# 张氏 AWS 简明手册

## 说明

* 从 Open Guide 得到灵感。
* 仅为个人爱好收集整理，不代表任何组织和官方意见，不承诺正确性及可能造成的后果。
* 使用方法：⌘ + F / ⌃ + F。

## 图例

* 🇨🇳 = 宁夏和北京 Region（中国大陆 Region，下简称「中国区」）相关
* 🚚 = 在 CFn / CDK 中的情况
* 🚥 = 在 CLI 工具中的情况
* ✅ = 建议执行的操作
* 🈲 = 不建议执行的操作
* 💢 = 需要注意的坑
* 💰 = 会产生额外费用

## 目录


## 🇨🇳 中国区特有情况

_「部分情况也适用于 GovCloud。」_

### 使用限制

* __仅限企业。__ 个人用户无法注册使用。
* 💢 __必须备案。__ 客户必须进行网站 ICP 备案，才能启用 Web 端口，否则会返回 `403`。

### 区域隔离

* __网络不互通。__ 中国区与其他 Region 没有亚马逊官方的光纤直联。
* __账号不互通。__ 中国大陆 Region 与其他 Region 的账号不处于同一体系。
* 💢 __资源不互通。__ ARN 中的第二段由「aws」改为「aws-cn」，与 Global 资源不在一个体系。

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
  * 🇨🇳 中文俗称「可用区」。
* __Region 指单个城市范围内的 AZ 集群。__ 大部分 Region 包含 3+ 个 AZ，少部分包含 2 个 AZ，极少部分仅有 1 个 AZ。
  * 🇨🇳 中文俗称「区域」，但容易混淆，所以本手册仍然使用「Region」。
  * 🇨🇳 中国宁夏 Region 以省命名而不是城市。
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

### Root Account

* __Root Account 是 AWS 账号体系下最高管理者账号。__ 即注册时的第一个账号，只需使用邮箱、密码登录，具备完整权限。
  * 🇨🇳 中国区没有 Root Account 概念，所有账号均为 IAM User，登录时需要同时输入 Account ID 以及邮箱、密码。
  * 💢 注意 IAM User 拥有删除自己的权限，删除后要恢复非常麻烦。
  * 🈲 __强烈建议不使用 Root Account 进行日常操作。__

### IAM User / Group

* __可以创建多个 IAM User 并置入 Group 之中。__ 
  * ✅ 应该为每个资源使用者创建 IAM User。
  * ✅ 应尽量避免为单个 IAM User 赋权，而是为 Group 赋权并将 IAM User 加入，方便管理。
* __IAM User 登录时需要提供 Account ID 以及邮箱、密码。__
  * 也可以使用专属登录页面链接。

### Role

* __Role 就是带权限的角色。__ 其他 IAM User、AWS 服务或者外部的已认证用户均可以担任 Role 并且获得相应权限。
  * 💢 __担任 Role 时访问者原来的身份消失。__ 这在 A 桶复制对象到 B 桶这样的场景下不方便，所以通常跨资源操作时会使用 RBP。
  * 跨账号使用 Role 时，会先将 Role 授权给目标账号，然后由目标账号再自己授权给单个 IAM User。
  * 权限名称为 `sts:AssumeRole`，因为担任 Role 实际上是 AWS Simple Token Service 出具了一个临时 Token 来给访问请求签名。

### MFA（Multi-Factor Authentication）

* ✅ __多因子认证要求用户在登录使用额外的一次性密码。__ 推荐使用，至少应该在 Root Account 或管理员账号上启用。

## S3（Simple Storage Service）

_「功能丰富的对象存储。」_

### 概况

* __第一个 AWS 服务。__ 2006 年 3 月 14 日上线。
* __3 AZ 多副本。__ 11 个 9 的数据耐久度；3 个 9 的 SLA。
* __事件触发。__ 使用 __Events__ 机制可以在文件上传、文件修改等事件发生时触发 __Lambda__。
* __支持使用 Service Gateway 不走公网访问。__ Service Gateway 不额外收费，推荐使用。
* __扁平化存储。__ 没有文件夹的概念，但是对结尾为 `/` 的文件做了特殊处理，可以当做文件夹使用。

