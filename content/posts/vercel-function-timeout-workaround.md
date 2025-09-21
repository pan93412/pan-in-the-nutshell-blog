+++
date = '2025-09-21T13:39:02+08:00'
title = '如何繞過 Vercel 的 Function 逾時？'
categories = ["comments"]
tags = ["vercel", "function", "zeabur"]
+++

[![original post](https://assets.blog.pan93.com/vercel-function-timeout-workaround/original-post-optimised.avif)](https://www.threads.com/@ruei_hua7th/post/DK7RWrLpgYV?xmt=AQF0ri-Ztw3Ofz9nb8FjUyki5lWSTc9FYgDZauUA5daUDg)

執行逾時了，Vercel 的免費方案 [只給你 60 秒回應](https://vercel.com/docs/fluid-compute#default-settings-by-plan)。假如你的 /upload 等待回應的時間超過 60 秒，那就會被 Vercel 殺掉哦。

有三種解法：

1. 如果不想花錢，就想辦法把任務放到背景，如圖一。不過我翻了一下 Vercel 的文件，它沒給你機會建立一個超過 60 秒的任務，所以你大概得在 Cloudflare Workers 用 JavaScript 重寫你 Gemini 等待回應那段邏輯，用 waitUntil 把任務移到背景執行，接著將回應寫進資料庫，你的前端再不停呼叫 response 等回應。注意 Cloudflare Workers 的免費方案有 100,000 次的觸發限制，而一輪會至少觸發 2 次。

   ![workaround](https://assets.blog.pan93.com/vercel-function-timeout-workaround/workaround-optimised.webp)

2. 升級到 20 美元的 Vercel Pro，他會慷慨的多給你 12 分鐘回應。

3. 試試看 5 美元的 Zeabur，不限制回應時間，你想處理多久就處理多久。
