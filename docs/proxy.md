# 服务器代理（FinalShell 远程隧道）

借本机 Clash 给服务器临时开代理。隧道随 FinalShell 会话存活，窗口关闭即失效。

机器 A 上这份文档在 `/srv/devops/docs/proxy.md`。最需要它的时刻恰恰是检出还不存在的时候 —— 克隆 GitHub 被墙时照着本文在本地读着做，下面的命令都不依赖服务器上的检出。

## 一、建隧道

FinalShell 里对连接右键 → 隧道 → 新建：

| 字段 | 值 |
|---|---|
| 名称 | `Clash Verge` |
| 类型 | **远程** |
| 监听端口 | `7898` |
| 绑定 ip | `127.0.0.1` |
| 目标地址 | `127.0.0.1` |
| 目标端口 | `7897` |

目标端口填 Clash Verge 的混合端口，默认 `7897`。绑定 ip 不要改成 `0.0.0.0`，那会把无认证代理暴露到公网。

## 二、验证

```bash
curl -x http://127.0.0.1:7898 -I https://www.google.com
```

返回 `HTTP/2 200` 或 `301` 即通。

## 三、启用

按需选，不必全做。

### 1 · 当前 shell

```bash
export https_proxy=http://127.0.0.1:7898 http_proxy=http://127.0.0.1:7898 all_proxy=socks5://127.0.0.1:7898 no_proxy=localhost,127.0.0.1,::1,registry.internal,10.8.0.0/24,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16,*.local
```

只对当前会话生效。`sudo`、systemd 服务、容器都读不到，需要各自配置，见下面各条。

### 2 · 装成开关函数

从 `cat >>` 到最后单独的 `EOF` 是一条命令，整段一次粘贴。

```bash
cat >> ~/.bashrc <<'EOF'

# >>> 代理开关（FinalShell 远程隧道 → 本机 Clash）>>>
proxy_on() {
  export http_proxy=http://127.0.0.1:7898
  export https_proxy=$http_proxy
  export all_proxy=socks5://127.0.0.1:7898
  export no_proxy=localhost,127.0.0.1,::1,registry.internal,10.8.0.0/24,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16,*.local
  echo "proxy on -> 127.0.0.1:7898"
}
proxy_off() {
  unset http_proxy https_proxy all_proxy no_proxy
  echo "proxy off"
}
# <<< 代理开关 <<<
EOF
```

`source ~/.bashrc` 后用 `proxy_on` / `proxy_off` 切换。不要把 export 直接写进 `~/.bashrc`，隧道断开后每条命令都会卡在连不上的代理上。

### 3 · git

```bash
git config --global http.https://github.com/.proxy http://127.0.0.1:7898
```

`--global` 只对当前用户生效。克隆和更新 `/srv/devops` 走的是 `sudo git`，root 读不到上面这条，要给 root 再配一次：

```bash
sudo git config --global http.https://github.com/.proxy http://127.0.0.1:7898
```

然后重试克隆，完整步骤见 [deploy.md](deploy.md) 第二节第 1 步：

```bash
sudo git clone https://github.com/opendesign-os/devops.git /srv/devops
```

### 4 · dnf

```bash
sudo sed -i '$a proxy=http://127.0.0.1:7898' /etc/dnf/dnf.conf
```

追加在文件末尾。`dnf.conf` 若 `[main]` 之外还有别的段，需手动把这行移到 `[main]` 下。

### 5 · Docker 拉镜像

```bash
sudo mkdir -p /etc/systemd/system/docker.service.d && printf '[Service]\nEnvironment="HTTP_PROXY=http://127.0.0.1:7898"\nEnvironment="HTTPS_PROXY=http://127.0.0.1:7898"\nEnvironment="NO_PROXY=localhost,127.0.0.1,registry.internal,10.8.0.0/24,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16"\n' | sudo tee /etc/systemd/system/docker.service.d/proxy.conf
```

```bash
sudo systemctl daemon-reload && sudo systemctl restart docker
```

`NO_PROXY` 里的 `registry.internal` 不能漏，漏了自建仓库的拉取全部失败。确认：

```bash
sudo systemctl show docker --property=Environment
```

### 6 · 容器内部

| 场景 | 做法 |
|---|---|
| 构建时联网 | `docker build --network host --build-arg HTTP_PROXY=http://127.0.0.1:7898 .` |
| 临时跑容器 | `docker run --network host ...` |

## 四、删除

对应上面启用过的项逐条撤销。

### 1 · 隧道

FinalShell 连接属性里删掉这条隧道。只关窗口的话下次连接会自动重建。

### 2 · 当前 shell

```bash
unset http_proxy https_proxy all_proxy no_proxy HTTP_PROXY HTTPS_PROXY ALL_PROXY NO_PROXY
```

### 3 · 开关函数

```bash
sed -i '/# >>> 代理开关/,/# <<< 代理开关/d' ~/.bashrc
```

### 4 · git

```bash
git config --global --unset http.https://github.com/.proxy
```

给 root 配过的那条要单独撤，撤完 `/srv/devops` 的 `git pull` 就走直连：

```bash
sudo git config --global --unset http.https://github.com/.proxy
```

### 5 · dnf

```bash
sudo sed -i '/^proxy=http:\/\/127.0.0.1:7898/d' /etc/dnf/dnf.conf
```

### 6 · Docker daemon

```bash
sudo rm -f /etc/systemd/system/docker.service.d/proxy.conf
```

```bash
sudo systemctl daemon-reload && sudo systemctl restart docker
```

重启 dockerd 会中断正在进行的拉取和构建，挑没有部署任务时做。

### 7 · 残留自检

```bash
env | grep -i proxy; git config --global --get-regexp proxy; sudo git config --global --get-regexp proxy; grep -rn 7898 /etc/dnf/dnf.conf /etc/systemd/system/docker.service.d/ ~/.bashrc 2>/dev/null; ss -lnt | grep 7898
```

全部无输出即清理干净。`ss` 仍有输出说明 FinalShell 会话还连着，回第 1 步。
