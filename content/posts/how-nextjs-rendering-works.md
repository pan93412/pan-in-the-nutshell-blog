+++
date = '2025-09-28T17:54:18+08:00'
title = 'Next.js 的渲染模式是怎麼運作的：SSR、CSR、SSG、ISR、PPR…'
categories = ["Developments"]
tags = ["nextjs", "react", "ssr", "csr", "ssg", "isr", "ppr"]
+++

> 看了一下 next.js ，感覺只是將原本 react 提供的 js 檔案先轉成 html 再輸出，還是獨立伺服器處理，不像 express-handlebars 可以跟 node.js 整合在一台伺服器內 [[MozTW](https://t.me/moztw_general/1/263038)]

我不太喜歡把 Next.js 的渲染模式歸結成任何形式 (SSR or CSR or SSG)，但想和你分享一下為什麼 Next.js 不是另一個 express-handlebars，以及 Next.js 團隊到底在渲染下做了多少事情。

express-handlebars 通常是純粹的 SSR，不考慮你在頁面中插入的 JavaScript，handlebars 不用往產物增加 JavaScript 恢復互動元素（也就是 Hydration）。

> Hydration 是指 React 接管由伺服器產生的靜態 HTML、並為其附加事件監聽器與狀態，使其從無生命的骨架變成可互動應用的關鍵步驟。

純粹的 React，就以搭配 Vite 來說的話，是 CSR。因為背後不用有配合的伺服器，放在可以 host 靜態網站的 web server 上就可以（比如 NGINX、Caddy 這種）。

## Next.js 的 Pages Router 有哪些渲染方式？

接下來回頭看 Next.js。如果只看 Pages Router，會相對簡單一點，可以分成 SSG、ISR、SSR、CSR 的部分：

如果頁面不用在伺服器端請求資訊，而且要產生的頁面是可以在編譯期推斷出來的，Next.js 會在編譯期 (build time) 產生好 HTML，使用者瀏覽時就會直接回傳，這叫 SSG (Static Site Generation)[^pages-ssg]。

如果編譯期產生的頁面還是需要在一段時間後更新，你可以設定 `revalidate` 讓 Next.js 在時間過後重新產生頁面後更新頁面快取，即使不用重新編譯也可以拿到新資料，這叫 ISR (Incremental Static Regeneration)[^pages-isr]。

![ISR](https://assets.blog.pan93.com/how-nextjs-rendering-works/isr.avif)

如果頁面需要在伺服器端請求資訊（比如在 `getServerSideProps` 做 fetch），Next.js 會在使用者請求後才能開始產生 HTML (request time)，這叫 SSR (Server-side Rendering)[^pages-ssr]。

最後，如果頁面的元素是需要在使用者瀏覽時，讓使用者的瀏覽器抓取後由瀏覽器渲染（比如 `useEffect` 中的 `fetch`），那就是 CSR (Client-side Rendering)[^pages-csr]。

![CSR (fetch)](https://assets.blog.pan93.com/how-nextjs-rendering-works/csr-fetch.avif)

另外，Next.js 預設會對所有頁面進行預渲染（Prerendering）[^prerendering]，無論是 SSG 還是 SSR 都屬於預渲染的範疇，目的是提供初始的 HTML 內容來改善 SEO 和首次載入速度，接著到使用者的瀏覽器後再進行 hydrate，讓頁面變得可以互動。你也可以使用 `next/dynamic` 搭配 `ssr: false` 來將元件從預渲染中排除掉，這樣元件就不會在伺服器上渲染，完全 CSR[^disable-ssr]。

![SSR hydrate](https://assets.blog.pan93.com/how-nextjs-rendering-works/ssr-hydrate.avif)

[^pages-ssg]: <https://nextjs.org/docs/pages/building-your-application/rendering/static-site-generation>
[^pages-isr]: <https://nextjs.org/docs/pages/guides/incremental-static-regeneration>
[^pages-ssr]: <https://nextjs.org/docs/pages/building-your-application/rendering/server-side-rendering>
[^pages-csr]: <https://nextjs.org/docs/pages/building-your-application/rendering/client-side-rendering>
[^disable-ssr]: <https://nextjs.org/learn/seo/dynamic-import-components>
[^prerendering]: <https://nextjs.org/learn/pages-router/data-fetching-pre-rendering>

## Next.js 的 App Router 有哪些渲染方式？

App Router 採用了更加自動化的渲染選擇機制，讓開發者無需明確指定使用哪種渲染模式。

Client Components 像是 Pages Router 既有的元件形式，而 Server Components 其實更像是 express-handlebars 這種傳統後端，頁面會在伺服器端渲染，並將渲染結果建構成 [RSC](https://nextjs.org/docs/app/getting-started/server-and-client-components#on-the-server)。這個 RSC 是在什麼時機產生的呢？可以分成兩種情況[^app-router-rendering]：

- 如果這個 Server Components 沒有使用動態 API（如 cookies()、headers()、searchParams）且所有資料取回都可在建置時確定，Next.js 就會採用「靜態渲染」策略。
- 如果這個 Server Components 需要用到請求期才能知道的資料，則是在請求期進行渲染，也就是「動態渲染」。

而靜態渲染的更新（revalidate）時機又有好幾種：

- 編譯期就把 Server Components 整個渲染好，也就是 [Full Route Cache](https://nextjs.org/docs/app/guides/caching#full-route-cache)。
- 如果你在 Server Components 中 `fetch` 的資料過期了（invalidated），那就會觸發 [Data Cache](https://nextjs.org/docs/app/guides/caching#data-cache) 的 Revalidation，這時候就會在請求期重新渲染後，將資料寫入 ISR 快取[^isr-note]。
- 當然還有傳統的 [ISR](https://nextjs.org/docs/app/guides/incremental-static-regeneration)，在編譯期宣告所有路由參數的組合（比如 `/posts/[id]` 中的 ID）。當 revalidate 時間到期時，會重新產生對應路由的內容。

如果是第一次載入頁面 (first load)，Next.js 會把 Server Components 的 RSC 和 Client Components 一起做 [預渲染](https://nextjs.org/learn/pages-router/data-fetching-pre-rendering)[^server-component-rendering-flow]，預先渲染成 HTML 後再進行 hydrate。後續切換路由時，Client Components 會改由瀏覽器端渲染（伺服器端不參與渲染，也就是 CSR），而 Server Components 繼續由伺服器端渲染之後回傳給用戶端進行渲染[^interleaving-server-and-client-components]。

> Next.js 負責渲染一堆 components 的後端伺服器，扮演的就是 [Backend for Frontend](https://nextjs.org/docs/app/guides/backend-for-frontend) 的角色。

![RSC](https://assets.blog.pan93.com/how-nextjs-rendering-works/rsc.avif)

[^app-router-rendering]: <https://nextjs.org/docs/app/getting-started/partial-prerendering#how-does-partial-prerendering-work>
[^server-component-rendering-flow]: <https://nextjs.org/docs/app/getting-started/server-and-client-components#how-do-server-and-client-components-work-in-nextjs>
[^interleaving-server-and-client-components]: <https://nextjs.org/docs/app/getting-started/server-and-client-components#interleaving-server-and-client-components>
[^isr-note]: Next.js 很多快取都是復用 ISR 的快取空間，所以即使 Data Cache 並不是 Next.js 所說的 ISR，但在 Vercel 上依然是進入 ISR cache 中：<https://nextjs.org/docs/app/guides/self-hosting#caching-and-isr>

### App Router 怎麼運用這些渲染方式

上面的介紹可能有些複雜——所以，如果我寫了一個 App Router，裡面有靜態渲染的 Server Components 和 Client Components，你應該可以看到：

- 大部分能被靜態渲染的 Server Components 都會在編譯期準備好 (SSG)。
- 打開頁面時，可以看到頁面被 Prerendering 了：Client Components 和 Server Components 會在伺服器端準備好後回傳 HTML 回來 (SSR)。
- 切換不同頁面時，Client Components 就是直接在瀏覽器裡面渲染了，而 Server Components 瀏覽器會透過 `?_rsc` endpoints 取得對應的 RSC payload，裡面包含 Server Components 的序列化資料和渲染指令。
- 如果你的 Client Components 呼叫了 `fetch` 相關的 hooks，那通常都是 CSR。如果有搭配 streaming SSR 的框架 (e.g. [@apollo/client-react-streaming](https://github.com/apollographql/apollo-client-integrations/tree/main/packages/client-react-streaming))，那也可能是 SSR 的。

如果想要看看每種渲染方式的觸發時機的話，在 Vercel 的 Observability 中可以分開來看：

![rendering methods](https://assets.blog.pan93.com/how-nextjs-rendering-works/rendering-methods.avif)

順帶一提，如果要省 Vercel 帳單的話，API Routes、Server Components、Middleware 這些都是 Function，會計入 [Fluid Compute](https://vercel.com/docs/functions/usage-and-pricing) 的費用。Build Logs 也能看到類似的報表，可以由此進行改善：

![build logs](https://assets.blog.pan93.com/how-nextjs-rendering-works/build-logs.avif)

## Next.js 的 PPR

Next.js 還有一個實驗性的魔法。通常來講伺服器端渲染都是「一次回傳完全 rendered 的結果」。如果你的 Server Components 裡面有用到 `cookies()` 這種 Next.js 無法事先渲染的動態 API，那頁面的載入就會很取決於這個 Server Components 渲染的速度。如果這個用到 `cookies()` 的 Server Components，需要 `fetch()` 幾個花費 5 秒鐘才能完成的 API，那使用者就得等上 5 秒鐘才能看得到你的頁面——但這個需要花費 5 秒鐘渲染的 Server Components，可能只是網頁中微不足道的區塊。

![thinking in PPR, src: https://nextjs.org/docs/app/getting-started/partial-prerendering](https://assets.blog.pan93.com/how-nextjs-rendering-works/vercel-assets/thinking-in-ppr.webp)

在 CSR 中這個事情很好處理：我們只要把需要花很久的元件，先顯示一個 Fallback 的 Loading 狀態，等到資料拉取完成之後再顯示渲染完成的資料就好。在 Suspense 的世界中，我們則會使用 [Suspense](https://react.dev/reference/react/Suspense) 來包住需要非同步渲染的元件，渲染完成之前顯示 `fallback`，`children` 渲染完成後再補回去顯示。但在 Prerendering 的 SSR 中，這個就不太好處理：我們要怎麼先回傳 Fallback 的 HTML，等到渲染完成之後再回傳 Children 呢？

Next.js 實驗性的 PPR 就在嘗試解決這個問題。假如你啟用了 [PPR (Partial Prerendering)](https://nextjs.org/docs/app/getting-started/partial-prerendering)，那就會看到 prerendering HTML 以一種很特別的形式回傳回來：你會看到 App 的框架和 Fallback 先回傳回來，然後後面組了一些好像不太合 HTML 規範的資料和一些 script，整個請求需要很長一段時間才會結束，但頁面卻不用等整個 HTML 下載完就能進行渲染。

簡單來說，PPR 運用了 [HTML 串流 (Streaming)](https://nextjs.org/learn/dashboard-app/streaming) 技術：

![server rendering with streaming, src: https://nextjs.org/docs/app/getting-started/partial-prerendering](https://assets.blog.pan93.com/how-nextjs-rendering-works/vercel-assets/server-rendering-with-streaming.webp)

- Next.js 的伺服器先回傳 App 外殼 (Shell)，通常是靜態渲染的部分，以及動態渲染外層 Suspense 的 fallback。
  ![PPR block](https://assets.blog.pan93.com/how-nextjs-rendering-works/ppr-block.avif)
- 接下來等待動態元件完成渲染後，就把對應的 HTML 回傳回來，交給瀏覽器做 hydrate。
  ![PPR injections](https://assets.blog.pan93.com/how-nextjs-rendering-works/ppr-injection.avif)
- 這樣子使用者可以先看到 App 框架，動態元素則是顯示 Suspense 的 fallback 後，才透過 Streaming HTML 的方式回傳回來。

這點又比傳統的模板語言更上一層樓了（或者說更複雜了）。

## Next.js 是不是 vendor lock-in

> next.js 感覺比較像是要把開發者綁在自己的生態系內，讓開發者對於 vercel 的服務產生依賴，內部實作許多其他外部套件的功能 [[MozTW](https://t.me/moztw_general/1/263052)]

順帶一提，因為這些事情 [Docker 裡面也能做](https://nextjs.org/docs/app/getting-started/deploying#docker)，似乎也沒有很明顯的 vendor lock-in。舉例來說，ISR 你可以自己寫一個 [快取儲存區](https://nextjs.org/docs/app/guides/self-hosting#caching-and-isr)、PPR 和 Middleware 這些也是 Docker [裡面支援的](https://nextjs.org/docs/app/guides/self-hosting#middleware)。不過 Next.js 的 Docker image 資源使用量真的很大⋯⋯

## 更新日誌和勘誤

- 感謝 OrkWard 提出文章中「[動態路由](https://nextjs.org/docs/app/api-reference/file-conventions/dynamic-routes)」、「[動態 API](https://nextjs.org/docs/app/getting-started/partial-prerendering#dynamic-rendering)」混用的問題，文章已經針對 Next.js 的正確概念整篇重寫：<https://t.me/c/1066867565/2166664>
- 使用了 Claude 4 Sonnet Thinking 和 Gemini 2.5 Pro 模型進行初步的 peer reviewing。
