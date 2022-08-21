# 权限模型

![](<../../../.gitbook/assets/image (13).png>)

常见的权限模型有下面这 5 种：&#x20;

权限控制列表（ACL，Access Control List）。&#x20;

自主访问控制（DAC，Discretionary Access Control）。&#x20;

强制访问控制（MAC，Mandatory Access Control）。&#x20;

基于角色的访问控制（RBAC，Role-Based Access Control）。&#x20;

基于属性的权限验证（ABAC，Attribute-Based Access Control）。

### 基于角色的访问控制（RBAC）

RBAC (Role-Based Access Control，基于角色的访问控制)，引入了 Role（角色）的概念，并且将权限与角色进行关联。用户通过扮演某种角色，具有该角色的所有权限。

![](<../../../.gitbook/assets/image (36).png>)

每个用户关联一个或多个角色，每个角色关联一个或多个权限，每个权限又包含了一个或者多个操作，操作包含了对资源的操作集合。通过用户和权限解耦，可以实现非常灵活的权限管理

```yaml
Permission:
    - Name: write_article
        - Effect: "allow"
        - Action: ["Create", "Update", "Read"]
        - Object: ["Article"]
    - Name: manage_article
        - Effect: "allow"
        - Action: ["Delete", "Read"]
        - Object: ["Article"]


Role:
    - Name: Writer
      Permissions:
        - write_article
    - Name: Manager
      Permissions:
        - manage_article
    - Name: CEO
      Permissions:
        - write_article
        - manage_article


Subject: Colin
Roles:
    - Writer


Subject: James
Roles:
    - Writer
    - Manager

```

RBAC 又分为 RBAC0、RBAC1、RBAC2、RBAC3。&#x20;

**RBAC0：**基础模型，只包含核心的四要素，也就是用户（User）、角色（Role）、权限（Permission：Objects-Operations）、会话（Session）。用户和角色可以是多对多的关系，权限和角色也是多对多的关系。&#x20;

**RBAC1：**包括了 RBAC0，并且添加了角色继承。角色继承，即角色可以继承自其他角色，在拥有其他角色权限的同时，还可以关联额外的权限。&#x20;

**RBAC2：**包括 RBAC0，并且添加了约束。具有以下核心特性： 互斥约束：包括互斥用户、互斥角色、互斥权限。同一个用户不能拥有相互排斥的角色，两个互斥角色不能分配一样的权限集，互斥的权限不能分配给同一个角色，在 Session 中，同一个角色不能拥有互斥权限。 基数约束：一个角色被分配的用户数量受限，它指的是有多少用户能拥有这个角色。例如，一个角色是专门为公司 CEO 创建的，那这个角色的数量就是有限的。 先决条件角色：指要想获得较高的权限，要首先拥有低一级的权限。例如，先有副总经理权限，才能有总经理权限。 静态职责分离(Static Separation of Duty)：用户无法同时被赋予有冲突的角色。 动态职责分离(Dynamic Separation of Duty)：用户会话中，无法同时激活有冲突的角色。&#x20;

**RBAC3：**全功能的 RBAC，合并了 RBAC0、RBAC1、RBAC2。

### 基于属性的权限验证（ABAC）

ABAC (Attribute-Based Access Control，基于属性的权限验证），规定了哪些属性的用户可以对哪些属性的资源在哪些限制条件下进行哪些操作。

跟 RBAC 相比，ABAC 对权限的控制粒度更细，主要规定了下面这四类属性：

用户属性，例如性别、年龄、工作等。

资源属性，例如创建时间、所属位置等。

操作属性，例如创建、修改等。

环境属性，例如来源 IP、当前时间等。

```yaml
Subject:
    Name: Colin
    Department: Product
    Role: Writer
Action:
    - create
    - update
Resource:
    Type: Article
    Tag:
        - technology
        - software
    Mode:
        - draft
Contextual:
    IP: 10.0.0.10
```

产品部门的 Colin 作为一个 Writer 角色，可以通过来源 IP 是 10.0.0.10 的客户端，创建和更新带有 technology 和 software 标签的草稿文章

### 相关开源项目

#### Casbin&#x20;

一个用 Go 语言编写的访问控制框架，功能强大，支持 ACL、RBAC、ABAC 等访问模型

{% embed url="https://github.com/casbin/casbin" %}

#### keto&#x20;

一个云原生权限控制服务，通过提供 REST API 进行授权，支持 RBAC、ABAC、ACL、AWS IAM 策略、Kubernetes Roles 等权限模型

{% embed url="https://github.com/ory/keto" %}

#### go-admin&#x20;

一个基于 Gin + Vue + Element UI 的前后端分离权限管理系统脚手架，它的访问控制模型采用了 Casbin 的 RBAC 访问控制模型

{% embed url="https://github.com/go-admin-team/go-admin" %}
