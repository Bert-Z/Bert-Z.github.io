---
title: Exponentially weighted moving average
date: 2020-11-27 09:23:48
tags: [Streaming,Algorithm]
categories:
cover: https://miro.medium.com/max/60/1*0YEXsYsYrHVE99iVmqmMPQ.png?q=20
---
<meta name="referrer" content="no-referrer" />

# Exponentially weighted moving average

This algorithm is one of the most important algorithms currently in usage. From financial time series, signal processing to neural networks , it is being used quite extensively. Basically any data that is in a **sequence**.

**We mostly use this algorithm to reduce the noise in noisy time-series data.** The term we use for this is called “**smoothing**” the data.

The way we achieve this is by essentially **weighing the number of observations and using their average.** This is called as **Moving Average.**

This method is **effective** however, it is not very easy to do as it involves holding the past values in memory buffer and constantly updating the buffer whenever a new observation is read. Here is where **EWMA** comes into picture.

EWMA solves this problem with a recursive formula.

![Image for post](https://miro.medium.com/max/60/1*0YEXsYsYrHVE99iVmqmMPQ.png?q=20)

![Image for post](https://miro.medium.com/max/1320/1*0YEXsYsYrHVE99iVmqmMPQ.png)

fig. EWMA formula

**Formula Explanation :**

The formula states that the value of the moving average(**S**) at time **t** is a mix between the value of raw signal(**x**) at time **t** and the previous value of the moving average itself i.e. **t-1**. The degree of mixing is controlled by the parameter **a** (**value between 0–1**)**.**

So, if **a = 10%(small),** most of the contribution will come from the **previous value** of the signal. In this case, “**smoothing**” will be **very strong**.

if **a = 90%(large),** most of the contribution will come from the **current value** of the signal. In this case, “**smoothing**” will be **minimum**.

So for a better understanding, we will consider an example of “temperature variation on the weekdays of a week” data.

![Image for post](https://miro.medium.com/freeze/max/60/1*fHfW4mJJhslB5G2_a9EMwA.gif?q=20)

![Image for post](https://miro.medium.com/max/1032/1*fHfW4mJJhslB5G2_a9EMwA.gif)

fig. Temp vs days of the week

Now as you can see, we need to “**smoothen**” out the above graph to have consistency in data.

![Image for post](https://miro.medium.com/max/60/1*IL8hOsSH-lIGOC7e4joZ6g.png?q=20)

![Image for post](https://miro.medium.com/max/1916/1*IL8hOsSH-lIGOC7e4joZ6g.png)

fig. The data for temp. vs weekdays

Now let us take a = 10%, considering the new formula will be:

![Image for post](https://miro.medium.com/max/60/1*PE142U0T5AdG_t0QmIPzGQ.png?q=20)

![Image for post](https://miro.medium.com/max/1320/1*PE142U0T5AdG_t0QmIPzGQ.png)

fig. EWMA formula for a=10%

![Image for post](https://miro.medium.com/max/60/1*elAoh7SVmQpsXk6NKBfF8w.png?q=20)

![Image for post](https://miro.medium.com/max/1288/1*elAoh7SVmQpsXk6NKBfF8w.png)

fig. calculation of EWMA

![Image for post](https://miro.medium.com/max/60/1*3CB9BDosSC5loq-4cBs6CA.png?q=20)

![Image for post](https://miro.medium.com/max/2468/1*3CB9BDosSC5loq-4cBs6CA.png)

fig. Original graph vs EWMA

As we can see, a =10% provides really strong smoothing.

To summarize, we have introduced EWMA and have solved a sample dataset to see how time series/sequential data is smoothed out for use in various algorithms.