+++
title = 'Capture'
date = 2024-08-16T10:28:03+08:00
draft = false
tags = ["抓包", "逆向"]
+++

简单记一下目前在用的几个抓包方案

# charles

遇到app有代理检测时，手机端需要用`SocksDroid`配合使用。`charles`需要启用socks代理，`SocksDroid`指到`charles`的socks代理端口即可。

安装证书需要配合`movecert`使用，因为默认会安装到用户目录成为用户证书，中间人抓包还是不可信的，需要通过`movecert`移动到系统目录成为系统证书。

# reqable

https://reqable.com/

mac端作为服务器，android端作为客户端，配合使用。

安装证书比较方便，有magisk模块。

![image-20240816105110417](https://raw.githubusercontent.com/0x540x59/piclist/master/image-20240816105110417.png)

# lamda

[firerpa/lamda: ⚡️ Android reverse engineering & automation framework | 史上最强安卓抓包/逆向/HOOK & 云手机/远程桌面/自动化取证框架，你的工作从未如此简单快捷。 (github.com)](https://github.com/firerpa/lamda)

这是个新玩意儿，感觉挺好用的，集成了frida，mitmproxy，可以通过自动化编程接口控制手机，功能挺全的。就是得花点时间学习一下，学习曲线可能有点陡峭。详细的安装和抓包过程就不说了，看文档或者视频吧。lamda功能有点多，如果没有完整阅读完文档，可能都不知道这个startmitm.py哪来的，我这里记录一下抓包工具的启动命令吧。

```shell
git clone https://github.com/firerpa/lamda.git
python3.10 -m venv .venv/
source .venv/bin/activate
cd lamda/tools
pip install -r requirements.txt
pip install lamda
python -u startmitm.py <your_phone_ip_address>
```

它自己的抓包是免证书的，应该是安装的时候已经集成了mitmproxy证书了。另外好像还可以通过它的接口来[安装系统证书](https://github.com/firerpa/lamda/wiki/安装证书)，所以上面的`charles`和`reqable`证书可能也可以通过它来安装，我也才看到，下次试试。

