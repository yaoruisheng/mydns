# 安装服务
mosdns service install -d 工作目录绝对路径 -c 配置文件路径
# 安装成功后手动运行服务。(服务仅设定为随系统自启，安装成功后并不会马上自动运行)
mosdns service start

# 卸载
mosdns service stop
mosdns service uninstall