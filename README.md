# domp
## 基于Docker快速部署openresty + php-fpm + MySQL，并且支持开启redis动、静态缓存支持。

### 一、环境要求
理论上可以基于任何支持docker的平台，不过domp内置的一些脚本是基于centos 7编写，所以如果是非centos 7系统，不可以通过脚本快速部署，请参见下面的附录。

### 二、目录及文件说明（必读）：
```
[root@localhost]# tree
.
├── docker-compose.yml   # docker 编排配置
├── Dockerfile           # php-fpm docker镜像编译文件（可自定义）
├── etc                  # 配置目录
│   ├── nginx
│   │   ├── cert         # 证书一级SSL公共配置存放目录，若启用https，请将证书放到此目录
│   │   │   └── options-ssl-nginx.conf
│   │   └── conf.d           # nginx 配置目录
│   │       ├── common.conf  # nginx 公共配置
│   │       ├── ext          # nginx 拓展配置
│   │       │   ├── header.conf  # header 通用配置
│   │       │   ├── proxy.conf   # proxy 通用配置
│   │       │   └── ssl.conf     # ssl 跳转配置
│   │       └── vhost            # 虚拟主机配置（必须修改的配置），已放置2个参考实例配置，仅供参考：
│   │           ├── yourdomain.com_cache.conf  # 站点配置文件（带缓存），实际使用需要将yourdomain.com改成实际域名。
│   │           └── yourdomain.com.conf        # 站点配置文件（无缓存），实际使用需要将yourdomain.com改成实际域名。
│   └── php-fpm.d
│       └── php-fpm.conf   # php-fpm 配置
├── init.sh  # domp 初始化并启动的脚本
├── LICENSE
├── opt
│   ├── backup
│   │   └── backup.sh # 基于 腾讯云COS备份脚本，需要先安装并配置 coscmd，参考：https://cloud.tencent.com/document/product/436/10976
│   ├── g_cache.sh    # 基于 shell 的预缓存脚本，需要自行修改并添加定时任务，参考：https://zhang.ge/5095.html
│   └── g_deathlink_file.sh # 基于 shell 的死链生成脚本，需要自行修改并添加定时任务，参考：https://zhang.ge/5038.html
├── README.md
└── reload_php.sh  # php reload脚本，build镜像的时候回ADD到镜像里面。
```
### 二、快速拉起domp环境
#### 1、 克隆代码
```
mkdir -p /data && cd /data
git clone https://github.com/jagerzhang/domp.git
```
#### 2、修改MySQL root密码
```
vim docker-compose.yml
找到：
- "MYSQL_ROOT_PASSWORD=yourpassword"
将 yourpassword 改成自定义密码
```
#### 3、修改php模块（Ps：绝大部分网站无需修改，除非出现php模块缺失报错）
```
vim Dockerfile
找到如下语句，若满足要求则无需修改，若缺少某模块，则加上 php72w-模块名称，pecl的可能要额外加上 pecl
yum install -y memcached php72w-fpm php72w-gd php72w-pecl-memcached php72w-opcache php72w-mysql php72w-mbstring php72w-pecl-redis
```
#### 4、启动domp
```
bash init.sh

docker ps # 查看是否正常启动
CONTAINER ID        IMAGE                        COMMAND                  CREATED             STATUS              PORTS                                 NAMES
2fdcd10d05c6        openresty/openresty:centos   "/usr/bin/openresty …"   2 hours ago         Up 2 hours                                                openresty
eb5684527e4b        php-fpm:7.2                  "php-fpm -F"             2 hours ago         Up 2 hours                                                php-fpm
41381dea3d7f        redis:latest                 "docker-entrypoint.s…"   2 hours ago         Up 2 hours          127.0.0.1:6379->6379/tcp              redis
1f6278298539        mysql:8.0                    "docker-entrypoint.s…"   4 days ago          Up 4 days           127.0.0.1:3306->3306/tcp, 33060/tcp   mysql
```
### 三、配置网站

#### 1、虚拟主机配置说明
domp 默认已经自带了2种虚拟主机配置：`yourdomain.com.conf` 和 `yourdomain.com_cache.conf`，第一个不带`redis`缓存，第二个带`redis`缓存，自行选择一个，然后删除另一个即可。然后参考这个配置文件来定制自己网站的配置文件。若看不懂这个配置文件，可以直接拷贝网站原来的`vhost`配置文件也可以。

#### 2、https证书配置说明
https证书请放置到 `domp/etc/nginx/cert` 目录，然后在vhost配置中引用即可，注意在vhost里面的配置要变为：/etc/nginx/cert/你放置到证书名字，而非domp/etc/nginx/cert目录，因为已经挂载到了docker里面了！！！

#### 3、待补充...

## 附录
### 非centos环境使用参考

1、安装docker，参考：https://docs.docker.com/install/

2、安装 docker-compose，参考：https://docs.docker.com/compose/install/
Ps：此处提供linux通用安装命令：
```
curl -L "https://github.com/docker/compose/releases/download/1.23.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/bin/docker-compose
chmod +x /usr/bin/docker-compose
```
3、编译php-fpm镜像，若需要自定义php的模块，请编辑Dockerfile，再执行如下命令：
```
docker build -t "php-fpm:7.2" ./
```
4、启动domp
```
docker-compose up -d
```
