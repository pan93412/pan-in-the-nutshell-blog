---
title: 109 高中資訊學科能力複賽的參加心得
tags:
  - "109"
  - algorighm
  - competition
  - high
  - idle
  - information
  - school
  - senior
  - TOI
  - visual studio
categories: The thoughts
date: 2020-11-02 08:57:07
lastmod: 2020-11-02 08:57:07
robotsNoIndex: true
---

**又墊底了。**

但不一樣的是，**這次比賽我收穫上不少**。

## 早先的選手培訓課程

**這是我覺得整個競賽當中，我最印象深刻而且最有收穫的課程。**

在參加這門課程之前，其實我很厭惡演算法。因為數學跟邏輯不太好，所以我當時會覺得演算法很恐怖，看不懂，不敢學。但在上完這幾門課程後，我才發現到，**很多演算法其實自己平時就在用，只是從未察覺。**

就例如動態規劃 (Dynamic Programming) 好了，概念這就像是我平常在搞得快取 (Cache)，把先前擷取 / 計算過的結果儲存下來，下次要用的時候，就直接把之前擷取 / 計算的結果拿出來。

```plain
hashmap cache_db;
get_something(id) {
  declare data;
  if id not in cache_db
    data = _get_something(id)
    cache_db[id] = data
  else
    data = cache_db[id]
  return data
}
```

還有圖論，這對我來講是個很新穎的演算法概念，而在聽邏輯之後覺得比想像中還要好懂。「廣度優先」和「深入優先」這兩個抽象的名詞，用老師的 GIF 講解之後就變得很好懂。但就是還不知道能用在哪些地方 😅

最後是逐個擊破（分而治，D&C）和貪婪演算法 (Greedy Method)，雖然一開始沒聽懂，但後來自己回家找了 Wikipedia 之後就看懂了。也是個自己寫程式偶爾會想到，但沒發覺是個演算法的東西。

![培訓課程的筆記 - 1](https://assets.blog.pan93.com/toi-competition-opitions/20201115_115415.webp)
![培訓課程的筆記 - 2](https://assets.blog.pan93.com/toi-competition-opitions/20201115_115345.webp)
![培訓課程的筆記 - 3](https://assets.blog.pan93.com/toi-competition-opitions/20201115_115415.webp)
![培訓課程的筆記 - 4](https://assets.blog.pan93.com/toi-competition-opitions/20201115_115448.webp)

培訓課程的筆記

## 比賽的過程與收穫

![ya](https://assets.blog.pan93.com/toi-competition-opitions/INFOP1.webp)

雖然跟上次一樣，看不懂題目，或是看懂卻不會做，但其實我還是有一些額外的收穫。

先說為何我「看懂卻不會做」，就例如「給三個點，求圍起來的三角形面積」。我知道三個點圍起是直角三角形的作法，但我不知道不規則三角形的解法。

後來聽了講解之後，才知道要用的是海龍公式。真是遺憾，競賽時不能用網路，所以我沒辦法上網查「三個點求三角形面積」的公式，而且我也忘了海龍公式。只能說是數學基礎不夠好。

還有下面這種「數字包數字」的題目，這題我至今還不會做，希望未來能看到能看懂的解法：

```plain
7 4444444
6 4333334
5 4322234
4 4321234
3 4322234
2 4333334
1 4444444
0 1234567
```

這一題中，N=4，而我們會給四個數字，分別為 x1 y1 和 x2 y2。求 (x1, y1) 和 (x2, y2) 的值

假設 N=4，而 (x1, y1) = (3,2) = n1，(x2, y2) = (5,3) = n2，那就應該回傳 `3 2` 分別指代 n1，n2 的值。

好像是這樣吧，我忘記了。

另外還有極大數字的題目和剪出時間的題目。這兩道題目我都有點忘了，找到考古題之後會再補上去。

雖然比賽 4hr 我大概 3hr 都不知道在幹嘛，但我在這 3hr 得出一個結論：

> 打競賽程式不要用 Visual Studio，有夠綁手綁腳。用 Python 內建的 IDLE 就好!
>
> Yi-Jyun Pan, 2020-11-14.

和

> 競賽用電腦好快ㄛ，開 VS 不用 10 秒耶。
>
> Yi-Jyun Pan, 2020-11-14.

以及我也已經大概擬定我自己是比較傾向於程式應用，也就是寫網頁、專案之類的東西，正是如此，我並沒有因為這次比賽的排名而傷透心，這就當作是一次嘗試吧（笑）
