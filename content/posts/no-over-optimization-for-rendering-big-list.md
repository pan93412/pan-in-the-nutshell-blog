+++
date = '2025-09-21T13:20:47+08:00'
title = '大量渲染不應該做太多無意義的最佳化'
categories = ["comments"]
tags = ["react", "virtual dom", "virtual list", "memo"]
+++

[![Original Post](https://assets.blog.pan93.com/no-over-optimization-for-rendering-big-list/thread-original-post.avif)](https://www.threads.com/@0mgdayvusb8oy/post/DOLk34Ykv-p?xmt=AQF0ri-Ztw3Ofz9nb8FjUyki5lWSTc9FYgDZauUA5daUDg)

`renderList` ……？討論串很多人針對這篇評論給出很多不同建議了，我想給幾個我還沒看到的 point。

關於「元件最佳化」這點：React 背後有 [Virtual DOM](https://hackmd.io/@Heidi-Liu/virtual-dom)，所以你 render 100K 遍一樣的 list，相同 key、相同內容的 element 是不會在實體 DOM 層被重建的；`useMemo` 如果 list 的內容會一直變動，反而可能會造成 memory leak。如果是新專案，我通常推薦不手寫 `useMemo`/`useCallback`/`memo` 等等的記憶函式，直接讓 [React Compiler](https://react.dev/learn/react-compiler) 分析哪些東西該被 memorized。

既然 React 不會一直重建那 100K 個 DOM 元素，那為什麼在非常大的 list 下還是會卡？因為瀏覽器不擅長處理過多 DOM 元素。這種情況下應該要用 Virtual List 來只 render 可見元素，壞消息是他有 trade off——無法選取超出 viewport 的內容、瀏覽器內建搜尋不能搜尋全文。所以就算 GitHub 那種程式碼 view 很適合 Virtual List，但實際上都沒有實作。
