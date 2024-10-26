---
title: 漫画图解 AWS KMS 与 AWS Secret Manager 的集成
date: 2023-09-19 00:00:44
updated: 2023-09-19 00:24:30
excerpt: 用有趣的漫画图解了二者的集成
categories: tech
math: true
tags: 
  - AWS
  - 安全
---

## 前言

在 AWS 中，用户时常容易混淆「AWS KMS」与「AWS Secrets Manager」—— 不仅是从名字上容易混淆，而且在实际使用的过程中，二者在<u>某些场景中</u>也存在一定的可互换性。

从顶层的设计出发点来说，KMS 是一种允许用户管理加密密钥的服务，用于加密、解密、签名和其他操作。

而「Secrets Manager」则更像是一种用来存储密码和 API 密钥等高等级数据的数据库。有了它，我们就可以避免应用程序中进行硬编码 API Key，也可以无需自行存储用户密码（除非你的团队有极强的专业水准和工程能力，否则自己维护一个包含了用户账号密码的数据库是极其困难且不安全的，而 AWS 提供的加密服务甚至武装到了硬件）。

与此同时，为了更好地保护机密数据，AWS Secrets Manager 也可以配合使用 KMS 对其内容进行加密来获取更高的安全性。

下面用一则漫画来演示二者集成的大致工作流程：

![漫画图解](/img/comic_kms/comic.png)

## 后话

在上面的漫画中，忽略了一个重要问题——谁有权限从 Secrets Manager 中获取解密的明文数据？

AWS 根据使用的密钥管理方式都实现了类似的授权机制：

1. 用户基于 KMS 自动托管的加密密钥可以使用 KMS 来管理授权范围，需要配合 IAM 系统在 AWS 平台上进行配置，客户端通过 AWS Credential 来表明自己的身份。
2. 用户基于自行管理的 KMS 密钥来加密的数据也可以使用类似的方式在平台上配置 Policy 和 IAM 等来实现鉴权，区别在于自行管理的 KMS 密钥的 id 是自定义的，需要自己找到对应密钥的配置进行更改。

综上所述，通过对二者的观察，可以看到一个比较明显的区别在于——Secrets Manager可以获取到存进去的数据的明文，而 KMS 只是提供了加解密等比较原子的功能，KMS 是无论如何都不会将设置进去的密钥返回出来的，它所做的只是在安全的黑盒中使用你最初配置的密钥对你的输入进行加解密操作（当然还有一些额外的功能，例如签名和验签等，这里不再赘述）。

本文只是大致介绍了两者的集成流程，并比较了二者的区别，有兴趣的读者可以阅读下面的参考资料和 AWS 的官方文档进行查阅。

## 参考资料

[Understanding the Integration Between KMS and Secrets Manager on AWS](https://blog.lightspin.io/understanding-the-integration-between-kms-and-secrets-manager-on-aws)
