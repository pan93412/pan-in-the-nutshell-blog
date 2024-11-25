---
date: '2024-11-24T00:56:03+08:00'
title: '搭建一個有圖床和統計功能的 Blog'
categories: ['Developments']
tags: ['blog', 'zeabur', 's3', 'hugo']
---

## tl;dr

- 使用 Zeabur 部署 Blog
- 使用 Cloudflare R2 當圖床，Cloudreve 管理
- 使用 Umami 進行網站資料統計
- 選擇性部署 CodiMD / Hedgedoc 方便行動裝置編輯

## 常見的方案有什麼問題

純粹的 GitHub Pages 架設靜態 Blog，搭配 Google Analytics（或 Cloudflare Web Analytics）雖然是最便宜的選擇，但對於媒體管理上和草稿編輯還是不太方便。

**統計工具方面**，除了 Google Analytics 之外，幾乎沒有什麼功能特別強大，可以看單一使用者流向的統計工具。

**圖床方面**，媒體放在 GitHub repository 上會造成 repo 迅速膨脹，但又擔心其他免費圖床如 Imgur 會倒閉。而且 Imgur 不能上傳圖片以外的資源！🥺

**草稿編輯方面**，GitHub 沒有提供一個比較好的 Markdown 書寫工具，要獲得好的書寫體驗，就需要在電腦上用 VS Code 等等的 Markdown editor 書寫，局限了在手機上完成草稿或者編輯文章的能力。

## 怎麼解決？

作為一個 self-hosted 跟偏好 Cloud Native 方案的使用者，我會這麼規劃我的 blog：