### Bucket

* __S3 是 Global 服务，但 Bucket 有区域之分。__ 区域选择会影响访问速度。
  * 🇨🇳 中国区除外。中国区的 S3 无法看到 Global 的 Bucket。
* __不同区域的 Bucket 可以通过 Cross-Region Replication（CRR）自动同步。__  
  * 🇨🇳 __CRR 不支持同步出、入中国区。__ 中国区的 CRR 仅能在中国区 Region 之间进行，Global Region 的 CRR 无法同步至中国区。

### 权限

* __有 Access Control List（ACL）、RBP 两种权限控制体系。__ ✅ 通常使用 RBP。
* 🈲 __ACL 以账号为单位来赋予权限。__ 需要使用 S3 专用的 Canonical ID 来指代账号，在 Bucket 的 ACL 页面可以找到。
  * __ACL 默认赋予 Bucket 拥有者全部权限。__ 所以拥有者不再需要额外设置 RBP 即可访问新建的 Bucket。
  * __ACL 需要在 Bucket 和每个文件上设置。__ 非常不方便。
  * __ACL 只有读、写、读取权限、写入权限、完全控制几种权限设置。__ 采用 XML 形式存储。
  * __ACL 也支持「已登录用户」、「所有人」以及「日志组」几个分组。__ 不建议使用。
  * __ACL 中的「所有人」指的是全世界。__ 即全天下的人都可以访问。
* __非 Bucket 拥有者需要在桶上设置 RBP 并持有对应的 IBP 才能访问。__ 除非在 ACL 里面专门允许。
* __有「Block Public Access」的设置。__ 针对 Bucket 的设置。✅ 建议打开。
  * 防止新上传对象公开访问，开启后会抛弃新上传对象的公开访问权限。
  * 防止所有对象公开访问，开启后会忽略对象所附着的公开访问权限。

### Endpoint

* __Global / Local Endpoint。__
  * 🇨🇳 __中国区不支持 Global Endpoint。__ 在中国区必须使用 Regional Endpoint，否则会返回 `404`。

### S3 Events

