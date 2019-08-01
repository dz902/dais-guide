# 张氏 AWS 简明手册

## 说明

* 从 Open Guide 得到灵感。
* 仅为个人爱好收集整理，不代表任何组织和官方意见，不承诺正确性及可能造成的后果。
* 使用方法：⌘ + F / ⌃ + F。

## 图例

* 🇨🇳 = 宁夏和北京 Region（下简称「中国区」）特有情况
* 🚚 = 服务在 CFn / CDK 中的情况
* ✅ = 建议执行的操作
* 🈲 = 不建议执行的操作

## 目录

* 中国大陆特有
* IAM
* S3
* Lambda
* ES
* CFn
* CDK

## 🇨🇳 中国区特有情况

_「部分情况也适用于 GovCloud。」_

### 使用限制

* __仅限企业。__ 个人用户无法注册使用。
* __必须备案。__ 客户必须进行网站 ICP 备案，才能启用 Web 端口，否则会返回 `403`。

### 区域隔离

* __网络不互通。__ 中国区与其他 Region 没有亚马逊官方的光纤直联。
* __账号不互通。__ 中国大陆 Region 与其他 Region 的账号不处于同一体系。
* __资源不互通。__ ARN 中的第二段由「aws」改为「aws-cn」，与 Global 资源不在一个体系。

### Service Endpoint / Principal 差异

## 安全

### 责任共担模型

* __AWS 负责基建设施的物理安全以及账号资源之间的隔离。__ 用户负责使用和应用层面的安全。
* __AWS 无法访问用户的应用数据。__ 也无法为用户重置 SSH 登录密码、操作系统用户密码。

## 账单

* ✅ __打开 Billing Alerts，才能在 CloudWatch 中看到 Billing 指标。__ 会有极少量费用。
  * __无法关闭。__ 此设置打开之后将无法关闭。
  * 如果在 Consolidated Billing 之前未打开，则无法再操作，只有付款账号能统一打开。

## Availability Zone（AZ）/ Region

_「全球基建，高速互联，同城容灾。」_

### 概况

* __AZ 指近距离、分布式数据中心集群。__ 一个 AZ 包含 2+ 个数据中心，数据中心之间采用高速光纤直连，另有冗余的网络集线中心确保 AZ 内数据中心之间的高速、不间断网络互通。
  * 🇨🇳 中文也俗称「可用区」。
* __Region 指单个城市范围内的 AZ 集群。__ 大部分 Region 包含 3+ 个 AZ，少部分包含 2 个 AZ，极少部分仅有 1 个 AZ。
* __Region 内 AZ 之间距离数十公里。__ 并且水、电、网络、安保均为各自独立，确保能做到同城灾备。
* __相邻 Region 之间有亚马逊官方铺设光纤直连。__ 确保体系内有足够网速和吞吐。
  * 🇨🇳 中国区除外。


## IAM

_「云上的访问权限管理体系。」_

### 概况

* __Identity-Based Policy（IBP）是附在用户、角色身上的访问权限。__ 如果没有 IBP 用户无法访问资源。
* __Resource-Based Policy（RBP）是附在资源身上的访问权限。__ 只有部分资源有 RBP。
* __Policy 包含多个 Statement。__
  * Statement 必须包含：Effect（允许 / 拒绝）、Action、Resource。
  * Statement 可选包含：Condition。
  * RBP 的 Statement 还包含 Principal（访问者）。可以是别的账号下的 IAM User。
* __账户有 Master Account 和 IAM User 两种。__ 一个用邮件登录，一个还需要输入 Account ID。
  * 🇨🇳 中国区没有 Master Account，只有 IAM User。

### IAM User / Group

* __可以创建多个 IAM User 并置入 Group 之中。__ 
  * ✅ 应该为每个资源使用者创建 IAM User。
  * ✅ 应尽量避免为单个 IAM User 赋权，而是为 Group 赋权并将 IAM User 加入，方便管理。

### Role

* __Role 就是带权限的角色。__ 其他 IAM User、AWS 服务或者外部的已认证用户均可以担任 Role 并且获得相应权限。
  * 担任 Role 时访问者原来的的身份消失，这在 A 桶复制对象到 B 桶这样的场景下不方便，所以通常跨资源操作时会使用 RBP。
  * 跨账号使用 Role 时，会先将 Role 授权给目标账号，然后由目标账号再自己授权给单个 IAM User。
  * 权限名称为 `sts:AssumeRole`，因为担任 Role 实际上是 AWS Simple Token Service 出具了一个临时 Token 来给访问请求签名。

## S3（Simple Storage Service）

_「功能丰富的对象存储。」_

### 概况

