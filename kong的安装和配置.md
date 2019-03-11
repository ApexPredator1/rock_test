

##  kong的安装和配置

### 1、通过安装包安装kong

```
wget https://bintray.com/kong/kong-community-edition-deb/download_file?file_path=dists/kong-community-edition-1.0.2.bionic.all.deb
sudo apt-get update
sudo apt-get install openssl libpcre3 procps perl
sudo dpkg -i kong-community-edition-1.0.2.bionic.all.deb
```

注：通过安装包安装kong完毕后：

1、配置文件模板：/etc/kong/kong.conf.default，我们通过命令cp /etc/kong/kong.conf.default /etc/kong/kong.conf创建一个默认配置文件/etc/kong/kong.conf

2、默认安装路径：/usr/local/kong

3、kong安装完毕后，其lua源码都被放在/usr/local/share/lua/5.1/kong/目录下，可以修改它们，重载或重启kong后生效

![](https://test-1256398113.cos.ap-guangzhou.myqcloud.com/20190311142614.png)



### 2、**安装postgres数据库：（**kong还支持Cassandra数据库）

```
sudo apt-get install postgresql pgadmin3 postgresql-client

初次安装后，默认生成一个名为postgres的数据库和一个名为postgres的数据库用户和一个名为postgres的用户。
sudo su - postgres             # 切换到postgres用户。
psql                           # 登录PostgreSQL控制台。
\> CREATE USER kong WITH PASSWORD '123456';
\> CREATE DATABASE kong OWNER kong;  数据库kong（其所有者设为数据库用户kong）                 
注意每条语句后面要加分号才可以
```



### **3**、配置kong数据库

sudo vi /etc/kong/kong.conf， 将以下几项取消注释

```
database = postgres             # Determines which of PostgreSQL or Cassandra
pg_host = 127.0.0.1             # The PostgreSQL host to connect to.
pg_port = 5432                  # The port to connect to.    # reading and writing.
pg_user = kong                  # The username to authenticate if required.
pg_password = 123456            # The password to authenticate if required.
pg_database = kong 
```



### 4、进行数据迁移，并启动kong

```
sudo kong migrations bootstrap -c /etc/kong/kong.conf     #将kong的数据迁移到postgres数据库
sudo kong start                   #启动kong
curl -i http://localhost:8001/    #查看kong是否正常运行
```



### 5、kong中nginx指令的配置

**不要直接在kong中的nginx.conf和nginx-kong.conf中进行指令的配置，因为在kong启动时，会完全使用kong.conf中的配置信息覆盖nginx.conf和nginx-kong.conf中的配置：所以单单在nginx.conf和nginx-kong.conf中配置指令是无效的！！！**

例1：在每次kong启动时，kong.conf中admin_listen = 172.18.150.168:8001, 172.18.150.168:8444 ssl指令都会覆盖nginx-kong.conf中的listen配置指令

例2：在kong.conf中配置admin_ssl_cert = /home/apex/cert/server.cert和admin_ssl_cert_key = /home/apex/cert/server.key，结果重启kong后，nginx-kong.conf配置文件中对应admin管理的server下多出了两条指令：ssl_certificate /home/apex/cert/server.cert; 和 ssl_certificate_key /home/apex/cert/server.key; 但是单单在nginx.conf或nginx-kong.conf中配置ssl on;指令却是无效的，每次重启，ssl on;指令都会消失。

 

**在kong.conf中配置nginx指令：**

```
注入在kong.conf配置文件中的以nginx_http_、nginx_proxy_、nginx_admin_等前缀开头的nginx指令会被去除前缀并添加到kong所使用的nginx配置文件中，其中：
1、以nginx_http_为前缀的指令最终会被添加到nginx配置文件中的http块中的# injected nginx_http_* directives注释下面
2、以nginx_proxy_为前缀的指令最终会被添加到nginx配置文件中的处理kong代理端口的那个server块中的http块中的# injected nginx_proxy_* directives注释下面
3、以nginx_admin_为前缀的指令最终会被添加到nginx配置文件中的处理kong admin api的那个server块中的http块中的# injected nginx_admin_* directives注释下面
```

 

例如：

在/etc/kong/kong.conf配置文件中添加如下3条指令（也可以把它们声明为环境变量，例如export KONG_NGINX_HTTP_OUTPUT_BUFFERS="4 64k"）

```
nginx_http_include = my-server.kong.conf  # 如果文件是相对路径的，则默认采用kong start的-p参数指定的路径作为相对路径，比如/usr/local/kong，
nginx_http_client_header_buffer_size = 4k
nginx_proxy_large_client_header_buffers=16 256k
```

执行sudo kong restart重启kong，查看/usr/local/kong/nginx-kong.conf配置文件，发现用于路由的那个server块（8000端口）中多了一条指令：

```
# injected nginx_proxy_* directives
large_client_header_buffers 16 256k;
include myserver.conf
```

 http块中多出了一条指令：

```
# injected nginx_http_* directives
client_header_buffer_size 4k;
```





## kong CLI

使用kong的命令行（command line interface）可以对kong实例进行一些操作。

全局Options：适用于所有命令

| 参数 | 含义         |
| ---- | ------------ |
| --v  | 显示更多信息 |
|--vv|启动调试模式，显示debug信息|
|--help|查看帮助信息|



kong version               查看kong的版本

| 参数 | 含义         |
| ---- | ------------ |
|-a,--all|打印所有依赖组件的版本|

​                                                                                      

kong start                        启动指定路径下的kong实例
| 参数 | 含义         |
| ---- | ------------ |
|-c,--conf|指定要加载的配置文件，默认为/etc/kong/kong.conf 或 /etc/kong.conf|
|-p,--prefix|指定kong实例所在目录的所在路径，默认为/usr/local/kong配置文件中的include等指令会参考该路径，|
|--nginx-conf|自定义nginx配置文件模板，默认路径是/usr/local/kong/下的nginx.conf和被nginx.conf所include的nginx-kong.conf，|
|--run-migrations|布尔值，指定是否在启动前运行数据迁移|
|--db-timeout|默认值为60，单位为秒，|
|--lock-timeout|默认值为60，单位为秒，指定等待主kong实例完成数据迁移的时间，|



kong stop                    停止指定路径下的kong实例
| 参数 | 含义         |
| ---- | ------------ |
|-p,--prefix|指定kong实例所在目录的所在路径|



kong reload [opts]     重载指定路径下的kong实例，该命令向nginx发送一个HUP信号，
| 参数 | 含义         |
| ---- | ------------ |
|-c,--conf|指定要重新加载的配置文件|
|-p,--prefix|指定kong实例所在目录的所在路径|
|--nginx-conf|自定义nginx配置文件模板|



kong restart                 重启指定路径下的kong实例，等价于分别执行kong stop和kong start命令，

| 参数 | 含义         |
| ---- | ------------ |
|-c,--conf|指定要重新加载的配置文件|
|-p,--prefix|指定kong实例所在目录的所在路径|
|--nginx-conf|自定义nginx配置文件模板|
|--run-migrations|布尔值，指定是否运行数据迁移|
|--db-timeout|默认值为60，单位为秒，|
|--lock-timeout|默认值为60，单位为秒，|



kong quit [opts]          优雅地退出指定路径下的kong实例，该命令向nginx发送一个SIGQUIT信号，意为会等待将当前所有的请求处理完，然后再关闭nginx，但如果超过了超时时间，也会强制关闭nginx
| 参数 | 含义         |
| ---- | ------------ |
|-p,--prefix|指定将指定路径下的kong退出|
|-t,--timeout|默认值为10，过了这个延迟时间就会强制退出|



kong check <conf-path>            查看指定的kong配置文件是否是一个有效的配置文件，

例如kong check /etc/kong/kong.conf 结果为configuration at /etc/kong/kong.conf is valid

 

kong health                                   检查当前kong实例的关键服务是否在运行，

例如：sudo kong health 结果为nginx.......running  Kong is healthy at /usr/local/kong

| 参数 | 含义         |
| ---- | ------------ |
|-p, --prefix|指定指定的kong文件夹的所在路径，例如/usr/local/kong|



kong migrations bootstrap               运行所有的迁移（包括已经迁移的和未迁移的），

kong migrations up                             运行新加的迁移（未迁移的），Run any new migrations.

kong migrations finish                        Finish running any pending migrations after 'up'.

kong migrations list                            列出已被执行的迁移，                        

kong migrations reset                        重置数据库，

| 参数 | 含义         |
| ---- | ------------ |
|-y,--yes|Assume "yes" to prompts and run non-interactively.|
|-q,--quiet|Suppress all output.|
|-f,--force|Run migrations even if database reports as already executed.|
|--db-timeout|默认值为60，Timeout, in seconds, for all database operations (including schema consensus for Cassandra).|
|--lock-timeout|默认值为60，Timeout, in seconds, for nodes waiting on the leader node to finish running migrations.|
|-c,--conf|指定一个配置文件|



kong prepare [opts]         Prepare the Kong prefix in the configured prefix directory. This command can be used to start Kong from the nginx binary without using the 'kong start' command.
| 参数 | 含义         |
| ---- | ------------ |
|-c,--conf|(optional string) configuration file|
|-p,--prefix|(optional string) override prefix directory|
|--nginx-conf|(optional string) custom Nginx configuration template|



​            

## kong API

kong有两个主要的端口：一个为管理端口8001，另一个是提供网关访问的8000端口

通过kong的rest api管理接口可以管理你的服务、路由、消费者等等，通过rest api传送给kong的数据会被存储在数据库中，下面是kong api的主要端口，它们都可以在kong的配置文件中自定义配置

| 端口 | 作用 |
| ---- | ---- |
| 8000 | 监听来自客户端的http请求，然后将它转发到上游服务器 |
|8443|监听来自客户端的https请求，然后将它转发到上游服务器|
|8001|基于http用来管理kong的配置|
|8444|基于https来管理kong的配置|



kong提供了RESTfull的admin api，发送到admin api的请求会被发送到集群中的所有节点，kong会保持集群中所有节点的配置一致性，该admin api提供了对kong的完全控制，所以应只在内部使用，不可把该接口暴露在外部环境，

**该admin api接受两种content-type（body数据类型）：**

1、application/x-www-form-urlencoded

![](https://test-1256398113.cos.ap-guangzhou.myqcloud.com/20190311144347.png)

![](https://test-1256398113.cos.ap-guangzhou.myqcloud.com/20190311144358.png)



2、application/json

![](https://test-1256398113.cos.ap-guangzhou.myqcloud.com/20190311144416.png)

![](https://test-1256398113.cos.ap-guangzhou.myqcloud.com/20190311144425.png)



kong对nginx中的逻辑进行了封装，并创建了很多封装对象，比如service、route、consumer等等



### service对象

**service对象的属性**

| 属性 | 含义 |
| ---- | ---- |
| "name" | 指定当前service的名字，例如my-service, |
|"id"|指定当前service的id，值为一个UUID，比如"173a6cee-90d1-40a7-89cf-0329eca780a6",|
|"created_at"|指定创建时间，是一个时间戳，例如1422386534,|
|"updated_at"|指定更新时间，是一个时间戳，例如1422386534,|
|"protocol"|指定与上游服务器的通信协议，默认值是http|
|"host"|指定上游服务器的host值，例如"example.com",|
|"port"|指定上游服务器的端口号，默认值是80,|
|"path"|指定向上游服务器转发请求时使用的path，默认值为空，例如"/some_api",|
|"retries"|在转发请求到代理服务器失败时的重试次数，默认值为5,|
|"connect_timeout"|指定与上游服务器的连接超时时间，单位是毫秒，默认值为60000，即60秒，|
|"write_timeout"|将请求发送到upstream服务器的两个连续的成功写操作之间的超时时间。默认为60000|
|"read_timeout"|将请求发送到upstream服务器的两个连续的成功读操作之间的超时时间。默认为60000|
|"url"|用于同时指定protocol, host, port, path属性，是它们的简写形式，该属性是只写的，不可读|



**service相关api：**

|http方法|路由|含义|
|-|-|-|
|POST|/services|创建一个service|
|PATCH|/services/{name or id}|更新指定name或id的service|
|PATCH|/routes/{route name or id}/service|更新指定name或id的route所指向的service|
|PATCH|/plugins/{plugin id}/service|更新指定name或id的plugin所依附的service|
|PUT|/services/{name or id}|更新或创建指定name或id的service，如果指定id的service存在，则更新它，如果不存在，则创建id为指定值的service，|
|PUT|/routes/{route name or id}/service|更新或创建指定name或id的route所指向的service|
|PUT|/plugins/{plugin id}/service|更新或创建指定name或id的plugin所依附的service|
|GET|/services|获得所有的service|
|GET|/services/{name or id}|获取指定name或id的service|
|GET|/routes/{route name or id}/service|获得指定name或id的route所指向的service|
|GET|/plugins/{plugin id}/service|获得指定name或id的plugin所依附的service|
|DELETE|/services/{name or id}|删除指定name或id的service|
|DELETE|/routes/{route name or id}/service|删除指定name或id的route所指向的service|



例如：

```
curl -i -X POST \
  --url http://localhost:8001/services/ \
  --data 'name=example-service' \
  --data 'url=http://mockbin.org'
```



### route对象

route对象用来设置service对象的路由，service和route是1:N的关系，

**route对象的属性**

| 属性   | 含义                               |
| ------ | ---------------------------------- |
| "name" | 指定当前route的名字，例如my-route, |
|"id"|指定当前route的id，值为一个UUID，比如"173a6cee-90d1-40a7-89cf-0329eca780a6",|
|"created_at"|指定创建时间，是一个时间戳，例如1422386534,|
|"updated_at"|指定更新时间，是一个时间戳，例如1422386534,|
|"protocols"|指定允许的请求协议，即匹配当前路由的请求必须要有的协议，可以设置多个，允许的值有"http"、"https"、"tcp"、"tls"，|
|"methods"|指定允许的请求方法，即匹配当前路由的请求必须要有的方法，可以设置多个，默认值为空，例如["GET", "POST"],|
|"hosts"|指定允许的请求host字段值，即匹配当前路由的请求必须要有的host字段值，可以设置多个，例如 ["example.com", "foo.test"],hosts的值中支持*通配符，且最多只能有1个*号，且只能位于域名的最左端或最右端，例如： *.example.com可以匹配a.example.com和x.y.example.comexample.*可以匹配example.com和example.org|
|"paths"|指定允许的请求URI，可以设置多个，可以含有正则表达式，paths是前缀匹配，且是贪婪匹配，匹配最长的一个paths值，例如: ["/foo", "/bar"]可以匹配/foo、/bar、/foo/hello/world等等，这里的正则表达式是PCRE（Perl Compatible Regular Expression），例如："paths": ["/users/\d+/profile", "/following"]可以匹配/following和/users/123/profile，paths也支持捕获，例如paths：/version/(?<version>\d+)/users/(?<user>\S+) 对于请求/version/1/users/john，可以通过以下代码获取捕获内容：local capture = ngx.ctx.router_matches.uri_captures  其值为表 { "1", "john", version = "1", user = "john" }|
|"regex_priority"|指定当前路由的正则表达式匹配的优先级，当有两个paths为正则表达式的路由同时匹配一个请求，且两个路由的优先级相同，则使用最先创建的那个路由（即created_at值最小的那个），paths为非正则表达式的路由的匹配是以贪婪模式进行匹配的，默认值为0|
|"strip_path"|默认值为true，表示当把匹配的请求转发给上游服务器时，去除请求URI中paths参数所指定的部分，false表示不去除，而是将客户端的原请求转发给上游，例如：service指向的url是http://172.18.150.179:9999/test/a，其route只指定了paths，且paths=/test，则：当strip_path为true时，发向kong的请求：http://172.18.150.167:8000/test/badfs，最终到达上游服务器的是GET /test/a/badfs，即将/badfs追加到service的url后面，当strip_path为false时，发向kong的请求：http://172.18.150.167:8000/test/badfs，最终到达上游服务器的是GET /test/atest/badfs，即将test/badfs追加到service的url后面|
|"preserve_host"|默认值为false，表示当把匹配的请求转发给上游服务器时，修改转发给上游的请求的头部Host字段值为当前route对应service的host属性值，true表示保留匹配的请求中的hosts值，即当把匹配的请求转发给上游服务器时，转发给上游的请求的头部Host字段值仍是匹配的请求中的Host字段值|
|service.id|指定当前路由所绑定的对应的服务，对于form-encoded格式：service.id=<service_id>. 对于json格式："service":{"id":"<service_id>"}，例如{"id":"f5a9c0ca-bdbb-490f-8928-2ca95836239a"}|

其中methods、hosts、paths属性用来指定请求和service之间的匹配规则，所以虽然三者都是可选的，但当protocols为http或https时，至少要指定hosts, paths, methods中的一个，且请求必须满足所有已经指定的属性（未指定的属性就不算了），这样才能通过匹配，而且如果一个请求和多个route都可以匹配，则优先匹配所匹配的属性最多的那个，例如路由1：{ "hosts": ["example.com"],},  路由2：{"hosts": ["example.com"],"methods": ["POST"],}，则请求 POST / HTTP/1.1  Host: example.com将匹配路由2， 

例如设置了一个service，指向172.18.150.167:9999/test/a，如果对应的route只指定了hosts为172.18.150.179（即kong的主机ip），那么任何类似于http://172.18.150.179:8000/xxxx的请求的结果都是相当于直接调用172.18.150.167:9999/test/a（strip_path为true）或 172.18.150.167:9999/test/axxxx（strip_path为false）所得到的结果，即这里只要host能对应，不管请求后面的url路径是什么，都不会影响结果，

即请求http://172.18.150.179:8000/xxxx的URI部分xxxx都会被追加到指定的service的URL后面，具体追加多少，还要看strip_path参数，



**route相关api：**

|http方法|路由|含义|
|-|-|-|
|POST|/routes|创建一个route|
|POST|/services/{service name or id}/routes|创建一个指向指定name或id的service的route|
|PATCH|/routes/{name or id}|更新指定name或id的route|
|PATCH|/plugins/{plugin id}/route|更新被指定name或id的plugin所绑定的route|
|PUT|/routes/{name or id}|更新或创建指定name或id的route|
|PUT|/plugins/{plugin id}/route|更新或创建被指定name或id的plugin所绑定的route|
|GET|/routes|列出所有的route|
|GET|/routes/{name or id}|列出指定name或id的route|
|GET|/services/{service name or id}/routes|列出指向指定name或id的service的route|
|GET|/plugins/{plugin id}/route|列出被指定name或id的plugin所绑定的route|
|DELETE|/routes/{name or id}|删除指定name或id的route|



### consumer对象

**consumer对象**

consumer和一些认证插件一起使用，用来保存指定的认证方式对应的用户名密码等验证凭据信息，如果某个api不需要访问验证，那么任何用户都可以访问该api，不需要用到consumer

 

**consumer对象的属性：**

|属性|含义|
|-|-|
|username|为当前consumer指定一个唯一的用户名，至少要指定username和custom_id中的一个参数，|
|custom_id|为当前consumer指定一个唯一的ID，|



**consumer相关api：**

|http方法|路由|含义|
|-|-|-|
|POST|/consumers|创建一个consumer|
|PATCH|/consumers/{username or id}|更新指定username或id的consumer|
|PATCH|/plugins/{plugin id}/consumer|更新与指定id的plugin相关联的consumer|
|PUT|/consumers/{username or id}|更新或创建指定username或id的consumer|
|PUT|/plugins/{plugin id}/consumer|更新或创建与指定id的plugin相关联的consumer|
|GET|/consumers|列出所有的consumer|
|GET|/consumers/{username or id}|列出指定username或id的consumer|
|GET|/plugins/{plugin id}/consumer|列出与指定id的plugin相关联的consumer|
|DELETE|/consumers/{username or id}|删除指定username或id的consumer|

 

### plugin对象

Kong Hub中有各种插件的文档

1、对于请求报文，kong在执行完所有的插件后，才会将请求转发给上游服务器

2、对于响应报文，kong会在转发给客户端之前，执行与route或service相绑定的实现于header_filter阶段的plugin，然后kong会添加以下头部字段到响应报文的头部，最后将其转发给客户端

|header|含义|
|-|-|
|Via: kong/x.x.x|x.x.x表示当前kong的版本|
|X-Kong-Proxy-Latency: <latency>|latency表示kong从接收到客户端请求到将其转发给上游服务器所经过的时间，单位为毫秒，|
|X-Kong-Upstream-Latency: <latency>|latency表示kong等待接收到来自上游服务器的第一个响应字节的时间，单位为毫秒，|

3、当头部被发送到客户端之后，kong开始执行与route或service相绑定的实现于body_filter阶段的plugin，这些插件可能会被执行多次，因为底层nginx是通过一个个数据块的形式处理响应报文的，

4、如果一个插件与route、service、consumer配置多次，则处理顺序为：rsc、rc、sc、rs、c、r、s、global（rsc分别表示route、service、consumer，global表示全局插件）



**plugin对象的属性：**

|属性|含义|
|-|-|
|name|指定要启用的已安装在kong实例中的插件的名字，不能自定义，必须是已存在的插件的名字，|
|route|optional，将当前插件与指定的route绑定，一旦绑定，所有匹配指定route的请求都会激活当前插件，form-encoded：route.id=<id>，JSON："route":{"id":"<id>"}|
|service|optional，将当前插件与指定的service绑定，一旦绑定，所有匹配指定service的请求都会激活当前插件，form-encoded：service.id=<id>，JSON："service":{"id":"<id>"}|
|consumer|optional，将当前插件与指定的consumer绑定，一旦绑定，所有能被consumer鉴权的请求都会激活当前插件，form-encoded：consumer.id=<id>，JSON："consumer":{"id":"<id>"}|
|config|optional，用来配置插件的属性，详见对应的插件|
|run_on|控制该插件运行在哪一个kong节点上，given a Service Mesh scenario. Accepted values are: * first, meaning “run on the first Kong node that is encountered by the request”. On an API Getaway scenario, this is the usual operation, since there is only one Kong node in between source and destination. In a sidecar-to-sidecar Service Mesh scenario, this means running the plugin|
|enabled|optional， 设置是否启用该插件，默认为true|



**plugin相关api：**

|http方法|路由|含义|
|-|-|-|
|POST|/plugin|创建一个公共的**全局的**插件，会在每一个请求上运行（创建完毕后会立即生效）|
|POST|/services/{service id}/plugins|创建一个与指定id的service相关联的plugin|
|POST|/routes/{route id}/plugins|创建一个与指定id的route相关联的plugin|
|POST|/consumers/{consumer id}/plugins|创建一个与指定id的consumer相关联的plugin|
|PATCH|/plugins/{plugin id}|更新一个指定id的plugin|
|PUT|/plugins/{plugin id}|更新或创建一个指定id的plugin|
|GET|/plugins|列出所有已经创建的plugin（被启用的插件不一定是被创建的与指定service、route或consumer想关联的插件）|
|GET|/plugins/{plugin id}|列出指定id的plugin的详细信息|
|GET|/services/{service id}/plugins|列出一个与指定id的service相关联的plugin的详细信息|
|GET|/routes/{route id}/plugins|列出一个与指定id的route相关联的plugin的详细信息|
|GET|/consumers/{consumer id}/plugins|列出一个与指定id的consumer相关联的plugin的详细信息|
|GET|/plugins/enabled|列出所有被启用的plugin列表|
|GET|/plugins/schema/{plugin name}|列出一个指定name的插件的schema，即插件对应的schema.lua文件的返回的表的内容，里面包括该插件的所有配置字段及其默认值等信息，|
|DELETE|/plugins/{plugin id}|删除一个指定id的plugin|



### 自定义插件编写流程                                                                                                                                     

 

#### 自定义插件的配置和文件结构

1、将你的插件名追加到kong配置文件：/etc/kong/kong.conf的plugins属性中，例如plugins = bundled,my-custom-plugin 

2、然后kong会加载路径kong.plugins.my-custom-plugin.<module_name>下的模块，如果该路径不存在，则报错，该路径下必须存在handler.lua和schema.lua两个基本的模块文件，其他模块文件可选，

3、一些插件与kong集成的比较深，比如它们有自己的数据库表、暴露于admin api中的端点等等，一个比较完整的插件可能包含以下模块：它们位于lua库中的/kong/plugins/<custom_plugin_name>/路径下
|文件名|作用|
|-|-|
|api.lua|用于定义与之相关的Admin API列表|
|daos.lua|用于定义与之相关的的数据库表（比如主键、字段等信息），DAOs (Database Access Objects)|
|migrations/*.lua|用于定义要在指定数据库中执行的SQL语句（比如创建表、删除表等），migration指的就是在数据库中创建指定的表，|
|handler.lua|用于定义要在一个请求的指定生命周期节点处要运行的自定义逻辑的处理函数，|
|schema.lua|用于定义该插件的配置属性，例如：配置oauth2插件时的scopes、mandatory_scope、enable_authorization_code等属性|

​    

##### schema.lua文件 

schema文件要返回一个定义了当前插件的配置属性的lua表，该表有以下键：                      

|键|含义|
|-|-|
|no_consumer|值为Boolean类型，默认值为false，true表示该插件只能被应用于Service和Routes，而不能被用于Consumer，|
|fields|值为Table类型，默认值为空表{}，指定插件配置的schema，即一个个属性，属性的值为一个表，表中的键详见下面的附表1|
|self_check|值为Function类型，默认值为nil，指定一个自定义验证函数，可用于在使用Admin API：POST /plugin创建该插件对其配置信息进行校验，校验通过就返回true，否则就返回false和错误信息字符串，self_check指定的函数的参数详见下面的附表2|

附表1：
|键|值类型|值含义|
|-|-|-|
|type|string|表示对应属性值的类型，可以是“id”, “number”, “boolean”, “string”, “table”, “array”, “url”, “timestamp”，id对应的是一个字符串，array对应一个元素均为整数索引的表，url对应一个合法的url，timestamp对应一个数字，|
|required|boolean|默认值为false，true表示对应属性必须要配置，|
|unique|boolean|默认值为false，true表示对应属性的值必须唯一，这对插件配置没有意义，只是用于数据库中的主键唯一性要求|
|default|any|指定对应属性的默认值，|
|immutable|boolean|默认值为false，true表示当前插件一旦被创建，对应属性不允许被更新，|
|enum|table|表的元素必须都是整数索引的，用于指定对应属性可接受的值，例如enum = {"GET", "POST"} 表示对应属性的值必须是GET或POST|
|regex|string|指定一个PCRE正则表达式，属性的值必须满足该正则表达式，|
|schema|table|一个嵌套的schema定义，用于定义对应属性的子属性，要求对应属性的type值为table，|
|func|function|指定一个函数，用于执行自定义验证，该函数有两个参数：value和config，value是用户配置的该属性的值，config是fields键对应的表，如果验证通过，则返回true、nil和一个设置属性默认值的表，见下例，否则返回false和错误信息字符串|


附表2
|参数|含义|
|-|-|
|schema|插件的schema表|
|config|An instance of the DAO (see DAO chapter).|
|dao|An instance of the DAO (see DAO chapter)|
|is_updating|A boolean indicating whether or not this check is performed in the context of an<br/>update.|



例如：schema.lua文件内容

```
local function server_port(given_value, given_config)
  if given_value > 65534 then return false, "port value too high" end
  if given_config.environment == "development" then return true, nil, {port = 8080} end   -- 设置port的默认值为8080
end

return {
  fields = {
    environment = {type = "string", required = true, enum = {"production", "development"}}
    server = {
      type = "table",
      schema = {  -- 实现嵌套的schema
        fields = {
          host = {type = "url", default = "http://example.com"},
          port = {type = "number", func = server_port, default = 80}
        }
      }
    }
  }
}
```



##### handler.lua文件

handler.lua文件要返回一个继承kong.plugins.base_plugin模块所返回对象的子类对象，且该对象可以重写父类对象的一系列函数，这些函数的作用见下方例子，最后handler.lua文件返回该子类对象即可，

kong会调用该扩展对象来处理请求，（kong使用rxi/classic模块模拟lua类），**自定义handler.lua文件大致内容如下，但大部分插件都只重写了下面部分函数，**

**该文件中实现的函数都是被放到ngxin-kong.conf配置文件中的xxxx_by_lua_block指令中执行的**

handler.lua文件模板：

```
local BasePlugin = require "kong.plugins.base_plugin"
local CustomHandler = BasePlugin:extend()  	-- 继承BasePlugin，这两句可以合成一句：local CustomHandler = require("kong.plugins.base_plugin"):extend()

function CustomHandler:new()				-- 对应的Lua-Nginx-Module上下文：init_by_lua_block，在Nginx Master进程加载nginx配置文件时执行
  CustomHandler.super.new(self, "my-plugin") 	-- 这里的super是其父类kong.plugins.base_plugin的父类kong.vendor.classic中的一个属性，
  -- 在这里实现自定义的逻辑
end

function CustomHandler:init_worker()			-- 对应的Lua-Nginx-Module上下文：init_worker_by_lua_block，在每个Nginx Worker进程启动时执行
  CustomHandler.super.init_worker(self)		-- 执行父类对该函数的实现，会将当前插件进入上下文的信息记录到日志，
  -- 在这里实现自定义的逻辑，如果在这里有sleep()函数，那么该sleep()函数会阻塞整个nginx的启动，因为这里的代码最终是被放到ngxin-kong.conf中的init_worker_by_lua_block指令中执行的，
end

function CustomHandler:certificate(config)		-- 对应的Lua-Nginx-Module上下文：ssl_certificate_by_lua_block，在SSL握手的SSL证书服务阶段执行
  CustomHandler.super.certificate(self)
  -- 在这里实现自定义的逻辑
end

function CustomHandler:rewrite(config)		-- 对应的Lua-Nginx-Module上下文：rewrite_by_lua_block，在每个请求中的rewrite阶段执行，注意：由于这个阶段中还未找到与当前请求相关的Service
  CustomHandler.super.rewrite(self)			-- 和Consumer，所以只有当前插件是一个全局插件时，该函数才会被执行，
  -- 在这里实现自定义的逻辑
end

function CustomHandler:access(config)		-- 对应的Lua-Nginx-Module上下文：access_by_lua_block，在被代理至上游服务前执行
  CustomHandler.super.access(self)
  -- 在这里实现自定义的逻辑，比如ngx.req.set_header("Hello-World", "this is on a request")
end

function CustomHandler:header_filter(config)	-- 对应的Lua-Nginx-Module上下文：header_filter_by_lua_block，在从上游服务器接收到所有Response headers之后执行
  CustomHandler.super.header_filter(self)
  -- 在这里实现自定义的逻辑，比如ngx.header["Bye-World"] = "this is on the response"
end

function CustomHandler:body_filter(config)		-- 对应的Lua-Nginx-Module上下文：body_filter_by_lua_block，在从上游服务接收到响应主体的每个块时执行
  CustomHandler.super.body_filter(self)
  -- 在这里实现自定义的逻辑
end

function CustomHandler:log(config)			-- 对应的Lua-Nginx-Module上下文：log_by_lua_block，当最后一个响应字节已经被发送到客户端时执行
  CustomHandler.super.log(self)
  -- 在这里实现自定义的逻辑
end

return CustomHandler

--[[
config参数：
上面的每个函数都有个参数config，它是一个由kong提供的包含插件的配置参数（定义于schema.lua模块中，用户创建插件时会配置这些参数的值）的lua表，config的用法和《schema.lua文件》章节中的例3相同，例如：
kong.log.inspect(config.environment) -- "development"
kong.log.inspect(config.server.host) -- "http://localhost"
kong.log.inspect(config.server.port)

CustomHandler对象的属性：
CustomHandler.PRIORITY		用于指定插件执行优先级，因为一些插件可能依赖于其他插件的执行，例如依赖于consumer认证插件的插件会运行在认证插件之后，值越大，优先级越高
						例如KeyAuthHandler.PRIORITY = 1003， 当前已经存在的插件的优先级可以到其handler.lua文件中查看，例如zipkin是100000，jwt是1005，oauth2是1004，
CustomHandler.VERSION		用于指定插件的版本号，例如KeyAuthHandler.VERSION = "1.0.0"
--]]
```



##### migrations目录下的文件

当确定了插件的配置参数等模型后，就可以创建迁移模块文件了，迁移模块用于在数据库中创建指定的表，

迁移模块文件返回一个表（假设为A），表中有一个没有键的表（假设为B），B中有以下3个键：

|键名|含义|
|-|-|
|name|值为一个唯一的字符串|
|up|值为一个字符串，当kong向后迁移的时候被执行，字符串值为SQL语句（Structured Query Language）或CQL语句（Cassandra Query Language） 或 Lua代码，|
|down|值为一个字符串，当kong向前迁移的时候被执行，字符串值同上，这种设计主要是考虑到插件版本更新可能要修改数据库表的原先设计，|
cassandra数据库的migration文件示例：

```
return {
  {
    name = "2015-07-31-172400_init_keyauth",
    up =  [[									-- 值完全为CQL语句
      CREATE TABLE IF NOT EXISTS keyauth_credentials(
        id uuid,
        consumer_id uuid,
        key text,
        created_at timestamp,
        PRIMARY KEY (id)
      );

      CREATE INDEX IF NOT EXISTS ON keyauth_credentials(key);
      CREATE INDEX IF NOT EXISTS keyauth_consumer_id ON keyauth_credentials(consumer_id);
    ]],
    down = [[
      DROP TABLE keyauth_credentials;
    ]]
  }
}
```



postgresql数据库的migration文件示例：

```
return {
  {
    name = "2015-07-31-172400_init_keyauth",
    up = [[								-- 值为SQL语句和Lua代码
      CREATE TABLE IF NOT EXISTS keyauth_credentials(
        id uuid,
        consumer_id uuid REFERENCES consumers (id) ON DELETE CASCADE,
        key text UNIQUE,
        created_at timestamp without time zone default (CURRENT_TIMESTAMP(0) at time zone 'utc'),
        PRIMARY KEY (id)
      );
	 -- lua代码格式：
      DO $$
      BEGIN
        IF (SELECT to_regclass('public.keyauth_key_idx')) IS NULL THEN
          CREATE INDEX keyauth_key_idx ON keyauth_credentials(key);
        END IF;
        IF (SELECT to_regclass('public.keyauth_consumer_idx')) IS NULL THEN
          CREATE INDEX keyauth_consumer_idx ON keyauth_credentials(consumer_id);
        END IF;
      END$$;
    ]],
    down = [[
      DROP TABLE keyauth_credentials;
    ]]
  }
}

```



##### daos.lua文件

该文件供kong创建对应的自定义DAO对象之用，kong会使用它实例化一个DAO对象，比如kong.db.keyauth_credentials就是kong基于key_auth插件的daos.lua文件创建的针对名为key_credentials的数据库表的DAO对象，该DAO对象的增删改查函数可以用于对数据库表key_credentials进行增删改查操作，

该文件返回一个表，表中有一个键，表示对应的数据库表名，键对应的值是一个表，表中有以下固定的键：

|键名|含义|
|-|-|
|primary_key|Table，是一个元素均为整数索引的表，表示主键名，值为表主要是为了支持复合主键（因为复合主键由多个键共同组成），|
|table|String，指定该表的表名，|
|cache_key|Table，指定该DAO对象实体被缓存时的键名，方便通过kong.cache获取，例如：cache_key = { "key" }，该键名应该使用对应表中具有唯一性约束的列|
|fields|Table，指定该表中的各个字段，其值中的键对应的表中的键详见附表3|



附表3：

|键名|含义|
|-|-|
|type|同附表1|
|required|同附表1|
|immutable|同附表1|
|dao_insert_value|Boolean，true表示该字段要被DAO自动生成，如果该字段的type值为id，则其值自动生成为uuid，如果type为timestamp，则生成一个timestamp，|
|queryable|Boolean，true表示指定Cassandra在指定的列上维护一个索引，这允许查询由该列过滤的column family|
|foreign|String，指定该列是一个外键（指向另一个实体的列）其值的格式为：dao_name:column_name，例如：foreign = "consumers:id"，这弥补了Cassandra不支持|



例如：key-auth插件的dao文件

```
return {
keyauth_credentials = {
  primary_key = {"id"},
  table = "keyauth_credentials",  -- the actual table in the database
  fields = {
    id = {type = "id", dao_insert_value = true}, -- a value to be inserted by the DAO itself (think of serial ID and the uniqueness of such required here)
    created_at = {type = "timestamp", immutable = true, dao_insert_value = true}, -- also interted by the DAO itself
    consumer_id = {type = "id", required = true, foreign = "consumers:id"}, -- a foreign key to a Consumer's id
    key = {type = "string", required = false, unique = true} -- a unique API key
  }
}
}
```



##### api.lua文件

该文件用于为自定义插件创建的admin api，文件返回一个表，该表描述了对应的路由和支持的http方法，最终kong会将该表反馈给Lapis第三方模块，由它实现该api，例如：

```
return {

  ["/my-plugin/new/get/endpoint"] = { GET = function(self, dao_factory, helpers)  -- 函数内容 end }

}
```

api.lua文件返回的表中的元素的键为路由，值为一个表，表中键有以下3种：

|键名|值含义|
|-|-|
|<http方法>|指定一个函数，用来处理http方法有关的请求，请求处理方法有3个参数：self(表示Lapis请求对象)、dao_factory（The DAO Factory. See the datastore chapter of this guide）、helpers(一个包含帮助信息的表)|
|before|指定一个函数，用于在执行http方法对应的函数之前执行，|
|on_error|指定一个函数，用来处理错误，它会重写由kong提供的错误处理函数，|

例如：

```
return {
  ["/consumers/:consumers/basic-auth"] = {
      GET = endpoints.get_collection_endpoint(credentials_schema, consumers_schema, "consumer"),
      POST = endpoints.post_collection_endpoint(credentials_schema, consumers_schema, "consumer"),
  },
  ["/consumers/:consumers/basic-auth/:basicauth_credentials"] = {
      before = function(self, db)  --....   end,
      GET  = endpoints.get_entity_endpoint(credentials_schema),
      PUT  = function(self, ...) -- ....      end,
      PATCH  = endpoints.patch_entity_endpoint(credentials_schema),
      DELETE = endpoints.delete_entity_endpoint(credentials_schema),
  }
}
```



##### 自定义插件的安装

**方法1：手动安装**

假设插件名为myplugin，插件相关的源码文件均存放于名为myplugin的文件夹中，那么：

1、修改/etc/kong/kong.conf，添加自定义的插件名myplugin，例如：plugins = bundled, myplugin

2、复制文件夹myplugin到kong的插件库目录中（例如/usr/local/share/lua/5.1/kong/plugin）

3、执行sudo kong restart重启kong，然后通过Postman就可以查看到该插件已经被添加到kong中了



**方法2：通过luarocks安装**

 1、在/home/apex/myplugin目录下创建handler.lua、schema.lua和kong-plugin-myplugin1-0.1.0-1.rockspec三个文件，前两个是要安装到lua库路径中的lua模块文件，![](https://test-1256398113.cos.ap-guangzhou.myqcloud.com/20190311171027.png)

其中kong-plugin-myplugin1-0.1.0-1.rockspec文件的内容如下：

```
package = 'kong-plugin-myplugin1'
version = '0.1.0-1'

supported_platforms = {"linux", "macosx"}
source = {	
  url = "https://github.com/ApexPredator1/rock_test/raw/master/test.tar.gz",	
-- test.tar.gz由来：首先创建一个包含handler.lua和schema.lua两个文件的test目录，然后通过命令tar -czf test.tar.gz test将test目录打包成test.tar.gz，并上传的github，
}

description = {
  summary = "Kong is a scalable and customizable API Management Layer built on top of Nginx.",
  homepage = "http://getkong.org",
  license = "Apache 2.0"
}

dependencies = {
}

build = {
  type = "builtin",
  modules = {
    ["kong.plugins."..'myplugin1'..".handler"] = "handler.lua",
    ["kong.plugins."..'myplugin1'..".schema"] = "schema.lua",
  }
}
```

2、执行sudo luarocks make kong-plugin-myplugin1-0.1.0-1.rockspec 或 sudo luarocks make命令来安装这两个模块文件，对于后者，它会自动在当前目录下搜索*.rockspec文件，![](https://test-1256398113.cos.ap-guangzhou.myqcloud.com/20190311171134.png)



##### 自定义插件的移除

1、首先要确保删除已经创建的全局的或与service、route或consumer相关联的该插件（如果贸然直接进行第2、3步，那么会导致kong无法启动，因为指定的插件还未被删除）

2、修改kong配置文件/etc/kong/kong.conf中的plugins属性值，将自己的插件名移除，

3、先通过luarocks list查看自己的插件的包名，然后根据包名删除它，然后需要重启kong才能生效

![](https://test-1256398113.cos.ap-guangzhou.myqcloud.com/20190311171549.png)

![](https://test-1256398113.cos.ap-guangzhou.myqcloud.com/20190311171602.png)



##### 自定义插件编写实例

**例1：给响应报文添加一个自定义内容的header**

1、创建文件schema.lua，内容如下：

```
local function validation(schema, config, dao, is_updating)
        if #config.content < 10 then
                return false, "length of content can not less 10"
        else
                return true
        end
end

local function(value, config)
		if #value < 10 then
                return false, 'length of content can not less 10'
         else
                 return true
         end
end

return {
        no_consumer = true, 
        fields = {
                content = {type = 'string', required = true}	-- 用来表示新header的内容
        },
        self_check = validation  -- 测试发现self_check和func指定的函数都不起作用
}
```

2、创建文件handler.lua，内容如下：

```
local CustomHandler = require("kong.plugins.base_plugin"):extend()
  
function CustomHandler:header_filter(config)
        CustomHandler.super.header_filter(self)
        ngx.header["custom"] = config.content
end

return CustomHandler
```

3、在/usr/local/share/lua/5.1/kong/plugins目录（目录/usr/local/share/lua/5.1/kong是kong的lua库目录）下创建名为myplugin的目录，并把上面两个文件复制到myplugin目录下，

4、修改/etc/kong/kong.conf，添加自定义的插件名myplugin，例如：plugins = bundled, myplugin，执行sudo kong restart重启kong，然后通过Postman就可以查看到该插件已经被添加到kong中了

![](https://test-1256398113.cos.ap-guangzhou.myqcloud.com/20190311171823.png)

5、将该插件配置为一个全局插件，并测试一个service，会发现其响应报文中多个一个自定义的header字段，

![](https://test-1256398113.cos.ap-guangzhou.myqcloud.com/20190311171910.png)



#### basic-auth插件

**创建basic-auth插件时需要配置的属性：**

|属性|含义|
|-|-|
|name|指定插件名，在这里为basic-auth|
|service_id|指定插件要绑定的service的id|
|route_id|指定插件要绑定的route的id|
|enabled|默认值为true，true表示启用该插件，false表示不启用，|
|config.hide_credentials|可选，默认值为false，设置是否对上游服务器隐藏credential，true表示在请求被转发到上游服务器前，其credential会被去除，即上游服务器就看不到认证信息了，|
|config.anonymous|可选，默认值为空，指定一个uuid值，表示如果认证失败，则将其用作一个匿名(anonymous)的consumer的id，如果参数为空，则认证失败的请求会返回4xx错误|



**例1：创建一个指向百度的service，并实现basic-auth认证**

 注：下面每一个POST请求中，Header头部的内容都是：

![](https://test-1256398113.cos.ap-guangzhou.myqcloud.com/20190311172121.png)

1、首先启用一个面向全局的basic-auth认证插件

![](https://test-1256398113.cos.ap-guangzhou.myqcloud.com/20190311172231.png)

2、创建一个指向百度的service，用来代理百度，并为该服务创建一个基于host的路由

![1552296181547](C:\Users\preda\AppData\Roaming\Typora\typora-user-images\1552296181547.png)

3、创建一个consumer对象consumer1，并将之与basic-auth插件相关联，并保存basic-auth认证的用户名和密码

![1552296261852](C:\Users\preda\AppData\Roaming\Typora\typora-user-images\1552296261852.png)

4、使用postman访问kong接口，只需要一个Authorization（或Proxy-Authorization）头部字段，其值为Basic <base64(username:password)> 或 basic <base64(username:password)>，，例如字符串"test:123456"的base64编码为dGVzdDoxMjM0NTY=，则对应Authorization头部字段的值为Basic dGVzdDoxMjM0NTY=

![](https://test-1256398113.cos.ap-guangzhou.myqcloud.com/20190311172434.png)

5、使用浏览器访问kong接口，输入用户名和密码即可

![](https://test-1256398113.cos.ap-guangzhou.myqcloud.com/20190311172547.png)

![](https://test-1256398113.cos.ap-guangzhou.myqcloud.com/20190311172600.png)



#### key-auth插件

**例1：创建一个指向百度的service，并实现key-auth认证（只需要key的认证）**

1、首先启用一个面向全局的key-auth认证插件

![](https://test-1256398113.cos.ap-guangzhou.myqcloud.com/20190311172650.png)

2、创建一个指向百度的service，用来代理百度，并为该服务创建一个基于host的路由

![1552296444477](C:\Users\preda\AppData\Roaming\Typora\typora-user-images\1552296444477.png)

3、创建一个consumer对象consumer1，并将之与key-auth插件相关联，并保存key-auth认证的apikey首部行的值test

![](https://test-1256398113.cos.ap-guangzhou.myqcloud.com/20190311172817.png)

4、通过设置首部行apikey=test来访问被代理的百度，浏览器无法访问，因为无法提供apikey

![](https://test-1256398113.cos.ap-guangzhou.myqcloud.com/20190311172845.png)





#### oauth2插件

**oauth2**：open authorization 2，是一个开放授权的标准，旨在让第三方应用访问用户在某服务器中的私有资源，而无需提供用户在某服务器上的账号密码给到第三方应用

**oauth2的运行流程：**

```
A、客户端向资源拥有者请求用户授权（这里用户要点击确认授权 或 选择静默授权，比如这里的客户端是某网站，资源拥有者是微信用户，某网站要获取微信用户的用户名头像等受保护资源）
B、资源拥有者向客户端提供用户授权
C、客户端将获得的用户授权发送给认证服务器，以请求访问令牌（token）
D、认证服务器验证用户授权无误后，向客户端提供访问令牌（token）
E、客户端通过访问令牌去访问资源服务器，
F、资源服务器验证令牌无误后，将受保护的资源响应给客户端

相关概念：
客户端：client：即访问资源服务器的浏览器或用户
资源拥有者：resource owner，即资源服务器的拥有者
认证服务器：authorization server，用于获取token，
资源服务器：resource server，客户端要访问的某个网站或服务
```

**用户授权有以下4种：**

1、授权码授权（authorization code grant）：授权码模式，

```
1、授权码请求：第三方应用使用用户代理（浏览器，或自己的服务器）访问微信授权服务器提供的用于获取授权码的url，同时第三方应用需要传递一些关于自身信息的参数

2、授权码返回：微信授权服务器验证第三方应用是否合法，确认无误后会提供一个页面给微信用户a登录或者手动确认授权操作，操作成功后微信授权服务器会将授权许可码code返回

3、访问令牌请求：第三方应用根据上一步拿到的授权码code以及app_id，重定向地址等参数，再次请求微信授权服务器，请求获取访问令牌access_token

4、访问令牌返回：微信授权服务器返回访问令牌和一些刷新令牌，令牌过期时间等信息给到第三方应用

5、受限资源获取：第三方应用根据访问令牌去微信资源服务器获取微信用户a的基本信息
```



 2、隐式授权（implicit grant）：是授权码授权的简化版本，其中省略了颁发授权码code给第三方应用的步骤，通过一次请求微信授权服务器直接返回访问令牌以及刷新令牌等信息，适用于没有server服务器来接受处理授权码的第三方应用，

```
1、访问令牌请求：第三方应用使用用户代理（浏览器，或自己的服务器）访问微信授权服务器提供的用于获取访问令牌的url，同时第三方应用需要传递一些关于自身信息的参数

2、访问令牌返回：微信授权服务器验证第三方应用是否合法，确认无误后返回访问令牌和一些刷新令牌，令牌过期时间等信息给到第三方应用

3、受限资源获取：第三方应用根据访问令牌去微信资源服务器获取微信用户a的基本信息
```



3、密码授权（resource owner password credentials grant）：密码模式，该授权模式是第三方应用直接使用微信用户的账号+密码来直接请求微信授权服务器获取access_token，该模式适用于微信资源服务器高度信任第三方client的情况，

```
1、访问令牌请求：请求参数中包含微信用户的用户名和密码等信息，

2、访问令牌返回：微信授权服务器验证用户名和密码，确认无误后返回访问令牌和一些刷新令牌，令牌过期时间等信息给到第三方应用
```



4、客户端凭证（client credentials）：客户端模式，第三方应用直接使用自己的名义去要求访问微信资源服务器的一些受保护资源，

```
1、授权请求：以自己的名义，请求参数中包含自己的信息

2、授权返回：微信授权服务器验证第三方应用是否合法，确认无误后返回访问令牌和一些刷新令牌，令牌过期时间等信息给到第三方应用
```



**oauth2插件的config属性**

| 子属性 | 含义 |
| ------ | ---- |
| config.scopes | 指定授权范围，其值为逗号分隔的scope名，例如config.scopes=email,phone,address， |
|config.mandatory_scope|optional，默认值为false，是否强制要求至少有一个scope要被终端用户授权|
|config.token_expiration|optional，默认值为7200，设置token的超时时间，单位是秒，当这个时间过后，客户端需要刷新token，设置为0表示禁止时间超时，即时间永不会超时|
|config.enable_authorization_code|optional，默认值为false，设置是否启用授权码授权|
|config.enable_client_credentials|optional，默认值为false，设置是否启用客户端凭证|
|config.enable_implicit_grant|optional，默认值为false，设置是否启用隐式授权|
|config.enable_password_grant|optional，默认值为false，设置是否启用密码授权|
|config.auth_header_name|optional，默认值为authorization，设置用来携带访问token的头部字段|
|config.hide_credentials|optional，默认值为false，设置是否对上游服务器隐藏credential，true表示在请求被转发到上游服务器前，其credential会被去除，即上游服务器就看不到token了|
|config.accept_http_if_already_terminated|optional，默认值为false， Accepts HTTPs requests that have already been terminated by a proxy or load balancer and the x-forwarded-proto: https header has been added to the request. Only enable this option if the Kong server cannot be publicly accessed and the only entry-point is such proxy or load balancer.|
|config.anonymous|optional，默认值为空，指定一个uuid值，表示如果认证失败，则将其用作一个匿名(anonymous)的consumer的id，如果参数为空，则认证失败的请求会返回4xx错误|
|config.global_credentials|optional，默认值为false，true表示允许在其他使用OAuth2插件的config.global_credentials值为true的service中使用同一个的OAuth凭证|
|config.refresh_token_ttl|optional，默认值为1209600，单位为秒，表示两周，An optional integer value telling the plugin how many seconds a token/refresh token pair is valid for, and can be used to generate a new access token. Set to 0 to keep the token/refresh token pair indefinitely valid|

**oauth2插件的api：基于service对应的path接口，例如172.18.150.167:8000/oauth2/token**

| http方法 | 路由              | 含义                                                         |
| -------- | ----------------- | ------------------------------------------------------------ |
| POST     | /oauth2/authorize | 授权服务器的接口，用于授权服务器为授权码授权模式流程的第一步提供授权码，或 为隐式授权模式流程提供访问令牌， |
|POST|/oauth2/token|授权服务器的接口，用于授权服务器为授权码授权模式 或 密码授权模式 或 客户端凭证模式提供访问令牌，|



**oauth2插件的密码授权模式：**

kong的官方设计是其oauth2插件的密码授权模式要求在外部搭建一个应用后端backend，用来来做客户端与kong中间层，如下图，当然也可以把backend放到kong的service中，由kong作代理，

![](https://test-1256398113.cos.ap-guangzhou.myqcloud.com/20190311174018.png)                                                  

这里的kong是验证服务器，client application是客户端，webapp backend是资源拥有者，client要访问的kong中的service是资源服务器，



**例1：为指定服务创建基于密码授权、授权码授权、隐式授权、客户端凭证授权的OAuth2验证**

**准备工作：**

1、因为请求token的接口必须是https接口，所以需要在kong配置文件：/etc/kong/kong.conf中进行如下配置：

![](https://test-1256398113.cos.ap-guangzhou.myqcloud.com/20190311185329.png)

2、首先需要关闭postman中的File > Settings > General中的'SSL certificate verification'项

3、创建一个指向http://www.baidu.com/的service，并创建与其关联的route和oauth2插件，此时由于没有token，所以访问该service会得到错误页面：{“error_description”: "The access token is missing"}

下面的oauth2插件同时启用了密码授权方式、授权码授权模式、隐式授权模式、客户端凭证授权模式，

![](https://test-1256398113.cos.ap-guangzhou.myqcloud.com/20190311185119.png)

**实验1：密码授权模式**

1、创建一个oauth2插件授权的客户端，该客户端需要使用token才能访问oauth2插件对应的service：创建一个consumer（代表实际的客户端的凭证信息），并将之与oauth2插件相关联，oauth2插件的消费者consumer就是客户端client，其中name、client_id和client_secret是授权服务器为app（客户端）提供的身份标识信息，其中client_id和client_secrets是可选的，如果不声明，则插件会自动生成它们的值，redirect_uris是APP向OAUTH2提供的回调地址，可以声明多个，在redirect_uris中users will be sent after authorization

![](https://test-1256398113.cos.ap-guangzhou.myqcloud.com/20190311185233.png)

2、因为没有条件创建backend，这里省略了kong的密码授权官方流程图中client向backend发送用户名和密码的验证流程，client直接冒充backend使用backend的身份信息（client_id、client_secret等）向kong请求token，其中的provision_key在将oauth2插件关联至service的结果中，也可以直接在指定的插件信息中看到，如果对应的service的路由有设置paths：xxxx，则形式应该是172.18.150.167:8000/xxxx/oauth2/token

3、获取token

 ![](https://test-1256398113.cos.ap-guangzhou.myqcloud.com/20190311185452.png)

此时就可以通过token访问了：curl -i -X GET-url https://172.18.150.167:8443/ --header "Authorization: Bearer W86Uxf1q3eKG3QGn2UnfTgEg4PUg8QLB"  或 使用postman如下：

![](https://test-1256398113.cos.ap-guangzhou.myqcloud.com/20190311185521.png)



**实验2：授权码授权模式**

1、如下左图，获取授权码，consumer与oauth2插件关联时指定了redirect_uris=https://www.baidu.com/，这里请求授权码时，授权服务器（kong oauth2插件）会将授权码作为请求参数来回调该uri，以将授权码传给该uri，

2、如下右图，获取token，根据授权码等参数获取token令牌，

![](https://test-1256398113.cos.ap-guangzhou.myqcloud.com/20190311190102.png)

3、根据token访问目标service

![](https://test-1256398113.cos.ap-guangzhou.myqcloud.com/20190311190037.png)



**实验3：隐式授权模式**

![](https://test-1256398113.cos.ap-guangzhou.myqcloud.com/20190311190007.png)



**实验4：客户端凭证授权模式**

![](https://test-1256398113.cos.ap-guangzhou.myqcloud.com/20190311190209.png)



**刷新token**：refresh token，通过refresh_token重新获取一个新的refresh_token和新的token，目前密码授权和授权码授权才有refresh_token，而隐式授权和客户端凭证没有refresh_token，猜测是因为前两者获取token的过程比较麻烦，所以才提供了refresh_token这种再次获取新token的简便方法，而后两者获取token的方法很简单，所以就没提供refresh_token，

![](https://test-1256398113.cos.ap-guangzhou.myqcloud.com/20190311190422.png)





### certificate对象

certificate对象表示一个SSL证书所使用的公钥/私钥对（cert/key），一个可选的功能是，certificate对象可以和sni对象关联在一起，来将一个公钥/私钥对绑定到一个或多个域名上，

kong 的 https 端口上默认绑着 localhost 的自签证书用于开发环境和测试

 

**certificate对象的属性：**

|属性|含义|
|-|-|
|cert|指定SSL密钥对中以PEM编码的证书内容|
|key|指定SSL密钥对中以PEM编码的私钥内容|
|snis|指定一个hostnames数组，kong会基于此数组自动创建一个与当前certificate相关联的SNI对象|



**certificate对象的api：**
|http方法|路由|作用|
|-|-|-|
|POST|/certificates|创建一个certificate对象|
|PATCH|/certificates/{certificate id}|更新指定id的certificate对象|
|PUT|/certificates/{certificate id}|更新或创建指定id的certificate对象|
|GET|/certificates|列出所有的certificate对象|
|GET|/certificates/{certificate id}|列出指定id的certificate对象|
|DELETE|/certificates/{certificate id}|删除指定id的certificate对象|
​                                           

 **例：详见openssl笔记，自签名证书的创建方法，版本2**

```
创建自签名CA根证书
sudo openssl genrsa -out cakey.pem 2048
sudo openssl req -new -key cakey.pem -out ca.csr -subj "/C=CN/ST=myprovince/L=mycity/O=myorganization/OU=mygroup/CN=www.certtest.com" 
openssl x509 -req -days 365 -sha1 -extensions v3_ca -signkey cakey.pem -in ca.csr -out  cacert.pem
sudo openssl x509 -req -days 365 -sha1 -extensions v3_ca -signkey cakey.pem -in ca.csr -out  cacert.pem
 

使用根证书签发服务器端的证书
sudo openssl genrsa -out key.pem 2048 
sudo openssl req -new -key key.pem -out server.csr -subj "/C=CN/ST=myprovince/L=mycity/O=myorganization/OU=mygroup/CN=www.certtest.com"
sudo openssl x509 -req -days 365 -sha1 -extensions v3_req -CA cacert.pem -CAkey cakey.pem -CAserial ca.srl -CAcreateserial -in server.csr -out cert.pem
sudo openssl verify -CAfile cacert.pem cert.pem   
sudo cat cert.pem               将证书内容复制作为下图中cert参数的值
sudo cat key.pem                将私钥内容复制作为下图中key参数的值
```

![](https://test-1256398113.cos.ap-guangzhou.myqcloud.com/20190311190848.png)



### SNI对象                                    

SNI：Server Name Indication，是为了解决一个服务器使用多个域名和证书的SSL/TLS扩展。工作原理是：在连接到服务器建立SSL连接之前先发送要访问站点的域名，而服务器则根据这个域名返回一个合适的证书。

 一个SNI对象表示一个域名与证书之间的多对一映射，即一个certificate对象可以与多个域名相关联，当kong接收到一个SSL请求时，它会通过客户端请求中的SNI字段寻找相应的证书，                    

**SNI对象的属性：**

|属性|含义|
|-|-|
|name|为当前SNI对象指定一个名字，例如my-sni，|
|certificate|用于指定与当前SNI对象关联的certificate对象，例如form-encoded格式：certificate.id=<certificate_id>. json格式："certificate":{"id":"<certificate_id>"}|


​                   

 **SNI对象的api：**
|http方法|路由|作用|
|-|-|-|
|POST|/snis|创建一个sni对象|
|POST|/certificates/{certificate name or id}/snis|创建一个与指定name或id的certificate对象相关联的sni对象|
|PATCH|/snis/{name or id}|更新指定name或id的sni对象|
|PUT|/snis/{name or id}|更新或创建指定name或id的sni对象|
|GET|/snis|列出所有SNI对象|
|GET|/snis/{name or id}|列出指定name或id的sni对象|
|GET|/certificates/{certificate name or id}/snis|列出与指定name或id的certificate对象相关联的sni对象|
|DELETE|/snis/{name or id}|删除指定name或id的sni对象|
​                                                           

### upstream对象                                            

**upstream相关属性：**

| 属性 | 含义 |
| ---- | ---- |
| name | 指定一个hostname，它必须等于对应service的host属性值， |
|hash_on|optional， What to use as hashing input: none (resulting in a weighted-round-robin scheme with no hashing), consumer, ip, header, or cookie. Defaults to "none".|
|hash_fallback|optional    What to use as hashing input if the primary hash_on does not return a hash (eg. header is missing, or no consumer identified). One of: none, consumer, ip, header, or cookie. Not available if hash_on is set to cookie. Defaults to "none".|
|hash_on_header|semi-optional   The header name to take the value from as hash input. Only required when hash_on is set to header.|
|hash_fallback_header|semi-optional   The header name to take the value from as hash input. Only required when hash_fallback is set to header.|
|hash_on_cookie|semi-optional   The cookie name to take the value from as hash input. Only required when hash_on or hash_fallback is set to cookie. If the specified cookie is not in the request, Kong will generate a value and set the cookie in the response.|
|hash_on_cookie_path|semi-optional   The cookie path to set in the response headers. Only required when hash_on or hash_fallback is set to cookie. Defaults to "/".|
|slots|optional    The number of slots in the loadbalancer algorithm (10-65536). Defaults to 10000.|
|healthchecks.active.https_verify_certificate|optional    Whether to check the validity of the SSL certificate of the remote host when performing active health checks using HTTPS. Defaults to true.|
|healthchecks.active.unhealthy.http_statuses|optional    An array of HTTP statuses to consider a failure, indicating unhealthiness, when returned by a probe in active health checks. Defaults to [429, 404, 500, 501, 502, 503, 504, 505]. With form-encoded, the notation is http_statuses[]=429&http_statuses[]=404. With JSON, use an Array.|
|healthchecks.active.unhealthy.tcp_failures|optional    Number of TCP failures in active probes to consider a target unhealthy. Defaults to 0.|
|healthchecks.active.unhealthy.timeouts|optional    Number of timeouts in active probes to consider a target unhealthy. Defaults to 0.|
|healthchecks.active.unhealthy.http_failures|optional    Number of HTTP failures in active probes (as defined by healthchecks.active.unhealthy.http_statuses) to consider a target unhealthy. Defaults to 0.|
|healthchecks.active.unhealthy.interval|optional    Interval between active health checks for unhealthy targets (in seconds). A value of zero indicates that active probes for unhealthy targets should not be performed. Defaults to 0.|
|healthchecks.active.http_path|optional    Path to use in GET HTTP request to run as a probe on active health checks. Defaults to "/".|
|healthchecks.active.timeout|optional    Socket timeout for active health checks (in seconds). Defaults to 1.|
|healthchecks.active.healthy.http_statuses|optional    An array of HTTP statuses to consider a success, indicating healthiness, when returned by a probe in active health checks. Defaults to [200, 302]. With form-encoded, the notation is http_statuses[]=200&http_statuses[]=302. With JSON, use an Array.|
|healthchecks.active.healthy.interval|optional    Interval between active health checks for healthy targets (in seconds). A value of zero indicates that active probes for healthy targets should not be performed. Defaults to 0.|
|healthchecks.active.healthy.successes|optional    Number of successes in active probes (as defined by healthchecks.active.healthy.http_statuses) to consider a target healthy. Defaults to 0.|
|healthchecks.active.https_sni|optional    The hostname to use as an SNI (Server Name Identification) when performing active health checks using HTTPS. This is particularly useful when Targets are configured using IPs, so that the target host’s certificate can be verified with the proper SNI.|
|healthchecks.active.concurrency|optional    Number of targets to check concurrently in active health checks. Defaults to 10.|
|healthchecks.active.type|optional    Whether to perform active health checks using HTTP or HTTPS, or just attempt a TCP connection. Possible values are tcp, http or https. Defaults to "http".|
|healthchecks.passive.unhealthy.http_failures|optional    Number of HTTP failures in proxied traffic (as defined by healthchecks.passive.unhealthy.http_statuses) to consider a target unhealthy, as observed by passive health checks. Defaults to 0.|
|healthchecks.passive.unhealthy.http_statuses|optional    An array of HTTP statuses which represent unhealthiness when produced by proxied traffic, as observed by passive health checks. Defaults to [429, 500, 503]. With form-encoded, the notation is http_statuses[]=429&http_statuses[]=500. With JSON, use an Array.|
|healthchecks.passive.unhealthy.tcp_failures|optional    Number of TCP failures in proxied traffic to consider a target unhealthy, as observed by passive health checks. Defaults to 0.|
|healthchecks.passive.unhealthy.timeouts|optional    Number of timeouts in proxied traffic to consider a target unhealthy, as observed by passive health checks. Defaults to 0.|
|healthchecks.passive.type|optional    Whether to perform passive health checks interpreting HTTP/HTTPS statuses, or just check for TCP connection success. Possible values are tcp, http or https (in passive checks, http and https options are equivalent.). Defaults to "http".|
|healthchecks.passive.healthy.successes|optional    Number of successes in proxied traffic (as defined by healthchecks.passive.healthy.http_statuses) to consider a target healthy, as observed by passive health checks. Defaults to 0.|
|healthchecks.passive.healthy.http_statuses|optional    An array of HTTP statuses which represent healthiness when produced by proxied traffic, as observed by passive health checks. Defaults to [200, 201, 202, 203, 204, 205, 206, 207, 208, 226, 300, 301, 302, 303, 304, 305, 306, 307, 308]. With form-encoded, the notation is http_statuses[]=200&http_statuses[]=201. With JSON, use an Array.|



**upstream相关api：**

|http方法|路由|作用|
| -    | -    | -    |
|POST|/upstreams|创建一个upstream对象|
|PATCH|/upstreams/{name or id}|更新指定name或id的upstream对象|
|PATCH|/targets/{target host:port or id}/upstream|更新与指定target相关联的upstream对象|
|PUT|/upstreams/{name or id}|更新或创建指定name或id的upstream对象|
|PUT|/targets/{target host:port or id}/upstream|更新或创建与指定target相关联的upstream对象|
|GET|/upstreams|列出所有的upstream对象|
|GET|/upstreams/{name or id}|列出指定name或id的upstream对象|
|GET|/targets/{target host:port or id}/upstream|列出与指定target相关联的upstream对象|
|GET|/upstreams/{name or id}/health/|获取指定name或id的upstream对象的各个target的健康情况|
|DELETE|/upstreams/{name or id}|删除指定name或id的upstream对象|
|DELETE|/targets/{target host:port or id}/upstream|删除与指定target相关联的upstream对象|
​                                                    

 

### target对象                                    

 **target对象的属性：**

|属性|含义|
|-|-|
|target|指定一个上游服务器的地址（ip或域名）和端口，格式为：<host|
|weight|可选，指定该上游服务器的权重，取值范围是[0, 1000]，，默认值为100，当域名被解析为SRV记录，则其权重会被DNS中的SRV记录中的权重所覆盖，|



**target对象对应的api：**

|http方法|路由|作用|
| -    | -    | -    |
|POST|/upstreams/{upstream host:port or id}/targets|创建与指定upstream相关联的target对象|
|POST|/upstreams/{upstream name or id}/targets/{host:port or id}/healthy|设置与指定upstream相关联的指定地址或id的target对象为健康的|
|POST|/upstreams/{upstream name or id}/targets/{host:port or id}/unhealthy|设置与指定upstream相关联的指定地址或id的target对象为不健康的|
|GET|/upstreams/{upstream host:port or id}/targets|列出与指定upstream相关联的所有target对象|
|GET|/upstreams/{name or id}/targets/all/|列出与指定upstream相关联的所有target对象|
|DELETE|/upstreams/{upstream name or id}/targets/{host:port or id}|删除与指定upstream相关联的指定地址或id的target对象|



**例1：负载均衡测试**

kong位于主机172.18.150.167上，4个target由主机172.18.150.179上的openresty中的4个虚拟主机模拟：

![](https://test-1256398113.cos.ap-guangzhou.myqcloud.com/20190311192525.png)                                  

1、添加一个upstream和与其绑定的4个target，                                                                   

 ![](https://test-1256398113.cos.ap-guangzhou.myqcloud.com/20190311192559.png)

 2、创建一个service，指向上面创建的upstream，其中url属性的值可以是http://upstream1/或http://upstream1:80/，upstream的默认端口是80，

并创建指向该service的route                  

 ![1552303586129](C:\Users\preda\AppData\Roaming\Typora\typora-user-images\1552303586129.png) 

3、访问https://172.18.150.167:8443/test 那么该请求会被转发到upsteam中的随意一个target主机，且该请求的URI：/test会被追加到那个target主机上，比如172.18.150.167:8888/test

结果每次刷新，访问返回结果是test6、test7、test8、test9中的任意一个，且无规律，kong网关的负载均衡好像并不像nginx那样严格的按照wergit的比例进行访问，但是总的来说也是能够看出访问到9000和9001 的次数成weight的比例

![](https://test-1256398113.cos.ap-guangzhou.myqcloud.com/20190311192644.png)               

 

### PDK                                      

**PDK：**Plugin Development Kit，插件开发包，pdk其实就是一些lua函数或变量，供用户开发自定义逻辑的插件，这些函数或变量可以通过全局变量kong来访问，                                           

**kong.version**

```
一个字符串，表示当前kong节点的版本号，例如print(kong.version) -- "0.14.0"
```



**kong.version_num**              

```
一个整数，表示当前kong节点的版本号，例如：if kong.version_num < 13000 then -- 000.130.00 -> 0.13.0   -- no support for Routes & Services  end
```



**kong.pdk_major_version**    

```
一个数字，表示当前PDK的主版本号，例如：if kong.pdk_version_num < 2 then  -- PDK is below version 2  end
```



**kong.pdk_version**                

```
一个字符串，表示当前PDK的版本号，例如print(kong.pdk_version) -- "1.0.0"
```



**kong.worker_events**            

```
kong的IPC模块的实例，用于worker间的内部通讯，且基于lua-resty-worker-events模块，注意：目前该模块的使用被保留给kong核心或高级用户，
```



**kong.cluster_events**            

```
kong的cluster events模块的实例，用于内部节点间的通讯，注意：目前该模块的使用被保留给kong核心或高级用户， 
```



**kong.configuration**

```
kong.configuration是一个只读的表，包含当前kong节点的配置信息，kong.configuration表中的键都是kong.conf中的配置项，
例如：kong.configuration.plugins结果为{"bundled","myplugin"}
例如：kong.configuration.prefix) 结果为"/usr/local/kong"  由于该表只读，所以语句kong.configuration.prefix = "foo" 会抛出异常
```



**kong.ctx.shared**

```
作用域为`rewrite`, `access`, `header_filter`, `body_filter`, and `log` 
kong.ctx.shared是一个表，具有与当前请求一样的生命周期，（所以它是面向请求的，不同的请求对应不同的kong.ctx.shared值）更重要的是它可以用来在所有的插件之间共享数据，在插件中可以向这个表中插入数据，以供其他的插件使用，
```



**kong.ctx.plugin**

```
作用域为rewrite, access, header_filter, body_filter, log
kong.ctx.plugin是一个表，具有与当前请求一样的生命周期，但是和kong.ctx.shared不同的是，该表不可以在插件之间共享，只能在当前的插件中共享数据，
如果将一个限速插件应用到多个service，则不同的service对应不同的kong.ctx.plugin表，但同一个service的每个请求都共用一个kong.ctx.plugin表，
主要是因为它的命名空间，这个表比kong.ctx.shared要更安全，因为它避免了命名冲突，
```



**kong.db**				

```
kong的DAO（Data Access Object）层，使用kong.db.<entity>可以获取kong数据库表<entity>（比如services、routes等所有数据库表名）的DAO对象，其中<entity>为所有的kong数据表名，这些对象都
有对数据库记录进行增删改查的select、page、each、insert、update、upsert、delete等函数，DAO对象的这些方法都是在kong.db.dao.init.lua模块中定义的，
```



#### **kong.ip**

```
源码位于kong.pdk.ip.lua，主要用来判断指定的ip（ipv4或ipv6地址均支持）是否在/etc/kong/kong.conf配置文件的trusted_ips属性值之中，

kong.ip.is_trusted(ip)          
作用域为init_worker, certificate, rewrite, access, header_filter, body_filter, log
参数ip为一个表示ipv4或ipv6地址的字符串，如果指定ip在/etc/kong/kong.conf配置文件的trusted_ips属性值之中，则返回true，否则返回false，

例如：if kong.ip.is_trusted("1.1.1.1") then kong.log("The IP is trusted") end
当然也可以通过调用kong.configuration.trusted_ips来实现，它返回一个表，例如：
trusted_ips = 172.18.150.199, 172.18.150.198
则kong.configuration.trusted_ips的结果为{"172.18.150.199","172.18.150.198"}      
```

​         

#### **kong.node**

```
kong.node.get_id()              
返回当前kong节点的v4 UUID字符串，例如kong.node.get_id()结果为d76aa4de-85c7-464c-9651-36fe75bf84de
```

 

#### kong.log

```
kong.log(arg1,arg2,...)                 
作用域为init_worker, certificate, rewrite, access, header_filter, body_filter, log，等价于

kong.log.notice(arg1, arg2, ...)，打印notice级别的日志，
将指定的一个或多个参数的内容打印至日志文件，其打印格式为：[kong] %file_src:%line_src %message，而对于插件生成的日志，其格式为： [kong] %file_src:%line_src [%namespace] %message，其中[%namespace]为插件名，
例如2019/03/06 02:23:16 [notice] 16366#0: *111 [kong] init.lua:251 tttttttttttttttttttttttable: 0x40822a18
而ngx.log()日志的打印格式为：[lua] %file_src:%line_src %message，例如2019/03/06 02:23:16 [debug] 16366#0: *111 [lua] init.lua:311: Loading API endpoints for plugin: myplugin
kong.log()有table类型的参数时，会使用tostring函数将其转化为字符串，而ngx.log()有table类型的参数时，会报错，

kong.log.alert(arg1,...)          打印alert级别的日志                  

kong.log.crit(arg1,...)           打印crit级别的日志

kong.log.err(arg1,...)           打印err级别的日志

kong.log.warn(arg1,...)         打印warn级别的日志

kong.log.info(arg1,...)          打印info级别的日志

kong.log.debug(arg1,...)     打印debug级别的日志


kong.log.inspect(arg1,...)    
作用域为init_worker, certificate, rewrite, access, header_filter, body_filter, log
它也是用来打印日志的，但是它能更好的方式输出变量内容，当然也会消耗更多的性能，所以该函数应该只在测试环境使用，而不应该在生产环境使用，
1、它输出的变量之间会空格隔开，而ngx.log()打印各个参数时，它们之间没有空格隔开，而是紧密连接的，
2、它可以输出表的实际内容，例如：kong.log.inspect(singletons.configuration)的部分结果：
例如：kong.log.inspect(config.key_names)        -- {"apikey"}    
kong.log.inspect(config.hide_credentials) -- false

kong.log.inspect.off()          
关闭inspect logging，这会使得kong.log.inspect()函数什么都不打印，即什么都不做，
```

​                    

#### kong.client

```
源码位于kong.pdk.client.lua，用于获取与当前kong连接的当前连接请求对应的客户端的信息，它所有函数都不运行于client-api阶段，

get_ip()				
作用域为certificate, rewrite, access, header_filter, body_filter, log
返回一个string，表示当前请求对应的与kong直接相连的客户端的IP地址，即如果真实客户端使用了代理服务器，那么函数返回的是代理服务器的IP地址


get_forwarded_ip()		
作用域为certificate, rewrite, access, header_filter, body_filter, log
返回一个string，表示当前请求对应的最原始的客户端的IP地址，该函数的实现依赖于kong.conf配置文件中的3个配置指令：trusted_ips、real_ip_header、real_ip_recursive，这3个配置指令实际分别配置的是nginx-kong.conf配置文件中ngx_http_realip_module模块的3个指令：set_real_ip_from、real_ip_header、real_ip_recursive，例如kong.conf中配置trusted_ips = 172.18.150.199, 172.18.150.198 则对应nginx-kong.conf文件中会多出两条指令：set_real_ip_from 172.18.150.199;   set_real_ip_from 172.18.150.198;
如果配置得当，则当真实客户端使用了代理服务器，那么函数返回的仍是真实客户端的IP地址，


get_port()
作用域为certificate, rewrite, access, header_filter, body_filter, log
返回一个number，表示当前请求对应的与kong直接相连的客户端的端口号，即如果真实客户端使用了代理服务器，那么函数返回的是与kong直接相连的代理服务器的端口号，


get_forwarded_port()
作用域为certificate, rewrite, access, header_filter, body_filter, log
返回一个number，表示当前请求对应的最原始的客户端的端口号，该函数的实现依赖于kong.conf配置文件中的3个配置指令：trusted_ips、real_ip_header、real_ip_recursive，同上面的get_forwarded_ip()


get_credential()
作用域为access, header_filter, body_filter, log
返回当前认证的consumer的credentials，如果没有设置，则返回nil，例如：
local credential = kong.client.get_credential()
if credential then
  consumer_id = credential.consumer_id
else
  -- request not authenticated yet
end


get_consumer()
作用域为access, header_filter, body_filter, log
返回当前认证的consumer实体对应的表，如果没有设置，则返回nil，例如：
local consumer = kong.client.get_consumer()
if consumer then
  consumer_id = consumer.id
else
  -- request not authenticated yet, or a credential without a consumer (external auth)
end


authenticate(consumer, credential)
作用域为access
设置认证consumer（或credential），两个参数都可以是nil，但至少要有一个不是nil，两个参数的数据类型都是表，
```

​            

#### kong.request

```
源码位于kong.pdk.request.lua

get_http_version()	 
返回一个string，表示当前请求所使用的http版本号，如果不存在，则返回nil，例如：1.1

get_method()		
返回一个string，表示当前请求所使用的http方法，例如GET

get_scheme()			
返回一个string，表示当前请求所使用的协议名称，例如https

get_host()				
返回一个string，表示当前请求URL中的host值，例如172.18.174.75、www.example.com

get_port()				
返回一个number，表示当前请求URL中的port值，例如8444

get_path()				
返回一个string，表示当前请求URL中的path值，例如/test

get_path_with_query()	
返回一个string，表示当前请求URL中的path+query值，例如/test?a=1&b=2&a=2

get_raw_query()			
返回一个string，表示当前请求URL中的查询字符串，例如a=1&b=2

get_query(max_args)		
查看源码可知，max_args取值范围是[1, 100]，默认值是100，返回一个table，表示所有的查询参数及其值，
例如对于请求URL：https://172.18.174.75:8444/test?a=1&b=2&a=2，其对应的get_query()结果为表{"b":"2","a":["1","2"]}

get_query_arg(name)		
返回指定name的查询字符串，如果某参数有多个，那么只返回它的第一个值的string值，如果指定的参数没有等号，则返回true，如果有等号但没有值，则返回空字符串，如果参数不存在，则返回nil，
例如：对于请求URL：https://172.18.174.75:8444/test?a=1&b=2&a=2，其对应的get_query_arg('a')结果为字符串1
例如：对于请求GET /test?foo=hello%20world&bar=baz&zzz&blo=&bar=bla&bar
-- kong.request.get_query_arg("foo") -- "hello world"
-- kong.request.get_query_arg("bar") -- "baz"
-- kong.request.get_query_arg("zzz") -- true
-- kong.request.get_query_arg("blo") -- ""

get_forwarded_scheme()	

get_forwarded_host()		
返回一个string，表示当前请求头部中的X-Forwarded-Host头部字段的值，如果不存在，则返回host头部字段的值

get_forwarded_port()		
返回一个string，表示当前请求头部中的X-Forwarded-Port头部字段的值，如果不存在，则返回port头部字段的值

get_header(name)		
返回指定name的header的值，这里的name是原始header字段的小写形式，如果指定name的header不存在，则返回nil，
例如：local authorization_header = kong.request.get_header("authorization")

get_headers()			
返回一个由指定数量的header键值组成的表，例如：{"host":"172.18.174.75:8444","accept-language":"zh-CN,zh;q=0.9,ru;q=0.8","accept-encoding":"gzip, deflate, br"}
参数[max_headers]指定获取的header的最大数量

get_raw_body()			
返回一个string，表示body原始内容，如果body是空的，则返回空字符串，如果body大小大于nginx指令client_body_buffer_size指定的值，则函数会报错，

get_body()
函数有3个返回值，table | nil表示body内容的表， string | nil表示错误信息字符串，string | nil表示解析body所使用的mimetype
参数[mimetype] 是一个string，指定body内容的类型，可选的值有multipart/form-data、application/x-www-form-urlencoded、application/json，函数会按照mimetype指定的方式来解析body
参数[max_args] 是一个number，指定mimetype值为application/x-www-form-urlencoded时，对应body中的要被解析的参数的最大数量，默认值为100

```



#### kong.response

```
源码位于kong.pdk.response.lua，

get_status()		获取响应报文的状态码，返回一个lua number类型的值，

set_status()		修改响应报文的状态码为指定的值，
	status			一个number，表示要设置的状态码

get_header()		返回一个string，获取指定header的值，如果不存在，则返回nil
	name		   一个string，指定header的名字

set_header()		给响应报文设置一个header，它会覆盖任何已存在的header
	name, 		   指定header名
    value			指定header的值

set_headers()		给响应报文设置多个header，它们会覆盖任何已存在的header
	headers		   一个table，用来存放header，例如{ ["Bla"] = "boo", ["X-Foo"] = "foo3", 
	               ["Cache-Control"] = { "no-store", "no-cache" } }

add_header()		给响应报文设置一个header，它不会覆盖任何已存在的header，而是将header追加进去
	name, 		  指定header名
    value			指定header的值

get_headers()		返回一个table，包含所有的header
	[max_headers]	指定要返回的header的最大数量，默认100，详见源码

clear_header()		删除指定name的header
	name		  指定header的name

get_source()		返回一个string，返回值有以下3种：
				  service表示当前响应来自于上游服务器
				  error表示当前响应不是来自上游服务器，或在处理请求的时候发生了错误，
				  exit表示之前已经调用了kong.response.exit()

exit()				结束对响应报文的处理和生成，在代理来自上游服务器的响应报文给客户端之前，kong自己向客户端发送了一个响应报文，该函数执行后，header_filter、body_filter、log等阶段的代码还会执行，
	status, 	 一个number，指定响应报文的状态码
    body, 		 一个table或string，指定响应报文的响应体，如果body是一个string，则发送给客户端的是一个字符串，如果body是一个table，则发送给客户端的是一个table的json版本的字符串
    headers		一个table，指定响应报文的响应头部

```



#### kong.service

```
源码位于kong.pdk.service.lua，用来处理kong与上游服务器之间的一系列事务，

set_upstream(host)
作用域为access，
创建一个service，该service的host属性值为该函数的host参数所指定的某个已存在的upstream的name值，即该函数用于为当前客户端请求设置期望的处理负载均衡的上游实体，
函数有2个返回值，如果函数执行成功则返回true和nil，如果找不到指定name的upstream，则返回nil和错误信息字符串，
例如：
local ok, err = kong.service.set_upstream("service.prod")
if not ok then
  kong.log.err(err)
  return
end


service.set_target(host, port)
作用域为access
直接为当前请求设置一个唯一的host和port所指定的上游服务器，所以对当前请求，不运行负载均衡阶段，也不会运行与负载均衡有关的retries和health-checks，
参数host为string类型，用来指定上游服务器的域名或ipv4地址或ipv6地址，port为number类型，用来指定上游服务器的端口，
例如：kong.service.set_target("service.local", 443)
  -- kong.service.set_target("192.168.130.1", 80)

```



#### kong.router

```
源码位于kong.pdk.router.lua
该模块中的函数主要用于处理当前请求的路由

get_service()
作用域为access, header_filter, body_filter, log，
返回一个表，表示当前的service的属性信息，
例1：
if kong.router.get_service() then
  -- routed by route & service entities
else
  -- routed by a legacy API entity
end

例2：kong.router.get_service()结果为：
{   
  connect_timeout = 60000,                    
  created_at = 1551335287,                    
  host = "www.baidu.com",                     
  id = "4ad380f9-ebbf-4b00-bf2e-5e5a0a4072c7",
  name = "service1",                          
  path = "/",                                 
  port = 443,                                 
  protocol = "https",                         
  read_timeout = 60000,                       
  retries = 5,                                
  updated_at = 1551335287,                    
  write_timeout = 60000                       
}        


get_route()
作用域为access, header_filter, body_filter, log
返回一个表，表示当前的router的属性信息，
例1：
if kong.router.get_route() then
  -- routed by route & service entities
else
  -- routed by a legacy API entity
end

例2：kong.router.get_route()结果为：
{         
  created_at = 1551335337,                             
  id = "d48ec4ed-e98b-4193-b282-cee3aa2c4bdb",         
  name = "service1",                                   
  paths = { "/", <metatable> = <1>{__class = {__base = <table 1>,__init=<function 1>, __name="PostgresArray", <metatable>={__call=<function 2>, __index=<table1>}},__index=<table 1> } },                                                   
  preserve_host = false,                               
  protocols = { "http", "https", <metatable> = { __index = <function 3> } },                                                   
  regex_priority = 0,                                  
  service = { id = "4ad380f9-ebbf-4b00-bf2e-5e5a0a4072c7" },                                                   
  strip_path = true,                                   
  updated_at = 1552011510                              
}                
```



#### kong.service.request

```
源码位于kong.pdk.service.request.lua，
该模块中的函数主要用于设置kong发送至上游服务器的代理请求，

set_method(method)
作用域为rewrite、access
设置要发送至上游服务器的代理请求的http方法，method参数为一个string，函数没有返回值，
method支持的值有：`"GET"`, `"HEAD"`, `"PUT"`, `"POST"`,  `"DELETE"`, `"OPTIONS"`, `"MKCOL"`, `"COPY"`, `"MOVE"`, `"PROPFIND"`, `"PROPPATCH"`, `"LOCK"`, `"UNLOCK"`, `"PATCH"`, `"TRACE"`.
例如：kong.service.request.set_method("DELETE")


set_scheme(scheme)
作用域为access
设置要发送至上游服务器的代理请求的协议scheme，scheme参数为一个string，支持的值有http或https，函数没有返回值，
例如：kong.service.request.set_scheme("https")


set_path(path)
作用域为access
设置要发送至上游服务器的代理请求的path，path参数为一个string，函数没有返回值，
例如：kong.service.request.set_path("/v2/movies")


set_raw_query(query)		比set_query()函数要更底层
作用域为rewrite、access
设置要发送至上游服务器的代理请求的查询字符串，query参数为一个string，函数没有返回值，
例如：kong.service.request.set_raw_query("zzz&bar=baz&bar=bla&bar&blo=&foo=hello%20world")


set_query(args)
作用域为rewrite、access
设置要发送至上游服务器的代理请求的查询字符串，args参数是一个表，表中元素的键是一个字符串，元素的值的类型可以是boolean、string、元素为boolean或string的table数组，而且所有的string必须是URL-encoded的，在函数内部，表args会被转换为查询字符串，且键值顺序以字典序排序，相同名的header的顺序保持不变，函数没有返回值，
例如：kong.service.request.set_query({ foo = "hello world",  bar = {"baz", "bla", true}, zzz = true,  blo = "" }) 等价于查询字符串“bar=baz&bar=bla&bar&blo=&foo=hello%20world&zzz“


set_header(header, value)
作用域为rewrite、access
设置要发送至上游服务器的代理请求的头部字段，该函数会重写请求中已经存在的头部字段的值，header参数为一个string，value参数的值的类型可以是string、boolean、number，函数没有返回值，
例如：kong.service.request.set_header("X-Foo", "value")


add_header(header, value)
作用域为rewrite、access
设置要发送至上游服务器的代理请求的头部字段，该函数不会重写请求中已经存在的头部字段的值，而是被追加到请求头部中，实现多个相同的请求头部字段共存，header参数为一个string，value参数的值的类型可以是string、boolean、number，函数没有返回值，
例如：kong.service.request.add_header("Cache-Control", "no-cache") 
  -- kong.service.request.add_header("Cache-Control", "no-store")


set_headers(headers)
作用域为rewrite、access
设置要发送至上游服务器的代理请求的头部字段，该函数会重写请求中已经存在的头部字段的值，headers参数是一个表，其内部元素的键为string，值为string或元素为string表（数组），结果的header是以字典序来生成的，相同名的header的顺序保持不变，函数没有返回值，
例如：
kong.service.request.set_header("X-Foo", "foo1")
kong.service.request.add_header("X-Foo", "foo2")
kong.service.request.set_header("X-Bar", "bar1")
kong.service.request.set_headers({ ["X-Foo"] = "foo3", ["Cache-Control"] = { "no-store", "no-cache" }, ["Bla"] = "boo" })
结果为：
X-Bar: bar1
Bla: boo
Cache-Control: no-store
Cache-Control: no-cache
X-Foo: foo3


clear_header(header)
作用域为rewrite、access
删除要发送至上游服务器的代理请求中所有字段值为header的头部，函数没有返回值，
例如：kong.service.request.set_header("X-Foo", "foo")
  -- kong.service.request.add_header("X-Foo", "bar")
  -- kong.service.request.clear_header("X-Foo")   此时当前请求中没有字段为X-Foo的header


set_raw_body(body)			比set_body()函数要更底层
作用域为rewrite、access
设置要发送至上游服务器的代理请求中的body内容，body参数是一个string，可以为空字符串，表示空的body，该函数同时会设置Content-Length头部，函数无返回值，
例如：kong.service.request.set_raw_body("Hello, world!")


set_body(args, [mime])
作用域为rewrite、access
args是一个表，函数会以mime参数（可选）所指定的编码类型对表args进行编码，如果执行成功，则返回true和nil，否则返回nil和错误信息字符串，
1、如果指定了mime参数，则函数会根据mime的值来设置当前请求中的Content-Type头部字段的值
2、如果不指定mime参数，则函数会参考当前请求中的Content-Type头部字段的值来编码，
mime参数支持的值有`application/x-www-form-urlencoded`、`application/json`、`multipart/form-data`
`application/x-www-form-urlencoded`:	将args加密为form-encoded，任何字符串都要是URL-encoded的，
`multipart/form-data`: 	将args加密为multipart form data.
`application/json`:		将args加密为JSON，等价于kong.service.request.set_raw_body(json.encode(args))，

例如：
kong.service.set_header("Content-Type", "application/json")
local ok, err = kong.service.request.set_body({ name = "John Doe", age = 42, numbers = {1, 2, 3} })		
结果生成的body为 { "name": "John Doe", "age": 42, "numbers":[1, 2, 3] }
local ok, err = kong.service.request.set_body({ foo = "hello world", bar = {"baz", "bla", true}, zzz = true, blo = "" }, "application/x-www-form-urlencoded")
结果生成的body为bar=baz&bar=bla&bar&blo=&foo=hello%20world&zzz

```



#### kong.service.response

```
源码位于kong.pdk.service.response.lua，
该模块中的函数主要用于设置上游服务器发送至kong的代理响应，和kong.response的一系列函数不同，该模块的函数只返回来自上游服务器的响应报文中的header，不会返回kong在接收到来自上游服务器的响应报文中后，又追加的响应头部字段内容，所以该模块中只有读取数据的函数，而没有设置数据的函数，

get_status()
作用域为`header_filter`, `body_filter`, `log`
返回来自上游服务器的响应报文中的状态码，是number类型，如果请求未被代理，则返回nil

get_header(name)
作用域为`header_filter`, `body_filter`, `log`
返回来自上游服务器的响应报文中的指定name的头部字段值，这里的header字段名都是大小写不敏感的，而且横杠会被转换为下划线，例如来自上游服务器的响应报文字段X-Custom-Header会被解析为x_custom_header，如果指定的头部字段未找到，则返回nil，如果有多个相同name的头部字段，则只返回第一个的值，

get_headers([max_headers])
作用域为`header_filter`, `body_filter`, `log`
返回来自上游服务器的响应报文中的指定数量的头部字段值，返回的是一个表，表元素的键是头部字段值，表元素的值是头部字段对应的string值或一个string数组（当有多个相同名的头部字段时），
表中的头部字段名是大小写不敏感的，而且横杠会被转换为下划线，
max_headers默认值为100，可选的参数max_headers的取值范围是[1, 100]，
函数有两个返回值，如果请求还未被代理至上游服务器，则该函数的返回值为nil，如果实际的headers数量大于100，则返回一个被截断的表和一个字符串"truncated"

get_raw_body()
还未实现

get_body()
还未实现
```



**kong.table**

```
kong.table.new() 		    就是返回一个空表{}
kong.table.clear(tbl)		清空表tbl的内容
kong.table.merge(t1, t2)	合并表t1和t2，如果键有冲突，则保留t2的那个，

例如local t3 = kong.table.merge({1, 2, 3, foo = "f"}, {4, 5, bar = "b"}) 结果为{4, 5, 3, foo = "f", bar = "b"}
```



**kong.cache**

```
kong的database caching对象的实例，来自kong.cache模块，注意：目前该模块的使用被保留给kong核心或高级用户，

设计缘由：频繁的查询数据库会使请求的处理速度变慢，也会使数据库负载加重，一个可行的方法是从数据库查询一次数据，并将之缓存到内存中，避免多次查询，缓存有以下两种：
1、Lua memory cache：用于nginx worker进程内部的共享，它可以存储任何类型的lua值，
2、Shared memory cache (SHM)：用于同一个nginx节点中worker进程之间的共享，它可以存储任何标量值，因此	它要求序列化和反序列化，

当使用kong.db.<dao_name>对象从数据库中获取一个数据时，该数据会被同时存储到上述两种缓存中，
1、当同一个worker进程再次请求该数据时，它会从Lua memory cache中获取数据（先反序列化数据，再将之返回），
2、当同一个nginx节点中的另一个worker进程请求该数据时，它会从Shared memory cache中获取数据（先反序列化数据，再将之返回），

获取实体的cache_key是哪个字段，例如kong.db.basicauth_credentials.schema.cache_key结果为{"username"}，但有的实体比如services就没有cache_key

kong.cache有以下函数：以冒号：索引
get(key, opts, cb, ...)		
返回值为value, err，从缓存中获取指定键对应的值，如果值不存在，则在protected模式中以参数...调用函数cb，函数cb必须返回一个值，且只能是一个值，该值会被缓存为键key对应的值，该函数不会缓存负的结果，比如nil，它可能会抛出错误，因为那会被kong捕获并正确的以ngx.ERR级别进行记录日志，第二个参数opts是一个表，用来设置缓存的TTL和negative TTL，例如：{ttl = 600, neg_ttl = 40 }表示如果有数据, 则缓存的时间为600秒，如果没有数据，则是40秒，比如cb函数可以是一个使用kong.db.<dao_name>从数据库获取数据的函数， 也可以是下例中用于返回错误信息，
key = '4ad380f9-ebbf-4b00-bf2e-5e5a0a4072c7'
local service, err,err2 = kong.db.services:select({id=key})
kong.cache:get(key, {}, function(key) return string.format('there is no key: %s in cache!', key) end, key))	结果为there is no key: 4ad380f9-ebbf-4b00-bf2e-5e5a0a4072c7 in cache!


probe(key)				
返回值为ttl, err, value，检查指定键对应的值是否已经被缓存，如果是，则返回TTL，如果不是，则返回nil，第3个返回值是被缓存的值，

invalidate_local(key)		
从当前节点中的cache中删除指定键对应的值，

invalidate(key)			
从当前节点中的cache中删除指定键对应的值，并将该删除事件传播到当前kong节点所在集群中的所有其他节点，

purge()				
从当前节点中的cache中删除所有的值，

例如：
local credential_cache_key = kong.db.basicauth_credentials:cache_key(username)
local credential, err      = kong.cache:get(credential_cache_key, nil, load_credential_into_memory, username)

local consumer_cache_key = kong.db.consumers:cache_key(credential.consumer.id)
local consumer, err      = kong.cache:get(consumer_cache_key, nil, load_consumer_into_memory, credential.consumer.id)

```

