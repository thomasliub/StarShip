# 企业应用平台体系架构

[TOC]

# 1. 企业应用平台概述
企业应用平台由企业应用云平台(简称云平台)和车间设备平台(简称设备平台)组成。应用部署在云平台之上，和车间设备组成整个企业应用解决方案。如下图所示：

![](/pics/paas-devops-1.png)

如图所示，自下而上分别为设备平台，PaaS云平台，IoT物联网平台和应用层组成。本文着重讲述车间设备通用平台和PaaS平台部分，对于IoT平台，有专门的文档详细阐述。对于具体企业应用只提及，并不做详细描述。

对于车间设备，它可以通过MQTT向物联网平台或者应用发送异步消息，同时也可以通过HTTP REST接口访问企业云上的应用接口。对于应用可以通过订阅MQTT消息或者websocket直接获取设备通报的数据。

# 2. 车间设备平台
车间设备平台负责连接现场设备，为现场应用提供一个统一的运维环境。在硬件方面，目前，X86系列有服务器工控机，Intel Nuc服务器, 第三方OEM服务器；ARM系列有树莓派等等。在这些硬件上可以运行Linux或者Windows操作系统。

车间设备平台为车间现场级应用提供服务有：
* 数据持久化服务
* 健康监控服务
* 日志服务
* 通信接口模块
    * 支持MQTT with TLS
    * 支持HTTP with TLS
* 设备OAM
    * 软件更新
    * 设备配置本地/远程
    * 设备远程登陆
    * 心跳监控

## 2.1 数据持久化服务
车间设备平台为各种代理或者现场级应用提供数据持久化服务，包括，文件的存储，关系数据库SQLite。一般而言，这些数据需要定期同步到云平台，以提供复杂数据服务和解析。

## 2.3 日志服务
在现场级应用设备，日志可以选择输出到网络ELK日志服务器或者本地保存。

## 2.4 通信接口模块
设备平台提供面向云平台的通信接口。包括MQTT消息接口和HTTP接口，同时支持TLS安全通道。

## 2.5 设备OAM
### 2.5.1 健康监控和恢复服务
应用健康监控和恢复服务提供应用一个可靠的运行环境。应用可以采取主动措施定期持久化运维数据，同时，当应用崩溃时（丢失心跳），设备平台会采用如下策略试图恢复应用（逐级累加）：
1. 应用告警
2. 尝试重新启动应用
3. 尝试重新启动系统

### 2.5.2 软件更新
支持设备登陆云平台，云平台根据配同步软件版本。
系统需要支持两种更新方式：
1. 设备登陆时，做软件和配置版本同步
2. 系统更改后，触发做软件版本同步

### 2.5.3 设备配置本地/远程
支持设备本地和远程配置参数，如网络地址，设备名称等等。

### 2.5.4 设备远程登陆
提供从云平台远程登陆到设备，执行运维操作。此过程中，需要验证用户权限，验证设备权限。

### 2.5.5 心跳监控
设备在登陆云平台后会按照配置定期报告心跳，云平台在收到设备心跳后会回复心跳确认，当心跳丢失时，云平台会采取相应恢复措施，如产生告警，事件，触发设备操作等等。

# 3. 企业应用云平台体系结构
PaaS是平台即服务，由一组云计算服务组成的一个平台，提供用户研发，运行和管理应用，而不需要触及复杂的下层基础设施IaaS（计算，存储，网络）。PaaS作为公有云/私有云/私有运行环境，包含基础设施，提供操作系统，中间件，数据库，等等所有与应用服务开发，测试和部署相关的服务。

企业云平台根据应用开发，测试，部署的需求，实现了PaaS层的必要组件，并且，会随着应用需求的变更逐渐扩展和完善。
如图1中所示，PaaS(PaaS)包括：
* HTTP接入网关
* API网关
* 负载均衡器
* MQTT代理
* 服务注册与发现
* 用户认证授权
* 设备认证授权
* 数据库服务
* Redis Cache服务
* 消息队列服务
* ETCD数据中心
* ELK日志服务
* 应用配置管理服务
* 公共服务，SMS,email, app push, 语音呼叫

未来将根据评估结果继续引入，数据分析服务，服务编排和分布式系统一致性保证等等。

## 3.1 HTTP接入网关
接入网关负责接入用户请求，承担着路由转发和过滤器的主要职能。
采用nginx作为接入网关，对用户请求进行认证，通过认证的用户请求，可以访问静态资源，对于API将请求转发给API网关。同时，接入网关会对api GET请求做cache以降低后端API资源的访问频率。

