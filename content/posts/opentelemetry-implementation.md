---
date: '2025-01-12T19:33:00+08:00'
title: '導入可監控性：OpenTelemetry 實踐和 Request ID 實作'
tags: ['microservice', 'telemetry', 'request id', 'observability']
draft: true
---

# 導入可監控性：OpenTelemetry 實踐和 Request ID 實作

## 前言：為什麼會需要 Trace ID？

透過有限的報告資訊，定位出是哪個程式出了問題，是很有難度的。舉例來說，假如我們收到一份內容只有「無法部署」的 ticket，要怎麼定位出「無法部署」的原因？

以往我們都會和客戶追問「發生了什麼事情」、「執行了什麼操作」這種 _context_，然後從一堆微服務根據 ID 從 Logs 和程式碼中大海撈針，找出哪裡發生錯誤，甚至一些比較棘手的情況，還需要復現或讓使用者重試一次。

![handle tickets from users without the telemetry](https://assets.blog.pan93.com/opentelemetry-implementation/handle-user-ticket-without-telemetry.png)

如果我們可以記錄使用者操作的動作，以及背後系統的運轉流程，並且用 ID 附加在工單上面，或許我們就可以節省許多 debug 的時間，把重點放在「解決問題上。」

![telemetry request timeline](https://assets.blog.pan93.com/opentelemetry-implementation/telemetry-request-timeline.png)

「從使用者的裝置和操作上，自動記錄並蒐集相關的線索」，其實就是「遙測 (Telemetry)」，目的就是透過提升可觀測性 (observability)，方便開發者了解情境資訊並加速除錯流程。

> 這個是之前 MOPCON 2024 的講題，把簡報重新脈絡化並重寫成 blog。

## Step 1: 我們怎麼記錄錯誤？

首先，我們要怎麼知道使用者出錯時發生了什麼情況？或許我們可以試試看記錄使用者當下操作的情境，例如網址、瀏覽器版本等等的資訊。

![Step 1: Logging on Error](https://assets.blog.pan93.com/opentelemetry-implementation/step-1-logging.png)

剛開始寫後端應用程式的開發者，會覺得「我遇到錯誤，就向上傳播 (propagate) 呀！」就以取回 GitHub 儲存庫為例，你自然而然的會寫出這樣的程式碼：

```go
repositories, err := fetchRepositories()
if err != nil {
  return err  // 原樣回傳錯誤
}
```

然後你 API 的最頂層，自然就是把錯誤格式化成 JSON，傳給使用者了。這樣會有什麼情況呢？我們模擬一下使用者收到的內容：

![propagate errors to users](https://assets.blog.pan93.com/opentelemetry-implementation/error-propagation-to-user.png)

你會發現前端噴出了一大堆只有後端看得懂的錯誤，這點對使用者的 UX 是很傷的，畢竟使用者看不懂也不在乎你的後端服務實際上出了什麼錯誤。

使用者看到這一堆錯誤，首先想到的或許是回報給開發者，不過很多使用者只會說出「系統無法使用」這樣的低意義描述，讓開發者幾乎無法 debug。此時開發者就會需要和使用者一來一回詢問出更完整的錯誤。

更何況，很多錯誤是需要情境的——比如你的程式碼區塊中有很多地方會呼叫到 GitHub API，我們要怎麼知道是哪個地方的 GitHub API 出了狀況呢？

所以，第一個問題，是 **我們可不可以把錯誤留在內部就好？**

### 實作最基本的 Log

我們要怎麼給使用者他能懂的敘述，同時留給開發者精確的錯誤呢？或許我們可以先試試看把錯誤輸出到 console 上，然後給使用者一個錯誤碼（或者是錯誤訊息）。

```go
repositories, err := fetchRepositories()
if err != nil {
  fmt.Println(err)
  return errutils.Code("GITHUB_ERROR")
}
```

好消息！現在使用者端的錯誤可以更看得懂了。壞消息是你的 `fmt.Println` 方式沒有辦法指出 **這個 Log 是誰觸發的**、**這個 Log 是怎麼觸發的**[^1]，以及相關的傳入參數。最後使用者告訴你「GitHub 出狀況了，」你還是很難知道 GitHub 什麼時候出得狀況。

![basic logging of error](https://assets.blog.pan93.com/opentelemetry-implementation/basic-logging-of-error.png)

[^1]: 有些語言的 error 會有 stack trace，這種就有辦法知道 Log 的觸發點。

所以，我們先試試看解決第一個問題吧：給每個錯誤打上一個 ID。

### 給 Log 加上 Request ID

給每個 Log **打上一個 ID**，然後 **把這個 ID 回傳給使用者**，或許我們就可以根據使用者回報的 ID，知道問題出在哪裡。

所以假設我每次請求，都產一個對應到這個請求的 UUID，然後在輸出錯誤以及回傳錯誤碼時附加這個 ID，這樣使用者回報 ID，我就可以找得出這個請求當時發生了什麼錯誤了！

```go
ctx = context.WithValue(ctx, KeyRequestUuid, uuid.New())   // psycho code
```

```go
repositories, err := fetchRepositories()
if err != nil {
  // psycho code, concept only
  fmt.Printf("[%s] %v", ctx.Get(KeyRequestUuid), err)
  return errutils.CodeWithContext(ctx, "GITHUB_ERROR")
}
```

![implementation of request ID](https://assets.blog.pan93.com/opentelemetry-implementation/request-id-impl.png)

不過這樣還是有個小問題：我們還是不知道「使用者是怎麼觸發這個錯誤的。」比如說他的請求連結和參數是什麼、他用的瀏覽器版本，甚至是他的使用者 ID。我們有沒有辦法記錄得更多？

### 讓 Log 更有情境脈絡

接著，我們試試看在這個 Request ID 記錄更多資訊，比如請求 URL、請求參數、UA 等等的資訊。

![contextful error](https://assets.blog.pan93.com/opentelemetry-implementation/contextful-error.png)

好消息是我們終於知道錯誤是在哪個網址（或函式）被觸發的了，並且也知道我們該用什麼環境復現了。但是隨著我們加入的錯誤愈來愈多，Console 的錯誤會愈來愈多、愈來愈亂，並且沒有結構可言，不適合程式分析以及蒐集。

JSON 作為一個廣為使用的資料交換格式，我們可不可以把錯誤的資訊寫入一列 JSON 當中，這樣一個 JSON 物件就包含上面這些豐富的情境資訊？

### 提升 Log 的結構性

用 JSON 物件來封裝記錄 log（然後每次都有相近的結構），稱之為「**結構化記錄 (structured logging)**」[^2]。

[^2]: https://cloud.google.com/logging/docs/structured-logging

![structured logs](https://assets.blog.pan93.com/opentelemetry-implementation/structured-logs.png)

假如我們是把這種結構化 Log 輸出到 Console 上的話，基本上會把它壓成一列，這樣子一列就會是一個 Log，相對會好處理很多，比如說我們可以寫 Python script 來逐列分析 Log 的內容。

不過 structured logs 輸出到 Console 有時候不好篩選和分析，假如我們想看某個子系統的 Logs，輸出到 Console 會需要我們讀取每一列 Logs 來進行篩選。如果我們把 logs 放到資料庫 (OLAP) 裡面，會不會更方便我們篩選甚至是快速查詢？

### 存入資料庫

Logs 首先很明顯是種 [時序資料](https://ithelp.ithome.com.tw/m/articles/10291212)，資料是有時間順序的。接著 Logs 的量很可能會很大，而且我們沒有關聯 (association) 的需求，所以我們可以把它放在專為時序資料設計的 NoSQL 資料庫上——比如 Clickhouse。

把 Logs 存到資料庫後，我們就能非常快速地根據使用者給出的 request ID 和情境資訊，檢索出需要的記錄。

![logs to database](https://assets.blog.pan93.com/opentelemetry-implementation/logs-to-database.png)
