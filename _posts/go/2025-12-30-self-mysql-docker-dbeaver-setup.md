---
title: MySQL 数据库准备（Docker + DBeaver）
tags: mysql docker dbeaver database
permalink: /go/mysql-docker-dbeaver-setup
---

在搭建后端服务的过程中，**数据库准备往往是第一步，也是最容易被忽略的一步**。这次我在本地使用 **Docker 启动 MySQL**，通过 **初始化脚本自动完成建库和建用户**，并使用 **DBeaver** 进行可视化连接和管理。整体流程不复杂，但中间踩到的坑都非常典型，值得完整记录下来，方便以后快速复用。

<!--more-->

## 一、为什么选择 Docker + MySQL

我的核心诉求其实很简单：

- 本地环境可以 **快速搭建、快速销毁**
- 不污染宿主机环境
- 数据需要 **持久化**
- 初始化流程尽量 **自动化、标准化**

在这种前提下，Docker 几乎是最优解；而 MySQL 作为最常见的关系型数据库之一，也非常适合作为后端服务的基础组件。

## 二、使用 Docker 启动 MySQL 容器

我使用的是官方 `mysql` 镜像，通过一条 `docker run` 命令完成容器启动、端口映射、数据目录挂载以及初始化脚本挂载。

```bash
docker run -d \
  --name mysql \
  -p 3306:3306 \
  -v /path/to/mysql/data:/var/lib/mysql \
  -v /path/to/mysql/init:/docker-entrypoint-initdb.d \
  -e MYSQL_ROOT_PASSWORD=******** \
  mysql
```

### 关键参数说明
- `-p 3306:3306`：将容器内 MySQL 的 3306 端口映射到宿主机，方便本地工具或服务连接。
- `-v /var/lib/mysql`：将 MySQL 的数据目录挂载到宿主机，实现数据持久化。否则一旦容器删除，数据也会一并丢失。
- `-v /docker-entrypoint-initdb.d`：MySQL 官方镜像支持在首次初始化时自动执行该目录下的 SQL 脚本，这是后面实现自动建库、建用户的关键。
- `MYSQL_ROOT_PASSWORD`：设置 root 用户密码（只在数据目录为空时生效）。

## 三、数据库初始化脚本设计

为了避免每次手动创建数据库和用户，我在项目中准备了初始化 SQL 脚本。

```sql
-- 01_db_user.sql

-- 创建数据库（如果不存在）
CREATE DATABASE IF NOT EXISTS name_db
  DEFAULT CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;

-- 创建业务用户
CREATE USER IF NOT EXISTS 'user_name'@'%' IDENTIFIED BY '********';

-- 授权该用户访问指定数据库
GRANT ALL PRIVILEGES ON name_db.* TO 'user_name'@'%';

-- 刷新权限
FLUSH PRIVILEGES;
```

### 我的设计思路
- root 用户只用于初始化和运维
- 业务服务使用独立账号
- 权限控制在指定数据库范围内
- 用户 host 使用 %，适配容器化环境

这种方式在后续迁移到测试或生产环境时，几乎可以原样复用。

## 四、初始化脚本的一个重要坑

这里有一个非常容易踩的坑：

/docker-entrypoint-initdb.d 中的脚本只会在 MySQL 第一次初始化时执行一次

也就是说：
- 如果 /var/lib/mysql 已经有数据
- 再修改 SQL 脚本是不会重新执行的

如果需要重新初始化数据库，必须：
```bash
docker stop mysql
docker rm mysql
# 必要时清空数据目录
```
这个机制在调试阶段一定要牢记，否则很容易误以为 SQL 没生效。

## 五、使用 DBeaver 连接数据库

在数据库可视化工具上，我最终选择了 DBeaver，原因也很直接：
- 表结构、索引、数据展示清晰
- SQL 执行体验好
- 连接配置灵活
- 跨平台体验一致

相比之下，VS Code 的数据库插件更偏向轻量级场景，而 DBeaver 更适合作为长期使用的数据库管理工具。

## 六、MySQL 8 + DBeaver 的经典问题

第一次使用 DBeaver 连接时，我遇到了一个非常常见的错误：
```
Public Key Retrieval is not allowed
```
这是 MySQL 8 默认认证方式（caching_sha2_password）与 JDBC 安全策略之间的冲突。

### 我的解决方式
在 DBeaver 的连接配置中添加驱动参数：
```ini
allowPublicKeyRetrieval = true
useSSL = false
```
添加后，连接流程可以正常进行。

## 七、关于 Docker 网络和用户 host 的理解

在排查连接问题时，我注意到一个细节：
```
Access denied for user 'nothing'@'192.168.xx.x'
```
这让我再次确认了一点：
- 在 Docker Desktop 环境中，MySQL 看到的客户端并不是 localhost 而是 Docker 的网关地址。

因此：
```sql
'nothing'@'%'
```
比：
```sql
'nothing'@'localhost'
```
更加适合容器化和本地开发环境。

## 八、最终稳定可用的连接配置

在所有问题解决后，我的本地连接配置如下：
- Host：127.0.0.1
- Port：3306
- Database：name_db
- User：user_name
- Driver properties：
  - allowPublicKeyRetrieval = true
  - useSSL = false

这套配置在本地开发环境下运行稳定，没有再遇到连接问题。

## 九、总结

这次 MySQL 数据库准备过程，让我再次确认了一些基础但非常重要的经验：
- Docker 化数据库是后端开发的基本功
- 初始化脚本要理解「只执行一次」的机制
- MySQL 8 的认证方式需要额外注意
- 合适的数据库工具可以极大提升效率

把这些基础设施提前理顺，后续写业务代码时会省下很多心力。这套流程经验也可以为后续引入 Docker Compose、Redis、消息队列等组件打下基础。