* __第一个 AWS 服务。__
* __3 AZ 多副本。__ 11 个 9 的数据耐久度；3 个 9 的 SLA。
* __事件触发。__ 使用 __Events__ 机制可以在文件上传、文件修改等事件发生时触发 __Lambda__。
* __支持使用 Service Gateway 不走公网访问。__ Service Gateway 不额外收费，推荐使用。
* __扁平化存储。__ 没有文件夹的概念，但是对结尾为 `/` 的文件做了特殊处理，可以当做文件夹使用。

### 权限

* __有 Access Control List（ACL）、RBP 两种权限控制体系。__ ✅ 通常使用 RBP。
* __ACL 以账号为单位来赋予权限。__ 需要使用 S3 专用的 Canonical ID 来指代账号，在 Bucket 的 ACL 页面可以找到。
  * __ACL 需要在 Bucket 和每个文件上设置。__ 非常不方便。
  * __ACL 只有读、写、读取权限、写入权限、完全控制几种权限设置。__ 采用 XML 形式存储。
  * __ACL 默认赋予 Bucket 拥有者全部权限。__ 所以拥有者不再需要额外设置 RBP 即可访问新建的 Bucket。
  * __ACL 也支持「已登录用户」、「所有人」以及「日志组」几个分组。__ 尽量不要使用。
* __非 Bucket 拥有者需要在桶上设置 RBP 并持有对应的 IBP 才能访问。__ 除非在 ACL 里面专门允许。
* __有「Block Public Access」的设置。__ 针对 Bucket 的设置。✅ 建议打开。
  * 防止新上传对象公开访问，开启后会抛弃新上传对象的公开访问权限。
  * 防止所有对象公开访问，开启后会忽略对象所附着的公开访问权限。

### 细节

* __Global / Local Endpoint。__

### S3 Select

* __直接对 S3 上存储的半结构化数据进行视图过滤。__

### 踩坑

* 🇨🇳 __不支持 Global Endpoint。__ 在中国区必须使用 Regional Endpoint，否则会返回 `404`。
  * 🚚 __RegionalDomainName。__ 在 CFn 中创建 Bucket 之后如果读取 `DomainName` 属性会返回 Global Endpoint，需要改用 `RegionalDomainName`。
* 🇨🇳 __Cross-Region Replication 不支持往非中国区复制。__ 跨区复制仅能在中国区 Region 之间进行。
* __ACL 中的 Everyone 指的是全世界。__ 即全天下的人都可以通过 HTTP 访问。

## Athena

_「直接针对 S3 上存的半结构化数据跑 SQL。」_

## EC2（Elastic Cloud Compute）

_「云上虚拟机。」_

### 总览

* ✅ __默认使用 SSH 并以 Key-Pairs 验证。__ 🈲 若要密码登录虚拟机，需要进行额外的操作。

### 购买方式

* __On-Demand（OD），按需。__
  * __收费精确到秒。__ 「Stop」之后就不收时间费用。
* __Reserved Instance（RI），预留。__
  * __Zonal RI。__
  * __Regional RI。__
* __Spot Instance，闲置。__

### 租户类型

* __Standard，共享型。__
* __Dedicated，独占型。__

### 实例类型

* __名称格式：（主类型）（代）.（特殊机能）。__
* __主类型。__
  * __M = Most Workloads，通用型。__
  * __T = Turbo，可超频。__
  * __C = Compute，高算力。__
  * __R = RAM，大内存。__
  * __G / P = GPU，大显卡。__
  * __I__
  * F
  * X
* __特殊机能。__
  * __n = Networking Enhanced，网络加强。__
  * __AMD。__
  * __metal = Bare Metal。__

### 踩坑

## ELB（Elastic Load Balancing）

_「托管的 4/7 层负载均衡器。」_

### ALB（Application Load Balancer）

* __OSI 7 层负载均衡。__ 类似 Nginx。
  * __支持按 Host 转发。__ 在管理控制台中需要先创建再设置。
* __实际上是弹性集群。__ 通过 DNS 来指向不同机器。
* ✅ __可以记录访问日志到 S3。__ 方便调试。

### NLB（Network Load Balancer）

* __OSI 4 层负载均衡。__ 基于 HyperPlane，10 微秒级请求转发速度。
* __静态 IP。__ 支持内部静态 IP，每个 Subnet 一个。

## AMI（Amazon Machine Image）

_「虚拟机镜像。」_

### Amazon Linux

* __基于 RHEL 的开源 Linux。__ 使用 `yum` 包管理工具。
* __经过审核、强化并预装 AWS 工具包。__
* 🇨🇳 __中国区暂时没有 Amazon Linux 2。__

## VPC（Virtual Private Cloud）

_「云上的虚拟专属网络。」_

### 总览

