---
title: Exort Important Tech Stack
date: 2020-05-22 01:18:53
tags: TechStack
categories:
cover: https://i.pinimg.com/474x/69/73/76/6973766435a3a5b50985ba9795c7f8cb.jpg
---
<meta name="referrer" content="no-referrer" />

# Exort Tech Stack Review

## 重新规范工作流程

基于第一周的敏捷开发流程的问题，周末我们根据本组的工作需求，重新梳理了工作流程，之后严格按照现在的工作流程进行。

[![1](https://github.com/exorteam/Exort/raw/master/doc/%E7%AC%AC%E4%BA%8C%E5%91%A8%E5%B7%A5%E4%BD%9C%E6%80%BB%E7%BB%93/images/1.png)](https://github.com/exorteam/Exort/blob/master/doc/第二周工作总结/images/1.png)

[![5](https://github.com/exorteam/Exort/raw/master/doc/%E7%AC%AC%E4%BA%8C%E5%91%A8%E5%B7%A5%E4%BD%9C%E6%80%BB%E7%BB%93/images/5.png)](https://github.com/exorteam/Exort/blob/master/doc/第二周工作总结/images/5.png)

## 组内学习机制

为了提高组内成员，让组员都能够在开发中提高，我们小组通过文档的形式来让各个组员都知道整个开发流程的细节。

[![2](https://github.com/exorteam/Exort/raw/master/doc/%E7%AC%AC%E4%BA%8C%E5%91%A8%E5%B7%A5%E4%BD%9C%E6%80%BB%E7%BB%93/images/2.png)](https://github.com/exorteam/Exort/blob/master/doc/第二周工作总结/images/2.png)

## 加强了审核流程

为了避免代码风格不一致等问题，我们有了严格的审核流程，每次工作都要pr，并且其他组员要进行CodeReview。

[![3](https://github.com/exorteam/Exort/raw/master/doc/%E7%AC%AC%E4%BA%8C%E5%91%A8%E5%B7%A5%E4%BD%9C%E6%80%BB%E7%BB%93/images/3.png)](https://github.com/exorteam/Exort/blob/master/doc/第二周工作总结/images/3.png)

## 非常正规的接口文档

我们组也严格按照k8s的http接口文档书写规范，有了非常正规且易于理解的接口文档。

[![4](https://github.com/exorteam/Exort/raw/master/doc/%E7%AC%AC%E4%BA%8C%E5%91%A8%E5%B7%A5%E4%BD%9C%E6%80%BB%E7%BB%93/images/4.png)](https://github.com/exorteam/Exort/blob/master/doc/第二周工作总结/images/4.png)

## 搭建了项目的公共类的远程仓库

由于微服务需要经常调用其他微服务的类，但是本地无法直接调取，再加上不同微服务也存在向ResponseCode这样的共有格式的类，于是也提取出来到公共类仓库，方便统一格式与调用，不需要本地再写了。

[![1](https://github.com/exorteam/Exort/raw/master/doc/%E7%AC%AC%E4%B8%89%E5%91%A8%E5%B7%A5%E4%BD%9C%E6%80%BB%E7%BB%93/images/1.png)](https://github.com/exorteam/Exort/blob/master/doc/第三周工作总结/images/1.png)

## K8s微服务架构

[![架构图](https://github.com/exorteam/Exort/raw/master/doc/images/architecture.png)](https://github.com/exorteam/Exort/blob/master/doc/images/architecture.png)

## CI / CD 

用之前在服务器上搭的drone做CI CD。