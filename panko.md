### Panko 文档
panko 是一个事件存储服务，用来存储和查询由Ceilometer产生的事件数据。这篇文档记录怎样使用Panko。
- 安装 Panko
    - 安装 development sandbox
    - 手动安装
    - 安装 the API behind mod wsgi
    - 安装 the API with uwsgi
- 贡献指南

#### 安装Panko
###### 安装development sanbox
**配置 devstack**
1. 下载 [devstack](https://docs.openstack.org/devstack/latest/)
2. 创建 **local.conf** 作为devstack的配置文件
3. 默认没有启用panko，so they must be enabled in local.conf before running stack.sh
```
[[local|localrc]]
# Enable the Panko devstack
pluginenable_plugin panko https://git.openstack.org/openstack/panko.git
```

###### 手动安装
**安装存储后端**
首先安装存储后端，可以安装如下的数据库：
*MongoDB*
[安装MongoDB](www.mongodb.org)，然后启动服务。需要的MongoDB最低版本为2.4.x。还需要安装 [pymongo](https://pypi.org/project/pymongo/) 2.4
若使用MongoDB来作为panko数据库，配置panko.conf
```
    [database]
    connection = mongodb://username:password@host:27017/panko
```

*SQLalchemy-supported DBs*
SQLalchemy支持的数据库，如MySQL 或者 PostgreSQL。
使用MySQL作为存储后端，修改panko.conf
```
    [database]
    connection = mysql+pymysql://username:password@host/panko?charset=utf8
```

**安装API Server**
> Note
> API server 需要能与keystone和panko的数据库交流。
> It is only required if you choose to store data in legacy database or if you inject new samples via REST API.

1. git clone 
```
    $ cd /opt/stack
    $ git clone https://git.openstack.org/openstack/panko.git
```
2. 使用root权限用户运行 panko 的安装脚本
```
    $ cd panko
    $ sudo python setup.py install
```
3. 在keystone中创建panko的service
```
    $ openstack service create event --name panko --description "Panko Service"
```
4. 在keystone中创建panko的endpoint
```
    $ openstack endpoint create $PANKO_SERVICE --region RegionOne --publicurl "http://$SERVICE_HOST:8777" --adminurl "http://$SERVICE_HOST:8777" --internalurl "http://$SERVICE_HOST:8777"
```
or 
```
openstack endpoint create --region RegionOne event public http://vip:8777
```
> Note
> PANKO_SERVICE 是之前创建的service的ID。SERVICE_HOST 是controller 或者 vip 这种host name。

5. 选择和启动 API server
Panko 有 panko-api 命令。
小型安装可以直接使用 panko-api 命令，大集群推荐使用wsgi安装 [点我](https://docs.openstack.org/panko/rocky/install/mod_wsgi.html)

> Note
> The development version of the API server logs to stderr, so you may want to run this step using a screen session or other tool for maintaining a long-running program in the background.


###### 安装 the API behind mod wsgi
Panko 自带很多例子，配置API service 在Apache中使用mod_wsgi 来运行。
**app.wsgi**
panko/api/app.wsgi 文件设置 V2 API WSGI应用。

**/etc/apache2/panko**
1. etc/apache2/panko 文件包含很多示例设置 that work with a copy of panko installed via devstack.
   基于deb的系统  copy 或者 symlink 这个文件到 /etc/apache2/sites-available
   基于rpm的系统  目标改为/etc/httpd/conf.d
2. Modify the WSGIDaemonProcess directive to set the user and group values to an appropriate user on your server. In many installations panko will be correct.
3. 重启httpd 或者 apache2

###### 安装 the API with uwsgi

Panko comes with a few example files for configuring the API service to run behind Apache with mod_wsgi.

**app.wsgi**
The file panko/api/app.wsgi sets up the V2 API WSGI application. The file is installed with the rest of the Panko application code, and should not need to be modified.

**example of uwsgi configuration file**
Create panko-uwsgi.ini file:
```
[uwsgi]
http = 0.0.0.0:8041
wsgi-file = <path_to_panko>/panko/api/app.wsgi
plugins = python
# This is running standalone
master = true
# Set die-on-term & exit-on-reload so that uwsgi shuts down
exit-on-reload = true
die-on-term = true
# uwsgi recommends this to prevent thundering herd on accept.
thunder-lock = true
# Override the default size for headers from the 4k default. (mainly for keystone token)buffer-size = 65535enable-threads = true# Set the number of threads usually with the returns of command nprocthreads = 8# Make sure the client doesn't try to re-use the connection.add-header = Connection: close# Set uid and gip to an appropriate user on your server. In many# installations ``panko`` will be correct.uid = pankogid = panko
```
...

**configuring with uwsgi-plugin-python on Debian/Ubuntu**
