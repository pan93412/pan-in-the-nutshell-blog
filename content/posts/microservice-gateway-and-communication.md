+++
date = '2025-02-05T22:29:00+08:00'
title = '聊聊微服務是什麼 – 閘道和服務通訊基礎'
tags = ['microservice', 'backend', 'SITCON 2025']
categories = ['Developments']
+++

今年我 SITCON 有投一篇微服務的議程：[「選課卡成狗？微服務架構帶你翻轉校園系統」](https://sitcon.org/2025/agenda/a03517/)。在 SITCON 之前，我打算每天在 Blog 上寫一篇和 Cloud Native 相關的短文，來當作議程的前導內容。當然針對每一篇短文的意見回饋（看不懂也是一種意見反饋 🥺），最終都有助於我產出更好的議程內容～

這個 Tag 的第三個主題，就來補充我們昨天微服務中一直沒有提到的部分：「Gateway」吧！我們昨天聊了服務的拆法（領域拆分），以及 RPC 分查詢和更動的目的（CQRS），但是你會發現到有個服務長得和其他服務不太一樣：Gateway。Gateway 沒有 RPC 方法，而且是唯一一個連接到前端的服務。為什麼需要 Gateway，以及 Gateway 是怎麼和其他服務通訊的呢？這篇文章就來詳細解釋這個問題。最後，我們也會簡單說明微服務之間的通訊流程。

![current microservice design](https://assets.blog.pan93.com/microservice-gateway-and-communication/current-microservice.png)

> 考慮到 Blog 本身不是一個很好的互動平台，我在每篇文章的底下都會留「**💬 互動區塊**」，連結到和這篇文章相關的社交媒體上。你可以在社交媒體上和這篇文章互動～

## 什麼是 Gateway？

「Gateway」（閘道）， 基本上就是你那些微服務的前台，負責給前端一組好用、封裝自 RPC 的 REST 或 GraphQL API，同時在轉送給微服務前進行正確的權限檢查（比如說「加入課程」只有管理員能執行）。這樣一來，前端就不需要知道所有微服務的位置、了解服務切分的細節，只需要和 Gateway 進行互動就好。

## 拆出 Gateway 的意義

你或許會疑惑「為什麼我們不要一步到位，讓前端直接連線這些微服務呢？」首先我們先看看讓前端連線微服務會發生什麼事情：

![microservices without gateway](https://assets.blog.pan93.com/microservice-gateway-and-communication/services-without-gateway.png)

現在前端會直接連線到這些微服務上。假如沒有 Gateway，如你所見，「認證」和「授權」的重責大任就落在每個微服務身上了。身分認證的目的是判斷請求者持有的 token 是否有效，確定是對應使用者登入的；授權的目的是檢查對應使用者是否可以取用這個方法，防止使用者進行越權的操作。

因為每個微服務都需要做一次認證和授權，單個微服務會需要處理自身領域外的事情，增加複雜度[^1]：首先，每個微服務都需要有自己一套進行認證和授權的程式碼，除了拖慢呼叫一個 RPC 方法的速度（現代微服務一個 Endpoint 可能會需要呼叫好幾十個 RPC 方法），微服務本身也會因為這些驗證而變得臃腫。接著，認證和授權都需要依賴「使用者服務」，如果「使用者服務」故障了，幾乎所有微服務都會停擺。最後，服務之間的通訊該怎麼認證呢？如果你的認證方式是 OAuth 的話，你很需要給所有服務組合產生一組 [M2M token](https://auth0.com/blog/using-m2m-authorization/)⋯⋯

[^1]: Authentication & Authorization in Microservices Architecture - Part I. <https://dev.to/behalf/authentication-authorization-in-microservices-architecture-part-i-2cn0>

除了後端一團糟外，前端也會一團亂。你的前端需要先知道「我需要的方法在哪個服務上」，然後還需要知道這個服務的連線網址。你的 API 呼叫時需要指向各種不同的網址，當你要撤換服務時，需要一一檢查前端有沒有地方引用到這個服務，增加你撤換和更新這些服務時的困難度。同時你的這些服務因為都跟使用者的前端（也就是用戶端）扯上關係了，所以設計上又需要考慮用戶端會怎麼搞壞你的服務，並且需要花上更多時間把你的 RPC 方法封裝得更適合讓任意用戶端呼叫（比如因此捨棄簡單的 gRPC，轉為使用 REST API，去提升各個用戶端的相容性）。

如果我們用 Gateway 的邏輯重新思考呢？上面的問題可以解決一大半：

![microservices with gateway](https://assets.blog.pan93.com/microservice-gateway-and-communication/services-with-gateway.png)

現在身分認證和授權的任務就完全交給 Gateway 了，微服務不再涉及身分認證的議題。這樣一來，我們其他的微服務（使用者服務、課程服務、選課服務）就不用認證了，可以專注在自己的領域上，不用再和「使用者服務」互動。

接著，因為現在只有 Gateway 會呼叫這些微服務，我們的微服務就可以設計得更簡單、用上更快但相容性不怎麼好的通訊技術，並且在更新或重構微服務時就只需要關心與 Gateway 的相容性就好。與此同時，因為微服務只跟 Gateway 互動，我們就可以把微服務鎖在只有 Gateway 能取用的內網，降低微服務被攻擊的可能性。

> 🤔 雖然微服務現在只放在內網，通常情況下不太會被未授權第三方存取；但假如有人攻入內網了，我們可以怎麼保證 API 的安全性？

前端只和 Gateway 通訊，所以你只需要在 Gateway 端關心用戶端連線上的相容性即可。另外 Gateway 也可以透過重新設計 API，將服務 RPC 方法和跨服務的細節隱藏掉。舉例來說，假如使用者想要知道自己有沒有被限修，沒有 Gateway 的情況下，用戶端需要被迫知道「限修規則」和「使用者資訊」是在兩個不同的服務，需要分開呼叫；Gateway 就可以封裝一個高層次的「是否限修」API，用戶端只需要呼叫這個 API 就可以知道結果。

> 🤔 Gateway 做法看起來很棒，但它有什麼缺點呢？

## 微服務的請求如何流動？

我們第二章和第三章介紹了微服務的拆法、RPC 方法的設計，以及 Gateway 的設計。但直到目前，我們都還沒談到「用戶端送出一個操作後，這個操作是怎麼在微服務之間流動的」。這一個小節，我們就以「選課」為例子，來看看「選課」例子在我們的架構下會經過哪些服務，以及這些服務是怎麼互相通訊的。

> 📚 之後我們會詳細介紹到具體通訊的方案，這裡只會簡單地用抽象的概念帶過～

![microservice communication](https://assets.blog.pan93.com/microservice-gateway-and-communication/microservice-communication-basic.png)

上面這種架構下，微服務溝通的流程大致是：

1. 使用者按下「加入選課志願」後，用戶端會往 Gateway 送出一個 `POST /api/v1/course/:course/selection` 的 HTTP 請求。與此同時，前端會把使用者的憑證 (token) 一起送出，讓 Gateway 可以知道是誰送出請求的。
2. 前端呼叫後，Gateway 服務會先檢查前端送出的 token 是否有效（認證）。如果 token 有效，我們確認一下這個 token 對應的使用者，是否有權限進行選課（授權）。
3. Gateway 接著呼叫選課服務的「插入志願 RPC」，讓「選課服務」負責將這門課程加入這個學生的志願名單中。
4. 選課服務接收到這個請求後，會先向「課程服務」查詢這門課程的限修規則，再向「使用者服務」查詢這個使用者的權重和選修學分。如果不符合限修規則，選課服務會往 Gateway 回傳錯誤；如果一切都符合條件的話，我們將這個志願記錄到與選課服務搭配的資料庫裡面。
5. 最後，選課服務會回傳一個成功的訊息給 Gateway。
6. Gateway 根據選課服務的回傳結果，建構回應給前端。

## 接下來怎麼連線到服務？

到這裡我們就介紹完「微服務」裡面抽象的概念了！看起來微服務是個很簡單、很直覺的東西：不就是把大的單體服務拆成幾個比較小的服務，呼叫這些服務的 RPC 方法嗎？但假如我們真的開始把微服務實作起來，你會發現有很多很多的議題需要解決。

我們下一章會講到「**服務探索**」，這個議題用一句話描述，就是「`gateway` 要怎麼知道 `selection-service` 的連線位置。」你或許會覺得這是個蠢問題：不就是這個服務的 IP 地址和連線埠嗎？但如果在微服務使用 IP 地址和連線埠互相連線，會遇到什麼問題？我們下一章就要來著手搞定這些服務之間的連線，看看有什麼方法可以讓我們更可靠的知道服務的連線方式。

> 🤔 在分散式系統下，用 IP 和連線埠互相串聯的問題是什麼？

## 💬 互動區塊

- Threads: <https://www.threads.net/@pan93412/post/DFsm41vyCto?xmt=AQGzl7pjfhsQTvC7cvx3dZ14EaAxQQRWNXBfOgqrMPa1IA>
- Telegram: <https://t.me/bystartw_chan/324>
- X / Twitter: <https://x.com/byStarTW/status/1887169833097375844>
- Bluesky: <https://bsky.app/profile/pan93.com/post/3lhgx4pmpl227>
