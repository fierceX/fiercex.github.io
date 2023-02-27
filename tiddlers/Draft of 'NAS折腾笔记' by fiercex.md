## 硬件
考虑使用TrueNAS系统，并且考虑价格原因，就开始走捡垃圾这条路。最终选购的硬件如下

| 组件           | 型号                  | 价格 |
| -------------- | --------------------- | ---- |
| 主板           | 华硕 P8B-M            | 328  |
| CPU            | E3 1220L V2           | 145  |
| 内存           | 金士顿 1333 ECC 8G X2 | 270  |
| 散热器         | 英特尔原装            | 19   |
| 电源           | 振华铜皇              | 269  |
| 硬盘（系统盘） | 闪迪 64G SSD          | 67   |
| 硬盘（数据盘） | 希捷银河 8T X2        | 2303 |

## TrueNAS
### 安装
### ZFS和数据池
#### ZFS
ZFS文件系统有很多优点，例如软Raid（Raid-Z），Mirror等，以及快照，写时拷贝，数据完整性和自动修复等。
ZFS相关的参考资料：
https://docs.freebsd.org/en/books/handbook/zfs/
https://docs.oracle.com/cd/E24847_01/html/819-7065/docinfo.html#scrolltoc
#### 数据池
由于只有两块盘，所以直接组成了Mirror，这样有一半的容错率（掉一块盘数据还在）
#### 快照
ZFS的快照系统是增量的，可以通过快照避免误删除，在任务->定时快照中，可以添加定时快照任务，以及快照的保存周期。
### 权限与共享
#### 权限
##### ACL
现在可以通过ACL进行权限配置，默认只有安装时的root用户，为了方便进行共享，需要新建用户。可以通过用户组的概念来管理不同用户的权限。这里可以参考linux用户组和用户。
##### jail 挂载权限
需要注意的是，如果给jail挂载相应的数据卷，那么一般来说，可以直接赋予777的权限，但是这样就会破坏原来设置的共享权限。一个比较好的办法是，在jail里新建用户组和用户，使其和系统中的用户组的名称和uid一致，这样就可以继承系统设置的访问权限，也就是可以通过ACL进行权限配置，例如：
```shell
pw groupadd -n group_name -g 1000
pw groupmod group_name -m user_name
```
#### 共享
为了方便，只开通了SMB共享，直接在服务中打开SMB共享服务，然后配置共享的数据卷即可，默认的服务没有开启smb1.0协议的兼容，有需要的可以再服务->smb的设置里，打开smb1.0协议兼容
### 系统
#### 网络
##### 链路聚合
链路聚合可以提升网络吞吐量，虽然达不到像2.5G的速度，但是双1G网口链路聚合也能有效解决多终端同时访问时的网络带宽情况。
链路聚合需要使用支持链路聚合的网管型交换机的支持，目前在用的是咸鱼淘的网件的GS108Tv2。
在truenas上，网络->接口，新建接口，类型选择Link Aggregation，聚合协议选择LACP，然后勾上两个网口，设置ip地址和自动DHCP ipv6地址。重启网络即可
##### IPV6
truenas是支持ipv6的，但是需要路由器和光猫开启ipv6支持。开启默认的DHCP分配即可。需要注意的是，IPV6只能通过DHCP分配一个接口，也就是说，需要关掉DHCP分配，配置好链路聚合后，再在链路聚合的连接上打开ipv6 的 DHCP。
#### 证书
默认的Web管理页面开启了HTTP和HTTPS的监听，但是HTTPS的证书是默认的，在系统->CAs可以新建CA证书，并且在系统->证书里可以用新建的CA证书签发HTTPS的证书，在这里，可以添加证书需要验证的域名或者IP。
因为是通过自建的CA证书签发的，所以需要将CA证书导出，在需要的设备上导入根证书信任，否则会提示不安全的证书。
##### FREEBSD 信任私有证书
1. 将证书移动至特定目录并赋予正确的权限：
    ```bash
    mv /var/tmp/ca.cert /etc/ssl/certs
    chmod 0644 /etc/ssl/certs/ca.crt && chown root:wheel /etc/ssl/certs/ca.crt
    ```
2. 计算CA证书的Hash
    ```bash
    openssl x509 -noout -hash -in /etc/ssl/certs/ca.crt
    ```
    记住输出的8位Hash值，例如：`5d3b9418`，下一步要用
3. 配置软链接
   用上一步得到的Hash值软链接CA证书
   ```bash
   cd /etc/ssl/certs
   ln -s ca.crt 5d3b9418.0
    ```
4. 测试是否正常
   ```bash
   openssl s_client -connect test.local.com:443 | grep -i -e verify
   ```
   如果正常，则会输出：
   ```bash
   verify return:1
   ```
