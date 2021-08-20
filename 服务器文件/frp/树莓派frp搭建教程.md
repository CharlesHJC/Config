# 基于frp的RaspberryPi内网穿透

#### frp相关链接

github: https://github.com/fatedier/frp<br>
下载地址: https://github.com/fatedier/frp/releases<br>
官方文档：https://github.com/fatedier/frp/blob/master/README_zh.md<br>

---

#### 服务器端frp配置
1. ssh连接到目标服务器
```
ssh root@47.94.202.25
//输入密码
```

2. 进入opt目录下，下载frp压缩包
```
cd /opt
wget https://github.com/fatedier/frp/releases/download/v0.32.0/frp_0.32.0_linux_amd64.tar.gz
```
![目录](0F1DA5885C504BB9ADF2F03419F055B7)

3. 对压缩包进行解压(修改文件夹名称为frp，方便后续操作)，进入frp目录
```
tar -xzvf frp_0.32.0_linux_amd64.tar.gz
mv frp_0.32.0_linux_amd64 frp
cd frp
```
![frp目录](3EC27404238F444E8585765140B4D884)

4. 在服务器端我们只对frps进行操作。在文本编辑器中修改frps.ini文件
```
[common]
bind_port = 7000
token = 20212021    #自行修改

dashboard_port = 7500   #控制台监控默认端口
dashboard_user = weikong    #管理员账号
dashboard_pwd = 20212021    #管理员密码
enable_prometheus = true

log_file = /var/log/frps.log
log_level = info
log_max_days = 3
```
5. 修改好frps.ini文件后保存。开放服务器指定端口(如7000，7500等)<br><br>
6. 通过"IP+7500"的方式访问控制台界面，输入管理员账号密码进入。若成功进入，则frps端配置成功。<br>
![控制台登录](056D8245FE13429D8724648FD76BF3BE)<br>
![控制台界面](8DA8AC52364144FA9A470367590AC2AE)<br>

>为了后续维护方便，接下来将设置frps开机自启动。<br>

7. 进入目录/etc/systemd/system(此文件夹下配置文件将会开机自启动)新建一个frps.service文件
```
cd /etc/systemd/system
touch frps.service
```

8. 编辑frps.service文件，编辑完毕后保存。
```
[Unit]
Description=The nginx HTTP and reverse proxy server
After=network.target remote-fs.target nss-lookup.target

[Service]
Type=simple
ExecStart=/opt/frp/frps -c /opt/frp/frps.ini   #frps.ini所在目录
KillSignal=SIGQUIT
TimeoutStopSec=5
KillMode=process
PrivateTmp=true
StandardOutput=syslog
StandardError=inherit

[Install]
WantedBy=multi-user.target
```

9. 重载服务，应用frps.service服务
```
sudo systemctl daemon-reload
sudo systemctl enable frps.service
```

10. 重启服务器后，可以通过访问"IP+7500"的方式验证自启动是否配置成功，或者查看7500端口是否正在被占用。

---
#### 客户端frpc配置
1. ssh连接到指定设备(此处以树莓派为例，网关等设备操作一致)
```
ssh pi@192.168.1.144
//输入密码
```

2. 进入opt目录下，下载frp压缩包(根据设备芯片架构下载合适的安装包，这里下载arm版本)
```
cd /opt
sudo wget https://github.com/fatedier/frp/releases/download/v0.32.0/frp_0.32.0_linux_arm.tar.gz
```
![rpi目录](99C95D2F264544E18C2A3FA0FC12B9C0)

3. 对压缩包进行解压(修改文件夹名称为frp，方便后续操作)，进入frp目录
```
tar -xzvf frp_0.32.0_linux_amd64.tar.gz
mv frp_0.32.0_linux_amd64 frp
cd frp
```
![rpi_frp目录](DE39A88EA6C942F4AEC158B54AAA8085)

4. 在客户端端我们只对frpc进行操作。在文本编辑器中修改frpc.ini文件
```
[common]
server_addr = 47.94.202.25  #公网IP
server_port = 7000
token = 20212021    #保持和frps.ini中的token一致

[ssh_rpi]       #服务名称
type = tcp      #代理模式
local_ip = 127.0.0.1    #客户端设备IP地址
local_port = 22         #本地服务端口号
remote_port = 7002      #映射端口号

[vnc_rpi]
type = tcp
local_ip = 127.0.0.1
local_port = 5900
remote_port = 7003
```
5. 修改好frpc.ini文件后保存。从服务器上开启指定端口(如7002,7003等映射端口号)<br><br>
6. 输入以下命令，运行frpc，若成功运行，则会出现以下信息。
```
/opt/frp/frpc -c /opt/frp/frpc.ini 
```
![成功运行](13B8FADEC3904ACD85C2424D39B7F0CF)<br>


>为了后续维护方便，接下来将设置frpc开机自启动。<br>

7. 进入目录/etc/systemd/system(此文件夹下配置文件将会开机自启动)新建一个frpc.service文件
```
cd /etc/systemd/system
touch frpc.service
```

8. 编辑frpc.service文件，编辑完毕后保存。
```
[Unit]
Description=frpc servic
After=network.target

[Service]
Type=simple
TimeoutStartSec=30
ExecStart=/opt/frp/frpc -c /opt/frp/frpc.ini    //输入frpc.ini所在目录
Restart= always
RestartSec=1min

[Install]
WantedBy=multi-user.target
```

9. 重载服务，应用frpc.service服务
```
sudo systemctl daemon-reload
sudo systemctl enable frpc.service
```

10. 重启客户端后可以通过"ssh 用户名@公网IP -p 端口号"的方式访问到局域网内设备。并且能够从控制台界面看到在线运行的端口以及相关信息。<br>

![控制台](6B0B835A1CAE4F5AB1FC4EEA6FFCA822)

**注：网关设备的frp开机自启动需要单独写一个.sh脚本，并且关联到网关的开机加载项当中，此处不作赘述，有疑问可以询问张工。**

---

#### 网关域名映射
>利用frp的“自定义子域名”功能实现网关管理界面的域名访问。

>官方文档
![自定义域名官方文档](D1CB5B24BF8C4A9DB9FB2A0877E55169)

1. 将服务器解析的域名添加在frps.ini中。
```
subdomain_host = frp.wkgywg.com
```
>修改后：
![frps_ini](1B4177701A0743D6A16587FC1519B55F)<br>


2. 将客户端的frpc也进行修改，添加subdomain，自定义的子域名(为了访问内网服务时候的安全，添加了http访问时候的用户登录)
```
subdomain = silvery
http_user = admin
http_pwd = 123456
```
>两台网关设备都进行了修改
![frpc_black](7DF7BDED6847408CBC4F5BE63B4D6E14)<br>
![frpc_silvery](7272B4F12C594E449D5D1E0336264D0F)<br>

>修改后重启设备，然后可以通过子域名+端口号的方式可以访问到网关的控制界面。
```
http://black.frp.wkgywg.com:8080/
```
![网关控制界面验证](8019722EF76645E996FA06ACF022DCAA)
输入http验证账户和密码 就进入到了网关登录界面。