![structure](https://assets.blog.pan93.com/create-a-blog-with-zeabur/structure.png "Structure")

這次我使用的 Blog SSG 是 **Hugo**，速度確實快，而且 template 比 Hexo 簡潔和清晰一點。

統計工具用 **Umami**，是一個隱私優先、可以自建的統計服務，可以看即時資料、單一使用者瀏覽動態，以及過往資料比較。而且 Umami 支援多網站、多帳號，所以可以把各種網站的統計資料放入同一個 Umami 實體裡面，而且統計資料基本上可以持續儲存，不用擔心 180 天後資料消失的問題。

圖床用 **支援 S3 API 為基礎的儲存空間** 搭建。S3 API 基本上是檔案儲存 API 的一種業界標準：

> The broad adoption of Amazon S3 and related tooling has given rise to competing services based on the S3 API. These services use the standard programming interface but are differentiated by their underlying technologies and business models. [^1]

[^1]: <https://en.wikipedia.org/wiki/Amazon_S3>

這裡使用 **Cloudflare R2**，它的輸出流量免費、寫入和儲存成本遠低於 Amazon S3，同時有著和 S3 幾乎一致的介面。不過純 R2 的 UI 稱不上 user-friendly，所以還會需要套一個 S3 的管理介面。這裡我選 Cloudreve，介面簡單、舒適，操作迅速，而且支援 Cloudflare R2。

而 Draft 方面，可以自己搭建一個 [CodiMD](https://zeabur.com/zh-TW/templates/TAZC8W) 或 [HedgeDoc](https://zeabur.com/zh-TW/templates/APYG3M)，也可以用 Craft、Notion 之類的筆記軟體寫上之後手動貼上 Markdown。不過我目前沒有這個需求，所以這篇文章只會簡單帶過。

## 將 Hugo 部署上 Zeabur

這部分主要看你的需求，留在 GitHub Pages 也行。我傾向統一管理 blog 的各種服務，同時有更大的 web server 自訂性，所以部署在 Zeabur 上。

Hugo 的搭建這部分不詳細贅述，可以參考現有的 Hugo 文件或者是別人的 [Blog Repository](https://github.com/pan93412/pan-in-the-nutshell-blog) 進行設定。

Zeabur 部署的話，可以用下面的推薦碼註冊一個帳號。

> 注意 Serverless 方案（免費帳號）的部署執行時間有限制，如果想要評估久一點可以綁卡升級，我們支援升級後不滿意退款的機制。另外 Zeabur 也有提供一些 [獎勵和贊助方案](https://zeabur.com/docs/zh-TW/billing/reward) 可以參考。

[![Deployed on Zeabur](https://zeabur.com/deployed-on-zeabur-dark.svg)](https://zeabur.com?referralCode=pan93412&utm_source=pan-blog)

建立專案之後，選擇「建立服務」，選擇 GitHub 的 blog repository。

![create service > deploy GitHub repository](https://assets.blog.pan93.com/create-a-blog-with-zeabur/zeabur-blog-select-github-repository.png)

Zeabur 會自動判斷出 Hugo 專案，直接按下「部署」即可。

![build plan preview of my Hugo blog](https://assets.blog.pan93.com/create-a-blog-with-zeabur/zeabur-blog-plan-meta-preview.png)

部署期間也可以綁定一下網域～

![networking configuration of my blog](https://assets.blog.pan93.com/create-a-blog-with-zeabur/zeabur-blog-binding-domain.png)

顯示「運作中」就代表部署成功，可以直接打開網域看看新架設好的 Blog 了。日後往 GitHub 推送 commits 也會自動更新。

![running status of blog service](https://assets.blog.pan93.com/create-a-blog-with-zeabur/zeabur-blog-status-running.png)

## 搭建圖床

### 設定 Cloudflare R2

「部署 Blog」說實話是整個架設流程裡面最無聊的部分了。我們接著來配置一個自己專享，而且還內建 CDN 的圖床吧！

![Cloudflare R2 的管理介面](https://assets.blog.pan93.com/create-a-blog-with-zeabur/r2-my-bucket.png)

首先我們先設定 Cloudflare R2。R2 會需要你將網域綁定在 Cloudflare 上，綁定完成後，點開 Cloudflare 帳號的「R2 物件儲存空間」，就能建立貯體（bucket）了。

![cloudflare r2 overview](https://assets.blog.pan93.com/create-a-blog-with-zeabur/r2-overview.png)

建立 bucket 的流程不多贅述，不過有幾個點要注意：

- **位置**：Cloudflare 的 R2 預設就會搭配全球 CDN，所以放在你上傳比較快的地方就行（或者是等會 Cloudreve 部署的區域）。我這裡選擇亞太地區。
- **儲存體類別**：考慮到圖床是一個 Read > Write 的服務，所以儲存體要選擇「標準」，這樣效能會比較好一些。

![creating r2 bucket](https://assets.blog.pan93.com/create-a-blog-with-zeabur/r2-new-bucket.png)

既然是圖床，那肯定是需要從外部存取的。打開建立好 bucket 的「設定」，並且在「公開存取」這裡填寫一個自己方便的域名。

![public access domain of bucket](https://assets.blog.pan93.com/create-a-blog-with-zeabur/r2-public-access.png)

然後我們還會需要設定一個 Cloudreve 管理介面用來上傳的 API token。打開概觀頁面的「管理 R2 API 權杖」，然後建立一個「系統管理員讀寫」級別的 token。

> 「系統管理員讀取和寫入」主要是方便 Cloudreve 日後設定 CORS 設定，**這個 scope 日後可以限縮到特定 bucket 的「物件讀取和寫入」**。

![generating api token for Cloudreve](https://assets.blog.pan93.com/create-a-blog-with-zeabur/r2-access-token.png)

建立完成之後儲存 API 的識別碼和秘密存取金鑰，稍後 Cloudreve 會用到。

### Cloudreve

打開 Zeabur 控制台，建立一個新專案，地區選擇離你 bucket 最近的區域（比如亞太地區就選擇 Tokyo 或 Hong Kong）。

然後「建立專案」，選擇「模板部署」，搜尋並部署 Cloudreve。其中會引導你建立免費網域，這部分可以先隨便寫一個，日後可以再換綁到自己的網域上。

![deploying cloudreve](https://assets.blog.pan93.com/create-a-blog-with-zeabur/cloudreve-create-service.png)

等到部署完成，服務顯示「運作中」後，打開「記錄」找到登入的帳號密碼，然後打開剛才建立好的網址登入到 Cloudreve 裡頭。

![find cloudreve login credential](https://assets.blog.pan93.com/create-a-blog-with-zeabur/cloudreve-find-credential.png)

登入完成後可以先到齒輪圖示的「個人設定」更改密碼和兩步驗證。完成之後點開個人頭貼，選擇「管理面板」：

> 點開後可能會看到「更新網域」的詢問對話框。直接讓 Cloudreve 更新就可以了～

![link to cloudreve admin panel](https://assets.blog.pan93.com/create-a-blog-with-zeabur/cloudreve-open-admin-panel.png)

接著選擇「儲存策略」。這裡點選「新增儲存策略」，[然後按照官方的教學設定 Cloudflare R2](https://docs.cloudreve.org/use/policy/s3#cloudflare-r2)。

![cloudreve storage policy overview](https://assets.blog.pan93.com/create-a-blog-with-zeabur/cloudreve-storage-policy.png)

注意幾個事項：

1. **參考「Public Bucket」的部署流程**，CDN Domain 則選擇你在 R2 設定的「公開存取 Domain」。
2. CORS 讓 Cloudreve 設定即可。Cloudreve 設定完成 CORS 後，你就可以把 token 改回讀寫特定 bucket 的權限了。
3. 「上傳路徑」可以更改成比較好懂的檔名。我自己是維持原始的資料夾和檔名結構。

![cloudreve upload path configuration](https://assets.blog.pan93.com/create-a-blog-with-zeabur/cloudreve-storage-upload-path.png)

接著是將預設上傳的位置改成 R2。Cloudreve 預設是存在 Zeabur 裡面，不能利用到 R2 免流量和低廉儲存費用的優勢，所以我們改成剛才設定好的 R2 上。打開「用戶組」，將 Admin 的「儲存策略」改成剛才設定的 R2 策略。「初始容量」也可以視情況更改，基本上 R2 到 10GB 儲存都還是免費的。

![CleanShot 2024-11-25 at 00.17.08@2x.png](https://assets.blog.pan93.com/create-a-blog-with-zeabur/cloudreve-user-group-modification.png)

到這裡管理介面就設定完成了。你可以視情況在「參數設定」的「註冊與登入」**禁止新使用者註冊**防止其他人把你的管理介面當免費圖床使用。「站點訊息」也可以視情況調整，基本上無妨。

Cloudreve 允許你上傳整個資料夾（並行上傳）和任何檔案，使用體驗接近 Google Drive，可以自行摸索～如果要取得直通 CDN 的網址，有兩種方式：

1. 使用 Cloudreve 的「獲取外鏈」功能。使用者打開圖片，會先進到你的 Cloudreve 再轉導到 R2。好處是網址經過 Cloudreve 整理，不會直接暴露你儲存圖片的資料夾結構，而且 Cloudreve 也允許你一次取得資料夾底下的所有外鏈；壞處是會多一個 301 轉導的流量費用和連線延遲。
    ![get external link from Cloudreve](https://assets.blog.pan93.com/create-a-blog-with-zeabur/cloudreve-get-external-link.png)
2. 點選「下載」功能，直接打開 R2 上的實體路徑。好處是沒有額外流量費用，直接連線，最大善用 Cloudflare 的 CDN；壞處是取得比「獲取外鏈」麻煩一點。

我自己會偏好先用 Cloudreve 的「獲取外鏈」功能取得所有圖片的連結，然後用瀏覽器點開讓瀏覽器跳轉，換成 R2 的直接連結。無論你最後選擇什麼連結，都可以用 Markdown 語法貼上連結：

```markdown
![alt text](https://my-r2-domain.blog.pan93.tld/your-path.png)
```

### 設定 Cloudflare 的防盜連結 (hotlink)

> ⚠️ Cloudreve 的「獲取外鏈」可能會導致 `Referer` 發生改變導致 hotlink 失效。如果需要這個功能，**建議先改回直接連結**。

點開 Cloudflare 網域的「Scrape Shield」，打開「Hotlink 保護」即可。

![cloudflare: enable scrape shield](https://assets.blog.pan93.com/create-a-blog-with-zeabur/cf-enable-scrape-shield.png)

這樣其他網站就無法引用你的圖片了。

![cloudflare: hotlink protection demo](https://assets.blog.pan93.com/create-a-blog-with-zeabur/cf-hotlink-protection.png)

## 設定統計工具

同樣開一個新的 Project（也可以沿用之前的），然後選擇模板部署 Umami。

![create Umami on Zeabur](https://assets.blog.pan93.com/create-a-blog-with-zeabur/umami-create-service.png)

部署完成、綁定完網域後，[輸入 `admin` 和 `umami` 帳密登入 Umami](https://umami.is/docs/login)。

接著打開 Settings > Website。Name 隨意輸入，Domain 輸入你 blog 的網址。

![umami-add-website.png](https://assets.blog.pan93.com/create-a-blog-with-zeabur/umami-add-website.png)

建立完成後點選 Edit > Tracking code，將這段貼到你 Blog 的 Header 裡面。以 Hugo 的 PaperMod 主題為例，在 Blog 根目錄建立 `layouts/partials/extend_head.html` 檔案並貼上這段即可。

![umami tracking code](https://assets.blog.pan93.com/create-a-blog-with-zeabur/umami-tracking-code.png)

貼完並重新 build Hugo 後，點開「Dashboard」就能看到自己造訪的資料了。

![umami statistics](https://assets.blog.pan93.com/create-a-blog-with-zeabur/umami-stats.png)

Umami 的操作，可以參考 [Umami 的官方文件](https://umami.is)。簡而言之：

- Overview 可以看到訪客的大致來源、頁面、瀏覽器版本和國家
- Events 可以看到每個去識別化使用者的造訪情況
- Sessions 可以看到造訪高峰時段和訪客畫像
- Realtime 顧名思義就是即時查看瀏覽情況，比 GA 即時許多。
- Compare 可以看到這週和上週的造訪情況。

## 設定 Markdown 線上編輯器

你依然可以用 Zeabur 一鍵部署這些編輯器。

- [CodiMD 的一鍵部署頁面](https://zeabur.com/zh-TW/templates/TAZC8W)。CodiMD 支援 GitHub 同步，功能跟上一代的 HackMD 很接近，但維護頻率不及 Hedgedoc。
- [Hedgedoc (v2) 的一鍵部署頁面](https://zeabur.com/zh-TW/templates/APYG3M)。Hedgedoc 是 CodiMD 早期的 fork 版本，維護頻繁，但是不支援 GitHub 同步。

下面是 Hedgedoc 的預覽截圖，可以參考：

![hedgedoc-preview.png](https://assets.blog.pan93.com/create-a-blog-with-zeabur/hedgedoc-preview.png)

## 計費參考

我的 Umami 和 Cloudreve 是多個服務共用的，而且都有各自的流量，所以資料可能不太準⋯⋯

> Zeabur 有更新計費方式。實際成本可能會和這裡列出的有稍微差距（可能更低）。

- Blog 本身的 Caddy：一天約 USD$0.02，一個月約 USD$0.61。
- Cloudflare R2：免費範圍。
- Cloudreve：一天約 USD$0.03，一個月約 USD$0.91。
- Umami：一天約 USD$0.07（似乎記憶體佔用比較大，可以設定 Auto Restart 定期排空記憶體），一個月約 USD$2.13。

加總起來大約是 USD$3.65，基本上能被 Zeabur 的 Dev Plan 抵扣額度完全 cover。另外 Umami 實際上應該會更低，只是我好幾天沒重開過了 🥺

## 結論

上面的 Blog 部署主要圍繞 Zeabur 出發，導入了這些服務：

- Cloudflare R2：作為圖床的儲存空間，並內建使用 Cloudflare 當 CDN 的功能
- Cloudreve：方便管理圖床
- Umami：方便統計網站瀏覽情況
- CodiMD 和 Hedgedoc：允許線上編輯

實際上 Zeabur 還有很多很不錯用的 self-hosted 服務。之後或許也會寫幾篇 solution-based 的文章讓大家參考 🤔