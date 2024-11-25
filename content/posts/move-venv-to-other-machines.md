---
date: '2024-11-25T13:14:26+08:00'
title: '把 Python 的 venv 移到其他機器'
tags: ['python', 'venv']
categories: ['Developments']
draft: true
---

## 前言

**要怎麼在不重新下載東西的情況下，把一整包 code 包含依賴本身，移到新的地方呢？**

舉個之前遇到的例子：我想在學校部署一個用 Python Streamlit 框架寫的程式碼評測工具，但是在測試部署時發現一個非常頭痛的問題：學校的連外網路特別慢，導致 pip 幾乎無法正常完成安裝。在嘗試了 `pip download` 以及一些打包方案後，發現 Streamlit 的執行最終都會缺幾個元件導致啟動失敗，最終都還是需要用 `pip install` 補全依賴。最後我想到一個 workaround：拿其他同學的 Windows 筆電先把程式準備好，再把準備好的程式複製到新的地方。

![The obstacle when installing with Pip](https://assets.blog.pan93.com/move-venv-to-other-machines/pip-install-obstacle.png)

不過要怎麼「準備好程式」然後「複製到新的地方」呢？

Python 不像 Go 和 Rust 可以編譯成靜態的執行檔，也不像 Node.js 和 PHP 有著各個專案獨立的 `node_modules` 或 `vendor` 資料夾，可以搬到其他地方而保持程式的依賴正常運作。不過 Python 有個很類似 `node_modules` 的東西——Virtualenv，搞不好我們真的能像 `node_modules` 一樣直接把整組專案複製到其他電腦上，專案就能跑了。

![Concept about making venv as an application bundle](https://assets.blog.pan93.com/move-venv-to-other-machines/venv-bundle-concepts.png)

但是 Stack Overflow 的文章又提到「venv 通常不能直接複製到其他電腦上」。

> **In general you can't copy virtual environments anywhere, Docker or otherwise.** They tend to be tied to a very specific filesystem path and a pretty specific Python installation. If you knew you had the exact same Python binary, and you copied it to the exact same filesystem path, you could probably COPY it in as-is, but the build system would be extremely fragile. [^1]

[^1]: https://stackoverflow.com/questions/56825265/how-to-use-a-python-virtual-environment-copied-to-a-docker-container

不過究竟「特定的檔案系統路徑」和「特定的 Python 安裝」是在哪裡被寫死的，又該怎麼突破這些限制來讓我可以把東西複製到新的地方呢？今天的講題主要就是分享怎麼克服 `venv` 的限制，成功在不執行 `pip install` 的情況下，把依賴複製到其他電腦上。

> 這個是之前 PyCon TW 2024 的講題，把逐字稿重新排版和整理成 blog。

## virtualenv 是什麼？

virtualenv 是一個可以將依賴環境與系統套件隔開的工具，它會建立一個獨立的資料夾，存放你在 virtualenv 安裝的依賴。舉例來說，你在 virtualenv 環境用 pip 安裝 `numpy` 這個套件，pip 會把依賴放在獨立的資料夾裡面，這樣系統的環境就不會受到影響。不過反過來就不一定，比如你在系統環境安裝 `openai`，virtualenv 環境基本上還是可以 `import openai` 的。

![Python venv concept](https://assets.blog.pan93.com/move-venv-to-other-machines/python-venv-concept.png)

它的建立方式其實相當簡單，甚至不用你安裝任何套件。你只需要輸入：

```bash
python3 -m venv myvenv
```

你的虛擬環境就建立完成了。至於切換進去虛擬環境，就只需要執行 myvenv 裡面的 `bin/activate` 檔案就好：

```bash
. ./myvenv/bin/activate
```

此時我們安裝任何依賴，都會存放在 myvenv 的 `lib` 和 `bin` 裡頭。

```bash
(myvenv) pan93412@ubuntu:~/myvenv$ pip install numpy
Collecting numpy
  Using cached numpy-2.1.1-cp312-cp312-manylinux_2_17_aarch64.manylinux2014_aarch64.whl.metadata (62 kB)
Using cached numpy-2.1.1-cp312-cp312-manylinux_2_17_aarch64.manylinux2014_aarch64.whl (13.7 MB)
Installing collected packages: numpy
Successfully installed numpy-2.1.1
(myvenv) pan93412@ubuntu:~/myvenv$ ls lib/python3.12/site-packages/
numpy/                 numpy-2.1.1.dist-info/ numpy.libs/            pip/                   pip-24.0.dist-info/
```

退出虛擬環境之後，可以發現到 numpy 並沒有影響到系統的 Python 依賴環境：

```bash
(myvenv) pan93412@ubuntu:~/myvenv$ deactivate
pan93412@ubuntu:~/myvenv$ python3
Python 3.12.3 (main, Jul 31 2024, 17:43:48) [GCC 13.2.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import numpy
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
ModuleNotFoundError: No module named 'numpy'
>>>
```

## virtualenv 是怎麼隔離環境的？

virtualenv 是怎麼讓你的 Python 乖乖把所有產物都放在你建立的虛擬環境裡面呢？或許我們能先做出這幾個假設。

1. virtualenv 對 venv 的 Python 做了 patch，讓他只會往固定的資料夾讀取套件
2. activate 腳本對 Python 載入了特殊的環境變數，引導 python 讀取特定目錄的資料夾
3. venv 資料夾中有一個檔案，引導這個目錄下的 python3 讀取特定的參數
4. Python 會偵測上層的目錄結構，判斷是否符合一個 /usr/lib/python3 的結構，如果是就當作 venv 看待。

首先我這裡建立一個虛擬環境，然後打開他的根目錄和 bin 目錄：

```bash
pan93412@ubuntu:~/test$ ls
bin  etc  include  lib  lib64  pyvenv.cfg  share
pan93412@ubuntu:~$ ls -al test/bin
total 48
drwxr-xr-x 1 pan93412 pan93412  164 Sep  9 17:40 .
drwxr-xr-x 1 pan93412 pan93412   56 Sep  9 17:40 ..
-rw-r--r-- 1 pan93412 pan93412 9033 Sep  9 17:40 Activate.ps1
-rw-r--r-- 1 pan93412 pan93412 2026 Sep  9 17:40 activate
-rw-r--r-- 1 pan93412 pan93412  913 Sep  9 17:40 activate.csh
-rw-r--r-- 1 pan93412 pan93412 2192 Sep  9 17:40 activate.fish
-rwxr-xr-x 1 pan93412 pan93412  236 Sep  9 17:40 pip
-rwxr-xr-x 1 pan93412 pan93412  236 Sep  9 17:40 pip3
-rwxr-xr-x 1 pan93412 pan93412  236 Sep  9 17:40 pip3.12
lrwxrwxrwx 1 pan93412 pan93412    7 Sep  9 17:40 python -> python3
lrwxrwxrwx 1 pan93412 pan93412   16 Sep  9 17:40 python3 -> /usr/bin/python3
lrwxrwxrwx 1 pan93412 pan93412    7 Sep  9 17:40 python3.12 -> python3
```

可以發現到他的 `python3` 是直接符號連結到目前安裝的 Python 版本上，而 `pip` 的檔案也和你的 `pip` 執行檔很相似：

```diff
--- /usr/bin/pip	2024-03-06 01:55:54.000000000 +0800
+++ /home/pan93412/test/bin/pip	2024-09-09 17:40:30.250961589 +0800
@@ -1,8 +1,8 @@
-#!/usr/bin/python3
+#!/home/pan93412/test/bin/python3
 # -*- coding: utf-8 -*-
 import re
 import sys
 from pip._internal.cli.main import main
-if __name__ == "__main__":
-    sys.argv[0] = re.sub(r"(-script\.pyw|\.exe)?$", "", sys.argv[0])
+if __name__ == '__main__':
+    sys.argv[0] = re.sub(r'(-script\.pyw|\.exe)?$', '', sys.argv[0])
     sys.exit(main())
```

除了引號不同之外，只差在 1 個地方：Shebang 不同。一個是指向 Virtualenv 裡面那個等於 `/usr/bin/python3` 的執行檔，另一個則是直接寫上 `/usr/bin/python3`。此時大家可能就會想到「如果我不執行 `activate`，直接輸入 `./bin/python3`，是不是也可以匯入虛擬環境中的相依套件？」正好我在虛擬環境中有安裝 streamlit，剛好可以用來比較是否可以取用虛擬環境的套件。

```bash
pan93412@ubuntu:~/test$ python3 -c "import streamlit"
Traceback (most recent call last):
  File "<string>", line 1, in <module>
ModuleNotFoundError: No module named 'streamlit'
pan93412@ubuntu:~/test$ ./bin/python3 -c "import streamlit"
pan93412@ubuntu:~/test$
```

實際測試過後，發現真是如此！所以 virtualenv 並沒有對Python 做修補，不用 `activate` 腳本也可以做到 virtualenv 的效果。

那究竟是什麼造就了 virtualenv 呢？實際上剛才的 ls 中就有端倪了：`pyvenv.cfg`。

```bash
pan93412@ubuntu:~/test$ mv pyvenv.cfg pyvenv.cfg.1
pan93412@ubuntu:~/test$ ./bin/python3 -c "import streamlit"
Traceback (most recent call last):
  File "<string>", line 1, in <module>
ModuleNotFoundError: No module named 'streamlit'
```

如果把這個檔案換成其他名字，`./bin/python3` 就也失去了可以讀取 venv 下套件的能力，即使呼叫 `./bin/activate` 亦同。實際上這個行為是可以在 PEP 405 中讀到的。簡而言之，`pyvenv.cfg` 這個檔案是 Python 虛擬環境的設定檔，實際上也是 Python 決定是否是虛擬環境的檔案，只要 Python 3 的執行檔能 **往上** 找到 `pyvenv.cfg`，那就會將這一整包視為 Python 的虛擬環境[^2]。

[^2]: <https://peps.python.org/pep-0405/#specification>

既然重點只是那個執行檔，`./bin/activate` 的用意為何？除了提供「目前在哪個 virtual environment 之下」的資訊之外，根據 Python 官方 venv 文件的說明，它最大的作用就是往 `PATH` 補上目前 `bin` 的路徑，這樣你打 `python3` 時就會優先啟動虛擬環境下的 `./bin/python3`[^3]。

[^3]: <https://docs.python.org/3.13/library/venv.html#how-venvs-work>

## virtualenv 為什麼不能直接遷移？

其實剛才的案例中，我們就找到 virtualenv 兩個寫死的地方了：**Shebang 和 Python 的執行檔路徑**。但除此之外，還有哪裡會導致 virtualenv 不能直接攜帶到同系統的環境呢？對整個資料夾掃了一下絕對路徑，發現還真的就只有 `bin` 檔案的 shebang、`activate` 跟 `pyvenv.cfg` 裡面有寫到絕對路徑而已。

其中，`pyvenv.cfg` 雖然帶有絕對路徑，但實際上整個檔案留空，Python 還是能正確辨識成虛擬環境的。下文我都會把 `pyvenv.cfg` 的資料清空，以免這裡的變數影響到測試：

```ini
home = /usr/bin
include-system-site-packages = false
version = 3.12.3
executable = /usr/bin/python3.12
command = /usr/bin/python3 -m venv /home/pan93412/test
```

當然，雖然寫死路徑的只有三個地方，但還是不能下「virtualenv 可以到處帶著走」的定論。所以我這次開了幾台虛擬機器，clone [一個 Streamlit 的範例 repo](https://github.com/daniellewisdl/streamlit-cheat-sheet) 來檢查行為。

![The VMs running Streamlit and Numpy demo](https://assets.blog.pan93.com/move-venv-to-other-machines/environment.png)

首先在 local 的 venv 的目錄執行 streamlit：

```bash
$ ./bin/python3 -m streamlit run app.py

Collecting usage statistics. To deactivate, set browser.gatherUsageStats to false.


  You can now view your Streamlit app in your browser.

  Local URL: http://localhost:8501
  Network URL: http://198.19.249.155:8501
  External URL: http://36.237.208.113:8501
```

![Streamlit demo](https://assets.blog.pan93.com/move-venv-to-other-machines/streamlit.png)

然後也測試一下需要用到 C library 的 `numpy`：

```bash
pan93412@ubuntu:~$ ./test/bin/python3
Python 3.12.3 (main, Jul 31 2024, 17:43:48) [GCC 13.2.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import numpy
>>>
```

看來都如預期般可以運作。接下來我們遷移到另一個也是用 glibc、同架構的系統——Arch Linux 上：

> 注意路徑和 Python 版本目前我都盡量維持一致，但假如路徑不同，是可以透過更改符號連結及 lib 版本來指向正確的路徑的。

![Streamlit on Arch Linux](https://assets.blog.pan93.com/move-venv-to-other-machines/streamlit-on-arch.png)

```
[pan93412@arch test]$ ./bin/python3
Python 3.12.6 (main, Sep 13 2024, 13:27:21) [GCC 14.1.1 20240507] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import numpy
>>>
```

那假如換到另一個架構的 Ubuntu 上，甚至是不同的 libc 系統（Alpine）呢？說實話就純 Python 的 _streamlit_ 來說是還能運作的。但是某些基於 C 的 library 就出現了不一樣的結果了：

```bash
pan93412@ubuntu-amd64:~/test$ ./bin/python3
Python 3.12.3 (main, Sep 11 2024, 14:17:37) [GCC 13.2.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import numpy
<略>
Importing the numpy C-extensions failed. This error can happen for
many reasons, often due to issues with your setup or how NumPy was
installed.
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "/home/pan93412/test/lib/python3.12/site-packages/numpy/__init__.py", line 119, in <module>
    raise ImportError(msg) from e
ImportError: Error importing numpy: you should not try to import numpy from
        its source directory; please exit the numpy source tree, and relaunch
        your python interpreter from there.
```

至於 macOS 和 Windows 就更不用說了，就我測試下來也是 Python library 能執行、但 C library 不能執行的狀態。

## virtualenv 在環境相同的情況下怎麼遷移？

簡單總結下來，一個 virtualenv 主要是仰賴系統的兩個部分：

- **仰賴系統的 Python 路徑**：主要是在 `bin` 和 `pyvenv.conf` 上面。其中 `pyvenv.conf` 的內容比較無所謂，而 `bin` 的話把 shebang 的路徑及 symlink 進行修改就能直接使用。
- **Python 的 C library 依賴系統執行環境**：如果這個依賴需要呼叫 C library，Python 版本、系統、架構甚至到 libc 都會影響到一個依賴是否可以正常載入。

![Concept of Moving venv to new environment](https://assets.blog.pan93.com/move-venv-to-other-machines/venv-move-concept.png)

其實換言之，**只要你能確保系統、架構、libc 和 Python 版本這幾樣東西可以相同，venv 只需要稍作修改就可以輕鬆移到另一個環境了**。我們的程式碼評測環境後來是怎麼解決的？其實就是找一台架構和版本都很相似的 Windows 系統，在這上面建立 virtualenv 並將 [virtualenv 的路徑進行微調](https://github.com/edwardgeorge/virtualenv-clone)，就能在保持主要功能的情況下部署到網路差的機器上了。

![Migrating our evalution runner to new environment](https://assets.blog.pan93.com/move-venv-to-other-machines/migrate-our-evalution-runner.png)

[Zeabur Pack](https://github.com/zeabur/zbpack) 的 Python Serverless runtime 同時也用上了這個原理。考慮到建構 Python 程式和實際執行 Python 程式的環境近乎一致，所以我們可以在編譯環境用 `virtualenv` 製作一個乾淨的 `site-packages`，然後將 `site-packages` 打入 Serverless 應用程式套件裡面，最後送入 Serverless Runtime 把這個 `site-packages` 寫入 `PYTHONPATH` ——程式就可以幾乎無縫執行了，而我們也不需要在 Runtime 中執行 pip install 的操作。

![Concept of Zeabur Pack's Python Runtime](https://assets.blog.pan93.com/move-venv-to-other-machines/zbpack-python-runtime-concept.png)

但你不可能要求其他客戶的環境與我們的一致，肯定還是要面對 C library 的問題，那又該如何解決呢？

## virtualenv 在環境不同的情況下如何遷移？

Python 的 Library 看來在遷移上其實不會有太大的狀況，但是 numpy 這種 C library 呢？不同架構乃至不同系統的機器碼不可能完全相同，拿一個 C library 走天下肯定是不好做到的，所以我們之前的 virtualenv 方法在這方面就比較麻煩了，得在每個目標系統都打一個 virtualenv。

![Why is migrating Venv difficult?](https://assets.blog.pan93.com/move-venv-to-other-machines/why-venv-migration-is-difficult.png)

不過事實上你應該不用這麼麻煩的，Python 的生態系統中已經有很多「打包」的輪子了，比如 pex、PyInstaller、briefcase 等等的工具。不過他們有處理 C 函式庫的問題嗎？或許我們可以都打包起來測試看看，但在開始之前，不妨先介紹這幾套軟體。

### pex

pex 是一個將 Python 虛擬環境（包括依賴、中繼資料，以及程式碼本身）包成 `.pex` 檔案的工具。`.pex` 基本上復用了 [PEP 441](https://peps.python.org/pep-0441/) 的 Python ZIP 執行檔格式。簡而言之，只要一個 zip 開頭有著 `#!/usr/bin/env python3` 這樣的 shebang line，Python 就會自己解壓縮，然後啟動裡面的 `__main__.py`。pex 這套工具的最大功用就是將程式碼的依賴也包進去，這樣無需先用 pip 安裝依賴也能執行。

![Pex structure](https://assets.blog.pan93.com/move-venv-to-other-machines/pex-structure.png)

某種程度上它和我們剛才手動改造的 virtualenv 有著異曲同工之妙：也是把依賴都準備在一個資料夾裡，這樣就不用 pip 安裝依賴了。但 pex 是 virtualenv 嗎？實際上 pex 確實用了 virtualenv 來隔離安裝依賴的環境，但 pex 的 runtime 又比我們剛才土炮的連結法更加細緻，同時 pex 也有內建一套針對 `requirements.txt` 等等依賴格式的 resolver，說實話肯定是比我們剛才那樣操作複雜許多。

pex 有兩種打包形態。一種是類似 `virtualenv` 一樣可以攜帶的虛擬環境，一種就是帶有 `__main__` 的執行檔。這裡我先打包虛擬環境的版本出來：

```bash
pan93412@ubuntu:~$ pex requests -o requests_demo.pex
pan93412@ubuntu:~$ ./requests_demo.pex
Pex 2.20.0 hermetic environment with 1 requirement and 5 activated distributions.
Python 3.12.3 (main, Jul 31 2024, 17:43:48) [GCC 13.2.0] on linux
Type "help", "pex", "copyright", "credits" or "license" for more information.
>>> import requests
>>>
```

基本上他會啟動一個 Python 的解釋器。不過就如剛才所說，他其實還是一個需要 Python 解釋器才能執行的檔案，這個可以從檔案標頭中看到，而檔案本體就是個普通的 zip 檔案而已。

```bash
$ head ./requests_demo.pex -c 50
#!/usr/bin/env python3.12

$ file ./requests_demo.pex
./requests_demo.pex: Zip archive data, made by v2.0 UNIX, extract using at least v2.0, last modified, last modified Sun, Jan 01 1980 00:00:00, uncompressed size 0, method=deflate
```

檔案結構則是剛才所說的，`__main__` 和 Bootstrap scripts 及依賴的組合：

```bash
pan93412@ubuntu:~$ unzip -v requests_demo.pex
Archive:  requests_demo.pex
 Length   Method    Size  Cmpr    Date    Time   CRC-32   Name
--------  ------  ------- ---- ---------- ----- --------  ----
       0  Defl:N        2   0% 1980-01-01 00:00 00000000  .bootstrap/
       0  Defl:N        2   0% 1980-01-01 00:00 00000000  .bootstrap/pex/
       0  Defl:N        2   0% 1980-01-01 00:00 00000000  .deps/
    1339  Defl:N      714  47% 1980-01-01 00:00 3e9642bc  PEX-INFO
    7925  Defl:N     2490  69% 1980-01-01 00:00 25632e6e  __main__.py
       0  Defl:N        2   0% 1980-01-01 00:00 00000000  __pex__/
    7898  Defl:N     2472  69% 1980-01-01 00:00 bd9c024f  __pex__/__init__.py
--------          -------  ---                            -------
 4368876          1224130  72%                            339 files
```

### pyinstaller

pyinstaller 和 pex 一樣也是一種打包工具，可是 pex 打出來的檔案依然需要一個 Python Runtime 來跑。pyinstaller 就不一樣了，他產出來的就是大家熟悉的 exe 執行檔，裡面就包含了 Python 的解析器、系統函式庫等等的東西，使用者就不用再安裝一套 Python runtime 來執行你的程式了。

![Structure of Pyinstaller](https://assets.blog.pan93.com/move-venv-to-other-machines/pyinstaller-structure.png)

除此之外，pyinstaller 的 runtime 又比 pex 更「進階」了些。pyinstaller 的 runtime 會在執行時把 Python 檔案釋放到暫存目錄裡面，另外依賴的找尋又更自動了些——你甚至不用給 `requirements.txt`，它也能分辨出哪些依賴是真正需要的，同時它也會自動的包含動態連結庫（DLL）並進行 hooking，適用的場景可以比 pex 更廣些。

![How do Pyinstaller hook libraries](https://assets.blog.pan93.com/move-venv-to-other-machines/pyinstaller-hooking.png)

我沒有很深入讀 pyinstaller 建構依賴的邏輯，不過至少就我目前 Code Search 所知，他是沒有什麼安裝依賴的步驟的。所以我們得先建立一個虛擬環境，然後安裝好 `requests`：

```bash
pan93412@ubuntu:~$ python3 -m venv pyinstallertest
pan93412@ubuntu:~$ . ./pyinstallertest/bin/activate
(pyinstallertest) pan93412@ubuntu:~$ pip install pyinstaller requests
```

接著我這裡也準備一個 requests 的小 demo：

```python3
import requests

r = requests.get('https://httpbin.org/basic-auth/user/pass', auth=('user', 'pass'))
print(r.json())
```

實際打包一下，會發現他確實產出了一個執行檔：

```bash
(pyinstallertest) pan93412@ubuntu:~/pyinstallertest_demo$ pyinstaller request_test.py
(pyinstallertest) pan93412@ubuntu:~/pyinstallertest_demo$ ./dist/request_test/request_test
{'authenticated': True, 'user': 'user'}
(pyinstallertest) pan93412@ubuntu:~/pyinstallertest_demo$ file ./dist/request_test/request_test
./dist/request_test/request_test: ELF 64-bit LSB executable, ARM aarch64, version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux-aarch64.so.1, BuildID[sha1]=0f94ea8d84caf6140edfd7338fd281905413e283, for GNU/Linux 3.7.0, stripped
```

當然，因為套件本身就是已經包在執行檔裡面的，所以就算退出虛擬環境，執行檔依然是可以使用的：

```bash
(pyinstallertest) pan93412@ubuntu:~/pyinstallertest_demo$ deactivate
pan93412@ubuntu:~/pyinstallertest_demo$ ./dist/request_test/request_test
{'authenticated': True, 'user': 'user'}
```

### briefcase

briefcase 本質上和 pyinstaller 是同類型的東西，最終都能打包出平台原生的執行檔。不過比起 pyinstaller 對依賴做了相當多的封存 (freeze) 和修補 (patch) 來達到原生跨平台的目標，briefcase 的操作就相對「原生」了許多——當然某些 edge cases 下的問題也能比 pyinstaller 少一些：

> **Briefcase doesn't do anything to make the app itself cross platform - it just packages the code.** It's up to you to make the application itself cross platform. There is another BeeWare project - Toga - to provide cross platform UI capabilities, but Briefcase doesn't require Toga, or vice versa. [^4]

[^4]: <https://github.com/beeware/briefcase/issues/145#issuecomment-397938450>

讀 briefcase 的原始碼也稍微印證了這個說法，因為他也沒有針對 numpy 等等的 C library 做出一些修補。**本質上他和 pex 比較接近**，都是把程式碼包成一個檔案而幾乎不多做修改——不過 pex 僅僅是把 zip 封存成 pyz，而 briefcase 是對 Python runtime 修補放入程式碼。

### 修補效果

那接著，我們就來測試一下這三個方案可不可以運作。首先我們測試 pymysql，是個純 Python 的函式庫。因為 Python 本身是跨平台了，我們可以測試看看平台本身有沒有跨平台的能力。然後還有一個 numpy，他的 library 是根據不同 OS 和 CPU 架構編譯的，所以也可以測試看看這個平台怎麼處理原生 library 的情況。而編譯出的東西會在兩個目標平台上跑，一個是 x86 的 Ubuntu 24.04，另一個是 aarch64 沒有裝 Python 的 Arch Linux。

![Environments for Experimenting the Packagers](https://assets.blog.pan93.com/move-venv-to-other-machines/venv-experiment-environments.png)

#### pex

首先，我用 pex 分別打 numpy 和 pymysql 的 Python 環境。

```bash
(pexenv) pan93412@ubuntu:~/pex-example$ pex numpy -o numpy_pex.pex
(pexenv) pan93412@ubuntu:~/pex-example$ pex pymysql -o pymysql_pex.pex
```

如同我們在 virtualenv 測試的結果相似，直接執行肯定都是能 import 的。

```bash
(pexenv) pan93412@ubuntu:~/pex-example$ ./pymysql_pex.pex
Pex 2.20.0 hermetic environment with 1 requirement and 5 activated distributions.
Python 3.12.3 (main, Jul 31 2024, 17:43:48) [GCC 13.2.0] on linux
Type "help", "pex", "copyright", "credits" or "license" for more information.
>>> import pymysql
>>> exit()

(pexenv) pan93412@ubuntu:~/pex-example$ ./numpy_pex.pex
Pex 2.20.0 hermetic environment with 1 requirement and 1 activated distribution.
Python 3.12.3 (main, Jul 31 2024, 17:43:48) [GCC 13.2.0] on linux
Type "help", "pex", "copyright", "credits" or "license" for more information.
>>> import numpy
>>> exit()
```

那如果我們搬到，比如說，不同 CPU 架構的 Ubuntu 上呢？

```bash
pan93412@ubuntu-amd64:~/pex$ ./pymysql_pex.pex
Pex 2.20.0 hermetic environment with 1 requirement and 1 activated distribution.
Python 3.12.3 (main, Sep 11 2024, 14:17:37) [GCC 13.2.0] on linux
Type "help", "pex", "copyright", "credits" or "license" for more information.
>>> import pymysql
>>>

pan93412@ubuntu-amd64:~/pex$ ./numpy_pex.pex
Failed to find compatible interpreter on path /opt/orbstack-guest/bin-hiprio:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin:/opt/orbstack-guest/bin:/opt/orbstack-guest/data/bin/cmdlinks.

Examined the following interpreters:
1.) /usr/bin/python3.12 CPython==3.12.3

No interpreter compatible with the requested constraints was found:

  A distribution for numpy could not be resolved for /usr/bin/python3.12.
  Found 1 distribution for numpy that do not apply:
  1.) The wheel tags for numpy 2.1.1 are cp312-cp312-manylinux_2_17_aarch64, cp312-cp312-manylinux2014_aarch64 which do not match the supported tags of /usr/bin/python3.12:
  cp312-cp312-manylinux_2_39_x86_64
  ... 1067 more ...
```

好吧，pex 看來沒有解決 C library 的問題，但 pex 確實挺適合用來取代我們剛才說的手動改 virtualenv 的方式：

```bash
pan93412@ubuntu-amd64:~/pex$ cat pymysql_test.py
import pymysql

pan93412@ubuntu-amd64:~/pex$ ./pymysql_pex.pex pymysql_test.py
<exit code 0>
```

#### pyinstaller

pyinstaller 的打包我就跳過了，基本上就是在 venv 上裝 pyinstaller、numpy 和 pymysql 之後分別做一個 import 的 Python 檔案，最終可以得到兩個⋯⋯ arm64 的執行檔 😅

![The artifacts of Pyinstaller](https://assets.blog.pan93.com/move-venv-to-other-machines/pyinstaller-cp-result.png)

那確實就沒辦法跨平台執行了，看來跨平台的部分也不太能用 pyinstaller 來做。

```bash
pan93412@ubuntu-amd64:~/pyi$ ./pymysql
-bash: ./pymysql: cannot execute: required file not found
```

不過好消息是同架構下，pyinstaller 可以讓你不用 Python 就可以順利地執行包含 C library 的 Python code 了。當然 `_internal` 執行依賴庫也要附上：

```bash
[pan93412@arch ~]$ python3
-bash: python3: command not found
[pan93412@arch pyi]$ ./numpy/numpy
[pan93412@arch pyi]$ ./pymysql/pymysql
```

#### briefcase

最後就是 briefcase 了。看起來他整合了 pex 的簡單，也整合了 pyinstaller 的單一執行檔特色。或許大家可以猜猜看他在跨平台的情況下會是什麼樣子。由於篇幅的關係，其實不太能帶大家看到結果。不過我最終可以給大家一個比較表格，來看看這些工具在各個不同環境的適用情況。

### 比較表格

|             | 同系統，有 Python | 同系統，沒 Python                | 不同系統或架構 |
| ----------- | ------------ | --------------------------- | ------- |
| pex         | ✅            | ❌                           | ❌       |
| pyinstaller | ✅            | ✅                           | ❌       |
| briefcase   | ✅            | ⚠️，可能依賴 Python runtime 庫 | ❌       |

## 總結

這篇文章簡單研究了 virtualenv 的原理，也某些程度上緩解了以往大家認為 virtualenv 不能遷移的困擾。簡單來說，假如你想搬去的系統大體上是一樣的（都使用同樣的 libc、動態依賴庫相近）、CPU 架構一樣、Python 版本相近，其實是可以放心把 virtualenv 複製到另一個系統上的。這部分的使用場景主要是大量部署或者是虛擬環境的環境建置。

而假如系統沒有這麼純粹，那就得看你有沒有用到需要 C 的函式庫了，比如說 numpy 就沒法在不同系統或不同架構下執行。老實說這次測試的 3 個打包工具都不能 handle 不同系統、不同架構的情況，但是假如系統和架構一樣，pyinstaller 透過它針對每個 library 特化的 hooking，通用性又更大了些。pex 則是更加類似於 virtualenv 可攜化的替代品，如果你覺得上面「環境相同」的情況很適合你，用 pex 可以讓你省下許多寫一個新 runtime 的麻煩。briefcase 其實更多是與 beeware 搭配，但如果 pyinstaller 不太適合你的 case，也是能試試看的。