HTTP接入网关主要功能：
* 静态HTTP web服务器
    * 提供静态HTTP资源访问功能，如html模板, js脚本，css样式文件，图片，媒体文件等等
* 虚拟主机
    * 单一nginx服务器，可以支持多虚拟主机服务不同的用户请求，如内部网络用户，外部APP用户等等
* 缓存服务器
    * 配置nginx缓存空间，降低访问后端服务的频率，减少系统响应时间
    * proxy_cache
* 连接池
    * 服务器可以同时接入连接：max clients = worker_processes * worker_connections / 2
* 测试/调试支持
    * 分配/跟踪/分析会话
    * 压力测试，配置压力测试模式，支持在线/离线测试
* 网络安全策略
    * 支持TLS加密通道，支持https连接
* 数据收集
    * 访问统计，收集用户访问系统metrics
    * 流量统计，收集用户访问系统流量metrics
    * 费用统计，收集用户访问系统费用数据
* 过滤器
    * 用户认证，鉴权，实施OpenIDConnect认证流程
    * 流量控制
        * 源IP限制流量

## 3.2 API网关
API网关承接系统所有API请求，参照负载均衡器给出的策略将请求转发给后端应用。同时，API网关提供后端接口发布和升级，支持蓝绿发布和比例升级。

API网关主要功能：
* 反向代理服务器
    * 路由转发，基于用户策略的路由转发，如，根据客户端类型，访问协议，url规则，等等。
* 服务平滑升级支持
    * 能够支持系统不同版本的API共存
    * 蓝绿发布
* 数据收集
    * 访问统计，收集用户访问系统metrics
    * 流量统计，收集用户访问系统流量metrics
    * 费用统计，收集用户访问系统费用数据
* 过滤器
    * 访问限流，自动减载， load shedding
    * 断路器
    * 流量控制

## 3.3 负载均衡器
负载均衡器根据定制的策略采用负载均衡算法给出下一个服务最合适的处理者。负载均衡器会和服务注册与发现交互，并根据服务健康状态调整权重因子。

负载均衡器主要功能：
* 负载均衡
    * 获取有效服务实例，计算权重
    * 获取服务实例健康状态，调整权重
    * RR策略，采用round roubin派发策略
    * 权重策略，根据调整后的权重决定转发的合适路由

## 3.4 OpenResty nginx扩展平台
OpenResty® 是一个基于 Nginx 与 Lua 的高性能 Web 平台，其内部集成了大量精良的 Lua 库、第三方模块以及大多数的依赖项。用于方便地搭建能够处理超高并发、扩展性极高的动态 Web 应用、Web 服务和动态网关。

OpenResty® 通过汇聚各种设计精良的 Nginx 模块（主要由 OpenResty 团队开发），从而将 Nginx 有效地变成一个强大的通用 Web 应用平台。这样，Web 开发人员和系统工程师可以使用 Lua 脚本语言调动 Nginx 支持的各种 C 以及 Lua 模块，快速构造出足以胜任 10K 乃至 1000K 以上单机并发连接的高性能 Web 应用系统。

OpenResty® 的目标是让你的Web服务直接跑在 Nginx 服务内部，充分利用 Nginx 的非阻塞 I/O 模型，不仅仅对 HTTP 客户端请求,甚至于对远程后端诸如 MySQL、PostgreSQL、Memcached 以及 Redis 等都进行一致的高性能响应。

关于OpenResty, 详细参考：
OpenResty官网: http://openresty.org/cn/


## 3.5 用户认证与授权
用户的认证与授权采用OpenIDConnect协议，该协议流程需要实现OpenIDConnect认证服务器和客户端。认证服务器基于开源项目实现，协议规范和操作过程支持。客户端直接在接入网关上实现，接入网关代理所有的应用服务，完成认证流程。并将认证结果附加在HTTP消息头，转发给后端服务器。后端服务器无条件信任接入网关的认证结果，完成领域内用户授权各项操作。

### 3.5.1 接入网关用户认证
* 用户首次登陆访问

