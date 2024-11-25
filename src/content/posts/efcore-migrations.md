---
title: EFCore迁移
published: 2024-11-18
description: ''
image: ''
tags: ["EFCore", "CSharp", "教程"]
category: '笔记'
draft: false 
lang: ''
---

## 简述

数据库的每次变更一般称它为迁移

## 安装

首先安装EFCore迁移工具: `dotnet tool install -g dotnet-ef`  
迁移需要安装nuget包: `Microsoft.EntityFrameworkCore.Design`  

## 使用

安装完成后, dotnet-ef就是迁移工具的命令了  
cd到项目目录(.csproj的目录), 否则需要指定 `--project` 参数  
输入 `dotnet-ef migrations add 迁移名` 就可以创建迁移了, 会在目录下生成一个Migrations文件夹, 里边有迁移的快照  
然后在代码中使用 `DbContext.Database.Migrate` 进行迁移, 千万不能使用 `DbContext.Database.EnsureCreated()`, 否则迁移不生效  
可以使用 `dotnet-ef database update` 更新数据库(效果和`DbContext.Database.Migrate`差不多)

每次变更数据库模型后, 都需要使用 `dotnet-ef migrations add 迁移名` 进行迁移  
