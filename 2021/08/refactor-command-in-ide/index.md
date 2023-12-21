# Refactor Command in Ide


IDE 常用重構快捷鍵

<!--more-->

# IDE 中提供的重構方法

此文件所使用的 IDE 環境為

* Visual Studio 2019 + JetBrains ReSharper
* JetBrains Rider

若為 Visual Studio 原生可使用的功能會在該功能下註記，但原生功能不一定會跟 ReSharper 所提供的功能有相同的方便性

Tips: ReSharper/Rider 可以用 Ctrl + Shift + R 來叫出重構用選單

此文件僅列出常用的重構命令，而這些常用的重構命令在不同情境下會有不同的效果

* 將選取的內容抽為方法：Ctrl + R -> Ctrl + M
    - VS 快捷鍵設定名稱：重構-提取方法 (Refactor.ExtractMethod)

* 將選取的變數、參數重新命名：Ctrl + R -> Ctrl + R
    - VS 快捷鍵設定名稱：重構-重新命名 (Refactor.Rename)

* 將選取方法拉到新的介面 (目前類別會掛上新的介面)：Ctrl + Shift + R -> X
    - VS 快捷鍵設定名稱：重構-提取介面 (Refactor.ExtractInterface)

* 提取類別：Ctrl + Shift + R -> E

* 將目前變數、方法攤回到使用端 (Inline)：Ctrl + R + I

* 安全刪除變數或參數：Alt + Del
    - 可以快速刪除不使用的變數，在有被使用的變數上不建議使用此命令，因為可能引發引用問題

* 將方法參數整合為一個類別
    - 在方法宣告上使用 Alt + Enter 後輸入 TranP 可找到 "Transform parameters"
    - 在方法宣告上使用 Ctrl + Shift + R -> 選擇 Transform parameters (或者按 P)

