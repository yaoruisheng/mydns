# 安装服务
mosdns service install -d 工作目录绝对路径 -c 配置文件路径
# 安装成功后手动运行服务。(服务仅设定为随系统自启，安装成功后并不会马上自动运行)
mosdns service start

# 卸载
mosdns service stop
mosdns service uninstall

# Mosdns readiness service（不修改 mosdns 的正统方案）

## Step 1：创建 readiness 探测脚本

``` bash
sudo nano /usr/local/bin/mosdns-ready.sh
```

内容：

``` bash
#!/bin/bash

# 等待 DNS 53 端口可用
until (echo > /dev/tcp/127.0.0.1/53) 2>/dev/null; do
  sleep 0.5
done

# 通知 systemd ready
systemd-notify --ready
```

赋权：

``` bash
chmod +x /usr/local/bin/mosdns-ready.sh
```

------------------------------------------------------------------------

## Step 2：创建 mosdns-ready.service

``` bash
sudo nano /etc/systemd/system/mosdns-ready.service
```

内容：

``` ini
[Unit]
Description=Wait for mosdns DNS port ready
After=mosdns.service
Requires=mosdns.service

[Service]
Type=notify
ExecStart=/usr/local/bin/mosdns-ready.sh
NotifyAccess=all
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

------------------------------------------------------------------------

## Step 3：修改 warp-svc.service 依赖

编辑：

``` bash
sudo nano /lib/systemd/system/warp-svc.service
```

在 \[Unit\] 中加入：

``` ini
[Unit]
After=network.target mosdns.service mosdns-ready.service
Requires=mosdns.service mosdns-ready.service
```

------------------------------------------------------------------------

## Step 4：重新加载 systemd

``` bash
systemctl daemon-reload
```

------------------------------------------------------------------------

## Step 5：启用服务

``` bash
systemctl enable mosdns
systemctl enable mosdns-ready
systemctl enable warp-svc
```

------------------------------------------------------------------------

## Step 6：启动与验证

``` bash
systemctl restart warp-svc
systemctl status warp-svc
systemctl status mosdns-ready
```

------------------------------------------------------------------------

## 最终结构

mosdns.service → mosdns-ready.service → warp-svc.service
