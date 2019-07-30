# 张氏 AWS 简明手册

## 说明

* 从 Open Guide 得到灵感。
* 仅为个人爱好收集整理，不代表任何组织和官方意见，不承诺正确性及可能造成的后果。
* 使用方法：⌘ + F / ⌃ + F。

## 图例

* 🇨🇳 = 中国大陆特有情况
* 🚚 = 服务在 CFn / CDK 中的情况

## 目录

* 中国大陆特有
* IAM
* S3
* Lambda
* ES
* CFn
* CDK

## 🇨🇳 中国大陆特有情况

_「部分情况也适用于 GovCloud。」_

### 使用限制

* __仅限企业。__ 个人用户无法注册使用。
* __必须备案。__ 客户必须进行网站 ICP 备案，才能启用 Web 端口，否则会返回 `403`。

### 区域隔离

* __网络不互通。__ 宁夏、北京 Region（下简称「中国大陆 Region」）与其他 Region 没有亚马逊官方的光纤直联。
* __账号不互通。__ 中国大陆 Region 与其他 Region 的账号不处于同一体系。
* __资源不互通。__ ARN 中的第二段由「aws」改为「aws-cn」，与 Global 资源不在一个体系。

### Service Endpoint / Principal 差异

## Availability Zone（AZ）/ Region

### 概况

* __AZ 指近距离、分布式数据中心集群。__ 一个 AZ 包含两个或以上的数据中心，数据中心之间采用高速光纤直连，另有冗余的网络集线中心确保 AZ 内的高速、不间断互通。
  * 🇨🇳 中文也俗称「可用区」。
* __Region 指单个城市范围内的 AZ 集群。__ 大部分 Region 包含 3 个 AZ，少部分包含 2 个 AZ，极少部分仅有 1 个 AZ。
* __Region 内 AZ 之间距离数十公里。__ 并且水、电、网络、安保均为各自独立，确保能做到同城灾备。


## IAM

_「云上的访问权限管理体系。」_

### 概况

* __Identity-Based Policy（IBP）是附在用户、角色身上的访问权限。__ 如果没有 IBP 用户无法访问资源。
* __Resource-Based Policy（RBP）是附在资源身上的访问权限。__ 只有部分资源有 RBP。
* __Policy 包含多个 Statement。__
  * Statement 必须包含：Effect（允许 / 拒绝）、Action、Resource。
  * Statement 可选包含：Condition。
  * RBP 的 Statement 还包含 Principal（执行人）。

## S3（Simple Storage Service）

_「功能丰富的对象存储。」_

### 概况

* __第一个 AWS 服务。__
* __3 AZ 多副本。__ 11 个 9 的数据耐久度；3 个 9 的 SLA。
* __事件触发。__ 使用 __Events__ 机制可以在文件上传、文件修改等事件发生时触发 __Lambda__。
* __支持使用 Service Gateway 不走公网访问。 __Service Gateway 不额外收费，推荐使用。
* __扁平化存储。 __没有文件夹的概念，但是对结尾为 `/` 的文件做了特殊处理，可以当做文件夹使用。

### 权限

* __有 Access Control List（ACL）、RBP 两种权限控制体系。__ 二选一即可，通常使用 RBP。
* __ACL 以用户为单位来赋予权限。__ 需要使用 S3 专用的 Canonical ID 来指代用户，在 Bucket 的 ACL 页面可以找到。
* __ACL 只有读、写、读取权限、写入权限、完全控制几种权限设置。__ 采用 XML 形式存储。
* __ACL 默认赋予 Bucket 拥有者全部权限。__ 拥有者不再需要额外设置 RBP。
* __非 Bucket 拥有者需要在桶上设置 RBP 并持有对应的 IBP 才能访问。__
* __有「Block Public Access」的设置。__ 针对 Bucket 的设置。
  * 防止新上传对象公开访问，开启后会抛弃新上传对象的公开访问权限。
  * 防止所有对象公开访问，开启后会忽略对象所附着的公开访问权限。

### 细节

* __Global / Local Endpoint。__

### S3 Select

* __直接对 S3 上存储的半结构化数据进行视图过滤。__

### 踩坑

* 🇨🇳 __不支持 Global Endpoint。__ 在中国必须使用 Regional Endpoint，否则会返回 `404`。
  * 🚚 __RegionalDomainName。__ 在 CFn 中创建 Bucket 之后如果读取 `DomainName` 属性会返回 Global Endpoint，需要改用 `RegionalDomainName`。
* 🇨🇳 __Cross-Region Replication 不支持非中国大陆 Region。__ 跨区复制仅能在中国大陆 Region 之间进行。

## S3 Athena

_「直接针对 S3 上存的半结构化数据跑 SQL。」_

## EC2（Elastic Cloud Compute）

_「云上虚拟机。」_

### 概览

* __默认密钥对登录。__ 若要密码登录虚拟机，需要进行额外的操作。

### Amazon Linux

* __基于 RHEL 的开源 Linux。__ 使用 yum 包管理工具。
* __经过审核、强化并预装 AWS 工具包。__
* 🇨🇳 __中国区没有 Amazon Linux 2。__

### 踩坑



## VPC（Virtual Private Cloud）

_「云上的虚拟专属网络。」_

### 踩坑

* __非 Amazon Linux 使用双 ENI 需要自己配置操作系统路由。__ 否则可能出现无法访问的情况。

## Lambda

_「无服务器计算资源。」_

### 概况

* __采用 Firecracker Micro-VM 内核。__
* __以「内存/100ms」为单位来收费。__

### 踩坑

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
  * 把 ARN 写成常量。（简易可行，推荐）
  * 写一个 Custom Resource，用 Lambda 来设置常量。
* __非空 Bucket 无法删除。__ S3 Bucket 如果已经放了东西，则无法在回滚、删除 Stack 时自动删除，需手动操作，如果在频繁测试，可以自行写 Shell 脚本进行删除。

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

* 🇨🇳 __Bootstrap 使用的 CFn 返回 Bucket 的 Global URL，导致不可创建 ChangeSet。__ CFn 中的返回值为 Bucket.DomainName 而不是 Bucket.DomainName，而中国不支持 Global URL。解决办法如下。（版本：v1.2.0，见 [#1459](https://github.com/aws/aws-cdk/issues/1459)）
  * 修改 `node_modules/aws-cdk/lib/api/bootstrap-environment.js` 中的 `"StagingBucket", "DomainName"` 为 `"StagingBucket", "RegionalDomainName"`。
* __S3、LogGroup 资源默认 UpdatePolicy / DeletionPolicy 为 Retain。__ 此项与 CFn 的默认行为相反。（版本：v1.2.0，见 [#2601](https://github.com/aws/aws-cdk/issues/2601)）

## IoT Greengrass

_「边缘计算的控制器。」_

## AppSync

_「托管的 GraphQL。」_

## RDS Aurora

_「云原生数据库。」_



