+++
date = '2025-09-21T13:12:21+08:00'
title = '弱 ETag 可以送到 If-Match 嗎？'
categories = ["comments"]
tags = ["http", "etag", "comments", "rfc"]
+++

> 如果伺服器傳送 `Etag: 1233`，過了 CDN Client 收到 `Etag: W/"1233"`
>
> 那麼 Client 回送 `If-Match` 應該傳送 `W/"1233"` 還是 `1233`？
>
> [Original Post](https://www.facebook.com/share/p/17TB3cW8wC/)

If-Match 的用意是在於防止「更新遺失」或「中途編輯衝突」（mid-air edit collision）。它的核心用途是在執行條件式請求時，確保用戶端操作的資源版本與伺服器上的目前版本完全一致[^1]。

如果 CDN 傳送 `Etag: W/"1233"`（weak validator），因為 `If-Match` 標頭要求使用強比較函式[^1]（即 `E/1` ≠ `E/1`[^2]），因此用戶端無法使用這個值來建立 `If-Match` 條件式請求[^1]。因此，如果伺服器傳送的是 `Etag: W/"1233"`，則不應該在 `If-Match` 中傳送任何值來進行條件式更新。

如果要做快取的話，直接記錄 ETag，日後用 `If-None-Match` header 讓伺服器比較版本就行[^3]。通常來說 `If-Match` 都是防止狀態改變的方法（POST 這些）[^1]，CDN 通常不會去快取 POST 這類主要做 mutation 的方法，也就沒必要把 ETag 改成 weak representation，也就不太會遇到這個問題。

[^1]: RFC 7232 – Hypertext Transfer Protocol (HTTP/1.1): Conditional Requests – If-Match: <https://datatracker.ietf.org/doc/html/rfc7232#autoid-15>
[^2]: RFC 7232 – Hypertext Transfer Protocol (HTTP/1.1): Conditional Requests – ETag Comparsion: <https://datatracker.ietf.org/doc/html/rfc7232#autoid-11>
[^3]: RFC 7232 – Hypertext Transfer Protocol (HTTP/1.1): Conditional Requests – If-None-Match: <https://datatracker.ietf.org/doc/html/rfc7232#autoid-16>
