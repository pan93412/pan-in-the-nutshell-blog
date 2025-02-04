+++
date = '2025-02-04T23:26:34+08:00'
title = '聊聊微服務是什麼 – 從單體服務進行擴縮'
tags = ['microservice', 'backend', 'SITCON 2025']
categories = ['Developments']
+++

今年我 SITCON 有投一篇微服務的議程：[「選課卡成狗？微服務架構帶你翻轉校園系統」](https://sitcon.org/2025/agenda/a03517/)。在 SITCON 之前，我打算每天在 Blog 上寫一篇和 Cloud Native 相關的短文，來當作議程的前導內容。當然針對每一篇短文的意見回饋（看不懂也是一種意見反饋 🥺），最終都有助於我產出更好的議程內容～

這個系列的第一個主題，就先從微服務這個主軸開始吧！但在這之前我們或許可以先聊聊「**單體架構可能會遇到什麼樣的瓶頸**」，以及我們要怎麼使用一些技術來稍微改善你的單體系統。

> 考慮到 Blog 本身不是一個很好的互動平台，我在每篇文章的底下都會留「**💬 互動區塊**」，連結到和這篇文章相關的社交媒體上。你可以在社交媒體上和這篇文章互動～

## 單體服務的瓶頸

假設你今天要開發一套用比序來選課的系統，你或許會像下圖這樣來設計你的系統。其中後端就是一個很大的應用程式，前端去呼叫後端來選課。

![Monolithic System](https://assets.blog.pan93.com/microservice-monolithic-scaling/monolithic-system.png)

乍看之下是個挺合理的設計吧？但我們設想一種情況：假如現在已經到了選課的尾聲，大家都會想要看看「自己選的課是否可以上得了」，所以「選課人數」Endpoint 的請求量也會隨時增加，然後系統的 CPU 資源就被「選課人數 Endpoint」吃光了。因為登入、選課、課程名單、志願序等等的 Endpoint 也在同個系統上，所以你的選課系統除了前端的部分會全部掛掉，學生們準備在 Dcard 上罵爆你的系統了。

![Monolithic on High Loading](https://assets.blog.pan93.com/microservice-monolithic-scaling/monolithic-on-high-load.png)

## 擴縮單體服務

此時你會想到：那我們多開好幾套單體的後端 (replicas)，然後前端隨機選擇 API（也就是所謂的「負載平衡」）呢？其實確實是個可行的方案，不過你首先要讓你的後端變成無狀態的 (Stateless) 的。「無狀態」是什麼概念呢？就以下圖來說，我們無論選到哪個 instance 的 Endpoint，呼叫結果都應該要是一致的。換言之，你的後端不可以儲存只有這個 instance 知道的東西，也就是所謂的「狀態」。當然取決於你的設計，**你可能多少會不小心存一些狀態在後端裡面（比如登入的 session 以及 lock 檔案⋯⋯）**，所以你可能會需要花點時間重構這些邏輯，讓這堆狀態不要跟後端放在一起（或甚至變成不用儲存狀態也能判斷的東西，比如 JWT）。

> 【🤔 想想看！】哪些東西可能會導致一個服務變成有狀態的（Stateful）？

![Load balancing for Monolithic System](https://assets.blog.pan93.com/microservice-monolithic-scaling/load-balancing-for-monolithic-system.png)

## 單體服務的擴縮問題

在完成相關的重構後，就算其中一個 instance 有著很大的負載，其他 instance 也能有效的分散掉請求，讓系統不至於完全停擺。不過這種方法粒度或許還是太大了——我們只有 1 個 endpoint 遇到瓶頸，但卻需要因此開出 5 個（甚至更多）完整的後端 instances，資源用量上會不會變得太多；而且所有 instances 最終還是連到一台資料庫上，遇到大量讀取、寫入的場景可能還是會 lag。如果我們用微服務、分散式系統的邏輯重新規劃後端，我們有沒有機會解決掉這個問題？

> 【🤔 想想看！】在不重構後端的前提下，有什麼辦法降低資料庫的負載呢？

## 💬 互動區塊

- Threads: <https://www.threads.net/@pan93412/post/DFnYWwbSQmV?xmt=AQGzOrH5bfvl9_sAcoKIHT3o3nwhyNpMfe6AJybnItrFAg>
- Telegram: <https://t.me/bystartw_chan/320>