#### 2FA和密钥
为了提升安全性，可以在系统->2FA中开启2FA验证，并且也可以开启SSH的2FA验证。需要注意的是，开启2FA的SSH验证，就会关闭SSH的密钥验证。密钥和密码+2FA的安全性区别我不太确定，但是为了方便，我关闭了密码登录，只开起了密钥登录。
SSH登录密钥不在系统->密钥对里配置，而是在账户->用户下的用户设置里，添加SSH的登录密钥
#### 系统微调
在FreeBSD中，需要开启的系统服务，一般在`/etc/rc.conf`文件中修改，但是TrueNAS系统禁止修改这些文件，每次重启会通过数据库恢复这些系统文件，但是TrueNAS也提供了修改的方法，即在系统->系统微调下，可以添加需要修改的Key和Value，并且可以选择是否是`rc.conf`这个文件。
#### 安装软件
同样，TrueNAS也禁止通过Pkg工具安装软件，不过可以通过修改系统文件进行临时允许安装：
1. 修改`/usr/local/etc/pkg/repos/local.conf`文件，将tes改成no
2. 修改`/usr/local/etc/pkg/repos/FreeBSD.conf`，将no改成yes

这样即可通过pkg安装软件，但是重启后这两个系统文件会重置回原来的内容，如需要再次安装，可再次修改上述两个文件。
## 组网
在没有公网IP的情况下，就需要一些打洞的组网工具。目前使用了ZeroTIer，Tailscale和Frp。
### ZeroTier
ZeroTier很简单，可以通过pkg安装
```bash
pkg install zerotier
```
然后启动ZeroTier
```bash
zerotier-one -d
```
启动后，可以加入网络
```bash
zerotier-cli join xxxx
```
如果需要添加自建的Moon节点，则运行
```bash
zerotier-cli orbit id id
```
需要注意的是，系统在重启的时候，会重置`/var/db/`下的的文件，所以重启后id会改变，可以通过备份db文件，并且在系统->任务中添加开机脚本解决。
```bash
cp -a /var/db/zerotier-one /usr/local
```
然后在系统->任务->初始化/关闭脚本中，添加一个初始化脚本任务，并选择`Pre init`
```bash
cp -a /usr/local/zerotier-one /var/db
```
然后再添加一个任务，并选择`Post init`
```bash
nohup /usr/local/sbin/zerotier-one -p >/dev/null 2>&1 &
```
这样重启就可以自动启动ZeroTier了。
### Tailscale
Tailscale很简单，可以通过pkg安装
```bash
pkg install tailscale
```
然后启动后台服务
```bash
tailscaled
```
后台服务启动后，可以启动加入网络
```bash
tailscale up
```
这是会弹出一个链接，需要将该链接复制到浏览器里打开，并进行登录。默认tailscale是通过第三方服务进行帐号认证的
然后，就可以接入tailscale网络了。和ZeroTier一样，默认保存的配置文件需要复制到别的地方，防止重启被覆盖
```bash
cp -a /var/db/tailscale /usr/local
```
但是好像和ZeroTier一样设置启动脚本，但不起作用。
> 这里可以使用Headscale替换Tailscale，Headscale是一个Tailscale控制器的开源版，目前没有UI界面。除了IOS的Tailscale客户端，别的Tailscale客户端都支持Headscale
#### 路由转发
使用Tailscale可以开启路由转发，这样就可以通过该节点访问整个局域网，而且通过Jail安装的应用，是可以申请独立的内网ip，不需要和主系统共享ip，这样就需要开启路由转发来远程访问Jail应用。  
在Tailscale的Web上可以开启对应的路由规则，将该节点设置为子网路由，并且加上子网路由的网段，例如 `192.168.1.0/24`  
在Headscale上，可以在启动的时候，加上`--advertise-routes=192.168.1.0/24`，并且在服务端将该路由开启。
这别的设备，连接上Tailscale后，可以直接用内网ip远程访问家里的各种内网服务。
## 服务
### jail 插件
jail是freebsd上的底层虚拟化技术，类似于docker，可以新建一个和系统隔离的虚拟子系统。
通过jail 插件，可以安装官方和社区提供的jail插件应用，这里我装了下面的一些插件
- qbittorrent
- duplicati
- DNSMASQ 
- syncthing 

