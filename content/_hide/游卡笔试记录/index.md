---
title: 游卡笔试记录
slug: "interview-youka-written-test"
description: "第一次笔试，心理压力较大。校园招聘看到的春招岗位，游戏服务器开发。我开始问有没有go的岗，hr说有(我在boss没看到)，随即加了微信，很快被捞了"
date: 2024-03-28
preview: ""
draft: true
tags: ["面试","笔试","游卡"]
categories: ["面试记录"]
---

笔试23道题，15单选，5多选，3算法，时间一个小时半，就是牛客网的笔试系统。

选择题我做的有点懵，有的知识已经忘了，还有一些没涉及过的领域题。有一些操作系统的计算题，cpu轮转还有LRU算法算缺页数的，梦回408，选择做了好久，应该快40分钟。。
说几道记得的：

- 匹配空行的正则表达式(实则是选出错误的正则表达式，正则都是查的 蒙了)
- tcp的ack和syn会放在一个包中吗(可以，握手第二次，ack和fin不可以）
- 一块一瓶水 三个瓶子换一瓶水 十块钱可以喝几瓶
- Raid10相比raid5的优势。(更安全，但效率低，成本高)
- 不属于应用层的网络攻击(DDOS)

三道编程题a了两道，最后时间不够了，第三题贪心都要出来了，哎，疏于练习
我全是用go写的

## 第一题：给定一个二维数组 合并数组所有重叠部分

应该是leecode的2580，开始我还理解错了，解了半天

```go
func mergeArrays(arr [][]int) [][]int {
 if len(arr) == 0 {
  return nil
 }

 // 对二维数组按照起始值进行排序
 sort.Slice(arr, func(i, j int) bool {
  return arr[i][0] < arr[j][0]
 })

 result := make([][]int, 0)
 current := arr[0]

 for i := 1; i < len(arr); i++ {
  next := arr[i]
  // 如果当前区间的结束值大于等于下一个区间的起始值，则合并区间
  if current[1] >= next[0] {
   current[1] = max(current[1], next[1]) // 更新当前区间的结束值
  } else {
   result = append(result, current) // 将当前区间加入结果
   current = next                   // 更新当前区间为下一个区间
  }
 }

 result = append(result, current) // 将最后一个区间加入结果

 return result
}
```

## 第二题： 输入字串符，检测是ipv4还是ipv6，都不是返回neither

用了go的net包，这题还是挺轻松的，

```go
func checkIP(input string) string {
 ip := net.ParseIP(input)
 if ip == nil {
  return "Neither"
 }

 if ip.To4() != nil {
  return "IPv4"
 } else if ip.To16() != nil {
  return "IPv6"
 }

 return "Neither"
}
```

第三题：一个地方连续n天长不同数量西瓜 西瓜会过期 一天吃一个 没过期的可以以后吃 给两个数组 西瓜生成数和过期天数 输出可以吃到多少个西瓜
可以直接贪心。

```go
func melons(watermelon []int, expire []int) int {
 // 记录每一天可用的西瓜数量
 available := make([]int, len(watermelon))
 // 记录已吃的西瓜数量
 eaten := 0

 for i := 0; i < len(watermelon); i++ {
  // 计算第 i 天可以吃的西瓜数量
  canEat := min(watermelon[i], available[i])
  eaten += canEat
  // 更新可用数量数组
  for j := i + 1; j < min(i+expire[i]+1, len(available)); j++ {
   available[j] += watermelon[i]
  }
 }

 return eaten
}
```

破防了，这么简单没写出来。。应该一个case没过，差一点，没时间了