* __逻辑隔离。__ VPC 之间不互通，除非额外使用 VPC Peering 等专门的对接方式。
* __IP 地址保留。__ CIDR 的 1 号 IP 地址是默认网关，2 号是 DHCP，3 号预留未来使用，255 号因为不支持 Broadcasting 而做了保留。
  * 实际上会只有整个 VPC 的 1 号会变成网关，但其他 Subnet 的 1 号也都会保留。
* __不支持 Broadcasting。__ 暂时也不支持 Multicasting。
* __不支持监听模式。__ 不支持 Promiscus 模式，避免了包嗅探的问题。
* __Network ACL 是针对整个 VPC 的无状态端口防火墙。__ 必须设置双向允许，网络包才能双向流动。
* __Security Group 是针对资源的有状态端口防火墙。__ 也可以用来代替 IP 地址、地址段，作为防火墙条目中的「源地址」。
* __Elastic Network Interface（ENI）。__ 虚拟网卡。
* __Elastic IP Address（EIP）。__ 虚拟公网 IP。

### Subnet

* __单 AZ 虚拟子网。__ Subnet 不能跨 AZ。
* __相互联通。__ 同一个 VPC 下的 Subnet 之间可以互通，但需要设置路由。
* __必须从 VPC 的 CIDR 中选择子集。__ 并且同一 VPC 下 Subnet 之间不能重叠。
* __每个 Subnet 至少带一个路由表。__ 路由条目中的规则越具体，优先级越高。

### Gateway

* __Internet Gateway（IGW）。__ 双向互联网网关，每个 VPC 一个。
  * __Subnet 路由表中有记录指向 IGW 则为 Public Subnet。__ 否则就是 Private。
* __NAT Gateway（NAT-GW）。__ 出口向互联网网关（IPv4），需置于 Public Subnet 中。
  * 设置路由后，Private Subnet 的实例可以通过 NAT-GW 单向访问互联网，用于下载更新包等。
* __Ingress / Egress Gateway。__ 入口 / 出口向互联网网关（IPv6）。

### 踩坑

* __非 Amazon Linux 使用双 ENI 需要自己配置操作系统路由。__ 否则可能出现无法访问的情况。

## Lambda

_「无服务器计算资源。」_

### 概况

* __采用 Firecracker Micro-VM 内核。__
* __以「内存/100ms」为单位来收费。__

### 踩坑

