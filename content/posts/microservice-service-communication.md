+++
date = '2025-09-05T20:22:00+08:00'
title = '聊聊微服務是什麼 - 服務通訊'
tags = ['microservice', 'backend', 'SITCON 2025']
categories = ['Developments']
+++

這個 Tag 的第六個主題是「服務通訊」，講白話些就是「我要怎麼呼叫另一個服務的功能？」我們在第三個主題講了「服務通訊的大致邏輯」，以及在第四個和第五個主題花了很大的篇幅講了「服務怎麼 **連線** 到服務」，但好像從來都沒有真正說「服務怎麼和服務 **互動**」、「RPC 是什麼」，以及具體要怎麼通訊、有哪些通訊方法，以及有哪些通訊手段。這裡就要開始介紹了！

> 這篇是之前 SITCON 2025 的草稿，不是後面寫的，也還沒有做過 peer reviewing！如果有任何問題的話，也歡迎到 Threads 或 X 上標我指正。

## 「通訊」是什麼？

在單體服務的世界，你要呼叫一個方法，比如「選課」，你通常是用函式或 HTTP API 呼叫完成的。舉例來說：

1. 你的前端往你的 API 發出了 `POST /api/v1/course/:course/selection`。我們會稱這個網址叫做端點 (endpoint)。
2. 你的 API 有一個「選課」Controller，裡面有一個處理選課端點的「方法」(method)，這裡叫做 `SelectCourse` 吧！
3. 接著你的 `SelectCourse` 方法會透過呼叫 `GetStudentInfo` 方法來取得這個學生的資訊，以及透過 `GetCourseContraints` 方法取得課程的限修資訊
4. 接著 `SelectCourse` 方法會根據這些資訊決定允不允許學生選這堂課，然後將選課結果插入資料庫當中。

