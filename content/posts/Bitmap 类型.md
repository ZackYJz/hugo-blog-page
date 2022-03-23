---
title: "Redis/Bitmap类型使用及原理"
date: 2021-02-01T13:31:23+08:00
draft: false
disableToC: true
featured_image: "https://images.unsplash.com/photo-1511406294398-718e9b6f85bc?ixlib=rb-1.2.1&ixid=MnwxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8&auto=format&fit=crop&w=2070&q=80"
tags: []
categories: Redis
description: 介绍 Bitmap 类型基本命令和使用场景
---

> BitMap 也叫位图
>
> 是一种使用一个字节的 8 位 bit 的 01 状态来统计二值状态的数据类型

- 二值统计：集合元素的取值就只有0和1两种，通常用于状态的统计

  例如：登录情况、打卡情况、日活统计等

## 原理

本质是由  0 和 1 两种状态表现组成的二进制位的bit数组，但底层使用 Redis/String 类型实现

- 存储时，实际存储的是一个 8bit bitmap 代表的 ASCII 码所对应的字符

- 使用 get 指令可以获得这个 二进制位对应的 ASCII 码符号

  <img src="https://logseq.oss-cn-chengdu.aliyuncs.com/noteImg/202203221430078.png" style="zoom:30%;" />

## 基本指令

- `setbit key offset value`
  - offset 位偏移，每八位一个byte 作为 bitmap,自动扩容
  - value 值只能是 0 或 1
    	<img src="https://logseq.oss-cn-chengdu.aliyuncs.com/noteImg/202203221412735.png" style="zoom:25%;" />

- `getbit key offset`

- `strlen`
  - 统计共占有的字节数(byte)
  - 长度不是字符串长度，而是byte 数
  - 超过 8 位后，就按 8 位扩容

- `bitcount key [start] [end]`
  - 统计这个键里面的位 1 的个数

- `bitop [operation] destkey key`
	- 对多个 key 的二进制数据进行位运算
	- [operation] 可取的二进制操作符 ：AND、OR、NOT、XOR

## 场景案例
#### 记录用户月签到数据
- 记录九月第一天签到 `setbit checkin:uid63:202109 0 1`
- 查看九月第一天签到情况 `getbit checkin:uid63:202109 0`

#### 统计日活月活
- 对每个用户 id 做一个全局的 id-位置映射
- 位置为 1 的用户当天访问过: `setbit 20210101 1 1`
- 当天日活：`BITCOUNT 20210101`
- 连续登录的用户位：`BITOP and destkey 202101015 202101016`