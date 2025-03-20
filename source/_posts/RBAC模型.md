---
title: RBAC模型
mermaid: true
math: false
comments: true
hide: false
excerpt: RBAC（Role-Based Access Control）即：基于角色的权限控制。通过角色关联用户，角色关联权限的方式间接赋予用户权限。

date: 2025-03-20 20:40:42
tags: MySQL
categories: 系统设计
---


# 什么是 RBAC 模型

RBAC（Role-Based Access Control）即：基于角色的权限控制。通过角色关联用户，角色关联权限的方式间接赋予用户权限。

> 在简单的系统里面，每个用户只存在一种角色，那么可以直接用户绑定权限。但是对于比较大的系统，很多用户拥有相同的一批权限，这时，我们如果不引入角色的关联，那么每次都需要批量修改全部的用户权限。而且角色关联，很方便的可以针对于多用户多角色的场景。


# RBAC 模型的分类

RBAC 模型可以分为：RBAC0、RBAC1、RBAC2、RBAC3 四种。其中 RBAC0 是基础，也是最简单的，相当于底层逻辑，RBAC1、RBAC2、RBAC3 都是以 RBAC0 为基础的升级。以下我们分别解释4种模型。

## RBAC0

> 最简单的用户、角色、权限模型。彼此之间可以`n:1`也可以是`n:n`。


**表结构设计**

1. **用户表（User）**：存储用户信息。
2. **角色表（Role）**：存储角色信息。
3. **权限表（Permission）**：存储权限信息。
4. **用户-角色关联表（User_Role）**：存储用户与角色的关联关系。
5. **角色-权限关联表（Role_Permission）**：存储角色与权限的关联关系。

**图例**

