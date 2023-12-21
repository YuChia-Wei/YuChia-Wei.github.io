# Git GUI Tools Memo


此篇筆記過往簡單測試過的一些 git gui 工具，時至今日應已有許多變革，此文僅作參考。

<!--more-->

# Git 工具

> 撰寫於 2021 / 08 / 20

## 首推

此節介紹的工具為建議使用工具，基本使用體驗良好

* Git cli (命令列)
    - 完全免費
    - 最直接的 Git 操作方式
    - 需要熟記相關命令
    - 對於擷選要加入的內容這件事情比較麻煩

* [ Fork ](https://git-fork.com/)
    - 收費軟體，已公開報價為 49.99 美元，免費版僅作為評估用，但是並未強制收費，僅會不定時提醒 Donate
    - 操作流暢
    - 方便確認變更內容與發佈 commit

## 次推

此節介紹的工具普遍體驗良好，但有部分問題導致評價下降 (會在內容說明)

* [ GitKraken ](https://www.gitkraken.com/)
    - 部分免費
    - **免費版無法連線到私人 Git Server / Repository**
        - 此點導致此軟體的評價下降至次推，因為這個限制會導致需要付費才可在公司環境使用
    - 非常美觀
    - 操作順暢

* [ SourceTree ](https://www.sourcetreeapp.com/)
    - 完全免費
    - 普遍推薦的軟體
    - 介面美觀
    - 容易閱讀 commit
    - **需要建立他們平台的帳號才能使用**
        - 此點導致此軟體的評價下降至次推，因為這個限制會導致為了使用這套軟體而註冊帳號，較為不便

## 依個人習慣可選擇

* Visual Studio 內建及其擴充功能
    * Visual Studio 中對 Git 的操作流程稍差
    * 有許多擴充套件可以擴充對 Git 操作的方便性
    * 安裝過多套件會導致效能低落
        - 有慣用 ReSharper 的人不會想要在 VS 上安裝其他套件讓 VS 更慢
    * 與 TFS 有良好整合性，方便操作 TFS 上的功能
    * 擴充套件 : GitFlow for Visual Studio
        - 在 Visual Studio 內使用 Git Flow 時需要的套件

* Visual Studio Code 及其擴充功能
    * 建議套件 : Git Graph、GitLens

* JetBrains Rider 及其擴充功能
    * 付費工具
    * 操作上需要特別開專案或選擇資料夾後再操作，稍嫌不便

## 待使用者提供心得

* [ sublimemerge ](https://www.sublimemerge.com/)
    - 付費 (99 鎂)
        - 偏貴，也有說明可免費評估，不確定是不是打算走跟 Fork 相同的套路
    - 操作體驗可能是 Fork / GitKraken 的強大競爭者 (依據官方提供的資訊判斷)

* GitFiend
    - UI 設計還可以，無操作經驗

* Cong
    - 主打指令操作，無按鈕 (未有太多使用經驗)

* Git Extensions
    - 撰文者不喜歡這套，UI 設計也很弱

* GitBlade
    - 基本功能免費，不少常用的功能都是付費的，不如直接使用 Git cli

* Git GUI
    - 安裝 Git for Windows 時會一同安裝，屬於官方自己的 Windows Desktop 工具，還算堪用

## 最後

最後提醒一件事情

_**不建議使用 TortoiseGit 操作Git**_

能理解從 SVN 轉換過來的使用者會偏好 TortoiseGit 的操作模式，但是其實 Git 的核心作業模式與 SVN 並不相同，因此不建議使用 TortoiseGit 來操作 Git，這會導致無法享受到 Git 的許多優勢。

