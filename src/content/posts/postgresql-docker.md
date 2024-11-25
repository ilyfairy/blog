---
title: Docker中安装PostgreSql
published: 2024-11-18
description: ''
image: ''
tags: ["Docker","笔记","PostgreSql"]
category: '笔记'
draft: false 
lang: ''
---

首先拉取镜像: `docker pull postgres`,  
如果要拉去指定版本的postgresql, 可以加后缀: `docker pull postgres:17`  
如果无法网络连接不良拉取失败, 可以在命令行中使用临时镜像源, 如果镜像源是`dockerpull.org`, 则命令行是: `docker pull dockerpull.org/postgres:17`  
或者在`/etc/docker/daemon.json`中设置镜像源: `{"registry-mirrors": [ "https://dockerpull.org" ]}`

## 指定数据路径

PostgreSql 的数据库默认路径通常是 `/var/lib/postgresql/data`  
通过`PGDATA`环境变量指定容器内的数据库路径: `-e POSTGRES_PASSWORD=/root/pgdata`  

通过-v参数可以将数据库映射到物理机目录: `-v /root/物理机:容器路径`  
也可以映射到docker卷中: `-v pgdata:/root/pgdata`  
如果映射到了docker卷中, 那么它在物理机的路径通常是: `/var/lib/docker/volumes`

## PostgreSql 环境变量

- `POSTGRES_PASSWORD`: 用户密码
- `POSTGRES_USER` – 指定具有超级用户权限的用户和具有相同名称的数据库。Postgres 在为空时使用默认用户。
- `POSTGRES_DB` – 指定数据库的名称，或者在留空时默认为 POSTGRES_USER 值。
- `POSTGRES_INITDB_ARGS` – 将参数发送到 postgres_initdb 并添加功能
- `POSTGRES_INITDB_WALDIR` – 定义 Postgres 事务日志的特定目录。事务是一个操作，通常描述对数据库的更改。
- `POSTGRES_HOST_AUTH_METHOD` – 控制主机连接到所有数据库、用户和地址的身份验证方法
- `PGDATA` – 定义数据库文件的另一个默认位置或子目录

## 完整命令

`docker run -d --name my-postgres -p 5432:5432 -v /home/ubuntu/pgdata:/var/lib/postgresql/data -e POSTGRES_PASSWORD=123456 postgres`

- `--name my-postgres` 容器名
- `-p 5432:5432` 暴露5432端口到物理机
- `-v /home/ubuntu/pgdata:/var/lib/postgresql/data` 映射数据库目录到物理机的 `/home/ubuntu/pgdata`
- `-e POSTGRES_PASSWORD=123456` 设置数据库密码为`123456`