### 下载
使用PT下载，必备的两个软件`qbittorrent`，和`transmission`，前者可以在jail插件中安装，后面的需要安装快校版，用来快速辅种。  
transmission快校版地址：https://github.com/ChisBread/transmission_pt_edition/releases/download/3.0-r7/transmission_x64-freebsd-13.zip  
手动编译方式：https://github.com/ChisBread/transmission_pt_edition/issues/4
### 照片备份
TrueNAS只是个NAS系统，不像群晖带有那么多的组件，一切都要靠自己。照片备份，我的方案是手机端用基于SMB的文件同步软件，将手机上的照片同步到NAS上，然后在NAS上搭一个照片展示的服务，用来查看。当然也可以用比较成熟的nextcloud，不过感觉这个有点复杂，在我这个机器上不一定流畅，就没采用。
#### photosview
使用新开源的photosview管理照片的查看问题，但是photosview只提供docker版，别的需要编译。
1. 安装Go：
    1. 通过官网下载FreeBSD版本额二进制包
    2. 解压并设置环境变量
2. 安装NPM
   通过pkg安装
   ```bash
   pkg install npm
   ```
3. 安装依赖
   在编译api的过程中，有很多依赖，一般通过查找对应的包，都可以安装，但是有几个需要注意
    1. `go-face`依赖的`dlib`
       首先通过pkg安装dlib
       ```bash
       pkg install dlib-cpp
       ```
       然后可能会出现`fatal error: 'dlib/graph_utils.h' file not found`的错误，可以添加环境变量解决
       ```bash
       setnev CGO_CXXFLAGS -I/usr/local/include
       setnev CGO_LDFLAGS -L/usr/local/lib
       ```
    2. libheif版本问题
       如果出现libheif问题，可以修改`api/go.mod`中的libheif的版本，从`v1.12.0`改成`v1.11.0`
4. 编译
   编译分两部分，编译api和编译ui
   1. 编译api
      进入api目录，运行：
      ```bash
      go build -v -o photoview .
      ```
      即可生成`photoview`可执行文件。
   2. 编译ui
      进入ui目录，运行：
      ```bash
      npm run build
      ```
      即可生成`build`静态文件。
5. 启动
    1. 修改api的配置文件
       复制api目录下的`example.env`文件为`.env`，然后修改内容
       `PHOTOVIEW_DEVELOPMENT_MODE`改为0
       `PHOTOVIEW_SERVE_UI`改为1
       `PHOTOVIEW_LISTEN_IP`改为监听ip，一版是`0.0.0.0`
       `PHOTOVIEW_LISTEN_PORT`为监听端口号。
       `PHOTOVIEW_DATABASE_DRIVER`和`PHOTOVIEW_MYSQL_URL`为Mysql的链接信息。
       `PHOTOVIEW_UI_PATH`为UI编译生成的静态文件目录
    2. 配置Mysql
       可以通过pkg安装`mariadb`
       ```bash
       pkg install mariadb103-server
       ```
       添加开启启动，在`/etc/rc.conf`中添加`mysql_enable="YES"`
       然后添加用户和数据库
       ```sql
       CREATE USER photoview@localhost IDENTIFIED BY 'password'
       CREATE DATABASE IF NOT EXISTS photoview;
       GRANT ALL ON photoview.* TO photoview@localhost;
       ```
       修改默认的字符集，编辑`/usr/local/etc/my.cnf`
       ```ini
       [mysqld]
       character_set_server=utf8mb4
       collation-server=utf8mb4_unicode_ci
       skip-character-set-client-handshake
       ```
       
       重启服务
       ```bash
       service mysql-server restart
       ```
       
    3. 运行可执行文件`photoview`启动服务
       ```bash
       ./photoview
       ```
#### PhotoPrism
使用PhotoPrism解决照片查看问题，已经有人做了FreeBSD的版本，可以直接安装：https://github.com/psa/photoprism-freebsd-port/releases  
注意该软件依赖tensorflow，但是作者也给出了FreeBSD的版本：https://github.com/psa/libtensorflow1-freebsd-port/releases  
先安装好libtensorflow1，再安装PhotoPrism。然后开始初始化  
```bash
photoprism --assets-path=/var/db/photoprism/assets --storage-path=/var/db/photoprism/storage --originals-path=/var/db/photoprism/storage/originals --import-path=/var/db/photoprism/storage/import passwd
```

然后就可以启动了
```bash
photoprism --assets-path=/var/db/photoprism/assets --storage-path=/var/db/photoprism/storage --originals-path=/var/db/photoprism/storage/originals --import-path=/var/db/photoprism/storage/import start
```

### 在线笔记
在线笔记，我选择了`MEMOS`和`TiddlyWiki`这两个，一个是flomo的开源版，一个是历史比较久的个人wiki。
#### Memos
同样官方只给了docker和linux版本，但是基于go的，可以很快的编译安装：
1. 编译前端
  ```bash
  cd web/
  yarn && yarn build
  ```

2. 编译后端
  ```
  cp -r web/frontend-build/dist /server/dist
  go build -o memos ./bin/server/main.go
  ```