* 🚚 __Log Group 不会自动删除。__ Lambda 所附带的 Log Group 是自动创建的，不在记录之列，所以不会在 CFn Stack 删除时自动删除。（见 [Link](https://blog.rowanudell.com/cleaning-up-lambda-logs-with-cloudformation/)）
  * 手动创建名为 `/aws/lambda/${lambdaFunctionName}` 的 Log Group 即可自动删除。

## ES（Elasticsearch Service）

_「托管的 Elasticsearch + Kibana 应用。」_

### 概况

* __两种网络模式。__ 置于公网，或者置于 VPC 内。
* __访问策略。__ 需要在访问策略中允许之后，才可访问。

### 踩坑

* __置于 VPC 中的 Domain 无法从公网访问。__ 因为 VPC 内的 ES 仅有私有 IP 地址，没有公网 IP 地址。（见 [Link](https://forums.aws.amazon.com/thread.jspa?threadID=279437)）
* __访问策略的 Resource 需加入路径。__ 支持确定的路径或 `*`，如没有路径会报错。正确示范：`arn:aws-cn:es:cn-northwest-1:1234567890:domain/my-es/*`，注意最后的 `/*`。
* __循环权限依赖。__ 要访问 ES，需要创建 Role，需要知道 ES 的 ARN；而创建 ES 的时候，需要设置访问策略，需要知道 Role 的 ARN。解决方案见 CloudFormation 踩坑部分。
* 🚚 __CFn 仅支持 1-AZ 和 2-AZ。__ `ZoneAwarenessEnabled: True` 代表 2-AZ，但管理控制台中支持 3-AZ。（见 [ElasticsearchClusterConfig.ZoneAwarenessEnabled](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-elasticsearch-domain-elasticsearchclusterconfig.html)）

## CFn（CloudFormation）

_「基建即代码工具。」_

### 踩坑

* __循环权限依赖。__ 通常发生在具备 Resource-Base Policy 的资源上，资源需要知道角色的 ARN 才能开放权限，而角色则需要知道资源的 ARN 才能设置权限，导致循环依赖。 解决办法如下。
  * ✅ 把 ARN 或资源名称写成常量。
  * 写一个 Custom Resource，用 Lambda 来增加、调整有依赖的部分。
* __非空 Bucket 无法删除。__ S3 Bucket 如果已经放了东西，则无法在回滚、删除 Stack 时自动删除，需手动操作，如果在频繁测试，可以自行写 Shell 脚本进行删除。
* __单向同步。__ CFn 模板修改后，资源会同步，但是资源创建完成后再进行修改，不会体现在模板上。
  * __有 Drift Detection 可以监测模板与资源之间的差异。__ 🇨🇳 中国区暂时没有。

## CDK（Cloud Development Kit）

_「封装成程序代码的 CFn。」_

### 总览

* __基于 Node 和 TypeScript。__ 命令行工具基于 Node。代码另支持 JavaScript、Python、Java、CSharp 等语言。
* __转译成 CFn 模板。__ CDK 代码仍然会转译成 CFn 模板进行部署。
* __助手函数。__ CDK 提供助手函数来简化模板编写。

### 概念

* __Construct。__ 即封装成类的资源。

### 命令

* `cdk bootstrap` = 创建用于存放 CFn 模板和材料的 S3 Bucket
* `cdk synth` = 将 CDK 代码转译成 CFn 模板
* `cdk deploy` = 部署转译好的 CFn 模板

### 踩坑

* 🇨🇳 __`cdk bootstrap` 中使用的 CFn 返回 Bucket 的 Global URL，导致不可创建 ChangeSet。__ CFn 中的返回值为 `DomainName` 而不是 `RegionalDomainName`，而中国区不支持 Global Endpoint，导致错误。解决办法如下。（版本：v1.2.0，见 [#1459](https://github.com/aws/aws-cdk/issues/1459)）
  * 修改 `node_modules/aws-cdk/lib/api/bootstrap-environment.js` 中的 `"StagingBucket", "DomainName"` 为 `"StagingBucket", "RegionalDomainName"`。
* __S3、LogGroup 资源默认 `UpdatePolicy` / `DeletionPolicy` 为 `Retain`。__ 此项与 CFn 的默认行为相反。（版本：v1.2.0，见 [#2601](https://github.com/aws/aws-cdk/issues/2601)）
* __缺乏 `DeletionPolicy` 抽象。__ 部分资源需要直接调用底层的接口才能修改，而且底层接口名字混乱，方法名字是 `RemovalPolicy`，值又是 `DESTROY`。
  * `(res.node.defaultChild as s3.CfnXXXX).applyRemovalPolicy(cdk.RemovalPolicy.DESTROY)` 

## IoT Greengrass

_「边缘计算控制器。」_

## AppSync

_「托管的 GraphQL。」_

## RDS（Relational Database Service）

## Aurora（RDS Aurora）

_「托管的云原生数据库。」_

* __有发表专门的论文。__

## Redshift

_「托管的数据仓库。」_

### Spectrum

* __直接从 S3 中扫描半结构化数据进行分析。__ 跟 Athena 的功能雷同。

### 踩坑

* __没有「Stop」这一说法。__ 虽然常听说「停机就不收费，要用时再打开」，但是 Redshift 没有「Stop」，只有「Delete」。

## CloudWatch

_「云上监控平台。」_

### 总览

* __Metrics 是应用指标。__
* __Events 是消息队列。__ 有相对独立的 ARN 和服务名字（events）。
  * __后续发展成了 EventBridge。__ 🇨🇳 中国区暂时没有。
* __Logs 是日志收集和查看器。__ 有相对独立的 ARN 和服务名字（logs）。
  * __有 Logs Insights 可以直接对日志进行查询。__ 🇨🇳 中国区暂时没有。

## CloudTrail

_「官方的 API 访问日志。」_

### 总览

* __默认打开。__

## Kinesis

_「托管的消息队列。」_

### Firehose

* __往各种地方输送数据。__ 支持 S3、Lambda、Elasticsearch 等等。
* __支持数据转换。__ 输送数据之前用 Lambda 进行转换。

### Video Streams

## ECS（Elastic Container Service）

_「托管的 Docker 集群治理工具。」_

## EKS（Elastic Kubernetes Service）

_「托管的 K8s Master Node。」_

### 踩坑

* 🇨🇳 __没有官方镜像。__ 中国区无法访问 Google 节点，所以无法从官方渠道下载 K8s 应用。
* __责任共担模型变化。__

## EMR（Elastic MapReduce）

_「托管的 Hadoop 生态。」_

## SageMaker

_「托管的 Jupyter Notebook。」_

## Shield

_「云的 4 层防火墙。」_

### 总览

* __通过 Shield Basic 免费提供 4 层 DDoS 防护。__
  * 🇨🇳 中国区暂时没有 Shield 服务，但有 4 层 DDoS 防护。

## WAF（Web Application Firewall）

_「云的 7 层防火墙。」_

## DynamoDB

_「托管的云原生 NoSQL 数据库。」_

* __从 Amazon.com 购物车需求发展而来。__
* __有发表专门的论文。__