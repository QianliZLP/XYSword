# XYSword
XYSword是一款强大的资产收集管理平台，主要用于src信息收集和红蓝对抗资产收集用，采用python celery分布式实现多节点部署，提高效率.

整个系统架构为 bootstrap + flask + celery(rabbitmq&redis) + mysql
实现以下功能

- [x] 域名扫描
- [x] 端口扫描
- [x] web站点扫描
- [x] 目录扫描
- [x] 漏洞扫描(nuclei, 自定义poc, xray+rad) 
- [x] host碰撞插件
- [x] apk域名提取插件
- [x] 备案信息查询
- [ ] fofa查询
- [ ] 多账号管理
### 系统搭建

```
git clone https://github.com/QianliZLP/XYSword.git
cd H && docker-compose build
```

#### 配置修改

建议修改下redis和rabbitmq的密码，在XYSword/docker-compose.yml中

```
version: '3'
services:
  server:
    restart: always
    build: .
    ports:
      - "5005:5005"
      - "5004:5004"
    tty: true
    restart: always
    networks:
      - H_network
  
  rabbitmq:
    image: rabbitmq:3.8.3-management
    container_name: rabbitmq
    restart: always
    ports:
      - 5672:5672
      - 15672:15672
    environment:
      - RABBITMQ_DEFAULT_VHOST=H_broker
      - RABBITMQ_DEFAULT_USER=[rabbitmq账号]
      - RABBITMQ_DEFAULT_PASS=[rabbitmq密码]
      - RABBITMQ_VM_MEMORY_HIGH_WATERMARK=2147483648
    networks:
      - H_network

  redis:
    image: redis
    container_name: redis
    command: redis-server --requirepass [redis密码]
    restart: always
    ports:
      - 6379:6379
    networks:
      - H_network

  worker:
    build: client/.
    tty: true
    environment:
      DEBUG: 'False'
      BROKER: 'amqp://[rabbitmq账号]:[rabbitmq密码]@rabbitmq:5672/H_broker'
      BACKEND: 'redis://:[redis密码]@redis:6379/2'
      TZ: Asia/Shanghai
    restart: always
    networks:
      - H_network

networks:
  H_network:
    driver: bridge
```

以及XYSword/config.ini

```
[CELERY_CONFIG]
CELERY_BROKER_URL = amqp://[rabbitmq账号]:[rabbitmq密码]@rabbitmq:5672/H_broker
CELERY_RESULT_BACKEND = redis://:[redis密码]@redis:6379/2

[DATABASE]
DB_HOST = 127.0.0.1
DB_USER = root
DB_PASSWD = root
DB_DATABASE = H
DB_PORT = 3306
```

对xray进行配置

在使用是注意xray的webhook地址，端口默认是5005也就是扫描器的web地址，而`h_server_1` 这个地址是docker镜像地址

如果是下载的main包，docker镜像名字可能就是`hmain_server_1`因此这里配置时要正确，xray才会工作

反链地址同理的，反链地址为server的5004端口
```
#配置位置 client/config.ini
vuln_xray_webhook = h_server_1:5005

#配置反链平台位置：
client/vuln_scan/xray_scan/tools/xray/config.yaml
123行:    http_base_url: "http://h_server_1:5004"  

```


#### 启动服务

搭建好后就是运行，服务端启动

```
#启动整个docker容器
docker-compose up -d

docker exec -it h_server_1 bash
cd /app && nohup python3 run.py &
cd /app/app/scan/lib/tools && chmod 755 xray
nohup ./xray reverse &
```


客户端启动

```
docker exec -it h_worker_1 bash

#如果你注释了开头的那段配置文件，你需要进入容器运行那一段代码
cd /tmp && wget https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb && apt install -y ./google-chrome-stable_current_amd64.deb

cd /app
python3 client_run.py

#需要启动哪些模块，可以在下面文件中配置
vim /app/config.ini
```

之后访问vps的5005端口，即可看到登录页面，默认账号为admin/123456

#### 多台部署