![](/pics/paas-devops-2.png)


    1. 用户首次访问服务
    2. 服务网关（OpenIDConnect RP client）识别用户首次访问，并redirect到用户认证服务器（OpenIDConnect OP Server)
    3. 用户认证服务器识别用户首次访问，要求用户登陆
    4. 用户完成登陆
    5. 用户认证服务器分配授权码，并redirect请求回源服务
    6. 服务网关识别用户请求，持授权码向用户认证服务器索取访问令牌，用户ID令牌，如成功则追加用户信息到请求头，并传递请求到后续服务
    7. 后续服务（前端服务或后端服务）完成请求
    8. 服务网关和用户建立有效会话（cookie）


* 用户已经登陆认证服务器，首次访问其他服务
本流程中，由于用户已经登陆过认证服务器，因次认证服务器和用户的认证过程省略掉

![](/pics/paas-devops-3.png)

    1. 用户访问服务
    2. 服务网关识别用户首次访问，并redirect到用户认证服务器
    3. 用户认证服务器识别用户已经登陆，则直接分配授权码，并redirect请求回源服务
    4. 服务网关识别用户请求，持授权码向用户认证服务器索取访问令牌，用户ID令牌，如成功则追加用户信息到请求头，并传递请求到后续服务
    5. 后续服务（前端服务或后端服务）完成请求
    6. 服务网关和用户建立有效会话（cookie）


* 用户在该服务拥有有效会话后的后续访问

![](/pics/paas-devops-4.png)

    1. 用户访问服务，携带有效会话ID(cookie)
    2. 服务网关识别会话，向认证服务器发起访问令牌验证请求
    3. 认真服务器识别访问令牌，回复用户信息
    4. 如有效访问令牌，则追加用户信息到请求头，并传递请求到后续服务；如无效访问令牌，则直接拒绝请求


* 用户登出

![](/pics/paas-devops-5.png)

    1. 用户向任意服务发起登出请求，携带有效会话ID(cookie)
    2. 服务网关识别并完成用户会话清除，随后redirect登出请求到认证服务器
    3. 认证服务器完成用户会话清楚，随后redirect回源服务缺省网页
        1. 方案一，认证服务器清除该用户所有已颁发的访问令牌，即立即使所有相关服务失效
        2. 方案二，认证服务器保留该用户所有访问令牌，待其自行失效。访问令牌有效期为1小时

### 3.5.2 OpenIDConnect认证服务器
单点登录为多个应用统一提供了用户登陆和验证机制，避免了在多应用环境下用户频繁登陆/登出的困境。单点登陆是整个企业云平台的安全核心，必须采用成熟的足够强度而又灵活的体系结构。

经过调研主要选项有CAS和基于OAuth2的OpenIDConnect技术。通过如下对比，最终选择OpenIDConnect技术。
* CAS不支持OAuth2. Fackbook, Google, Yahoo采用OAuth2做认证, 而且目前85%认证系统采用OAuth2
* 大部分采用OAuth2的系统将来会采用OpenIDConnect
* CAS缺失很多功能，不支持feature
* 动态client注册，发现
* 用户claim, client claim
* Even SAML support is weak
* CAS实现多步骤认证困难，大部分是基于用户名/口令方式

