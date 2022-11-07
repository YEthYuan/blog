---
title: 编译原理LL(1)文法总结
date: 2021-12-21 13:03:32
tags: [CN,编译原理,知识整理]
categories: Compiler
toc: true
mathjax: true
---

# 编译原理-LL(1)文法总结

---

## 1 First集求法总结

两种情况：

1. **A->aB**：以终结符开头，直接把这个终结符放到A的First里

2. **A->CD**：以非终结符开头， 先把C的First放到A的First里(要谨记ε的情况)

   再看如果C的First中有ε的话就把D的First放到A的First里，如果D也有ε的话往后依次类推，直到ε本身就在first（A）中

---

## 2 Follow集求法总结

先在候选式（右边）中找到该非终结符，如L（注意例中只有一个定义，但找Follow要看到所有右边出现该非终结符的）

算法：

- 对于文法G[S]，首先将右端结束标记 `$` 放到 FOLLOW(S) 中

- 按照下面两个规则**不断迭代**，直到所有的FOLLOW集合都不再增长为止
  - 如果存在产生式`A -> αBβ` ，那么 FIRST(β)中所有非ε的符号都在FOLLOW(B)中；
  - 如果存在产生式`A -> αB`，或者`A -> αBβ` 且FIRST(β)包含ε，那么FOLLOW(A)中的所有符号都加入到FOLLOW(B)中
