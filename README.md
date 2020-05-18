# ss服务器+haproxy中转服务器配置说明

## 简介

因为使用ss工具的时候直接访问外网服务器速度会很慢，但使用国内中转服器进行一次转发后数据速度会非常快，所以这里使用了一个国内服务器的中转代理方法进行上网

本地<-->国内服务器<-->外网ss服务器

## 用法

#### 配置外网ss服务器

其实服务器上已经有VPN和SSH,VPN是全局代理,不是很方便,SSH在关键时候会断开,手机上没有很好的客户端.今天忽然见到shadowsocks这个方案,比较方便,而且是IOS ANDROID win linux 全平台通用.索性就搞一个备用了.
具体参考这里
https://github.com/clowwindy/shadowsocks/wiki/Shadowsocks-%E4%BD%BF%E7%94%A8%E8%AF%B4%E6%98%8E
首先检查下Python 版本，要有 2.6 or 2.7.
``` bash
python --version
Python 2.7.4
```
这个ubuntu的服务器上一般都有吧.
然后官网上直接要用pip装shadowsocks，有些人可能会出现命令错误，还要安装些东西
``` bash
apt-get install python-gevent python-pip
    ```
然后就可以
``` bash
pip install shadowsocks
```
安装shadowsocks了。
接下来配置也比较简单，
新建一个 config.json，或者其他名字的都行，位置可以放在/etc/shadowsocks/下（默认没有这个文件，你要自己创建一个），或者home或者其他地方。
内容是
``` json
 {
     "server":"my_server_ip",
     "server_port":8388,
     "local_port":1080,
     "password":"barfoo!",
     "timeout":600,
     "method":"table"
}
 ```
具体含义wiki上给的也很清楚
```
server       服务器 IP (IPv4/IPv6)，注意这也将是服务端监听的 IP 地址
server_port  服务器端口
local_port   本地端端口
password     用来加密的密码
timeout      超时时间（秒）
method       加密方法,默认是一种不安全的加密，推荐用 "aes-256-cfb"
```
我只更改了加密方式和密码，加密方式推荐用bf-cfb，因为aes-256-cfb系统默认貌似不支持，会报错。想支持这些加密方式你还要安装
``` bash
1at-get install python-m2crypto
```
然后就可以启动服务了。
``` bash
ssserver -c /etc/shadowsocks/config.json
nohup ssserver -c /etc/shadowsocks/config.json &gt; log &amp;
```
然后可以关了SSH。
或者更直接的开机自启动，添加到rc.local
``` bash
/usr/local/bin/ssserver -c /etc/shadowsocks/config.json
```
安卓手机可以安装“影梭”然后配置，其他手机自行google。
更多客户端在这儿
https://github.com/clowwindy/shadowsocks/wiki/Ports-and-Clients
###### 服务器端其他设置：
后台长期启动shadowsockts
``` bash
nohup ssserver -c /usr/local/lib/python2.7/dist-packages/shadowsocks/config.json > log &
```
查看后台启动任务： jobs
关掉 fg %n
 开机自动启动：
``` bash
cd /etc/
sudo vim rc.local
```
加上一行：
``` bash
/usr/local/bin/ssserver -c /usr/local/lib/python2.7/dist-packages/shadowsocks/config.json
```
另外，如果你的服务器安装有端口过滤软件iptables,你改下端口设置.
客户端的相关使用.
客户端可以在github找到：支持ios,android,mac os,linux,windows等等各种平台：
https://github.com/fengmm521/shadowsocks-ios（在这里也可以找到mac os,linux,windows，android客户端）

###### ios的客户端设置方法：

先打开shadowsocks的app，这个app在app store上是下载不下来的，你要用我上边的源码用xcode去编译安装到你的ios设备上.然后像下边这样在wifi网络设置的代里。

![](https://raw.githubusercontent.com/fengmm521/nettool/master/other/img.png)

输入的内容是:http://127.0.0.1:8090/proxy.pac
 这个可以一直都设置上，不使用代理也不用改。这样以后想要代码上网，只打开shadowsocks就可以了。

#### 关于ubuntu报libcrypto.so.1.1: undefined symbol错误

openssl升级到1.1.0版本后，shadowsocks2.8.2版本不对应

官网中说：
``` bash
EVP_CIPHER_CTX was made opaque in OpenSSL 1.1.0. As a result, 
EVP_CIPHER_CTX_reset() appeared 
and EVP_CIPHER_CTX_cleanup() disappeared.
EVP_CIPHER_CTX_init() remains as an alias for EVP_CIPHER_CTX_reset().
```
影响的只有openssl.py文件
该文件路径python3.6/site-packages/shadowsocks/crypto/openssl.py
修改其中两条函数调用的名称：
``` python
# libcrypto.EVP_CIPHER_CTX_cleanup.argtypes = (c_void_p,)
libcrypto.EVP_CIPHER_CTX_reset.argtypes = (c_void_p,)
# libcrypto.EVP_CIPHER_CTX_cleanup(self._ctx)
libcrypto.EVP_CIPHER_CTX_reset(self._ctx)
```

原文链接：https://blog.csdn.net/shelldawn/article/details/83578218


### shadowsocks中转ss代理服务器中转方法
##### 1.使用haproxy进行数据转发
安装haproxy,ubuntu下
```
sudo apt-get install haproxy
```

安装好haproxy后配置文件默认是
```bash
/etc/haproxy/haproxy.cfg
```
使用vi修改配置为以下内容
``` bash
# this config needs haproxy-1.1.28 or haproxy-1.2.1
global
    ulimit-n  30000

defaults
        log     global
        mode    tcp
        option  dontlognull
        timeout connect 5000
        timeout client  50000
        timeout server  50000

frontend ss-in
        bind *:中转服务器监听端口
        default_backend ss-out

backend ss-out
    server server1 ss服务器地址:ss服务器端口 maxconn 1024
```

改好后用:wq来保存haproxy配置

##### haproxy运行

使用下边命令来直接在终端运行
``` bash
#运行
haproxy -f 配置文件路径
#测试配置文件是否正确
haproxy -c 配置文件路径
#帮助
haproxy -h
```
注意：端口不能被占用，要不然会报bind socket错误

##### haproxy后台和开机运行

会了上边的命令行运行，那开发运行和后台运行方法也是和ss服务器运行方法类似了