* __在上传、修改等事件时触发 Lambda。__
  * 💢 __不确保及时、准确性。__ 即可能会有 1 分钟以上的延时，可能会有重复的事件，可能会漏发事件。（见 [Link](http://www.hydrogen18.com/blog/aws-s3-event-notifications-probably-once.html?ck_subscriber_id=527212891)）
  * 🈲 __不推荐用来处理有强一致性需求的任务。__ ✅ 可以用于做一些简单的统计，特别是可耐受少量信息冗余、缺失的。

### S3 Select

* __直接对 S3 上存储的半结构化数据进行视图过滤。__ 基于 PartiQL 语言。

### 数据湖

* __数据湖核心。__ S3 是 AWS 数据湖的核心存储机制，也是数据湖的「湖」本体，大部分分析类服务的持久化存储由 S3 提供。

### 踩坑

  * 🚚 __RegionalDomainName。__ 在 CFn 中创建 Bucket 之后如果读取 `DomainName` 属性会返回 Global Endpoint，需要改用 `RegionalDomainName`。
  * 🚥 __无法从 CLI 获得 ARN。__ S3 的 CLI 没有提供任何获得 ARN 的方法，用户只能从 Console 复制，或者自己组成 ARN，需考虑中国区或 GovCloud 的名称差异问题。


## Athena

_「直接针对 S3 上存的半结构化数据跑 SQL。」_

## EC2（Elastic Cloud Compute）

_「云上虚拟机。」_

### 总览

* ✅ __默认使用 SSH 并以 Key Pairs 验证。__ 🈲 若要密码登录虚拟机，需要进行额外的操作。

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

### Nitro

* __Nitro 是 AWS 自研虚拟机硬件载体卡。__
  * __芯片解决方案由 A 公司提供，已被亚马逊收购。__
* __Nitro 承担了网络编码传输、读写、加解密等工作。__ 虚拟机可以使用到更完整的算力。

### Windows

* __管理员账号跟 Windows 默认语言有关。__ 比如，可能是 `Adminstrator`，也可能是 `Adminstrateur`。
* __需要从 Console 获取默认管理员密码。__ 针对 Windows 实例，选择「Actions > Get Windows Password」，需要提供 SSH Private Key。
  * ✅ 建议首次登录后使用 Ctrl + Alt + Delete 修改默认密码。
  * 💢 在 OS X 上请按 Fn + Ctrl + Option + Delete。

### SSH

* __SSH 卡死时请使用 `~` 大法。__ 先按 `Enter`，然后按 `~`，然后按 `.`，即可强制终止 SSH。

## EBS（Elastic Block Storage）

_「块存储服务。」_

### 安全

* ✅ __支持 KMS 全盘加密。__ 不影响 I/O 性能。

### 踩坑

* __`growpart` 在 CentOS 上可能执行成功但是仍然报错。__ 用户可尝试直接重启机器再看效果。

## ELB（Elastic Load Balancing）

_「托管的 4/7 层负载均衡器。」_

### ALB（Application Load Balancer）

* __OSI 7 层负载均衡。__ 类似 Nginx。
  * __支持按 Host 转发。__ 需要先创建 ALB 再设置转发规则。
  * __支持身份验证。__ 在转发规则中设置。🇨🇳 中国区不支持。
* __实际上是弹性集群。__ 一个 ALB 对应多个节点，自动伸缩，通过 DNS 记录来指向不同节点。
* ✅ __可以记录访问日志到 S3。__ 💰 会产生费用。

### NLB（Network Load Balancer）

* __OSI 4 层负载均衡。__ 基于 HyperPlane，10 微秒级请求转发速度。
* __静态 IP。__ 支持内部静态 IP，每个 Subnet 一个。
* 💢 __实例无法通过 NLB 访问自己。__ 网络包可以成功发出，但是因为 NLB 会保留源地址和目标地址，导致两个地址一致，被网卡判断是无效包，从而访问失败。
  * 由于 ALB 会替换源地址，所以不存在这个问题。
* __每选择 1 个 AZ，即会在这个 AZ 中置入一个 NLB 并附带一个 IP 地址。__ 面向 Internet 的 NLB 可以选择 EIP 或随机分配地址，内部的 NLB 只能随机分配。
  * __NLB 默认只会向自己 AZ 中的目标。__ ✅ 每选择一个 AZ，都应该确保 AZ 中有健康的目标。
  * 💢 __如果某个 NLB 所处的 AZ 中没有目标，而用户正好访问到了这个 NLB，就会导致 NLB 无法访问。__ NLB 依靠 DNS 记录做负载均衡，而客户端可能会缓存这个记录，导致反复访问同一个没有目标的 NLB，导致失败。
  * __如果想让 NLB 能转发流量到任意 AZ 中的目标，可以开启 Cross-Zone Load Balancing。__

### Classic Load Balancer

* __前 VPC 时代的负载均衡器。__ 同时支持 TCP 和 HTTP 层的负载均衡。

### 踩坑

* __必须置于公开 Subnet。__ ALB 必须置于公开 Subnet，外部才能访问。

## AMI（Amazon Machine Image）

_「虚拟机镜像。」_

### Amazon Linux

* __基于 RHEL 的开源 Linux。__ 使用 `yum` 包管理工具。
* __经过审核、强化并预装 AWS 工具包。__
* __RHEL 对应的 [EPEL](https://access.redhat.com/solutions/3358) 工具包是 Amazon Linux Extras。__ 见 [Link](https://amazonaws-china.com/amazon-linux-2/faqs/#Amazon_Linux_Extras)。
* ✅ __可以打开 TCP BBR 阻塞算法增加网络吞吐。__ 见 [Link](https://aws.amazon.com/amazon-linux-ami/2017.09-release-notes/)。

## VPC（Virtual Private Cloud）

_「云上的虚拟专属网络。」_

### 总览

* __逻辑隔离。__ VPC 之间不互通，除非额外使用 VPC Peering 等专门的对接方式。
* __IP 地址保留。__ VPC CIDR 的 1 号 IP 地址是 VPC 路由器（默认网关），2 号是 DNS，3 号预留未来使用，255 号因为不支持 Broadcasting 而做了保留。
  * 每个 Subnet CIDR 的第 1 个和最后 1 个 IP 地址也会保留，不准使用。
  * 实际上会只有整个 VPC CIDR 的 2 号会预留做 DNS，但其他 Subnet CIDR 的 2 号也都会保留。
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

### EIP（Elastic IP）

* __同时固定公网和私网 IP。__ 公网 IP 从 AWS 自有 IP 池子中随机选取。

### ENI（Elastic Network Interface）

* __虚拟网卡。__
* 💢 __非 Amazon Linux 使用双 ENI 需要自己配置操作系统路由。__ 否则可能出现无法访问的情况。

### Flow Logs

* __记录 VPC、Subnet 内所有 ENI 上网络 Packet 的流动。__ 也可只针对单个 ENI 做记录。
* __有 ACCEPT / REJECT 两种状态。__ ACCEPT 是正常放行的流量，REJECT 是被 Security Group / ACL 阻挡的流量。
* __可保存至 CloudWatch Logs 或 S3 Bucket。__ 并自动创建所需权限。 
  * 🇨🇳 __S3 Bucket 如有原来带有 `aws-cn` 的权限会出错。__ `aws-cn` 中间的 `-` 会被丢掉，导致「MalformedPolicy - Invalid principal in policy」错误。
  * 可以到 CloudTrail 中找到该条 API 调用，将 Bucket Policy 部分复制出来，加上 `-`，手动覆盖原权限。
* __忘记打开 Flow Logs 但是产生大流量账单时可通过 CloudWatch 简单排查流量大头。__ 使用 NetworkOut 等 Metric。

### Site-to-Site VPN

* __AWS 提供托管 VPN 方案打通多个隔离网络。__
* __VGW（Virtual Gateway） 是 AWS 侧的 VPN 网关。__ Customer Gateway 是客户侧的网关。
  * 💢 __只能由 Customer Gateway 一侧发起请求建立 VPN 管道。__ AWS 侧无法发起请求，但请求发起后网络包可以双向发送。（见 [Link 1](https://docs.aws.amazon.com/vpc/latest/adminguide/Introduction.html)、[Link 2](https://forums.aws.amazon.com/thread.jspa?threadID=106147)）
  * 🇨🇳 中国区暂时不支持 Customer Gateway，可使用 Openswan。
* __用户需自行准备 Customer Gateway 硬件或软件。__ 在 AWS 上「创建 Customer Gateway」指的是「生成 Customer Gateway 对应的配置」。
  * __AWS 提供已经测试过可用的软硬件路由列表。__
* __Tunnel 1 是主管道，Tunnel 2 是备用管道。__ 见 [Link](https://community.spiceworks.com/how_to/143768-how-to-set-up-a-site-to-site-vpn-for-aws)。

### Amazon DNS Server

* __VPC CIDR 的第 2 个 IP 地址是 AWS 官方提供的 DNS 服务器。__ 也可以使用 `169.254.169.253`，见 [Link](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-dns.html)。
* __官方 DNS 服务器可以解析 `.internal` 私有域名，也可以解析 Route 53 Private Hosted Zone 中的地址。__


## Lambda

_「无服务器计算资源。」_

### 概况

* __采用 Firecracker Micro-VM 内核。__
* __以「内存/100ms」为单位来收费。__
* __默认不放在 VPC 内。__

### Function

* __支持多种语言环境。__
* __可使用 Layers 来附加程序所需的库。__
* __经过程序触发。__ 不能直接触发，需经过程序触发，比如 S3 Events、API Gateway 等。
* __可以通过 Environment 变量传入参数。__ Function 将可以访问这些参数。
  * 🇨🇳 中国区暂时不支持。

### Layer

* __可以复用的依赖。__ 不用每个函数去打包依赖。
* __上传 zip 包来创建 Layer。__
* 💢 __打包有固定格式。__ 不按格式则无法正常引入依赖。（见 [Link](https://docs.aws.amazon.com/lambda/latest/dg/configuration-layers.html)）
  * 比如 Node.js 的依赖目录必须是 `nodejs/node_modules`。

### 踩坑

* 🚚 __Log Group 无法在 Stack 删除或回退时自动删除。__ Lambda 所附带的 Log Group 是自动创建的，不在 Stack 资源之列，所以不会在 Stack 删除时自动删除。（见 [Link](https://blog.rowanudell.com/cleaning-up-lambda-logs-with-cloudformation/)）
  * 可手动创建名为 `/aws/lambda/${lambdaFunctionName}` 的 Log Group，即可自动删除。
* 🚚 __注意 Lambda 函数 Role 需要有足够权限。__ 特别注意对 Log Group 的操作，以及对各项资源的操作。
  * 💢 如果没有 Log Group 创建权限则不会自动创建 Log Group，如果没有写入 Log Group Stream 权限则不会自动写入，但二者均不会报错。
  * ✅ 可使用部分 Managed Policy，比如 AWSLambdaBasicExecutionRole。
  * 🚚 注意 Managed Policy 名字有的前面有前缀（比如 `service-role/`、`job-role/`），有的没有前缀。在 Console 中不会显示前缀，但在 CDK 中使用 `fromAwsManagedPolicyName()` 时需要把前缀加上。

## ES（Elasticsearch Service）

_「托管的 Elasticsearch + Kibana 应用。」_

__快速通道__ >> [使用手册](https://docs.aws.amazon.com/elasticsearch-service/latest/developerguide/what-is-amazon-elasticsearch-service.html)

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
* __非空 Bucket 无法删除。__ S3 Bucket 如果已经放了东西，则无法在回滚、删除 Stack 时自动删除，需手动操作。
  * 可以自行写 Shell 脚本进行删除。
* __单向同步。__ CFn 模板修改后，资源会同步，但是资源创建完成后再进行修改，不会体现在模板上。
  * __有 Drift Detection 可以监测模板与资源之间的差异。__ 🇨🇳 中国区暂时没有。

## CDK（Cloud Development Kit）

_「封装成程序代码的 CFn。」_

__快速通道__ >> [API 文档](https://docs.aws.amazon.com/cdk/api/latest/versions.html)

### 总览

* __基于 Node 和 TypeScript。__ 命令行工具基于 Node。代码另支持 JavaScript、Python、Java、CSharp 等语言。
* __转译成 CFn 模板。__ CDK 代码仍然会转译成 CFn 模板进行部署。
* __助手函数。__ CDK 提供助手函数来简化模板编写。

### 概念

* __Construct。__ 即封装好的资源和配置。
  * 💢 __很多服务还没有 Construct。__ 可退回到使用 `Cfn-` 前缀的原始类。
* __使用 JavaScript 开发。__ 使用 [jsii](https://github.com/aws/jsii) 框架来接入其他语言。

### 命令

* `cdk bootstrap` = 创建用于存放 CFn 模板和材料的 S3 Bucket
* `cdk synth` = 将 CDK 代码转译成 CFn 模板
* `cdk deploy` = 部署转译好的 CFn 模板
  * 💢 部署之前记得先运行 `cdk synth`。

### 踩坑

* 🇨🇳 __`cdk bootstrap` 中使用的 CFn 返回 Bucket 的 Global URL，导致不可创建 ChangeSet。__ CFn 中的返回值为 `DomainName` 而不是 `RegionalDomainName`，而中国区不支持 Global Endpoint，导致错误。解决办法如下。（版本：v1.2.0，见 [#1459](https://github.com/aws/aws-cdk/issues/1459)）
  * 修改 `node_modules/aws-cdk/lib/api/bootstrap-environment.js` 中的 `"StagingBucket", "DomainName"` 为 `"StagingBucket", "RegionalDomainName"`。
* 🇨🇳 __中国区 Service Principal / ARN 格式错误。__ Global 区域研发者对中国区命名规则误判，写错规则，导致 CDK 基本无法使用。（见 [Link](https://github.com/aws/aws-cdk/issues/2198)）
  * 可调用内部接口自行覆盖内置规则临时解决。（见 [Link](https://gist.github.com/bnusunny/090e65be682b4703b72b41e4f648c51c)）
* __S3、LogGroup 资源默认 `UpdatePolicy` / `DeletionPolicy` 为 `Retain`。__ 此项与 CFn 的默认行为相反。（版本：v1.2.0，见 [#2601](https://github.com/aws/aws-cdk/issues/2601)）
* __部分资源缺乏 `DeletionPolicy` 抽象。__ 部分资源需要直接调用底层的接口才能修改，而且底层接口名字混乱，方法名字是 `RemovalPolicy`，值又是 `DESTROY`。
  * ✅ 对于 Construct，可以在传入参数时直接带 `removalPolicy: cdk.RemovalPolicy.DESTROY`

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

### Events（CloudWatch Events）

* __Events 是消息队列。__ 有相对独立的 ARN 和 API 名字（events）。
  * __后续发展成了 EventBridge。__ 🇨🇳 中国区暂时没有。

### Logs（CloudWatch Logs）

* __Logs 是日志收集和查看器。__ 有相对独立的 ARN 和 API 名字（logs）。
  * __有 Logs Insights 可以直接对日志进行查询。__ 🇨🇳 中国区暂时没有。
* __可以把日志转发到 Lambda。__
  * 🇨🇳 __中国区暂不支持。__
* __可以把日志转发到 Elasticsearch。__ 仅限 AWS 托管的 Elasticsearch。
  * 🇨🇳 __中国区暂不支持。__

## CloudTrail

_「官方的 API 访问日志。」_

### 总览

* __默认打开。__

## Kinesis

_「托管的消息队列。」_

### Data Streams

* 💢 __传递延迟较高。__ 目前仅支持亚秒级的传递延迟（200-1000ms）。（见 [Link](https://aws.amazon.com/about-aws/whats-new/2015/03/amazon-kinesis-propagation-delay-reduction/)、[Link](https://docs.amazonaws.cn/en_us/streams/latest/dev/building-consumers.html)）
  * 如需最大限度降低延迟，还应在客户端处降低轮询间隔。（见 [Link](https://docs.amazonaws.cn/en_us/streams/latest/dev/kinesis-low-latency.html)）
  * 使用 Enhanced Fan-Out 推送机制可以降低到约 75ms。（见 [Link](https://docs.amazonaws.cn/en_us/streams/latest/dev/building-consumers.html)）
  * 与之相对的，是 Apache Kafka 可以优化到个位数 ms 级的传递延迟。（见 [Link](https://engineering.linkedin.com/kafka/benchmarking-apache-kafka-2-million-writes-second-three-cheap-machines)）

### Data Firehose

* __输送数据专用。__ 消息队列常会充当传输数据时的缓存，所以专门提供了这个功能。
  * 支持往 S3、Lambda、Elasticsearch 等目标位置输送数据。
* __支持数据转换。__ 输送数据之前可以用 Lambda 进行格式转换。

### Video Streams

* 🇨🇳 中国区暂未上线。

## ECS（Elastic Container Service）

_「托管的 Docker 集群治理工具。」_

### ECS Container Agent

* __EC2 在安装 ECS Container Agent 之后才能接入 ECS 集群。__ ECS 管理的 EC2 实例或 ECS-Optimized 型实例已经自带 ECS Container Agent。
* __ECS Container Agent 是开源的。__ 见 [Link](https://github.com/aws/amazon-ecs-agent)，目前仅支持 EC2。

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

__快速通道 >>>__ [JavaScript API 手册](https://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/DynamoDB.html)

* __从 Amazon.com 购物车需求发展而来。__
  * 有论文。

### DynamoDB Streams

### 踩坑

* __Primary Key 如果包含 sort key，则意味着在 `getItem` 时必须同时提供 partition key 和 sort key。__
  * Primary Key 是确保数据唯一性的依据，而 `getItem` 是取出单条数据，所以必须完整提供两个 key，否则会报「The provided key element does not match the schema」错误。
  * 如果只想按照 partition key 来查询，可以使用 `query`。
* __DynamoDB 使用特定数据格式。__ 如 `{ id: { 'N': 100 } }`，其中的 `N` 代表数据是数字格式。
  * 在 SDK 中可使用 `AWS.DynamoDB.Converter.unmarshall()` / `marshall()` 函数在 DynamoDB 格式与原生格式之间做转换。
* __索引不等于唯一。__ DynamoDB 的索引对唯一性没有要求。


## HyperPlane

* __AWS 内部的负载均衡服务。__
* __从 S3 发展而来。__

## KMS（Key Management Service）

### CMK（Customer-Managed Keys）

* __使用 `GenerateDataKey` 命令来返回明文 Key + 密文 Key。__ 明文 Key 用来加密数据，密文 Key 和数据存一起，取出数据时先解密 Key，然后用 Key 解密数据。
* __使用 `GenerateDataKeyWithoutPlainText` 命令只返回密文 Key。__ 适用于需要延迟一段时间再加密的情况，届时需先解密 Key，然后再加密。
  * 也可用于权限隔离，比如 A 组件负责存放对应的密文 Key，B 组件负责解密 Key 并用 Key 来加密数据。即 A 组件无需知道密文 Key。

## CLI（Command Line Interface）

_「命令行界面。」_

* ✅ __可使用内置 Autocompleter 提升体验。__ 参考 [Link](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-completion.html)。
  * 🈲 原来的 AWS-Shell 工具已经没有在活跃开发，且集成不便，不推荐使用。

### 踩坑

* 🈲 __如果自行 `export AWS_DEFAULT_REGION=xxx` 而 `xxx` 不存在，则后续所有的命令都可能报错。__ 错误信息：`botocore.exceptions.ProfileNotFound: The config profile (xxx) could not be found`。

## Beanstalk（Elastic Beanstalk）

_「编程 PaaS 平台。」_

* __可直接上传代码形成应用。__ 可选择多种编程语言运行环境，以及 Web 和 Worker 两种模板。
* __支持多种部署方式。__
  * Canary
  * Green-Blue
* __会使用 S3 桶来存储应用配置。__ 每个部署应用的 Region 会放一个桶。
  * 💢 默认不做存储加密，需自行开启加密。
* __使用 `.ebextensions` 文件夹中的 `.config` 文件来定制环境。__ 

### EB CLI

* __Beanstalk 提供类似 `npm` 的工具。__

### 踩坑

* __Beanstalk 是一种部署 + 运行时服务。__ 不对接 CodeDeploy，而是直接对接 CodeBuild 的产出物。
  * 在 CodePipeline 里面也以部署服务的形式供选择。


## API Gateway

_「托管的 API 网关。」_

* __Edge-Optimized Endpoint 就是 API Gateway + CloudFront，使用全局统一的域名。__ Regional Endpoint 需要你按需自行配置 CloudFront。
  * 🇨🇳 中国区暂时没有 Edge-Optimized Endpoint。
  * 💢 如使用 Edge-Optimized Endpoint，由于使用了全局统一域名并仅路由到单个 Region 的 API Gateway，所以无法做双活容错、基于延迟的 DNS 路由等等。要达到这个效果可使用多个 Regional Endpoint。（见 [Link](https://aws.amazon.com/blogs/compute/building-a-multi-region-serverless-application-with-amazon-api-gateway-and-aws-lambda/)）

### API 配置

* __API 设置好之后需要部署才能使用。__ 旧版 API 可以和新版并存。
* __API 部署后会在 URI 后加入部署的 Stage 名作为路径。__ 可以通过自定义域名来避免。
* __原来不支持把后台服务放到私有子网中。__ 现在可以通过 PrivateLink 以及 NLB 来支持。
* __使用代理模式时可使用 `{proxy+}` 作为路径参数。__ 这是一个贪婪式参数，会尽量多地匹配所有路径上的字符，并存入 `{proxy}` 这个参数中。
  * 在 Console 中使用 `{proxy+}` 会自动创建 `ANY` 方法并只能选择代理模式，可删除 `ANY` 方法并自行创建需要的方法。
  * `{proxy+}` 和代理模式实际上是两个独立的功能。
  * `{proxy+}` 中的 `proxy` 可以改成其他名字，但建议保留 `proxy` 以便识别。
* __不能使用 `/res/{proxy+}` 的形式。__ 因为资源路径只允许纯参数或者纯静态，不允许混杂二者。
  * 可先创建 `/res` 然后在其之下创建 `/{proxy+}`。

### Lambda

* __Lambda Proxy 打开之后路径和请求中的参数可以从 `event` 中读取。__ 分别是 `event.pathParameters` 和 `event.queryStringParameters`。
* __Console 中的设置变更有诸多 bug。__ 建议更改 Proxy 和 Execution Role 等设置的时候采取删除 Method 重建的方式。
* 💢 __使用 Lambda Proxy 集成的 API 在返回值不符合格式要求时会产生 `502 Bad Gateway` 错误。__ 见 [Link](https://forums.aws.amazon.com/thread.jspa?threadID=246541)。
  * 请仔细检查 `status` 和 `body`，尤其是 `body` 是否已经 `JSON.stringify()`。
* 💢 __默认的超时只有 3s。__ 函数在初次执行时可能会超时。
  * 在后续再执行函数时会效率会逐步提高到最优，所以为了避免性能损耗，会做 [Pre-warming](https://forums.aws.amazon.com/thread.jspa?threadID=232882)。
* 💢 __Console 中的「Test」功能偶尔会使用旧代码。__ 修改函数并保存后，点击「Test」，日志现实仍然测试的是旧函数。

### 踩坑

* __路径信息不会自动添加到 Endpoint 上。__ 必须手动在 Endpoint 上添加静态路径，或者用路径参数捕捉后再添加动态路径。
  * 即便使用代理模式也不会自动添加。
* __测试 `POST` 方法时将默认使用 `application/json` 格式。__ 不是 Web 常见的 `application/x-www-form-urlencoded` 格式。
  * 可在 Method Execution 界面手动添加 `Content-Type: application/x-www-form-urlencoded` 的 header 来覆盖。


## SQS（Simple Queue Service）

_「托管的极简消息队列。」_

* __最早的 AWS 服务之一。__ 2004 年即[存在](http://jeff-barr.com/2014/08/19/my-first-12-years-at-amazon-dot-com/)但未用于生产，2006 年 6 月 13 日[上线](https://amazonaws-china.com/blogs/aws/amazon_simple_q/)。


## Data Pipeline

* __传输的源和目标称作「Data Node」。__

## Glue

_「数据归拢工具。」_

* __使用爬虫获取内部数据并发布到 Glue Data Catalog。__


## X-Ray

_「托管的分布式追踪系统。」_

* __Interceptors。__

## AWS SDK

_「用编程方式来调用 AWS 服务接口。」_

### JavaScript

* 💢 __几乎所有服务接口调用均为异步，返回 `AWS.Request` 对象。__
  * ✅ 可使用 `AWS.Request.promise()` 函数将其转换成 `Promise` 然后使用 `await` 修饰符来转化成同步调用。注意 `await` 仅能在带 `async` 修饰符的函数中使用。

## CodeBuild

* 💢 __默认不支持 VPC 内的资源，需要额外的配置。__ 见 [Link](https://docs.aws.amazon.com/codebuild/latest/userguide/vpc-support.html)。

## CodeDeploy

* 💢 __EC2 的权限错误会导致部署步骤全部被「跳过」。__ 如果你发现所有部署步骤都被 Skip 掉，可先检查 EC2 是否有足够权限访问 S3 桶。
  * 可在 EC2 实例上查看日志，位置是 `/var/log/aws/codedeploy-agent/codedeploy-agent.log`。

## Organizations

* 💢 __被关联的账号无法下载详细的账单报表。__