![monolithic service communication](https://assets.blog.pan93.com/microservice-service-communication/01-monolithic-service-communication-optimised.avif)

> 📚 小提醒：在 MVCS 架構下，我們通常會叫這種功能性質的 class 為 **Service**。不過為了防止跟微服務的 Service 混淆（雖然其實概念是一樣的），這裡就不會用 Service 這個名詞。

如果你能理解上面這張圖，那我們把它換成微服務，你就很清楚我們要解決什麼「通訊」問題了：

![microservice communication](https://assets.blog.pan93.com/microservice-service-communication/02-microservice-communication-optimised.avif)

上面這張圖的青色線段，就是我們這次要探討的問題了：**這些服務都不在一起，我們要怎麼呼叫對方的方法**——也就是「青色」這個線條，究竟怎麼實作？

> 🫨 選課服務是怎麼連線到使用者服務的？你可以往回看看「服務探索」這個章節！從這裡開始，我們都假定我們已經知道這些服務的 IP 和連線方式了，要處理的只是應用層的溝通問題。

## 用 HTTP 設計自己的 RPC

首先，最直覺的方法，就是你給每個服務開一個 HTTP 的 API，用 HTTP 的呼叫邏輯和其他服務溝通。因為這些 HTTP 方法只是內部使用的，你不用像 REST API 一樣，需要認真思考每個端點的命名，只需要讓每個服務看得懂就好。因此，你這樣設計你的微服務溝通方法：

```plain
POST /getStudentInfo
Host: 使用者服務
Content-Type: application/json

  --> {"studentID": "A123456789"}
  <-- {"studentID": "A123456789", "gender": "male", ...}
```

![communicate based on HTTP](https://assets.blog.pan93.com/microservice-service-communication/03-communicate-based-on-http-optimised.avif)

其實上面這樣就是一個 RPC 了——根據 [AWS 的定義](https://aws.amazon.com/tw/compare/the-difference-between-rpc-and-rest/)，RPC 其實就是隨意奔放、主要「專注於功能或動作」的 API。實際上這種以 HTTP 為基礎的 RPC 很常見，甚至有很多 RPC 框架是根據上面的邏輯為基礎設計的。比如說，JSON-RPC 就是一個類似上面這種設計的 RPC 標準：

```plain
POST /rpc
Host: 使用者服務
Content-Type: application/json

 --> {"jsonrpc": "2.0", "method": "getStudentInfo", "params": {"studentID": "A123456789"}, "id": 1}
 <-- {"jsonrpc": "2.0", "result": {"studentID": "A123456789", "gender": "male", ...}, "id": 1}
```

> 🤔 你知道哪些 RPC 框架是以 JSON-RPC 為基礎的嗎？

## 為什麼需要 RPC 框架？

不過你會發現上面的 RPC 有幾個小問題：

1. 我是怎麼把外部呼叫的 `POST /getStudentInfo`，跟使用者服務裡面實際的 `GetStudentInfo` 連結的？就目前沒有框架的設計，你得自己在 HTTP 框架寫一個「`POST /getStudentInfo`」路由對應到 `GetStudentInfo` 的規則。
2. 我的「選課服務」是怎麼知道「使用者服務」可以用 `POST /getStudentInfo` 呼叫「取得使用者資訊」，然後可以傳入 `studentID` 參數的？就目前沒有框架的設計，這單純是靠人腦保證的：我在開發「選課服務」的當下，知道「使用者服務」有一個這樣的方法和參數能呼叫，所以我就在「選課服務」呼叫 `getStudentInfo` 方法。

你或許就發現跟「服務探索」很相似的問題了：假如對方變動了——舉例來說，我把 `POST /getStudentInfo` 改名成 `POST /getStudentInformation` 了呢？這樣子「選課服務」當然就沒有辦法呼叫「使用者服務」取得使用者資訊的方法了。

![what if endpoints changed](https://assets.blog.pan93.com/microservice-service-communication/04-what-if-endpoint-changed-optimised.avif)

說實話上面這兩件事情是廢話：你在單體服務呼叫不存在的方法，肯定也是會出錯的。但不同的是，你可以在程式編寫的期間就很明顯地知道「沒有這個方法」，以及會很清楚「這個方法可以傳入哪些參數」。

所以，我們希望引入框架來解決上面兩個問題：

1. 我們可不可以請框架自動把內部的方法和外部呼叫的 RPC 方法對應起來？
2. 這個框架可不可以產出一些程式碼，讓我可以方便的呼叫框架產出的 RPC 方法？
3. 同時，這些程式碼可不可以讓開發者方便知道（甚至是驗證）這個方法接受的參數？

## 引入 RPC 框架

RPC 框架的目的，就是讓你不用再關心底層的通訊方式（JSON、HTTP POST），透過產生程式碼（stubs），讓我們可以像單體服務一樣用高層次（high-level）的函式，呼叫遠端服務的方法。這裡就不詳細展開「RPC 框架有什麼類型」，我們就直接跳入重點：主流的 RPC 框架要怎麼用？

> 🤔 RPC 框架有什麼類型，以及他們分別的代表是什麼？

這裡就介紹一個常見而且頗為主流的 RPC 框架：gRPC。

1. 首先你給每個服務用 gRPC 的 proto 語言寫出他的方法、他接受的參數以及回傳值：

    ```protobuf
    // 不是完整的 proto 定義！

    service UserService {
      rpc GetStudentInformation(GetStudentInformationRequest) returns (GetStudentInformationResponse)
    }

    message GetStudentInformationRequest {
      string student_id = 1;
    }

    message GetStudentInformationResponse {
      string student_id = 1;
      string gender = 2;
      // ....
    }
    ```

2. 用 `protoc` (proto 的編譯器) 分別產出他的用戶端 (client) 和伺服器 (server)。
3. protoc 的編譯器會產生伺服器端的介面宣告 (interface)，你負責在產出的「使用者服務」伺服器實作 (implement) `GetStudentInformation` 方法。
4. 然後，你在「選課服務」使用編譯器產出的用戶端，連線到「使用者服務」的 gRPC 伺服器，然後像一般的函式呼叫般呼叫 `GetStudentInformation`——傳入 `GetStudentInformationRequest` ，等待伺服器回傳 `GetStudentInformationResponse` 。
5. 完成！

![gRPC](https://assets.blog.pan93.com/microservice-service-communication/05-grpc-optimised.avif)

gRPC 有幾個特點：首先他使用 Protocol Buffers 來把結構編碼成二進位的格式，會比 JSON 緊湊，並且格式和類型也會更精確、嚴格得多。除此之外，它用上了許多 HTTP/2 的特性來傳輸編碼後的結果，來盡量降低遠端呼叫的延遲。不過它也有個小缺點：`protoc` 不是所有語言都支援的，遇到不支援的語言，可能就會出一些小狀況——不過主流的語言基本上 `protoc` 都有支援（或者是第三方實作）啦。

> 🤔 不支援 gRPC 協定的語言，我們作為伺服器端可以怎麼優雅地提供備用介面 (HTTP) 呢？

## 非同步訊息通訊

到這裡，我們其實就學會了服務之間怎麼互相通訊了。不過你會發現到我們目前講到的都是「同步通訊」，也就是我請求「使用者資訊」**然後等待回應**。但有些方法其實你完全不需要真的等待。比如說「選課」我們可不可以讓他變成放在背景執行，使用者按下去之後可以先做其他事情，而我們處理好之後再告知使用者呢？

> 🤔 你直覺上會怎麼實作「背景執行」的功能？

我們下一個主題正是要講到「非同步訊息通訊」[^1]。而且有個好消息：這個「非同步訊息通訊」是單體服務也能實作的！

[^1]: 這篇沒有寫完，所以沒有發佈。
