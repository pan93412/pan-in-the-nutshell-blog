+++
date = '2025-09-21T13:30:40+08:00'
title = '分頁應該在用戶端還是伺服器端進行？'
categories = ["comments"]
tags = ["pagination", "api", "client", "server"]
+++

> 我有看過一種做法是 api 直接在 server side 直接吐完整包列表資料，前端在自己做 pagination, 我不知道這樣做跟 api 做 pagination 比有沒有優勢
>
> [Post](https://www.threads.com/@wildfrontend/post/DOLnNBQErwz?xmt=AQF0ri-Ztw3Ofz9nb8FjUyki5lWSTc9FYgDZauUA5daUDg)

原則上，如果抓資料的開銷和筆數有關，我會設計 API 層級的分頁：API 比較不需要讓資料庫查太多東西 (`SELECT` all) 而造成資料庫壓力，也不會一次下載太多沒必要的資料回來。

如果資料量沒這麼大（比如一些 Tags），我會讓 client 分頁，純粹只是為了改善前端體驗。如果資料量大一點（比如 1K），我就會傾向在 API 設計基礎的分頁功能（即便對後端來說，取多少筆資料都不會影響速度），來減少需要回傳的內容。

By the way 後端分頁也有他的學問，傳統的 offset-based pagination 在資料筆數非常大的情況下，可能會有效能問題。可以看看比如 [Cursor-based pagination](https://tec.xenby.com/36-%E9%BE%90%E5%A4%A7%E8%B3%87%E6%96%99%E5%BA%AB%E5%88%86%E9%A0%81%E6%96%B9%E6%A1%88-cursor-based-pagination) 的知識，或許對以後設計超大資料的分頁 API 會有幫助。