然后就可以启动了
#### TiddlyWiki
这个很简单，用基于Node.js版本的，可以直接用npm安装
```bash
npm install tiddlywiki -g
```
然后启动
```
tiddlywiki data --listen host=0.0.0.0 port=8989 "readers=(anon)" writers=fiercex username=fiercex password=passowrd
```
这样可以匿名访问，但是只有`fiercex`这个用户可以写入
### 密码管理器
密码管理器推荐用`vaultwarden`，`Bitwarden`的开源版，git地址：https://github.com/dani-garcia/vaultwarden  
同样没有FreeBSD版本，基于rust的，可以自己编译。
```bash
cargo build --features sqlite --release
```
然后，就可以启动了
```bash
./vaultwarden
```
需要注意的是，vaultwarden默认拒绝http请求登录，必须要用https，这点在后面详细说下。
### 个人导航页

## 打通内外网
上面已经提到了使用`tailscale`做了远程组网的服务，那么可以更进一步，将本地服务通过vps反代，达到公网访问的目的  
首先在vps上也安装tailacale客户端，加入虚拟网络，然后就可以通过vps访问家里的nas上的服务，特别针对TrueNAS系统，在TrueNAS上新建的Jail可以使用VNET，也就是拥有完整的网络，能够连接到路由器分配到该Jail独立的IP地址，那这样的话，就需要在nas系统上开启路由转发，否则无法通过`tailscale`访问到独立ip上的web服务。
### 外网访问
1. 为了安全起见，建议使用SSL，有免费的SSL证书，可以通过`.acme.sh`自动申请证书，记得打开80端口用于验证：
```bash
/root/.acme.sh/acme.sh --issue -d test.test.com --debug --standalone --keylength ec-256 --server letsencrypt
```
2. 安装并配置nginx，最简单的一个例子如下：
```conf
server {
        #SSL 访问端口号为 443
        listen 443 ssl;
        #填写绑定证书的域名
        server_name test.test.com;
        #证书文件名称
        ssl_certificate /root/.acme.sh/test.test.com_ecc/fullchain.cer;
        #私钥文件名称
        ssl_certificate_key /root/.acme.sh/test.test.com_ecc/test.test.com.key;
        ssl_session_timeout 5m;
        #请按照以下协议配置
        ssl_protocols TLSv1.2 TLSv1.3;
        #请按照以下套件配置，配置加密套件，写法遵循 openssl 标准。
        ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE;
        ssl_prefer_server_ciphers on;
        location / {
           #网站主页路径。此路径仅供参考，具体请您按照实际目录操作。
            proxy_pass http://192.168.0.3:8080/;
            proxy_set_header  X-Real-IP  $remote_addr;
            proxy_set_header  X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
            proxy_set_header Host $http_host;
        }
    }
```
其中`location`下的`proxy_pass`字段即为nas系统上的web服务，由于用了`tailscale`，这里可以直接使用内网ip访问。  
nginx在同一个端口号监听多个域名，所以加域名前缀然后申请证书，可以把想要的服务全部暴露出来，记得暴露出来服务要带帐号密码验证，否则有风险。
### 内网访问
内网可以直接输入ip:port访问，但是这样就和外网访问的方式发生了变化，在家和在外打开的网址不一样，这样体验很不好。有没有办法，能够让在家里输入外网访问的网址，但是不走外网，直接局域网连接呢？有，那就是DNS劫持。
#### 内网搭建NGINX服务
内网搭建nginx服务和外网其实一样的，但是注意一点，如果要内外网用同一个域名访问，那需要内网的nginx的ssl证书和外网一样对该域名认证，因为内网没有公网，而且也不开放80端口，所以没办法用内网机器进行验证。解决办法就是直接拷贝外网机器的ssl证书就行了。
#### DNS劫持
1. DNS映射
安装`DNSMASQ`，这个在插件里就能找到，一键安装。  
在`/usr/local/etc/dnsmasq.conf`文件下，添加下面一组DNS映射，就可以将该域名映射到后面的ip地址，这里全部将域名映射到`nginx`服务所在的ip地址
```
address=/test.test.com/192.168.0.2
address=/test2.test.com/192.168.0.2
```
2. 修改默认DNS地址
这一步有两个办法
   -  修改路由器上的DNS地址，将路由器上的DNS地址指向`DNSMASQ`服务的ip地址，这样的好处是接入路由器的设备不需要修改任何东西，就能自动将该域名指向内网ip。缺点是需要路由器支持，遗憾的是我的路由器不支持
   -  修改需要的设备的DNS地址，将DNS地址指向`DNSMASQ`服务的ip地址，这样的好处是只在修改的设备上起作用，缺点是如果设备多，需要挨个修改