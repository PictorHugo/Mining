# 前言
周四中午，惊闻自己机器下线，以为又拉闸限电玩呢，跑到现场却发现无事发生。简单debug过后发现，原因是无法连接到鱼池导致的。924之后星火、蜜蜂都相继退出，鱼池肯定也避免不了，各位矿友应该都知道这一天早晚会到来。
目前鱼池仍然可以使用，但要通过IP而不是域名的方式访问，这种方式明显不是长久之计，所以受此影响的朋友要早做准备。
今天得空，把自己如何将机器连接到ethermine的方式做一个简单记录，仅供朋友们参考。
# 环境
- 5.4.0-hiveos #140
- NBMiner
# 准备
SSR梯子，其他具有sock5代理应该也可以使用类似方法，但本文章的一切操作都是基于SSR的
# 步骤
## 更新软件源
HiveOS默认的软件源比较慢，或者根本无法正常使用，这里我们将其修改成清华的软件源头。
根据机器上/etc/apt/sources.list的文件内容，这台hiveos的机器使用的ubuntu软件源为16.04版本的即可。

机器上的sources.list文件
```
deb http://security.ubuntu.com/ubuntu bionic-security main restricted
# deb-src http://security.ubuntu.com/ubuntu xenial-security main restricted
deb http://security.ubuntu.com/ubuntu bionic-security universe
# deb-src http://security.ubuntu.com/ubuntu xenial-security universe
deb http://security.ubuntu.com/ubuntu bionic-security multiverse
# deb-src http://security.ubuntu.com/ubuntu xenial-security multiverse
```
更新后的sources.list文件，可以直接复制下面的内容，或者从清华镜像源官网下载https://mirrors.tuna.tsinghua.edu.cn/help/ubuntu/。注意我将https替换成了http，否则可能在稍后的步骤出错。
```
deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial main restricted universe multiverse
deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial-updates main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial-updates main restricted universe multiverse
deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial-backports main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial-backports main restricted universe multiverse
deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial-security main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial-security main restricted universe multiverse
```
将/etc/apt/sources.list文件内容更新好后，直接执行下面命令即可
```
sudo apt update
```
## 安装pip3
执行下面命令即可
```
sudo apt install python3-pip
```
## 安装shadowsocksr-cli
shadowsocksr-cli是一个命令行版本的ssr客户端，这样即使在没有桌面的Linux系统上我们也可以使用自己的ssr。项目地址https://github.com/TyrantLucifer/ssr-command-client。在这里也感谢作者的贡献！
执行下面命令即可，安装过程可能会提示缺少setuptools包，如果遇到直接通过pip3安装这个缺少的包即可。
```
pip3 install install shadowsocksr-cli
```
## 通过shadowsocksr-cli配置ssr
### 更换订阅地址
```
shadowsocksr-cli --setting-url <your-sub-url>
```
### 手动获取ssr server
```
shadowsocksr-cli -u
```
## 配置systemd
如各位矿友所知，矿机可能经常因为显卡的驱动等原因需要重启，那么这时候如果还需要重新配置一便ssr就太不方便了，也耽误挖矿效率，这一步骤就是为了让矿机重启后自动加载ssr相关服务。
### 添加systemd形式的service文件
在/etc/systemd/system目录下创建一个autossr.service文件，文件内容如下
```
Description=Autossr keepalive daemon
Wants=network-online.target
After=network.target

[Service]
## here we can set custom environment variables
ExecStartPre=/usr/local/bin/shadowsocksr-cli -u
ExecStart=/usr/local/bin/shadowsocksr-cli -s 1
ExecStop=/usr/local/bin/shadowsocksr-cli -S 1
Type=simple
RemainAfterExit=yes
# don't use 'nobody' if your script needs to access user files
# (if User is not set the service will run as root)
#User=nobody

# Useful during debugging; remove it once the service is working
StandardOutput=console

[Install]
WantedBy=multi-user.target

```
### 将autossr.service设为enable
设置成enable后，systemd会按照配置的依赖，在开机完成后的合适时间（在这里是网络服务已经加载成功后）启动ssr相关服务。
```
systemctl enable autossr.service
```
### 启动autossr服务
```
systemctl start autossr.service
```
### 检查ssr服务是否正常启动
服务正常启动的话是可以看到监听了本地的1080端口
```
root@root:/etc/systemd/system# netstat -tnlp|grep 1080
tcp        0      0 127.0.0.1:1080          0.0.0.0:*               LISTEN      3149/python3
```
# 配置NBMiner通过socks5 proxy连接矿池
NBMiner直接通过proxy参数即可指定socks5 proxy的套接字地址（如127.0.0.1:1080）即可完成。具体参考官方文档https://github.com/NebuTech/NBMiner