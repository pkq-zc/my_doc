# windows使用vnc远程连接Deepin

## windows软件安装

通过RealVnc下载客户端软件.选择合适你电脑系统的即可,然后无脑点击安装.[下载地址]('https://www.realvnc.com/en/connect/download/viewer/')

## Deepin下Vnc Server 安装

- 更新源:```sudo apt-get update```
- 安装:```sudo apt-get install x11vnc```

## Deepin下服务启动

- 初始化设置密码:```x11vnc -storepasswd```,执行完了之后会要你设置密码,并将密码存储在```~/.vnc/passwd```中
- 启动x11vnc:```x11vnc -forever -shared -rfbauth ~/.vnc/passwd```

## windows下运行客户端

windows下运行```VNC Viewer```程序,输入Deepin的ID,然后输入密码登录即可
