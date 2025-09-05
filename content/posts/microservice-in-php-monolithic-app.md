---
date: '2024-11-28T07:58:43+08:00'
title: '從 PHP Symfony 應用程式分出微服務'
tags: ['symfony', 'microservice', 'php', 'go', 'api']
categories: ['Developments']
draft: true
---

## Background

其實這有點標題黨，本質上就是 PHP Symfony 怎麼連接 HTTP 服務而已。

手頭上有一個 [幫教授寫的 SQL 語法練習平台](https://github.com/database-playground/app-sf)，這個練習平台裡面有一個以 SQLite 為基礎的 SQL 執行器。以往這個執行器是跟主平台放在一起，每當要執行 SQL 語法時，都會 [呼叫 `php` 啟動一個執行器子程序](https://github.com/database-playground/app-sf/blob/v0.3.0/src/Service/DbRunner.php#L73)，用標準輸入 / 輸出 [傳輸序列化的資料](https://github.com/database-playground/app-sf/blob/v0.3.0/src/Service/Processes/ProcessService.php#L35-L40)。

說實話這樣沒有問題，練習平台運作的 4 個月期間，這種 IPC 模式都沒有出過大狀況。不過有幾個小問題，可能會對未來主程式的擴展造成一些障礙：

- 每個請求啟動一個 PHP 處理程序，在高使用量下會造成突發的記憶體增加。
- SQL 執行器的快取和主程式用同個快取介面（雖然是不同 cache pool），清除 SQL 執行器的快取需要在主程序操作。
- 不方便日後在其他環境單獨使用 SQL 執行器（比如 script 測試）。
- SQL 執行器出問題時（比如記憶體突然暴增時）可能會連累執行 app 的 container。

關鍵契機是打算把練習平台用 Next.js 重寫，所以先把 SQL Runner 拆成一個獨立的服務。後來 Next.js 的計劃被拋棄了，但也剛好把 SQL Runner 拆出了。日後也可以單獨對 SQL Runner 做 load balancing，不過現在的 SQL Runner 吞吐量面對現在的需求應該已經夠好了。

## Choosing RPC method

[新版的 SQL 執行器](https://github.com/database-playground/sqlrunner-v2) 是用 Go 寫的。理論上在 Go 底下，用 gRPC 做 RPC 介面是最直覺的選擇。相較於 REST API，gRPC 有這幾個特點：

- 通常使用 Protocol Buffer 這種二進位的資料格式傳輸的，在序列化 / 反序列化上的速度會遠高於 JSON 結構。[^1]
- 支援全雙工傳輸，可以 input / output 同時標為 stream 來互相串流資料。[^2]
- 在 Go 底下有很成熟的 codegen。[^3]

[^1]: <https://grpc.io/docs/what-is-grpc/introduction/#working-with-protocol-buffers>

[^2]: <https://grpc.io>

[^3]: <https://grpc.io/docs/languages/go/quickstart/>

不過 gRPC 在其他語言上就不是這麼理想的選擇了。舉例來說：

- gRPC 的 Python[^4] 和 PHP[^5] Type Hint 還不夠好，很多 methods 還是需要猜測的，文件也比較簡略[^6]。
- gRPC 的 PHP extension 可能會跟 FrankenPHP 的 worker 模式發生衝突，用 `glibc` 在我的 case 下也會出現 crash 的問題[^7]。
- gRPC 在 Rust 上沒有官方支援，`prost` 與現有的 Rust ecosystem 整合不算太好[^8]，而第三方的 WKT 解法不是很優雅[^9]。
- gRPC 預設對於大 Payload 有著 4MiB 限制[^10]，而轉成 Streaming API 會引入其他開發負擔[^11]。
  - HTTP 的 streaming 相對來說比較 naive，而且可以用支援 stream 的 JSON decoder 處理。
- gRPC 沒有 HTTP API 可以方便 debug 或在不支援 gRPC 的語言進行呼叫（雖然有相容 gRPC 的 [connectrpc](https://connectrpc.com) 可以使用）

[^4]: <https://github.com/grpc/grpc/issues/29041>

[^5]: <https://github.com/grpc/grpc/issues/33431>

[^6]: <https://grpc.io/docs/languages/php/basics/#calling-service-methods>

[^7]: <https://github.com/dunglas/frankenphp/issues/676#issuecomment-2138660167>

[^8]: <https://github.com/tokio-rs/prost/issues/537>

[^9]: <https://github.com/fdeantoni/prost-wkt>

[^10]: <https://learn.microsoft.com/zh-tw/aspnet/core/grpc/security?view=aspnetcore-8.0#message-size-limits>

[^11]: <https://grpc.io/docs/languages/go/basics/#server-side-streaming-rpc-1>

另外配置 `protoc` 也是一筆不小的時間開銷，而且 JSON 序列化 / 反序列化的時間並不是 SQL Runner 的瓶頸。所以，我最終選擇用很簡單的 HTTP API + JSON 做一個 interface。

## Implementing the Server

自從 Go 1.21 [有了路由功能](https://pkg.go.dev/net/http#hdr-Patterns-ServeMux) 之後，用 Go 寫一個簡單的 REST API 已經不太需要引入 HTTP 相關的套件了。

```go
http.HandleFunc("GET /healthz", func(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusOK)
    _, _ = w.Write([]byte("OK"))
})
```

如果你的 API 有一些依賴或者是狀態（比如快取或者是互斥鎖），也可以讓 struct 實作 [`http.Handler`](https://pkg.go.dev/net/http#Handler)，然後傳入 `http.Handle` 當中：

```go
service := &SqlQueryService{ /* stateful */ }

http.Handle("POST /query", service)
```

`http.Handle` 是並行 (concurrently) 接受傳入請求的，而我們初始化的程式碼會將資料寫入硬碟，不希望有超過兩個 goroutine 同時進行初始化，造成磁碟資料毀損的情況。這時候我們可以使用 [`singleflight`](https://pkg.go.dev/golang.org/x/sync/singleflight) 套件，這樣包住的程式碼只會 **由 1 個 goroutine** 初始化 **1 次**。

```go
type SqlQueryService struct {
	sfgroup      singleflight.Group
}

func (s *SqlQueryService) findRunner(schema string) (*sqlrunner.SQLRunner, error) {
    result, err, _ := s.sfgroup.Do(schema, func() (any, error) {
		newRunner, err := sqlrunner.NewSQLRunner(schema)
		if err != nil {
			return nil, fmt.Errorf("create SQLRunner: %w", err)
		}

		return newRunner, nil
	})
	if err != nil {
		return nil, err
	}

    typedResult := result.(*sqlrunner.SQLRunner)
	return typedResult, err
}
```

注意 `sfgroup` 不會自動 GC，不過可以用 [`Group.Forget(key)`](https://pkg.go.dev/golang.org/x/sync/singleflight#Group.Forget) 來刪除對應的項目，可以根據這個原理實作 LRU cache。