![RBAC0.png](https://img.stuartyang.site/2025/03/b4aaacb65f593b3cb7b585ee8de1e936.png)


**数据示范**

1. 用户表（User）

| user_id | username | password  |
| ------- | -------- | --------- |
| 1       | userA    | passwordA |
| 2       | userB    | passwordB |
| 3       | userC    | passwordC |

2. 角色表（Role）

|role_id|role_name|
|---|---|
|1|admin|
|2|editor|
|3|viewer|

3. 权限表（Permission）

|permission_id|permission_name|resource|action|
|---|---|---|---|
|1|user:read|user|read|
|2|user:create|user|create|
|3|user:update|user|update|
|4|user:delete|user|delete|

4. 用户-角色关联表（User_Role）

| user_id | role_id | 解释               |
| ------- | ------- | ---------------- |
| 1       | 1       | # userA 是 admin  |
| 2       | 2       | # userB 是 editor |
| 3       | 3       | # userC 是 viewer |

5. 角色-权限关联表（Role_Permission）

| role_id | permission_id | 解释                      |
| ------- | ------------- | ----------------------- |
| 1       | 1             | # admin 拥有 user:read    |
| 1       | 2             | # admin 拥有 user:create  |
| 1       | 3             | # admin 拥有 user:update  |
| 1       | 4             | # admin 拥有 user:delete  |
| 2       | 1             | # editor 拥有 user:read   |
| 2       | 2             | # editor 拥有 user:create |
| 2       | 3             | # editor 拥有 user:update |
| 3       | 1             | # viewer 拥有 user:read   |

## RBAC1

> RBAC1在RBAC0的基础上<mark style="background: #FFB8EBA6;">引入了角色继承的概念</mark>，即子角色可以继承父角色的所有权限。这种设计可以简化权限管理，同时支持更复杂的权限分配逻辑。


**表结构设计**

1. **用户表（User）**
2. **角色表（Role）**
3. **权限表（Permission）**
4. **用户-角色关联表（User_Role）**
5. **角色-权限关联表（Role_Permission）**
6. **<mark style="background: #FFB8EBA6;">角色层次表（Role_Hierarchy）</mark>**：表明角色之间的层级关系

**图例**

![RBAC1](https://img.stuartyang.site/2025/03/bb027075a9d34214197c4823b5eb66c7.png)

**数据示例**

角色层次表（Role_Hierarchy）

| parent_role_id | child_role_id | 解释                     |
| -------------- | ------------- | ---------------------- |
| 1              | 2             | # admin 是 editor 的父角色  |
| 2              | 3             | # editor 是 viewer 的父角色 |


## RBAC2

> RBAC1在RBAC0的基础上<mark style="background: #FFB8EBA6;">引入了约束（Constraints）的概念</mark>，用于增强权限管理的安全性和灵活性。

常见的约束包括：

1. **互斥角色（Mutually Exclusive Roles）**：
    - 用户不能同时拥有两个互斥的角色。
    - 例如：一个用户不能同时是“会计”和“审计员”。
2. **基数约束（Cardinality Constraints）**：
    - 限制用户拥有的角色数量，或者角色被分配的用户数量。
    - 例如：一个用户最多只能拥有 3 个角色。
3. **先决条件约束（Prerequisite Constraints）**：
    - 用户必须拥有某个角色，才能被分配另一个角色。
    - 例如：用户必须先拥有“初级工程师”角色，才能被分配“高级工程师”角色。
4. **时间约束（Temporal Constraints）**：
    - 限制角色或权限的有效时间。
    - 例如：某个角色只能在工作日使用。

**表结构设计**

1. **用户表（User）**
2. **角色表（Role）**
3. **权限表（Permission）**
4. **用户-角色关联表（User_Role）**
5. **角色-权限关联表（Role_Permission）
6. <mark style="background: #FFB8EBA6;">互斥角色表（MutuallyExclusiveRoles）</mark>：存储互斥角色的关系。
7. <mark style="background: #FFB8EBA6;">基数约束表（CardinalityConstraints）</mark>：存储角色分配的限制条件。
8. <mark style="background: #FFB8EBA6;">先决条件约束表（PrerequisiteConstraints）</mark>：存储角色分配的先决条件。

**图例**
![RBAC2.png](https://img.stuartyang.site/2025/03/be96f8196ae0365b9942f3cee2771871.png)

**数据示例**

互斥角色表（MutuallyExclusiveRoles）

| role_id_1 | role_id_2 | 解释                     |
| --------- | --------- | ---------------------- |
| 1         | 2         | # admin 和 editor 是互斥角色 |

基数约束表（CardinalityConstraints）

| role_id | max_users | max_roles_per_user | 解释                                   |
| ------- | --------- | ------------------ | ------------------------------------ |
| 1       | 10        | 2                  | # admin 角色最多分配给 10 个用户，每个用户最多 2 个角色  |
| 2       | 20        | 3                  | # editor 角色最多分配给 20 个用户，每个用户最多 3 个角色 |

先决条件约束表（PrerequisiteConstraints）

| role_id | prerequisite_role_id | 解释                               |
| ------- | -------------------- | -------------------------------- |
| 2       | 3                    | # 必须先拥有 viewer 角色，才能拥有 editor 角色 |

## RBAC3

> RBAC1+RBAC2 = RBAC3，即在RBAC0的基础上同时<mark style="background: #FFB8EBA6;">引入约束和角色层次的概念</mark>。

![CleanShot 2025-03-20 at 20.33.09@2x.png](https://img.stuartyang.site/2025/03/470acd12d7e24f817376c779c09e93fd.png)


![CleanShot 2025-03-20 at 20.32.57@2x.png](https://img.stuartyang.site/2025/03/6e73b184ba2c19e0d1f7372a9ef5cb8f.png)

# 我们该又如何设计权限系统？

站在RBAC0的基础上，我们要考虑：
1. 是否需要支持角色继承（RBAC1，又或者是否需要支持约束条件（RBAC2）；
2. 考虑用户规模大小，用户角色是否经常变动；
3. 权限的粒度，权限是粗粒度的（如模块级别）还是细粒度的（如按钮级别）；
4. 扩展性：是否需要支持动态添加角色和权限，是否需要支持多租户；


> 多租户模型设计，放在下一篇！
