---
layout: post
title: Kotlin: tailrec 修饰符优化尾递归
description: Kotlin: tailrec 修饰符优化尾递归
categories: [kotlin]
keywords: kotlin, tailrec
---

函数中所有递归形式的调用都出现在函数的末尾的情形称为尾递归。
<!-- more -->
Kotlin 支持尾递归，这允许一些通常用循环写的算法改用递归函数来写，而无堆栈溢出的风险。 当一个函数用 tailrec 修饰符标记并满足所需的形式时，编译器会优化该递归，留下一个快速而高效的基于循环的版本。

## 实例：单链表的查找

{% highlight kotlin %}

data class Node(val value: Int, var next: Node? = null)

tailrec fun searchNode(startNode: Node?, value: Int): Node? {
    startNode ?: return null
    if (startNode.value == value) return startNode
    return searchNode(startNode.next, value)
}

fun main(args: Array<String>) {
    val NODE_COUNT = 1000000
    val head = Node(0)
    var node = head
    for (i in 0 until NODE_COUNT) {
        node.next = Node(i + 1)
        node = node.next!!
    }
    println(searchNode(head, 123456)?.value)
}

{% endhighlight %}

## 官方文档实例

计算余弦的不动点（fixpoint of cosine），这是一个数学常数。 它只是重复地从 1.0 开始调用 Math.cos，直到结果不再改变，产生 0.7390851332151607 的结果。
{% highlight kotlin %}
tailrec fun findFixPoint(x: Double = 1.0): Double
        = if (x == Math.cos(x)) x else findFixPoint(Math.cos(x))
{% endhighlight %}

最终代码相当于这种更传统风格的代码：
{% highlight kotlin %}
private fun findFixPoint(): Double {
    var x = 1.0
    while (true) {
        val y = Math.cos(x)
        if (x == y) return x
        x = y
    }
}
{% endhighlight %}

