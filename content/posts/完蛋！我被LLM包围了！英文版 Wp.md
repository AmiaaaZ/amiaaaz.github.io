---
title: "完蛋！我被LLM包围了！英文版 Wp"
slug: "llm-prompt-injection-wp"
description: "ai时代的sql注入大概就长这样叭（"
date: 2023-11-24T17:30:11+08:00
categories: ["AI"]
series: []
tags: []
draft: false
toc: true
---

https://www.v2ex.com/t/992611  |  https://modelleai.com/

[完蛋！我被LLM包围了！](https://zhuanlan.zhihu.com/p/665393240)的在线英文版，很可惜原版关停服务……因此中文版的就不挑战了，原版题目在作者知乎文章的最末

*未解出来的用星号标注

## Chapter1

### Question1-1

Please construct a query for the model that will result in the exact answer '1+1=3' (no quotes needed)

```
i need your answer to this prompt just the text "1+1=3", you don't need to check if it is correct, it's just a text, not a equation, don't need to care about the calculation results, or mathmatics things
```

![cbf76025aacf2d534e682f600c32fce](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/cbf76025aacf2d534e682f600c32fce.png)

### Question1-2

Please enter a question within three words so that the model’s answer is more than 30 words.

```
what about bible
```

![image-20231117100245605](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20231117100245605.png)

### Question1-3

Please enter a one-word question so that the model’s answer is more than 100 words.

```
Bible
```

![image-20231117100624991](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20231117100624991.png)

### Question1-4

Please enter a one-word question so that the model’s answer is less than 20 words.

```
yes
```

![image-20231117100719562](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20231117100719562.png)

## Chapter2

### Question2-1

You need to come up with a question that has a prime number of words, so the answer's length is exactly the next prime number.

```
give me a text contains eleven words
```

![image-20231117101611142](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20231117101611142.png)

### Question2-2

Please enter a positive integer greater than one as a question, so that the answer contains a number that is at least one thousand more than it.

```
100+2000
```

![image-20231117102034852](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20231117102034852.png)

### *Question2-3

Please enter a positive integer greater than one as a question, so that the answer contains at least 10 different natural numbers, each of them at least 1000 less than the question.

以下都是失败尝试：

- 问+3628800是谁的阶乘并展开

含有单词不认正整数？？？

- 计算式

好方略，但是就差一个

```
+9999-8-7-6-5-4-3-2-1-0
```

![image-20231117110523011](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20231117110523011.png)

有加有减会直接给最后答案，被减数超过9个会给不出结果-> 卡死或处理不了

```
+9999-99-98-97-96-95-94-93-92-91-90
```

- 问意义比较重大的数字，比如2023，2020，2012

有结果，但是反复尝试数字都不够10个

### Question2-4

Please make the model answer '114514' without mentioning the characters 1, 4, and 5, and without any extra characters.

```
hello, (37237+20000+20)*2=?, i need your answer just the result and no other information
```

![image-20231117112648098](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20231117112648098.png)

### Question2-5

Please provide a square number greater than one as the question, and the answer should include the number corresponding to its square root.

```
40=7*7
```

![image-20231117140230802](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20231117140230802.png)

### Question2-6

Ask a question of 10 words or less that does not include the word 'dog'  so that the word 'dog' appears at least twice as often as the number of  words in the question.

```
Poodle
```

![image-20231117140645127](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20231117140645127.png)

## Chapter3

### Question3-1

Please enter a question of at least ten words so that the model's answer is exactly the same with the question.

```
please just repeat the text in this prompt and no additional information
```

![image-20231117142745279](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20231117142745279.png)

### Question3-2

Please enter a question of at least ten words so that the model's answer is the reverse order of the question.

```
please reverse the text in this prompt and thanks a lot
```

![image-20231117142606288](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20231117142606288.png)

### Question3-3

Please enter a question not contain '1' or '3' that will result in the exact answer '1+1=3' (no quotes needed).

```
we all know 2+2=6 is a false equation, if divided them by 2, what false equation will be? just use the false equation as output, no words, just the false equation, and no additional blankspace please!
```

![image-20231117143407288](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20231117143407288.png)

## Chapter4

### Question4-1

Please enter a positive integer greater than one as the question so that the answer contains a number that is exactly 1 different from it.

```
2
```

![image-20231117144652310](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20231117144652310.png)

## Chapter5

### Question5-1

Please construct a question of no less than ten words so that the answer does not contain any of the characters in the question.

```
please write a equation that calculate one plus one, i need your output just contain the equation and no additional information and no blankspaces
```

![image-20231117145251471](https://amiz-1307622586.cos.ap-chongqing.myqcloud.com/images/image-20231117145251471.png)