单点登陆服务器基于开源项目[django-oidc-provider](https://github.com/juanifioren/django-oidc-provider)，首先，实现了OpenIDConnect技术的标准框架以及流程，支持服务访问点的发现和授权码认证方式；同时扩展了用户应用授权和服务导引功能。在单点登陆服务器上，分配用户应用权限，当用户登陆后，可以根据其权限列出可以访问的企业应用。需要注意的是应用内部的子授权还由应用各自维护和控制。


![](/pics/paas-devops-6.png)


![](/pics/paas-devops-7.png)

### 3.5.3 添加用户和用户授权
* 用户授权批量导入
    1. IT人员从OA系统中导出用户基本信息，并制作用户应用授权
    2. IT人员将用户信息和应用授权批量导入到认证服务器，完成应用授权初始加载
    3. IT人员将用户信息和应用子授权信息导入到各个应用服务器，完成应用的子授权初始加载  


* 用户授权单独处理流程
在第一个release仅支持如下方式创建用户和用户授权，在第二个release（10月初）中，实现用户和用户授权的集中配置和管理，同时保留一阶段的分散配置以提供系统灵活性。
    1. HR人员在OA系统首先录入用户基本信息，OA系统提供访问用户信息REST接口
    2. IT人员在用户管理系统即认证服务器添加用户和用户应用权限，在此过程中，需要查询OA REST接口确认用户身份；认证服务器提供用户应用授权查询REST接口
    3. 应用管理员在各个应用添加用户和应用子授权，此过程中需要查询和确认认证服务器用户应用授权REST接口
    4. 最终用户登入系统，应用检查用户各项操作授权，进行访问/操作控制

参考：
单点登陆SSO项目：
OpenIDConnect官网：[http://openid.net/connect/](http://openid.net/connect/)

## 3.6 设备认证与授权
设备认证与授权提供设备的认证，id标识和权限检查服务。目前，实现了基于用户名和密码的简单验证机制。即在云平台配置允许接入的设备序列号，设备名称，密码等信息，当设备登陆MQTT代理时，由代理调用云平台设备识别接口判定设备是否被允许，否则，直接拒绝设备连接请求。

未来可以支持基于X.509数字证书的设备验证和识别，云平台为用户签发用户根证书，用户则利用根证书进一步签发所有设备证书。设备持设备证书访问云平台服务。
在权限鉴别方面，未来可以考虑实现细粒度的权限划分和控制。即划分，设备登陆权限，设备获取权限，设备发布权限，设备接收控制权限，设备远程控制权限等等。

## 3.7 服务注册与发现
服务注册与发现是服务编排和横向扩展的基础支撑技术，在采用微服务架构中，尤为重要。服务注册与动态发现解决了应用服务可以随时在任何云节点上创建或删除，系统功能具有弹性扩张能力。有了服务的动态注册/发现，才可以使服务HA得以实施，为服务动态载荷调整提供前提。同时，实施服务的动态注册与发现是实现serverless应用的前提条件。

方案采用etcd+confd。etcd 是一个分布式一致性k-v存储系统作为服务注册数据中心，可用于服务注册发现与共享配置。confd用来读取etcd的服务信息并动态生成nginx的配置文件，并执行重新加载nginx进程。

服务动态注册/发现流程如下图：

![](/pics/paas-devops-8.png)


1. confd作为nginx代理，注册数据中心服务名称，如serivce_erp, service_oa
2. 应用服务创建服务实例，注册服务实例；服务实例停止，数据中心注销
3. 数据中心根据订阅信息，将服务变动通知nginx代理confd
4. confd根据nginx模板重新生成nginx.conf
5. confd通知nginx重新加载

### 3.7.1 etcd
采用etcd多节点组成集群，并监控集群状态。应用服务向etcd注册服务实例IP地址。etcd集群同步所有etcd节点数据。

### 3.7.2 confd
confd连接etcd，并监听服务节点变化，当有变化时，etcd主动通知confd，confd会读取相应服务地址重新生成nginx配置文件，最后重新加载nginx进程。
自此，服务的变化，实时地反映到nginx网关，当有服务增加/减少时，网关会动态调整后端服务器，从而实现应用服务的自动注册和发现。

## 3.8 日志服务（待补充）
* Logstash: 收集存储各类日志
* Elasticsearch: 分布式的，基于JSON的数据搜索分析引擎
* Kibana: web化接口，多种形式索引在Elasticsearch和Logstash中的数据
PaaS搭建了ELK日志系统，采用异步消息进行日志收集，并采用Elasticsearch对日志进行高效检索和分析，最后通过Kibana将数据呈现。

## 3.9 系统公共服务
系统公共服务指在服务运维平台上运行的各种异步的公共服务，应用只需要通过REST或者消息接口将服务请求提交到系统公共服务，不需要关心请求处理结果，由系统公共服务异步地执行。  

目前，支持的系统公共服务有：
* 云平台短消息发送服务
* 云平台邮件发送服务
* 云平台APP推送发送服务
* 云平台语音电话拨打服务

应用服务通过REST接口或者异步消息通知系统公共服务，后者调用内部功能接口实现所需功能，当发送失败时，采取重试策略，保证尽最大努力完成任务。对于最后仍然失败的任务，进行告警。

## 3.10 集中配置与部署服务
在企业云中，将PaaS和应用服务的配置集中管理，目的是实现配置和代码隔离。虽然采用了容器技术，使软件的开发和运行的技术环境相一致，但仍然面临这用户IT环境的不统一。因此，需要动态的配置软件环境。在企业云环境中，由各个应用服务分别配置会带来整个系统的混乱和易错。因此，有必要实施统一的配置中心来完成这一任务。
集中配置服务会和DevOps的部署结合起来，实现，集中配置和一键启动。
系统配置分两大部分，系统服务和应用服务。

## 3.11 数据库服务
数据库泛指在企业云中数据持久化层，包括各种数据库存储技术，包括各种结构化数据存储和非结构化数据存储。
结构数据存储：
* postgres
* SQLite

非结构化数据存储：
* Redis, 提供键值存储
* MongoDB, 提供类Json文档存储，可以对某些字段建立索引，实现关系数据库的某些功能
* InfluxDB, 时序数据库服务
    * 时序数据是基于时间的一系列的数据。
    * 在有时间的坐标中将这些数据点连成线，往过去看可以做成多纬度报表，揭示其趋势性、规律性、异常性;
    * 往未来看可以做大数据分析，机器学习，实现预测和预警。
    * 时序数据库就是存放时序数据的数据库，并且需要支持时序数据的快速写入、持久化、多纬度的聚合查询等基本功能。
    * 单机版（开源），集群版（收费）

PaaS提供数据持久化服务，应用根据自身需要选择不同的数据持久化技术，其自身不需要构建数据库服务器容器，统一访问PaaS提供的数据库容器。                                                 
                                                                                                                                                                                                                                                                                                                                                                                                                                 
### 3.11.1 数据库集群维护(待展开)
不同的数据库技术有不同的集群方案。
Postgres集群方案：
* Pgpool II，一个位于 PostgSQL  服务器和 client 之间的中间件。
    * 连接池  (无需多说)。
    * 复制 ： pgpool-II 可以管理多个 PostgreSQL服务器，利用复制功能可以实时的备份多个福利磁盘，保障了在磁盘失效情况下的连续服务。（最多128 数据节点)）
    * 负载均衡：复制生效时，执行 select 命令时，无论那个服务其将返回相同的结果。pgpool-II 利用复制的优势来减少服务器的负载--在多个服务器之间选分发select指令，从而增加了系统吞吐量。 当大量的用户执行select 指令时候这一优势将得到充分的体现。
	* 超过载链接：PostgreSQL 有一个最大的同时链接数，超过时就拒绝链接。设置大的链接数，增加了资源的消耗，影响系统的性能，. pgpool-II 有最大连接数限制，对超过的连接请求做队列处理，或者立即返回错误。 
	* 平行的Query: 使用平行 Query 功能，数据可分配到多个服务器，所以 查询会同时在多个服务器执行，减少了总运行时间。平行查询在大规模数据查询时候体现出最好的表现

* Postgres-XL，基于无共享结构的,多主，写扩展的 PostgreSQL集群
    * 写扩展的 PostgreSQL 集群，和纯PostgreSQL相比，用五个服务器可获得超过三倍的性能增强（1.0版本），提高扩展性的方法是众所周知的。
    * 同步的多主配置，对主的任何更新对其他主都是可见的。
    * 表位置是透明的，可以继续使用同一的应用，事物处理无需改变。
    * 基于PostgreSQL。使用和PostgreSQL相同的API。 V 1.2.1 已经可用了。

### 3.11.2 数据备份与恢复
要提供增量和总量的数据库备份与恢复策略。

### 3.11.3 数据库客户端访问
要求提供数据库Schema和数据的便捷访问，查询，操作，备份恢复等操作。

### 3.11.4 数据库调优
根据数据库使用情况不断优化（？？？）

## 3.12 MQTT代理服务
MQTT（Message Queuing Telemetry Transport，消息队列遥测传输）是IBM开发的一个即时通讯协议，已成为物联网的重要组成部分。PaaS采用Mosquitto开源MQTT代理实现了消息代理功能，采用QoS保证的基于订阅/发布的消息通信机制连接物联网设备层中的各种设备。是一种低开销的近实时的物联网协议。同时，该协议可以采用TLS安全连接，用以提高安全性。

## 3.13 消息队列服务
消息队列服务为云平台上所有服务提供统一的消息代理，目前采用RabbitMQ，支持消息发送模式有：
* 消息队列发送
* exchange定向广播，fanout
* exchange定向路由，direct模式
* exchange定向消息主题订阅和发布模式，topic模式

同时，为了实现消息传递的隔离机制，可以规划虚拟主机vhost。

## 3.14 数据统计与分析服务(待评估)
待后续随着系统演进做进一步评估。

## 3.15 分布式系统事务一致性服务（待评估）
待后续随着系统演进做进一步评估。

## 3.16 服务编排（待评估）
待后续随着系统演进做进一步评估。