如果你要部署多台，可以在第二台主机上这样运行，修改H/docker-compose.yml，只保留worker的配置，并且rabbitmq和redis的ip设置为主服务器(搭建了rabbitmq和redis的那台服务器的ip)

```
services:
  worker:
    build: client/.
    tty: true
    environment:
      DEBUG: 'False'
      BROKER: 'amqp://[rabbitmq账号]:[rabbitmq密码]@[rabbitmq服务器的ip]:5672/H_broker'
      BACKEND: 'redis://:[redis密码]@[redis服务器的ip]:6379/2'
      TZ: Asia/Shanghai
    restart: always
    networks:
      - H_network

networks:
  H_network:
    driver: bridge
```



### 使用简介

#### 首页

当有客户端节点接入的时候，会显示celery节点信息，记事本预留内容为每个模块的搜索语法，你也可以自己记录自己想要的内容
#### 系统设置模块

该模块几个参数

同时扫描任务上限，这里的2意思是可以2个项目一起并行扫，但是也需要节点数量够，大部分节点都是多线程，多进程，部分如域名扫描时单进程（后续优化）

自定义poc地址，这里填你poc管理的github或者gitee地址，国内vps建议用gitee，更新拉取的时候不会出错，poc格式参照nuclei的格式即可，在扫描模式勾选了自定义poc会从你在此次设定的github地址拉取你的poc，你只需要维护好自己的github的poc，不会存储在本地

邮箱账号和邮箱密码(不是你邮箱的登录密码，发件密码需要申请一个独立的密码)还有邮箱服务器地址，可以在如126.com和qq.com邮箱获得

接收邮箱可以填多个，每个一行即可，都会接收到

之后就是配置推送信息时间和内容选择

#### 扫描模式

扫描模式预设了2个模式，full和normal，里面有组件勾选

注意：如果选择了对应的组件，但该组件没有节点连接，则会项目卡在该阶段一直等待
#### 定时模块

定时模块主要作用是，比如一个src项目需要每个月扫一次，则会自动重新扫描，发现新资产并记录

### 项目添加

添加内容可分为主域名， 指定域名和ip， 黑名单分为domain，ip，title

黑名单设置需要设置title如：

```
domain: xxx.alibaba.com

title: 店铺
```

添加了后进入项目详细点击开始扫描， 暂停扫描此处是中断扫描，导出结果会保存成Excel
### 信息展示

每个页面均有一个搜索框，搜索语法可以看初始化时候，首页记事本有记载，简单的来说就可以通过&&符号拼接多个条件，搜索框查多条信息

#### 子域名

#### 端口
#### 站点

设置了标记按钮，如果浏览了该站点，可以点击未标记，该条信息会自动跑到列表最后，可以通过该方式来过滤已经看过的站点

### 路径扫描
### 漏洞扫描

nuclei分等级，有一个漏洞等级的选项卡，该选项卡如果是nuclei扫描的话，会返回poc的等级，如果是xray扫描的话，不会划分等级统一归为xray

### 备案信息查询

调用了https://github.com/canc3s/cSubsidiary， 该工具会根据天眼查查询第一个公司并获取子公司，经过我的改良，实现了通过关键字获取子公司和备案域名信息

特别注意，该查询功能被做成了celery，在client中有`icpget`模块，之所以做成celery模块是因为vps对天眼查发起请求时会被判定为海外ip，无法访问到数据，这边建议可以部署在本地，或者一台不关机的设备上即可

使用该模块需要一个auth_token 该值为登录天眼查后cookie中的一个字段，经过测试，天眼查获取数据只需要该字段值即可，并且该cookie长时间有限，实测半个月还有效

### host碰撞

host碰撞调用该工具

https://github.com/cckuailong/hostscan

输入主域名到内网域名生成，会自动生成一些可能是内网的域名，当然你也可以自定义输入收集到的域名，ip地址需要自行准备，这边可以通过站点信息模块中nginx返回403的站点ip来作为一个目标

碰撞了后有记录，点击后会显示碰装钱的域名和ip，如果有结果在右侧会出来，反之没有成功则不会显示

### apk域名提取

原理：使用grep命令提取
