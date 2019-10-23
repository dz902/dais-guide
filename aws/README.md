# 张氏 AWS 简明手册

## 说明

* 从 Open Guide 得到灵感。
* 仅为个人爱好收集整理，不代表任何组织和官方意见，不承诺正确性及可能造成的后果。
* 使用方法：⌘ + F / ⌃ + F。

## 图例

* 🇨🇳 = 宁夏和北京 Region（中国大陆 Region，下简称「中国区」）相关
* 🚚 = 在 CFn / CDK 中的情况
* 🚥 = 在 CLI 工具中的情况
* 👀 = 在 CloudWatch Metrics 中的情况
* ✅ = 建议执行的操作
* 🈲 = 不建议执行的操作
* 💢 = 需要注意的坑
* 💰 = 会产生额外费用
* 🎓 = 考试中常见

## 🇨🇳 中国区特有情况

_「部分情况也适用于 GovCloud。」_

> [服务](https://www.amazonaws.cn/en/about-aws/regional-product-services/) | [案例](https://aws.amazon.com/cn/solutions/case-studies/china/)

### 使用限制

* __仅限企业。__ 个人用户无法注册使用。
* 💢 __必须备案。__ 客户必须进行网站 ICP 备案，才能启用 Web 端口，否则会返回 `403`。

### 区域隔离

* __网络不互通。__ 中国区与其他 Region 没有亚马逊官方的光纤直联。
* __账号不互通。__ 中国大陆 Region 与其他 Region 的账号不处于同一体系。
* 💢 __资源不互通。__ ARN 中的第二段由「aws」改为「aws-cn」，与 Global 资源不在一个体系。

### Service Endpoint / Principal 差异

* 💢 __中国区的服务命名与 Global 其他 Region 有些许差异。__ 请查看 [AWS Service Endpoints](https://docs.amazonaws.cn/en_us/general/latest/gr/rande.html) 来确定具体名字。

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

> [手册](https://docs.aws.amazon.com/IAM/latest/UserGuide/introduction.html)

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

### Policy

* 🚥 __使用 `simulate-custom-policy` 可以模拟该 Policy 的效果。__ 见 [Link](https://docs.aws.amazon.com/cli/latest/reference/iam/simulate-custom-policy.html)。

### MFA（Multi-Factor Authentication）

* ✅ __MFA（多因子身份验证）要求用户在登录时使用额外的一次性密码。__ 推荐使用，至少应该在 Root Account 或管理员账号上启用。
* __也可使用 MFA 来保护 API 调用。__
  * 🎓 __`AssumeRole` 和 [`GetSessionToken`](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_mfa_sample-code.html#MFAProtectedAPI-example-getsessiontoken) 均可[传递 MFA 信息](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_mfa_configure-api-require.html)。__

### 测试 Policy

* __使用 [`SimulateCustomPolicy`](https://docs.aws.amazon.com/cli/latest/reference/iam/simulate-custom-policy.html) 命令可以测试 Policy。__
* __需要提供 Context Keys。__ 比如 IP 地址、日期。
  * 为在 `Condition` 中的判断条件提供测试值。

## S3（Simple Storage Service）

_「功能丰富的对象存储。」_

> [手册](https://docs.aws.amazon.com/AmazonS3/latest/dev/Welcome.html) | [CLI](https://docs.aws.amazon.com/cli/latest/reference/s3/index.html) | [FAQ](https://www.amazonaws.cn/en/s3/faqs/) | [价格](https://www.amazonaws.cn/en/s3/pricing/)

### 概况

* __第一个 AWS 服务。__ 2006 年 3 月 14 日上线。
* __3 AZ 多副本。__ 11 个 9 的数据耐久度；3 个 9 的 SLA。
* __事件触发。__ 使用 __Events__ 机制可以在文件上传、文件修改等事件发生时触发 __Lambda__。
* __支持使用 Service Gateway 不走公网访问。__ Service Gateway 不额外收费，推荐使用。
* __扁平化存储。__ 没有文件夹的概念，但是对结尾为 `/` 的文件做了特殊处理，可以当做文件夹使用。
  * 这种 `/` 结尾的特殊文件被称作 Prefix。

### Bucket

* __S3 是 Global 服务，但 Bucket 有区域之分。__ 区域选择会影响访问速度。
  * 🇨🇳 中国区除外。中国区的 S3 无法看到 Global 的 Bucket。
* __必须使用 Region-specific Endpoint（`bucket-name.s3.region-name.amazonaws.com`）来访问。
  * 如果不使用正确的 Endpoint，2019 年 3 月之前建立的桶将会收到 `307 Permanently Moved` 以及正确的 Endpoint。
  * 2019 年 3 月后建立的桶将不支持不带 `region-name` 的 Endpoint，并直接返回 `400 Bad Request` 错误。
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
* __有「Block Public Access」的设置。__ 针对 Bucket 的设置。
  * ✅ 强烈建议打开。
  * 防止新上传对象公开访问，开启后会抛弃新上传对象的公开访问权限。
  * 防止所有对象公开访问，开启后会忽略对象所附着的公开访问权限。
  * 💢 __需要特别注意该选项仅会阻止完全公开的 Bucket Policy。__ 一旦加上 `Condition` 语句，即便是非常公开的条件语句，也不会被阻止。
  * 💢 __比如 `aws:SourceIp: 52.0.0.0/*` 等极其宽泛的 Bucket Policy 也不会被阻止。__

### 并发

* 💢 __S3 不是为大并发设计。__ 每个 Prefix 下的所有文件[仅支持 5500 次/秒的 `GET` 总并发](https://docs.aws.amazon.com/AmazonS3/latest/dev/optimizing-performance.html)，以及 3500 次/秒的 `PUT` 并发。
  * S3 对 Prefix 数量没有限制，所以理论上并发总数没有限制。
  * 超过 5500 次/秒的并发需求需要在架构上进行调整，或者使用 CloudFront。
* 💢 __S3 会限制单 IP 并发请求。__ 当单并发请求数到达 95 左右时，会出现流量控制的情况。

### Endpoint

* __Global / Local Endpoint。__
  * 🇨🇳 __中国区不支持 Global Endpoint。__ 在中国区必须使用 Regional Endpoint，否则会返回 `404`。

### S3 Events

* __在上传、修改等事件时触发 Lambda。__
  * 💢 __不确保及时、准确性。__ 即可能会有 1 分钟以上的延时，可能会有重复的事件，可能会漏发事件。（见 [Link](http://www.hydrogen18.com/blog/aws-s3-event-notifications-probably-once.html?ck_subscriber_id=527212891)）
  * 🈲 __不推荐用来处理有强一致性需求的任务。__ ✅ 可以用于做一些简单的统计，特别是可耐受少量信息冗余、缺失的。

### S3 Select

* __直接对 S3 上存储的半结构化数据进行视图过滤。__ 基于 PartiQL 语言。

### Batch Operations

> [手册](https://docs.aws.amazon.com/AmazonS3/latest/dev/batch-ops-basics.html)

* __使用 Inventory 来[创建桶内对象列表快照](https://docs.aws.amazon.com/AmazonS3/latest/dev/storage-inventory.html)。__ 可选择包含文件尺寸、修改时间等。
  * 可导出为 CSV / Parque / ORC 等格式。
  * 可以方便地在 Amazon Athena 中查询。
* __使用对象列表作为输入来执行批量操作。__

### 上传

* __使用 `PUT` 可上传最大 5GB 的文件。__ 使用 Multipart Upload 可上传最大 5TB 的文件。
* __Multipart Upload 支持文件块串行、并行上传。__ ✅ 网络不好时可以降低上传失败风险，网络好时可以最大限度利用带宽。
* 💢 __尚未完成的 Multipart Upload 文件块不会自动删除，但是会占用空间并产生费用。__ 可使用 Bucket Lifecycle Policy 删除一定天数后仍未完成的文件块，见 [Link](https://docs.aws.amazon.com/AmazonS3/latest/dev/mpuoverview.html#mpuploadpricing)。

### 数据湖

* __数据湖核心。__ S3 是 AWS 数据湖的核心存储机制，也是数据湖的「湖」本体，大部分分析类服务的持久化存储由 S3 提供。

### S3 Analytics

* __可使用 S3 Analytics 分析桶内对象存储的规律。__ 特别是不常访问的对象。

### 踩坑

* 🚚 __RegionalDomainName。__ 在 CFn 中创建 Bucket 之后如果读取 `DomainName` 属性会返回 Global Endpoint，需要改用 `RegionalDomainName`。
* 🚥 __无法从 CLI 获得 ARN。__ S3 的 CLI 没有提供任何获得 ARN 的方法，用户只能从 Console 复制，或者自己组成 ARN，需考虑中国区或 GovCloud 的名称差异问题。
* 👀 __`BucketSizeBytes` 和 `NumberOfObject` 的数字包括所有版本以及暂时未完成的 Multipart Upload 文件块。__ 即这些 CloudWatch Metrics 体现的数字会大于 S3 Console 中体现的数字，见 [Link](https://docs.aws.amazon.com/AmazonS3/latest/dev/cloudwatch-monitoring.html)。
* __`PUT` 操作既用于上传文件，又用于修改现有文件的属性。__ 见 [Link](https://docs.aws.amazon.com/cli/latest/reference/s3api/put-object.html)。


## Athena

_「Serverless 版的 Hive。」_

* 💢 __[仅支持 `EXTERNAL` 表](https://docs.aws.amazon.com/athena/latest/ug/creating-tables.html)。__ 且数据需存储在 S3。

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
* 🎓 __EC2Rescue 是一个 Instance 诊断工具。__ 可以[离线诊断 Windows 实例](https://docs.aws.amazon.com/AWSEC2/latest/WindowsGuide/ec2rw-gui.html)。
  * 方法是在另一台诊断实例上安装 EC2Rescue，把无响应/无法连接的 Windows 实例的盘挂载到诊断实例上，运行该工具。
  * 也有 Linux 版本但是功能较弱。

### SSH

* __SSH 卡死时请使用 `~` 大法。__ 先按 `Enter`，然后按 `~`，然后按 `.`，即可强制终止 SSH。
* __不同系统的默认 SSH 用户名不同。__ Amazon Linux 系列是 `ec2-user`，CentOS 是 `centos`，也有一些是 `root`。

### Instance Metadata

* __在实例上通过 `http://169.254.169.254/latest/meta-data/` 可以获得 Instance Metadata。__ 包括 AMI-ID、IP 地址、MAC 地址等等。
  * 此项功能由 EC2 提供，无论实例上运行的是哪种系统都可以获取。

### EC2 Instance Connect

* __安装 `ec2-instance-connect` 包之后支持针对制定用户上传临时公钥进行登录。__ 60 秒内有效。
  * 该包会修改 sshd daemon 并在用户远程登录时以 instance metadata 里面的临时公钥来做验证。
  * EC2 Instance Connect 服务在 instance metadata 中录入临时公钥并在 60 秒后删除。
  * 🇨🇳 中国区暂时没有 EC2 Instance Connect 服务所以此项功能无法使用。
  * 💢 __Amazon Linux 2 默认安装并开启 `ec2-instance-connect`。__
  * __Ubuntu 也支持安装。__ 其余系统不支持。

### RI（Reserved Instance）

* __RI 是一个账单机制不是产品机制。__ 所以无法在产品中看到某个 EC2 是否使用了 RI。
  * 可以在 Cost Explorer 中[查看使用情况统计](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/usage-reports.html)。🇨🇳 中国区暂时没有。
  * 也可以在账单中查看 RI 小时数和非 RI 小时数。


## EBS（Elastic Block Storage）

_「块存储服务。」_

### 安全

* ✅ __支持 KMS 全盘加密。__ 不影响 I/O 性能。

### 备份

* __不提供定期备份机制，需要手动备份。__ 或使用脚本。
* __备份快照基于用量。__ 快照的尺寸等于实际数据使用量。
* __备份为增量。__ 对卷进行首次快照后，后续新增快照仅会存储变动的部分。

### 踩坑

* __`growpart` 在 CentOS 上可能执行成功但是仍然报错。__ 用户可尝试直接重启机器再看效果。
* __`st1` 和 `sc1` [不能用作启动盘](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EBSVolumeTypes.html)。__

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
* __默认每 Region 可以有 5 个 EIP。__
* __在运行中的实例上使用单个 EIP 不收费。__ 申请了但闲置不用反而收费。
  * 在实例上使用超过 1 个 EIP，多出部分按个收费。

### NACL

* __无状态的双向端口防火墙。__
* __每个 NACL 只对应一个 VPC，但可以对应多个 Subnet。__

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

### Traffic Miroring

* __在不安装 agent 的情况下将一台虚机的进出口流量镜像到另一台虚机上。__ 用于调试和监控。

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


## R53（Route 53）

### `ALIAS` 记录

* __`ALIAS` 是 R53 对 DNS 的[私有扩展](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resource-record-sets-choosing-alias-non-alias.html)。__
  * 🎓 只能指向带公开访问地址的 AWS 资源，比如 ELB、Beanstalk 环境、S3 桶等。
  * 支持裸域名（zone apex）指向。DNS 原生不支持在裸域名上创建 `CNAME` 指向其他域名，[使用 `ALIAS` + `A` 记录的方式](https://stackoverflow.com/questions/22053472/how-do-i-redirect-a-naked-apex-domain-to-www-using-route-53)可以指向使用其他域名的 AWS 资源。
  * 💢 `ALIAS` 也[不支持在裸域名上做 `CNAME`](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resource-record-sets-values-alias.html)，因为 `ALIAS` 与其对应的记录，名字和类型都必须相同，而裸域名上本身就不支持 `CNAME`。
  * 会在资源 IP 地址更新时自动同步记录。
  * 🎓 也可支持指向其他账号内的 AWS 资源。
  * 🎓 不收费，对比 `CNAME` 记录会收取请求费。


## Lambda

_「无服务器计算资源。」_

> [文档](https://docs.aws.amazon.com/lambda/latest/dg/welcome.html) | [价格](https://www.amazonaws.cn/en/lambda/pricing/) | [FAQ](https://www.amazonaws.cn/en/lambda/faqs/)

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

### 异步调用

* __支持异步调用。__ 实际上是 Lambda 自己管理了一个事件队列来做异步调用。
  * 💢 __用户无法获取异步调用的函数的返回值。__ 暂时[没有相关API](https://docs.aws.amazon.com/lambda/latest/dg/invocation-async.html)。
  * 💢 __Lambda 异步调用与编程语言自己的 `async` 行为是两个完全不同的东西。__ 后者仍然可以是同步调用，Lambda 会等待函数所有异步执行完毕后再返回。
* __异步调用的函数返回错误时会重试 2 次。__ 分别间隔 1、2 分钟。
  * 如果均失败则会尝试把事件[发送至 Dead Letter Queue](https://docs.aws.amazon.com/lambda/latest/dg/invocation-async.html#dlq)，如果有设置。
  * 如果因为触及并行限额而被限流，事件会被放回到队列中[在 6 小时内自动重试](https://docs.aws.amazon.com/lambda/latest/dg/invocation-async.html#dlq)。
* 💢 __异步调用的队列是最终一致的。__ 即同样的事件可能被发送两次，也可能因为函数处理不及而事件已经过期所以还没发送给函数就被删掉。
  * ✅ 函数应该能很好地处理重复事件。
  * ✅ 应该配置 Dead Letter Queue 来避免事件丢失。

### 踩坑

* 🚚 __Log Group 无法在 Stack 删除或回退时自动删除。__ Lambda 所附带的 Log Group 是自动创建的，不在 Stack 资源之列，所以不会在 Stack 删除时自动删除。（见 [Link](https://blog.rowanudell.com/cleaning-up-lambda-logs-with-cloudformation/)）
  * 可手动创建名为 `/aws/lambda/${lambdaFunctionName}` 的 Log Group，即可自动删除。
* 🚚 __注意 Lambda 函数 Role 需要有足够权限。__ 特别注意对 Log Group 的操作，以及对各项资源的操作。
  * 💢 如果没有 Log Group 创建权限则不会自动创建 Log Group，如果没有写入 Log Group Stream 权限则不会自动写入，但二者均不会报错。
  * ✅ 可使用部分 Managed Policy，比如 AWSLambdaBasicExecutionRole。
  * 🚚 注意 Managed Policy 名字有的前面有前缀（比如 `service-role/`、`job-role/`），有的没有前缀。在 Console 中不会显示前缀，但在 CDK 中使用 `fromAwsManagedPolicyName()` 时需要把前缀加上。
* 🈲 __请勿在生产环境使用内置的包。__ 虽然可以在函数中直接 `import boto3`，但是内置的包无法保证版本且随时可能更新，导致莫名错误。
  * ✅ 在生产环境中请使用 Layer 引入外部依赖。
* __函数对应的 IAM 权限修改后并不一定会立刻生效。__ 因函数的 IAM 权限实际是附着于容器上，而容器的释放时间不定。
  * ✅ 更新 IAM 权限后应尝试修改函数内容或配置并保存，强行使容器释放更新，以应用新的 IAM 权限。
* __Lambda 具备[自动重试](https://docs.aws.amazon.com/lambda/latest/dg/retries-on-errors.html)机制。__ 需要注意。

## ESS（Elasticsearch Service）

_「托管的 Elasticsearch + Kibana 应用。」_

> [手册](https://docs.aws.amazon.com/elasticsearch-service/latest/developerguide/what-is-amazon-elasticsearch-service.html)

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

> [手册](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/Welcome.html) | [资源](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/AWS_EC2.html) | [CLI](https://docs.aws.amazon.com/cli/latest/reference/cloudformation/index.html#cli-aws-cloudformation)

* __Template 必须上传到 S3 桶中。__ 选择本地文件[也会自动上传到 S3](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-whatis-howdoesitwork.html)。

### Parameter

* 🎓 __使用单个 Template 时，可以使用 Parameter 来增加复用性。__

### Nested Stack

* __需要复用局部架构时，可嵌套其他 Template，形成 Nested Stack。__

### CLI

* __`package` 命令可以上传 Assets 到 S3 桶，并自动将 Template 中指向本地文件的部分修改为指向 S3 的地址。__

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

> [API](https://docs.aws.amazon.com/cdk/api/latest/versions.html)

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

* 🇨🇳 __`cdk bootstrap` 中使用的 CFn 返回 Bucket 的 Endpoint 错误，导致不可创建 ChangeSet。__ CFn 中的返回值为 `DomainName` 而不是 `RegionalDomainName`，而中国区的 CloudFormation 无法返回正确 Endpoint，导致错误。解决办法如下。（版本：v1.2.0，见 [#1459](https://github.com/aws/aws-cdk/issues/1459)）
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

_「托管的关系型数据库。」_

* __支持 Multi-AZ。__ 高可用副本，实时同步。
  * 可读写。
  * 平时不启用，当主实例失效时自动启用并更新域名指向。
* __支持 Read Replica。__ 只读副本，非实时同步。

### 系统维护

* 🎓 __[由 RDS 负责](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_UpgradeDBInstance.Maintenance.html)操作系统、数据库的补丁及版本升级。__
* __可以设置 Maintenace Window。__ RDS 将在该维护窗口期对执行维护。
  * 遇到部分高危问题时补丁会立即应用。
* __部分补丁需要数据库下线一段时间。__ 比如需要操作系统重启。
* __使用 Multi-AZ 可以降低对可用性的影响。__
  * 先在 Standby 上执行维护，将 Standby 升级为 Primary，然后在原 Primary（现 Standby）上执行维护。

### Automated Backup

* __支持自动备份。__ 在选定的时间窗口进行备份。
* 🈲 __可以禁用。__ 设置 Rentention Period 为 0 时可以禁用。
  * 不推荐禁用。
* __备份尺寸小于预留数据库时[不收费](https://amazonaws-china.com/rds/mysql/pricing/?pg=pr&loc=2)。__ 超过时才收费。

## Aurora（RDS Aurora）

_「托管的云原生数据库。」_

* __有发表专门的论文。__

### Endpoint 类型

* __Cluster Endpoint 是唯一可执行写操作的节点。__ 连接到数据库主实例。
  * 也可以读。
  * 每个集群固定有 1 个。
  * Aurora 管理，不可自建。
* __Reader Endpoint 是只读节点。__ 连接到数据库的 Read Replica。
  * 有连接负载均衡，无查询负载均衡。
  * 每个
* __Custom Endpoint 是自选节点。__ 连接到你制定的一组实例中的一个。有连接带负载均衡。
* __Instance Endpoint 是实例节点。__ 连接到单个实例。每个实例均有自己的专属 Instance Endpoint。

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
* __Dashboard 不分地区。__
* __自定义 Metric 支持 [High-Resolution](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/publishingMetrics.html#high-resolution-metrics)。__ 默认数据收集频率 1 分钟 1 次，High-Resolution 数据收集频率为 1 秒钟 1 次。
  * __区别于 EC2 [Detail Monitoring](https://docs.amazonaws.cn/en_us/AWSEC2/latest/UserGuide/using-cloudwatch.html)。__ EC2 服务每 5 分钟发送一次实例指标给 CloudWatch，开启 Detailed Monitoring 之后每 1 分钟发送一次。

### Events（CloudWatch Events）

* __Events 是消息队列。__ 有相对独立的 ARN 和 API 名字（events）。
  * __后续发展成了 EventBridge。__ 🇨🇳 中国区暂时没有。

### Logs（CloudWatch Logs）

* __Logs 是日志收集和查看器。__ 有相对独立的 ARN 和 API 名字（logs）。
  * __有 Logs Insights 可以直接对日志进行查询。__ 🇨🇳 中国区暂时没有。
* __可以把日志转发到 Lambda。__
  * 🇨🇳 中国区暂不支持。
* __可以把日志转发到 Elasticsearch。__ 仅限 AWS 托管的 Elasticsearch。
  * 🇨🇳 中国区暂不支持。

### Alarms

* __有 OK、INSUFFICIENT_DATA 和 ALARM 三种状态。__
* __有 Period、Evaluation Period 和 Datapoints to Alarm 三个参数。__
  * Period = 隔多少时间收集一次数据。
  * Evalution Period = 统计最近几次的数据。
  * Datapoints to Alarm = 统计结果要超标多少次才触发警报。
* __Regular Alarm，Period = 60 秒。__ High-Resolution Alarm，Period = 10 / 30 秒。

## CloudTrail

_「官方的 API 访问日志。」_

### 总览

* __默认打开。__

### Event

* __区分 Management Event 和 Data Event 类型。__ 
  * __Management Event 指的是管理类操作。__ 通常指的是配置或权限的修改。
  * __Data Event 指的是对资源的操作。__ 比如上传文件到 S3。
  * 💢 __默认不记录 Data Event。__

## Kinesis

_「托管的消息队列。」_

### Data Streams

> [手册](https://docs.aws.amazon.com/streams/latest/dev/introduction.html) | [限制](https://docs.amazonaws.cn/en_us/streams/latest/dev/service-sizes-and-limits.html) | [价格](https://www.amazonaws.cn/en/kinesis/data-streams/pricing/) | [FAQ](https://www.amazonaws.cn/en/kinesis/data-streams/faqs/)

* 💢 __默认按收取时间来为消息排序，[不确保精准](https://docs.aws.amazon.com/kinesis/latest/APIReference/API_PutRecord.html)。__ 如需精准排序，需在消息发送时手动提供 `SequenceNumberForOrdering` 参数，消息会按此参数从小到大排序。
  * 如不提供，Sequence Number 会自增并保持唯一。
* 💢 __排序仅针对单个 Shard。__ Sequence Number 也[仅在单个 Shard 内唯一](https://docs.aws.amazon.com/streams/latest/dev/key-concepts.html)。
* 💢 __消息不能手动删除。__ 只能等自动过期。
  * 在取数据时[提供 Sequence Number](https://docs.aws.amazon.com/kinesis/latest/APIReference/API_GetRecords.html) 来确保只取 Sequence Number 之后的新消息。
* 💢 __传递延迟较高。__ 目前仅支持亚秒级的传递延迟（200-1000ms）。（见 [Link](https://aws.amazon.com/about-aws/whats-new/2015/03/amazon-kinesis-propagation-delay-reduction/)、[Link](https://docs.amazonaws.cn/en_us/streams/latest/dev/building-consumers.html)）
  * 如需最大限度降低延迟，还应在客户端处降低轮询间隔。（见 [Link](https://docs.amazonaws.cn/en_us/streams/latest/dev/kinesis-low-latency.html)）
  * 使用 Enhanced Fan-Out 推送机制可以降低到约 75ms。（见 [Link](https://docs.amazonaws.cn/en_us/streams/latest/dev/building-consumers.html)）
  * 与之相对的，是 Apache Kafka 可以优化到个位数 ms 级的传递延迟。（见 [Link](https://engineering.linkedin.com/kafka/benchmarking-apache-kafka-2-million-writes-second-three-cheap-machines)）

#### KCL（Kinesis Client Library）

* __KCL 是 Kinesis Data Stream 的[官方高阶 SDK](https://docs.aws.amazon.com/streams/latest/dev/developing-consumers-with-kcl.html)。__ 比原始 API 更抽象一层。
* 🎓 __可使用 KCL 的 [`checkpoint`](https://docs.aws.amazon.com/streams/latest/dev/kinesis-record-processor-implementation-app-java.html) 方法来设置数据指针。__ 类似书籍的页码，用于指示目前已读取并处理到该 Shard 的哪一条记录。
  * 在原始 API 中，数据指针被称作 [Shard Iterator](https://docs.aws.amazon.com/kinesis/latest/APIReference/API_GetShardIterator.html)。

#### Sharding

* __消息会按 Parition Key 被送到不同的 Shard。__ 每个 Shard 提供每秒 1MB 输入，2MB 输出，1000 次 `PUT`。
* 🎓 __Consumer 的数量不应超过 Shard 数量。__ 虽然没有明确限制，但是在手册针对 KCL 中有[明确建议](https://docs.aws.amazon.com/streams/latest/dev/kinesis-record-processor-scaling.html)和[说明](https://docs.aws.amazon.com/streams/latest/dev/troubleshooting-consumers.html#records-belonging-to-the-same-shard)。
  * __可理解为 1 Shard 最多支持 1 Consumer。__ 但 1 Consumer 可以处理多个 Shard。

#### 自动重试

* __有 Producer 重试和 Consumer 重试两种[情况](https://docs.aws.amazon.com/streams/latest/dev/kinesis-record-processor-duplicates.html)。__

### Data Firehose

> [手册](https://docs.aws.amazon.com/firehose/latest/dev/what-is-this-service.html)

* __输送数据专用。__ 消息队列常会充当传输数据时的缓存，所以专门提供了这个功能。
  * 支持往 S3、Lambda、Elasticsearch 等目标位置输送数据。
* __支持数据转换。__ 输送数据之前可以用 Lambda 进行格式转换。
* __[支持 SSE 静态加密](https://docs.aws.amazon.com/firehose/latest/dev/encryption.html)。__
  * __数据源是 Kinesis Data Stream 时 Firehose 不存储数据到硬盘。__ 加密由 Kinesis Data Stream 执行，解密传输到 Firehose 缓存入内存，然后直接存至目标。
  * __数据源是 `PUT` 时，可以使用 `StartDeliveryStreamEncryption` 打开 Firehose 的 SSE 加密功能。

### Video Streams

* 🇨🇳 中国区暂未上线。

### Data Analytics

* 🇨🇳 中国区咱未上线。
* __使用 SQL 语句对队列中的数据进行流处理并输出结果。__ 也可以使用 Java。

## ElastiCache

_「托管的 Redis/Memcached 数据库。」_

* __仅供云上访问。__ 默认不能从 AWS 之外访问。
  * 🈲 除非[使用 NAT](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/accessing-elasticache.html)，但不安全且配置麻烦。

## ECS（Elastic Container Service）

_「托管的 Docker 集群治理工具。」_

### ECS Container Agent

* __EC2 在安装 ECS Container Agent 之后才能接入 ECS 集群。__ ECS 管理的 EC2 实例或 ECS-Optimized 型实例已经自带 ECS Container Agent。
* __ECS Container Agent 是开源的。__ 见 [Link](https://github.com/aws/amazon-ecs-agent)，目前仅支持 EC2。

## ECR（Elastic Container Repository）

_「托管的 Docker 镜像存储服务。」_

* 🎓 __使用 `docker` 命令之前，先运行 `aws ecr get-login` 并[执行其输出的内容](https://docs.aws.amazon.com/AmazonECR/latest/userguide/ECR_AWSCLI.html#AWSCLI_push_image)。__ 这可以让你登录到 ECR 而不是 Docker 官方的镜像存储服务。

## EKS（Elastic Kubernetes Service）

_「托管的 K8s Master Node。」_

### 踩坑

* 🇨🇳 __没有官方镜像。__ 中国区无法访问 Google 节点，所以无法从官方渠道下载 K8s 应用。
* __责任共担模型变化。__

## EMR（Elastic MapReduce）

_「托管的 Hadoop 生态。」_

> [价格](https://www.amazonaws.cn/en/elasticmapreduce/pricing/)

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

> [手册](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Introduction.html) | [API](https://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/DynamoDB.html)

* __从 Amazon.com 购物车需求发展而来。__
  * 有[论文](https://www.dynamodbguide.com/the-dynamo-paper/)，后采用了 [Paxos 算法](https://en.wikipedia.org/wiki/Paxos_%28computer_science%29)。
* __DynamoDB Accelerator（DAX）指的是内存缓存层。__ 打开 DAX 即可使用内存缓存提升效率。
* __读写均有体量限制。__ 每单位读 = 4KB/条目/秒，每单位写 = 1KB/条目/秒。
  * 使用最终一致性读取时，每单位读 = 2 × 4KB/条目/秒。
* __数据默认加密。__ 可以在创建时[选择](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/EncryptionAtRest.html) DynamoDB 管理密钥（免费），或 KMS 管理密钥。
  * 也可以创建后修改。

### Partition

* __数据表会根据 Partition Key 而[分成 10GB 大小的 Partition](https://amazonaws-china.com/blogs/database/choosing-the-right-dynamodb-partition-key/)。__

### DynamoDB Streams

* 🎓 __DynamoDB 的时序增删改查事件合集。__
* __有几种 View Type。__ 代表 Stream 中的通知数据结构。
  * `KEYS_ONLY`，只记录条目的 Key。
  * `NEW_IMAGE`，只记录条目修改后的值。
  * `OLD_IMAGE`，只记录条目修改前的值。
  * `NEW_AND_OLD_IMAGES`，同时记录条目修改前后的值。

### Capacity Unit

* 🎓 __分 Read Capacity Unit（RCU）和 Write Capacity Unit（WCU）。__ 时间单位为秒。
  * 1 RCU = 1 次读取请求（强一致性），或 2 次读取请求（最终一致性），每次请求可读取最多 4KB 数据。
  * 1 WCU = 1 次写入请求，最多 1KB 数据。
* __RCU 和 WCU 均向上舍入。__ 即不足 1 Unit 的按 1 Unit 计算。
* __使用 `ReadItem` 和 `BatchReadItem`， 每读取一个 Item 即算一个读取操作。__ 即 1.5KB 的 Item 会向上舍入成 4KB，6.5KB 的 Item 会向上舍入到 8KB。
* __读取不存在的 Item 也会消耗读取操作。__ 和正常的读取一致。

### 索引

* __Primary Key 是数据唯一性的依据。__ 可以是单 Partition Key，或者是 Partition Key + Sort Key。
  * 🎓 __Partition Key 是数据分区的一句，应尽量选择高唯一性的字段。__
  * __Sort Key 不参与数据分区，但可以用于 Query。__
* 🎓 __Secondary Index 是单张表的[第二套 Primary Key](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/SecondaryIndexes.html)。__ 包含 Global 和 Local 两种。
  * __可以理解为创建了一张子表。__
  * 💢 __Secondary Index [的值可以重复](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/GSI.html)。__ 没有唯一性要求。
* __Global Secondary Index = 与原来完全不同的 Primary Key。__ 
  * 可随时添加和删除。
  * 不支持强一致性查询。
  * 每个 Global Secondary Index 自带 RCU 和 WCU。
  * 不可请求不在 Index 中的字段。
* __Local Secondary Index = 与原来的 Primary Key 共享 Partition Key，但提出新的 Sort Key。__
  * 只能在表创建时创建。
  * 强一致性或最终一致性。
  * 与表共享 RCU 和 WCU。
  * 可请求不在 Index 中的字段。

### 查询

* 🎓 __主要有 Query 和 Scan 两种查询方式。__
  * Query 针对 Index。
  * Scan 针对全表或 Secondary Index。
* 🎓 __Query 按照返回数据消耗 RCU，而 Scan 按照扫描数据量消耗 RCU。__
  * 二者皆有过滤结果的方式，但是 Scan 的过滤只是把[已经取回的结果扔掉](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Scan.html#Scan.FilterExpression)。
* __取单条数据时可使用 Get。__ 需提供 Primary Key。
  * __Primary Key 如果是 Paritition Key + Sort Key，则意味着在 `getItem` 时必须同时提供二者。__
  * 否则会报「The provided key element does not match the schema」错误。
  * 如果只想按照 Partition Key 来查询，可以使用 Query。
* __根据 Primary Key 来取多条数据时可以使用 BatchGet。__

### Global Tables

* __可在指定 Region 创建 Replica 并统一管理。__ 自动同步数据，一般[数秒内](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/globaltables_HowItWorks.html)可同步。
  * __需打开 DynamoDB Streams。__
* __每个 Replica 均支持读写。__ 写冲突时后者覆盖前者。
* __不同 Region 的 Replica 之间没有强一致性选项。__ 仅保证最终一致性。

### 踩坑

* __DynamoDB 使用特定数据格式。__ 如 `{ id: { 'N': 100 } }`，其中的 `N` 代表数据是数字格式。
  * 在 SDK 中可使用 `AWS.DynamoDB.Converter.unmarshall()` / `marshall()` 函数在 DynamoDB 格式与原生格式之间做转换。
* __索引不等于唯一。__ DynamoDB 的索引对唯一性没有要求。


## HyperPlane

* __AWS 内部的负载均衡服务。__
* __从 S3 发展而来。__

## KMS（Key Management Service）

_「托管的密钥管理服务。」_

> [手册](https://docs.aws.amazon.com/kms/latest/developerguide/overview.html) | [FAQ](https://www.amazonaws.cn/en/kms/faqs/) | [价格](https://www.amazonaws.cn/en/kms/pricing/)

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
* __使用 `.ebextensions` 文件夹中的 `*.config` 文件来定制环境。__ 支持多种定制，比如安装某些包，创建用户、用户组等等。
* __使用 [Saved Configuration](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/environment-configuration-savedconfig.html) 可以将当前配置存为 `.yml` 文件。__ 可以用于创建多个类似的环境，比如 DEV / PROD。
  * 默认存放在 Beanstalk 创建的 S3 桶中。

### 部署模式

* __Rolling Updates = 逐台更新。__
* __Rolling with Additional Batch = 创建 n 台新机器更新。__ 健康检查成功后关掉 n 台旧机器，如此往复直到全部更新完毕。
* __Immutable Updates = 开一个新的 ASG 然后开新机器。__ 全部健康检查通过后，机器转移到原来的 ASG，删掉空的 ASG，停掉旧机器。
* __[Blue/Green Deployment](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/using-features.CNAMESwap.html) = [克隆新环境](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/using-features.CNAMESwap.html)并部署到新机器。__ 成功后切换 CNAME 到新的环境。

### VPC

* 💢 __实例需要[通过公网](https://forums.aws.amazon.com/thread.jspa?threadID=164178)与 Beanstalk 通讯。__
  * 如果实例在 Public Subnet，则需要设置 Public IP。
  * 如果实例在 Private Subnet，则需要[设置 NAT](https://aws.amazon.com/premiumsupport/knowledge-center/elastic-beanstalk-instance-failure/)。
* 💢 __实例需要[打开 UDP 123](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/vpc.html) 端口来同步时钟。__ 否则健康检查会失效。

### ELB

* 🎓 __ELB 的类型[只能在创建环境时选择](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/using-features.managing.elb.html)__。如果要修改 ELB，不能通过修改环境、克隆环境来达成，必须新建环境。
* 💢 __Classic Load Balancer 也需要 VPC。__ 虽然 Classic Load Balancer [不必须使用 VPC](https://docs.aws.amazon.com/elasticloadbalancing/latest/classic/elb-getting-started.html)，但是在 Beanstalk 中创建 Classic Load Balancer 时需要选择 VPC 或者在该区域有默认 VPC，否则会报错。

### Docker

* 🎓 __Multi Container Docker 环境底层[使用了 ECS](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/create_deploy_docker_ecs.html)。__
  * 配置方式与 ECS 一致，使用 Task Definition。
  * Single Container Docker 就是普通 Docker 环境。可使用 Dockerfile 或者 Task Definition 来配置。

### EB CLI

* __Beanstalk 提供类似 `git` 的工具 `eb`。__

### 踩坑

* __Beanstalk 是一种部署 + 运行时服务。__ 不对接 CodeDeploy，而是直接对接 CodeBuild 的产出物。
  * 在 CodePipeline 里面也以部署服务的形式供选择。
* __安全组在 Beanstalk 之外被关联可能导致环境无法终止。__
  * Beanstalk 会[为外部的 RDS / ElastiCache 等资源创建安全组](https://forums.aws.amazon.com/message.jspa?messageID=591163)，而这些安全组可能因为无法删除而导致 Beanstalk 无法重建、终止环境。


## API Gateway

_「托管的 API 网关。」_

* __Edge-Optimized Endpoint 就是 API Gateway + CloudFront，使用全局统一的域名。__ Regional Endpoint 需要你按需自行配置 CloudFront。
  * 🇨🇳 中国区暂时没有 Edge-Optimized Endpoint。
  * 💢 如使用 Edge-Optimized Endpoint，由于使用了全局统一域名并仅路由到单个 Region 的 API Gateway，所以无法做双活容错、基于延迟的 DNS 路由等等。要达到这个效果可使用多个 Regional Endpoint。（见 [Link](https://aws.amazon.com/blogs/compute/building-a-multi-region-serverless-application-with-amazon-api-gateway-and-aws-lambda/)）
* __可以直接设置为 AWS 服务代理。__ 直接[集成 AWS 服务](https://amazonaws-china.com/blogs/compute/using-amazon-api-gateway-as-a-proxy-for-dynamodb/)。

### Method & Integration

* __Method Request / Response。__ Method 是与前端对接的部分，比如 `GET` / `POST`。

### API 配置

* 💢 __API 设置好之后需要部署才能使用。__ 旧版 API 可以和新版并存。
* __API 部署后会在 URI 后加入部署的 Stage 名作为路径。__ 可以通过自定义域名来避免。
* __原来不支持把后台服务放到私有子网中。__ 现在可以通过 PrivateLink 以及 NLB 来支持。
* __使用代理模式时可使用 `{proxy+}` 作为路径参数。__ 这是一个贪婪式参数，会尽量多地匹配所有路径上的字符，并存入 `{proxy}` 这个参数中。
  * 在 Console 中使用 `{proxy+}` 会自动创建 `ANY` 方法并只能选择代理模式，可删除 `ANY` 方法并自行创建需要的方法。
  * `{proxy+}` 和代理模式实际上是两个独立的功能。
  * `{proxy+}` 中的 `proxy` 可以改成其他名字，但建议保留 `proxy` 以便识别。
* __不能使用 `/res/{proxy+}` 的形式。__ 因为资源路径只允许纯参数或者纯静态，不允许混杂二者。
  * 可先创建 `/res` 然后在其之下创建 `/{proxy+}`。

### Model

* __用于指示 Request 和 Response 的格式。__

### Lambda

* __Lambda Proxy 打开之后路径和请求中的参数可以从 `event` 中读取。__ 分别是 `event.pathParameters` 和 `event.queryStringParameters`。
* __Console 中的设置变更有诸多 bug。__ 建议更改 Proxy 和 Execution Role 等设置的时候采取删除 Method 重建的方式。
* __使用 Lambda Proxy 集成的 API 在返回值不符合格式要求时会产生 `502 Bad Gateway` 错误。__ 见 [Link](https://forums.aws.amazon.com/thread.jspa?threadID=246541)。
  * ✅ 请仔细检查 `status` 和 `body`，尤其是 `body` 是否已经 `JSON.stringify()`。
* __默认的超时只有 3s。__ 函数在初次执行时可能会超时。
  * 在后续再执行函数时会效率会逐步提高到最优，所以为了避免性能损耗，会做 [Pre-warming](https://forums.aws.amazon.com/thread.jspa?threadID=232882)。
* __Console 中的「Test」功能偶尔会使用旧代码。__ 修改函数并保存后，点击「Test」，日志现实仍然测试的是旧函数。

### Stage

* __Stage 用于区分同一套 API 的不同上下文环境。__ 比如 DEV / PROD。
* __Stage 名字会出现在 URI 中。__ 比如 `https://1x2x3x4x.execute-api.cn-northwest-1.amazonaws.com.cn/dev` 中的 `dev`。
* __可使用 Stage Variable 来存储变量。__ 可以在 URI 中使用这些变量，以便根据 Stage 来做一些简单的定制。

#### Canary Release

* __在 Stage 上可以开启 Canary Release。__ 
  * 开启后部署会先到 Canary Release。
  * 可设定固定百分比的流量指向 Canary Release。
  * 需手动 Promote Canary Release 才能真正部署到 Stage。

### 错误代码

* __`429 Too Many Requests` 用户请求数量超过了限额。__ 默认每秒 10000 次调用。

### 并发限制

* 💢 __单个 Method 的并发限制[默认等于账号区域并发限制](https://theburningmonk.com/2019/10/the-api-gateway-security-flaw-you-need-to-pay-attention-to/?ck_subscriber_id=527212891)。__ 意味着对一个 Method 进行攻击，就能把单个账号下所有 API 的限额用完。

### 踩坑

* __路径信息不会自动添加到 Endpoint 上。__ 必须手动在 Endpoint 上添加静态路径，或者用路径参数捕捉后再添加动态路径。
  * 即便使用代理模式也不会自动添加。
* __测试 `POST` 方法时将默认使用 `application/json` 格式。__ 不是 Web 常见的 `application/x-www-form-urlencoded` 格式。
  * 可在 Method Execution 界面手动添加 `Content-Type: application/x-www-form-urlencoded` 的 header 来覆盖。


## SQS（Simple Queue Service）

_「托管的极简消息队列。」_

> [手册](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/welcome.html) | [FAQ](https://amazonaws-china.com/sqs/faqs/)

* __最早的 AWS 服务之一。__ 2004 年即[存在](http://jeff-barr.com/2014/08/19/my-first-12-years-at-amazon-dot-com/)但未用于生产，2006 年 6 月 13 日[上线](https://amazonaws-china.com/blogs/aws/amazon_simple_q/)。
* 💢 __消息处理完后不会自动删除。__ 必须手动删除。
  * __设置队列的 VisibilityTimeout 可以确保在初次返回消息之后一段时间内该消息不会再被返回，避免被不同客户端重复处理。__ 默认 30 秒，最短 0 秒，最长 12 小时。
* 🎓 __设置对列的 DelaySeconds 参数，可以[让消息延迟返回](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-delay-queues.html)。__ 默认 0 秒，最长 15 分钟。
  * 也可在单条消息上设置 message timer 参数来覆盖 DelaySeconds 参数。

### [Short / Long Polling](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-short-and-long-polling.html#sqs-long-polling)

* __使用 `ReceiveMessage` 来获取消息。__ 有 Short 和 Long 两种轮询模式。
* __默认使用 Short Polling 模式。__ 在 SQS 集群上随机取样并返回消息。
  * 💢 __可能无法一次取回完整的待处理消息。__ 需要多次调用才能取到完整消息。
* __可在使用 `ReceiveMessage` 时设置 `WaitTimeSeconds > 0` 则使用 Long Polling 模式。__ 可以避免反复轮询又取不完数据造成的浪费。
  * __一旦有新的信息会立刻返回。__

### FIFO Queue

## Data Pipeline

* __传输的源和目标称作「Data Node」。__

## System Manager

_「系统管理工具集。」_

### Parameter Store

* __安全地存储密码等机密信息。__ 免费存储 10000 条，无额外费用。

### SSM Agent（System Manager Agent）

* 🎓 __可以安装在 EC2 或云下的机器上，让 System Manager 可以统一管理。__ 也[支持 Raspberry Pi](https://amazonaws-china.com/blogs/mt/manage-raspberry-pi-devices-using-aws-systems-manager/) 的 Raspian OS。

## Secret Manager

_「秘密管理工具。」_

* __安全存储数据库密码等秘密信息。__
* __可自动轮换密码。__ 与 RDS 自动集成。
* 🇨🇳 __中国区暂时没有。__

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

* __支持「Build 命令」、「项目配置」、「`buildspec.yml`」三个地方设置环境变量。__ [优先顺序](https://docs.aws.amazon.com/codebuild/latest/userguide/build-spec-ref.html)：Build 命令 > 项目配置 > `buildspec.yml`。
  * 🎓 __构建参数的长度有限制。__ 推荐使用 System Manager Parameter Store 才存储，可以在 `buildspec.yml` 中[直接引用](https://docs.aws.amazon.com/codebuild/latest/userguide/build-env-ref-env-vars.html)。
* 💢 __默认不支持 VPC 内的资源，需要额外的配置。__ 见 [Link](https://docs.aws.amazon.com/codebuild/latest/userguide/vpc-support.html)。

## CodeDeploy

### 部署方式

* __All-at-once = 全部机器同时部署。__ 服务会中断。
* __Rolling = 每次部署 n 台，完成后继续部署 n 台。__ 算力 = 总量减去 n 台。
* __Rolling with additional batch = 额外开 n 台机器进行部署，完成后继续部署 n 台。__ 算力在部署期间不变。
* __Immutable = 额外开一个 Auto Scaling 组，并完整复制整个应用。__ 算力在部署期间 × 2。

* 💢 __默认的回滚方式是重新部署。__ 对于蓝绿部署来说，可以手动做环境 URL 切换。
* 💢 __必须安装 Agent。__ 否则无法部署。
* 💢 __EC2 的权限错误会导致部署步骤全部被「跳过」。__ 如果你发现所有部署步骤都被 Skip 掉，可先检查 EC2 是否有足够权限访问 S3 桶。
  * 可在 EC2 实例上查看日志，位置是 `/var/log/aws/codedeploy-agent/codedeploy-agent.log`。

## Organizations

* 💢 __被关联的账号无法下载详细的账单报表。__

## STS（Simple Token Service）

_「登录令牌发放服务。」_

### `AssumeRole`

* __返回 AK/SK 以及 Session Token。__ 调用 API 时需要同时提供。

### `AssumeRoleWithWebIdentity`

* __先登录 Facebook 并拿到 OAuth 2 令牌，然后用令牌来调用 `AssumeRoleWithWebIdentity`。__

### `DecodeAuthorizationMessage`

* __用户执行未授权操作时，AWS 服务会返回 `403 Unauthorized` 并附加一条加密的信息。__ 这条信息可以用 `DecodeAuthorizationMessage` 来解密。

## CloudFront

_「CDN 服务。」_

> [价格](https://www.amazonaws.cn/en/cloudfront/pricing/)

* __免费支持 Server Name Indication（SNI）。__ 如果[使用专属 IP 的 SSL 证书](https://docs.amazonaws.cn/en_us/AmazonCloudFront/latest/DeveloperGuide/cnames-https-dedicated-ip-or-sni.html)（Dedicated IP SSL Certificate），会产生费用。


## SNS（Simple Notification Service）

_「消息推送服务。」_

* __系统推送支持 Lambda、SQS、EventBridge 和 HTTP/S Endpoint。__
* __用户推送支持 Mobile Push 和短信。__

### Topic

* __用户必须先订阅 Topic 才能收到消息。__ 消息会推送给所有订阅者。

### Filter

* __可以为每个订阅者添加 Filter 来过滤消息。__ 比创建多个 Topic 更高效。

### Fanout Pattern

* ✅ __用一个消息队列（如 SQS）来解耦发布者和订阅者。__ 订阅者订阅消息队列，而不直接订阅 Topic。
  * 牺牲部分实时性，换取并行处理、弹性伸缩等好处。

## Cognito

_「托管的 Facebook、Google、Amazon 用户身份服务。」_

> [手册](https://docs.aws.amazon.com/cognito/latest/developerguide/what-is-amazon-cognito.html)

* __🇨🇳 中国区暂时没有。__
* __支持 MFA。__

### Cognito Sync

* 🎓 __可在多个设备间[同步和推送用户信息](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-sync.html)。__ 无需自建后台。

## CodePipeline

> [手册](https://docs.aws.amazon.com/codepipeline/latest/userguide/welcome.html)

* __🇨🇳 中国区暂时没有。__

## CodeBuild

### Build Phases

* 🎓 __构建分为[不同 Phase](https://docs.aws.amazon.com/codebuild/latest/APIReference/API_BuildPhase.html)。__
  * `SUBMITTED`，收到构建任务。
  * `PROVISIONING`，申请构建资源。如失败直接跳到 `COMPLETED`。
  * `DOWNLOAD_SOURCE`，下载源代码。如失败直接跳到 `FINALIZING`。
  * `INSTALL`，安装需要的包。如失败直接跳到 `FINALIZING`。
  * `PRE_BUILD`，执行构建前的操作。如失败直接跳到 `FINALIZING`。
  * `BUILD`，执行构建操作。
  * `POST_BUILD`，执行构建后的操作。
  * `UPLOAD_ARTIFACTS`，上传构建好的应用。构建失败的产物也会上传，方便你调试。
  * `FINALIZING`，收尾工作。
  * `COMPLETED`，完成构建。
* __构建的 Phase 转换规则请见[此链接](https://docs.aws.amazon.com/codebuild/latest/userguide/view-build-details.html#view-build-details-phases)。__

## CodeCommit

_「托管的 Git 代码仓库。」_

* 🇨🇳 __中国区暂时没有。__

### Git

* 🎓 __如果需要通过 HTTPS 方式来使用 CodeCommit，则[需要生成 Git 身份验证信息](https://docs.aws.amazon.com/codecommit/latest/userguide/setting-up-gc.html#setting-up-gc-iam)。__

### Notifications

* __对接 CloudWatch 进行通知。__
* __主要针对 Pull Requests 和 Comments 进行通知。__ 后来[增加了状态变化的通知](https://aws.amazon.com/about-aws/whats-new/2017/08/aws-codecommit-now-sends-state-changes-to-amazon-cloudwatch-events-saves-user-preferences-and-adds-tag-details-view/)，即可以在新代码提交时触发。
  * 💢 新增的状态变化通知并没做进 CodeCommit 一侧的 Notifications 设置中，而需要在 CloudWatch 中选择。

### Triggers

* __在代码提交、创建/删除 Branch 时发送 SNS 通知或触发 Lambda。__
* 💢 __与 Notification 的功能有重合。__


## CodeStar

_「基于 Code* 系列的软件研发管理环境。」_

* 🇨🇳 __中国区暂时没有。__
* __直接集成了 CodePipeline、CodeCommit、CodeBuild、CodeDeploy 等服务。__

### Member Roles

* __内置不同的 Role 类型。__ 每个项目设置一组。
  * Owner，即管理员，可管理项目成员。
  * Contributor，即可读写和执行操作的人员。
  * Viewer，即只读人员。

## Rekgnition

_「托管的视频和图片识别服务。」_

* __🇨🇳 中国区暂时没有。__
* 💢 __封装的高阶 API 可能会报意义不明的错误。__ 
  * `CompareFaces` 会[抛出 `InvalidParameterException`](https://forums.aws.amazon.com/thread.jspa?threadID=261754)，但实际上仅仅是因为图片中没有包含人脸，而非参数错误。
  * `DetectLabels` 会[抛出 `InvalidParameterException`](https://forums.aws.amazon.com/thread.jspa?threadID=250949)，但实际上是因为图片尺寸小于 Rekgnition 支持的最小尺寸（80px × 80px）。


## Config

_「托管的配置管理监控服务。」_

* __根据 Rule 评估次数及记录的配置项数量来收费。__

### Rule

* __Change-triggered Rule。__ 在 Config 监测到配置改变时触发。
* __Periodic Rule。__ 定时触发。

### Scope

* __Rule 创建后必须设置生效范围。__ Rule 仅针对范围内的资源进行评估。

### Aggregated View

* __所有区域的所有资源列表。__


## Service Catalog

_「自定义 AWS 服务列表和入口。」_

* 🇨🇳 中国区暂时没有。
* __基本概念是 Portfolio 和 Product。__ 前者是列表，后者是产品。









