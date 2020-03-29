---
title: "鬼谷子问徒"
layout: post
date: 2018-04-22 22:45
# image: /assets/images/markdown.jpg
headerImage: false
tag:
- puzzles
category: blog
author: junhao
description: N.A.
---

## 题

> 一天鬼谷子出了这道题目：他从2到99中选出两个不同的整数，把积告诉孙，把和告诉庞；庞说：我虽然不能确定这两个数是什么，但是我肯定你也不知道这两个数是什么。孙说：我本来的确不知道，但是听你这么一说，我现在能够确定这两个数字了。庞说：既然你这么说，我现在也知道这两个数字是什么了。请问这两个数字是什么？为什么？

## 解

庞涓确认孙不知道因为相加为他所知的和的所有组合必然都不是质数，否则孙有可能推出。

{% highlight python %}
A = [] # 包含两数和的可能

def check_prime(n):
	if n == 2:
		return True
	i = 2
	while i < n:
		if (n % i) == 0:
			return False
		i = i + 1
	return True

for i in range(5, 198):
	j = 2
	flag = True
	checked = set()
	while j < i - 2 and j not in checked:
		x = j
		y = i - j
		checked.add(x)
		checked.add(y)
		if check_prime(x) and check_prime(y):
			flag = False
			break
		j = j + 1
	if flag:
		A.append(i)
{% endhighlight %}

记录所有积的可能性，并存入list B。

{% highlight python %}
mul = set()
for i in range(2, 100):
	for j in range(2, 100):
		if i != j:
			mul.add(i * j)
B = list(mul)
{% endhighlight %}

孙能够确定因为所有的一对因子只有一对的和是出现过在A里的。

{% highlight python %}
results = [] # 包含积经过第二句话筛选后的可能性

for b in B:
	j = 2
	fac = set()
	counter = 0
	while j < b:
		if (b % j == 0) and (b / j != j) and (j not in fac):
			fac.add(b / j)
			fac.add(j)
			a = j + b / j
			if a in A:
				counter += 1
		j = j + 1
	if counter == 1:
		results.append(b)
{% endhighlight %}

庞也知道了因为所有的一对相加项的积只有一对出现在results里。

{% highlight python %}
f_A = [] # 记录筛选后的和的可能性

for a in A:
	j = 2
	counter = 0
	while j < a - 2:
		x = j
		y = a - j
		b = x * y
		if b in results:
			counter += 1
		j = j + 1
	if counter == 2:
		f_A.append(a)
{% endhighlight %}

接下来找出这两个数

{% highlight python %}
for s in f_A:
	j = 2
	checked = set()
	while j < s - 2 and j not in checked:
		x = j
		y = s - j
		checked.add(x)
		checked.add(y)
		p = x * y
		if p in results:
			print("这两个数为：{0}, {1}".format(x, y))
		j = j + 1
{% endhighlight %}

结果为：

{% highlight bash %}
>$ time python guiguzi.py
 这两个数为：4, 13

 real 0m2.285s
 user 0m2.219s
 sys  0m0.027s
{% endhighlight %}