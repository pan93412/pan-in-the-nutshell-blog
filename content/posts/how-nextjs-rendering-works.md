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

## Next.js 的 Client Components 是怎麼做 SSR 的？

接下來回頭看 Next.js。如果只看 Client Component，會相對簡單一點，可以分成 SSG (+ISR)、SSR、CSR 的部分[^1] [^2] [^3]：

- 如果是靜態路由（瀏覽器輸入什麼，產物都相同），Next.js 會在編譯期 (build time) 產生頁面，這叫 SSG (Static Site Generation)。另外還有一個叫 ISR (Incremental Static Regeneration) 增量渲染，如果 SSG 的頁面到期了，或者是遇到 SSG 不能推斷的頁面，就重新渲染。
  ![ISR](https://assets.blog.pan93.com/how-nextjs-rendering-works/isr.avif)
- 如果是動態路由（比如 `?q=xxx` 或 `/products/[id]`），Next.js 會在存取當下 (request time) 先行渲染不需要瀏覽器的部分（e.g. useEffect），也就是 SSR (Server-side Rendering) 然後留下一些 metadata 指示前端做 hydrate。
  ![SSR hydrate](https://assets.blog.pan93.com/how-nextjs-rendering-works/ssr-hydrate.avif)
- 如果你在 `useEffect` 裡面做了 `fetch`，部分頁面元素是在瀏覽時才渲染，那就是 CSR (Client-side Rendering)。
  ![CSR (fetch)](https://assets.blog.pan93.com/how-nextjs-rendering-works/csr-fetch.avif)

從這點來看，Next.js 只是把一些瀏覽器該做的事情，放到伺服器上運作，來改善 SEO 和瀏覽器渲染的效能。你也可以大膽猜測，如果沒有 Next.js，你的那堆元件在瀏覽器上依然還是可以正常渲染——舉例來說，如果你用 `next/dynamic` 搭配 `ssr: false` 把整個元件變成不 SSR 的懶載入，那元件就不會在伺服器上渲染[^4]。

[^1]: <https://nextjs.org/learn/pages-router/data-fetching-pre-rendering>
[^2]: <https://nextjs.org/learn/pages-router/data-fetching-two-forms>
[^3]: <https://nextjs.org/docs/pages/building-your-application/rendering/client-side-rendering>
[^4]: <https://nextjs.org/learn/seo/dynamic-import-components>

## Next.js 的 Server Components

但如果我們提到 Server Components 呢？Server Components 其實更像是 Express.js 這種傳統後端，可以去讀 Cookies、Headers，甚至可以去和資料庫互動。也因此，Server Components 的元件是純伺服器端渲染的，當然裡面可以去套 Client Component 去增加其可互動性，但這時候沒有 Next.js 的伺服器，元件就渲染不起來了。

到這裡其實都挺符合常識的：SSR 就是事先渲染、CSR 就是日後補上。但 Next.js 的魔法這時候就開始了：如果你有一個 Client Components 裡面有一個 Server Components，那會發生什麼事呢？答案是 Client Components 會去 fetch Server Components 的 RSC 回來[^5]。你可以簡單把 RSC 想像成 React 的 HTML，反正就這點來看，他有點像是 CSR 了 SSR 的內容 🫠。

![RSC](https://assets.blog.pan93.com/how-nextjs-rendering-works/rsc.avif)

[^5]: <https://nextjs.org/docs/app/getting-started/server-and-client-components#on-the-client-first-load>

### Next.js 怎麼運用這些渲染方式

下次你可以寫幾個 Server Components + Client Components + SSR 看看。如果沒有加入什麼動態元素（比如 cookies），你可能會看到：

- 大部分的靜態頁面和 Server Components 都會被 SSG。
- 有些動態路由（比如 `/posts/[id]`），可以在 Vercel 中看到被 SSR 後進入 ISR cache 了。這樣下次瀏覽就會是 SSG，不用再重新渲染一遍。
- 如果你的 Server Components 是動態的，且在 Client Components 裡面，你會看到有一些 `?rsc` 的 fetch 請求被打出去，這裡有點像是 CSR。
- 如果你的 Client Components 裡面有 `fetch` 東西後進行渲染，那通常都是 CSR。如果有搭配 streaming SSR 的框架 (e.g. [@apollo/client-react-streaming](https://github.com/apollographql/apollo-client-integrations/tree/main/packages/client-react-streaming))，那也可能是 SSR 的。

實務上 Next.js 可能會運用非常多的渲染方式，在 Vercel 的 Observability 中可以分開來看：

![rendering methods](https://assets.blog.pan93.com/how-nextjs-rendering-works/rendering-methods.avif)

順帶一提，如果要省 Vercel 帳單的話，盡量不要做動態 (dynamic) 路由 (static 愈多愈好，Server Component 也是一種 function)。Build Logs 也能看到類似的報表，可以由此進行改善：

![build logs](https://assets.blog.pan93.com/how-nextjs-rendering-works/build-logs.avif)

## Next.js 的 PPR

Next.js 還有一個實驗性的魔法：通常來講 SSR 都是一個請求打回來，也就是一次就是完全 rendered 的結果。所以如果你在 Server Components 裡面用了動態元素，即便有沒有套上 Suspense 都不能 SSG，每次瀏覽都必須等待帶有動態元素的元件 rendered 完後才能看到頁面。

但假如你啟用了 [PPR (Partial Prerendering)](https://nextjs.org/docs/app/getting-started/partial-prerendering)，那就會看到 initial HTML 以一種很特別的形式回傳回來：你會看到 App Frame 先回傳回來，然後後面組了一些好像不太合 HTML 規範的資料和一些 script，整個請求需要很長一段時間才會結束，但頁面卻不用等整個 HTML 下載完就能進行渲染。

![PPR block](https://assets.blog.pan93.com/how-nextjs-rendering-works/ppr-block.avif)
![PPR injections](https://assets.blog.pan93.com/how-nextjs-rendering-works/ppr-injection.avif)

簡單來說，PPR 運用了 [HTML 串流 (Streaming)](https://nextjs.org/learn/dashboard-app/streaming) 技術：

- Next.js 的伺服器先回傳 App 外殼 (Shell)，可能是已經 SSG 完的部分。
- 接下來動態元素的元件開始渲染。渲染完一個，就把對應的渲染結果、hydrate 資料和插入方式回傳回來。
- 瀏覽器持續 hydrate 伺服器渲染好的 HTML。
- 這樣子使用者可以先看到 App 框架，動態元素則是顯示 Suspense 的 fallback 後，才透過 Streaming HTML 的方式回傳回來。

這點又比傳統的模板語言更上一層樓了（或者說更複雜了）。

> next.js 感覺比較像是要把開發者綁在自己的生態系內，讓開發者對於 vercel 的服務產生依賴，內部實作許多其他外部套件的功能 [[MozTW](https://t.me/moztw_general/1/263052)]

順帶一提，因為這些事情 [Docker 裡面也能做](https://nextjs.org/docs/app/getting-started/deploying#docker)，似乎也沒有很明顯的 vendor lock-in（除了 Next.js 的 Docker image 資源使用量真的很大）。
