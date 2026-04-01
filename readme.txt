# 安装服务
mosdns service install -d 工作目录绝对路径 -c 配置文件路径
# 安装成功后手动运行服务。(服务仅设定为随系统自启，安装成功后并不会马上自动运行)
mosdns service start

# 卸载
mosdns service stop
mosdns service uninstall

# Mosdns + systemd 正统启动方案（wrapper + 端口探测 + notify）

## Step 1：创建 wrapper 脚本

创建文件：

``` bash
sudo nano /usr/local/bin/mosdns-wrapper.sh
```

写入内容：

``` bash
#!/bin/bash

/usr/local/mosdns/mosdns start \
  --as-service \
  -d /usr/local/mosdns \
  -c /usr/local/mosdns/config_v4.yaml &

PID=$!

# 等待 DNS 端口就绪（53）
until (echo > /dev/tcp/127.0.0.1/53) 2>/dev/null; do
  sleep 0.5
done

# 通知 systemd：服务 ready
systemd-notify --ready

# 等待主进程退出
wait $PID
```

赋予执行权限：

``` bash
chmod +x /usr/local/bin/mosdns-wrapper.sh
```

------------------------------------------------------------------------

## Step 2：修改 mosdns.service

编辑 service 文件：

``` bash
sudo nano /etc/systemd/system/mosdns.service
```

内容如下：

``` ini
[Unit]
Description=A DNS forwarder
ConditionFileIsExecutable=/usr/local/mosdns/mosdns
After=network.target

[Service]
Type=notify
NotifyAccess=all

ExecStart=/usr/local/bin/mosdns-wrapper.sh

Restart=always
RestartSec=120

EnvironmentFile=-/etc/sysconfig/mosdns

[Install]
WantedBy=multi-user.target
```

------------------------------------------------------------------------

## Step 3：重新加载 systemd

``` bash
systemctl daemon-reload
```

------------------------------------------------------------------------

## Step 4：启动 mosdns 并验证

``` bash
systemctl restart mosdns
systemctl status mosdns
```

查看日志：

``` bash
journalctl -u mosdns -f
```

------------------------------------------------------------------------

## Step 5：配置 warp 依赖 mosdns

编辑 warp service：

``` bash
sudo nano /etc/systemd/system/warp-svc.service
```

关键内容：

``` ini
[Unit]
Description=Cloudflare Zero Trust Client Daemon
After=network.target mosdns.service
Requires=mosdns.service
```

------------------------------------------------------------------------

## Step 6：重新加载并启动 warp

``` bash
systemctl daemon-reload
systemctl restart warp-svc
systemctl status warp-svc
```

------------------------------------------------------------------------

## Step 7：验证

查看 DNS 端口：

``` bash
ss -lntup | grep :53
```

查看依赖关系：

``` bash
systemctl list-dependencies warp-svc
```

------------------------------------------------------------------------

## 总结

该方案实现：

-   mosdns 真正 ready 才通知 systemd
-   systemd 正确管理依赖关系
-   warp 依赖 mosdns 启动
-   避免 DNS 未就绪问题
