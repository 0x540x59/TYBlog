+++
title = '代理中继'
date = 2024-08-27T18:21:02+08:00
draft = false
ShowToc = true
+++

遇到个比较奇葩的需求，要跨VPC下载文件，还跨了两个，如图。Account1和 Account2是两个不同云厂商的独立账号，账号内的VPC是网络是互通的，账号间本来是不通的。我们组由于要支持两个account的业务，所以把我们的vpc用vpn隧道打通了，即vpc2和vpc3是通的。但现在另外一组在vpc1说需要从account2那边定期下载文件，而文件提供方在vpc4中，因此他们的网络是不通的。

![image-20240827183519193](https://raw.githubusercontent.com/0x540x59/piclist/master/image-20240827183519193.png)

按理说你有这个需求你就把网络打通就好了，或者你提供公网访问呗。我也不知道为啥，成本？安全？总之这么个奇葩需求就落到我手里了。那还能咋弄，想辙呗。直接当二传手太费劲了，关键还得传两次。要么给vpc1加路由，给vpc4加返程路由，但我也不敢随便动他们vpc配置啊，万一影响了线上服务呢。突然想到要不建个代理做中转呗，一次中转应该没问题，两次就得叫中继了吧，可以的吗？想起科学上网中常常用到的v2ray的配置里面有inbound，outbound，好像配置挺灵活的。网上搜索了一下，可行，记录一下。

# 安装V2Ray

https://www.v2fly.org/guide/install.html

```shell
bash <(curl -L https://raw.githubusercontent.com/v2fly/fhs-install-v2ray/master/install-release.sh)
# 启动服务
systemctl enable --now v2ray
```

# 配置Socks5代理

vpc3中代理配置

```json
{
  "inbounds": [{
    "port": 10086,
    "protocol": "socks",
  }],
  "outbounds": [{
    "protocol": "freedom",
    "settings": {}
  }]
}
```

vpc2中代理配置

```json
{
  "inbounds": [{
    "port": 10086,
    "protocol": "socks",
    "settings": {
      "auth": "password",
      "accounts": [
        {
          "user": "admin",
          "pass": "123456"
        }
      ]
    }
  }],
  "outbounds": [{
    "protocol": "socks",
    "settings": {
      "servers": [{
        "address": "10.10.2.100",
        "port": 10086
      }]
    }
  }]
}
```

# 重启服务

```shell
systemctl restart v2ray
```

# 测试代理

```shell
curl --socks5 10.131.4.182:10086 --proxy-user "admin:123456" {{ download_url }}
```



如果是看是不是走了代理，可以对比一下使用和不使用代理，看下外网IP是不是变化了。

```shell
curl https://checkip.amazonaws.com

curl --socks5 10.131.4.182:10086 --proxy-user "admin:123456" https://checkip.amazonaws.com
```

python使用socks代理需要额外安装一下模块`pysocks`

```python
# pip install pysocks
def check_outbound_ip(proxy=False):
    '''
    检查出口ip
    '''
    proxies = {
        'http': 'socks5://admin:123456@10.131.4.182:10086',
        'https': 'socks5://admin:123456@10.131.4.182:10086',
    } if proxy else {}
    response = requests.get(
        url='https://checkip.amazonaws.com',
        proxies=proxies
    )
    print(f'outbound ip: {response.text}')
```
