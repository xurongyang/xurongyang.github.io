---
title: Go总结
date: 2019-01-28 17:30:03
tags:
---

优点：
1.生成直接可执行的二进制文件
2.gofmt
3.内置的channel（消息队列）
4.构造函数很灵活
5.加强版for循环

缺点：
1.莫名其妙的类型系统（interface vs Java的Object基类和interface）
2.奇怪的数组定义（[5]string）
3.自动绑定接口实现（不知道实现了哪个接口）
4.错误处理和泛型（已被大家所熟知）