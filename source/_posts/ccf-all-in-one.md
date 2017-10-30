title: CCF目录单页版
category: [misc]
tags:
date: 2017-02-25

---
CCF目录有[网页版](http://history.ccf.org.cn/sites/ccf/paiming.jsp)和[2015版的PDF](http://history.ccf.org.cn/sites/paiming/2015ccfmulu.pdf)。
前一段时间整理论文列表，感觉分成多页的目录用起来不太方便。于是就粘贴复制，把它们合并到一起。
Markdown排版比较麻烦，于是把表格单独放在一个html文件里了，链接是 **[CCF目录完整列表2017-02-25](/doc/ccf-all-in-one-2017-02-25.html)** 。
此外还有 **[Excel格式](/doc/ccf_all_in_one_2017-02-25.xlsx)**。

CCF网站最近改版了，原来的链接发生了变化(2017-02-25访问，上面的链接已经更新过了)。新版的链接是[CCF目录](http://webtest.ccf.org.cn/xspj/gyml/) ，HTML版不太完整，而且没有提供PDF文件。

<!--more-->

<!-- TOC -->

- [说明](#%E8%AF%B4%E6%98%8E)
- [CCF目录中存在的问题](#ccf%E7%9B%AE%E5%BD%95%E4%B8%AD%E5%AD%98%E5%9C%A8%E7%9A%84%E9%97%AE%E9%A2%98)
    - [同一刊物同时被列入不同方向](#%E5%90%8C%E4%B8%80%E5%88%8A%E7%89%A9%E5%90%8C%E6%97%B6%E8%A2%AB%E5%88%97%E5%85%A5%E4%B8%8D%E5%90%8C%E6%96%B9%E5%90%91)
    - [同一方向中的重复会议](#%E5%90%8C%E4%B8%80%E6%96%B9%E5%90%91%E4%B8%AD%E7%9A%84%E9%87%8D%E5%A4%8D%E4%BC%9A%E8%AE%AE)

<!-- /TOC -->

# 说明
表格中的类别简写分别是:
+ AI：人工智能
+ 系统：计算机体系结构/并行与分布计算/存储系统
+ 软工：软件工程/系统软件/程序设计语言
+ 数据库：数据库/数据检索/内容检索
+ 安全：网络与信息安全
+ 多媒体：计算机图形学与多媒体
+ 网络：计算机网络
+ 理论：计算机科学理论
+ 其它：交叉/综合/新兴
+ 交互：人机交互与普适计算

表格中是按拼音排序的，排序字段依次是：方向，类型（会议/刊物），级别（A/B/C）。

# CCF目录中存在的问题

## 同一刊物同时被列入不同方向
多个研究方向经常出现交叉，所以这种情况也是可以接受的。

+ B类刊物 `DKE: Data and Knowledge Engineering` 分别被列入了 AI 和 数据库 方向
+ C类刊物 `IJIS: International Journal of Intelligent Systems` 分别被列入了 AI 和 数据库 方向
+ C类刊物 `IPL: Information Processing Letters` 分别被列入了 理论 和 数据库 方向
+ B类刊物 `TOMCCAP: ACM Transactions on Multimedia Computing, Communications and Applications` 分别被列入了 网络 和 多媒体 方向

## 同一方向中的重复会议
+ 安全方向的 C类会议 `ASIACCS`和 A类会议 `CCS`给出的链接都是 [http://dblp.org/db/conf/ccs/] ，这个不是错误，因为dblp上确实是在同一页面显示了 `ASIACCS` 和 `CCS`，不过列表中 `ASIACCS`的全称不太恰当
+ 交互方向的 C类会议 `DIS: ACM Conference on Designing Interactive  Systems` 在第2条和第11条重复出现了两次

<img src="/img/ccf_sum.png" alt="刊物分方向汇总">
