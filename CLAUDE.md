# vTool

台灣房屋投資報酬率計算機。純前端、單檔 HTML（無 build step、無框架、無 npm 依賴），直接用瀏覽器打開即可用。

## 檔案總覽

| 檔案 | 說明 |
|---|---|
| `vTool_v3.html` | **單一案件**版本，目前功能最完整（IRR + 租金所得稅），非對照版的維護重點應以此檔為準 |
| `vTool_compare_v3.html` | **A/B 雙案對照**版本，功能與 `vTool_v3.html` 同步（IRR + 租金所得稅），對照版的維護重點應以此檔為準 |
| `vTool.html` / `vTool_v2.html` / `vTool_compare.html` / `vTool_compare_v2.html` | 較舊的迭代版本，稅率級距較簡化（例如 v1/v2 沒有 >10年 15% 級距、沒有自住免稅判定），除非使用者特別指名，否則不需要跟著更新 |

`v3` 兩個檔案的程式邏輯是刻意保持平行的：任何一邊改公式，另一邊通常也要比照修改，除非使用者明確說只改其中一個。

## 核心計算邏輯（v3 系列）

### 費用假設
- 買入費用固定 2%（1% 仲介費 + 1% 代書規費雜支）
- 賣出費用固定 4%（賣方仲介費）

### 房貸攤還模擬
- 逐月模擬：寬限期內「純付息」（`loanAmount * monthlyInterestRate`），寬限期後用標準年金公式 `P × r(1+r)^n / ((1+r)^n − 1)` 全額攤還，`n = 貸款總月數 − 寬限期月數`
- 逐月遞減 `remainingLoan`；貸款提前還清時自動停止累計利息與還款

### 房地合一稅級距（現行法規，故意保留 off-by-one）
```
holdingYears >= 6 && monthlyRent === 0  → 10%（自住滿6年，400萬免稅額，超額部分課10%）
holdingYears < 2                        → 45%
holdingYears < 5                        → 35%
holdingYears < 10                       → 20%
else                                    → 15%
```
嚴謹來說法條邊界是「以內/未逾」（inclusive），所以剛好持有滿 2/5/10 年整數年時，程式碼算出的稅率其實比法規少一級（例如剛好5年應為35%，程式算20%）。**這是使用者已確認過的已知取捨**——因為 UI 只能輸入整數年，若要精確處理邊界會需要顯示「5年+1天」這種難以閱讀的措辭，所以維持現狀，未來不需要再提這件事或主動修正。

### 實質淨獲利（netProfit）
```
netProfit = (估計售價 - 買入總價)              // propertyAppreciation
          + 累積租金收入 (totalRentIncome)
          - 累積租金所得稅 (totalRentIncomeTax)  // 見下方「租金所得稅」
          - 累積利息成本 (totalInterestExpense)
          - 買入費用(2%)
          - 賣出費用(4%)
          - 房地合一稅 (capitalGainTax)
```
這個公式已經用現金流展開驗證過代數正確性：不論貸款成數多少，只要正確扣除「利息」（消耗性支出）而非「本金」（本金攤還等於把現金轉成房屋權益，賣屋清償貸款時會拿回來），此淨利公式在槓桿倍數變動下依然自洽。**不要因為「本金沒有算進支出」而覺得是bug**——這是刻意設計，定義區塊 (`.definition-box`) 裡有說明。

### 房地合一稅稅基
```
capitalGain = 估計售價 - 買入總價 - 買入費用 - 賣出費用
```
不含租金收入（租金所得稅是分開課的，見下方）。

### IRR（實質年化報酬率，已取代舊版簡化 CAGR）
不要退回「頭尾金額算 CAGR」的簡化版！現在的做法是建構完整逐月現金流，再用 bisection 解出使 NPV=0 的月報酬率：
```
cashFlow[0]        = -期初總投入現金 (totalInitialCash)
cashFlow[1..M-1]   = 每月租金 - 每月租金所得稅 - 每月房貸還款 (monthlyNetFlowBase)
cashFlow[M]        = 上面的月現金流 + (估計售價 - 賣出費用 - 房地合一稅 - 持有期滿時剩餘貸款餘額)
irr_annual = (1 + monthlyIRR)^12 - 1
```
`solveMonthlyIRR()` 用二分逼近法，下界會依總期數動態調整（`Math.pow(1e-100, 1/n) - 1`）避免長年期（例如100年持有）時 `(1+rate)^t` 數值下溢造成 NaN。已用 Playwright 驗證過：
- 沒有中途現金流時，新 IRR 與舊 CAGR 公式數值完全吻合（退化情況一致）
- 100年持有、0%利率等極端情境不會出現 NaN/Infinity
- 全虧損（NPV 無解）時回傳 `null`，顯示為「虧損過大」

### 租金所得稅
台灣租金收入是併入個人綜合所得稅，不是單獨稅率，工具用以下簡化模型：
```
租金所得稅(年) = 月租×12 × (1 - 43%)  × 使用者自選的邊際稅率
```
- 43% 是法定必要費用推計比例（未提供實際費用單據時的標準做法），寫死在程式碼裡（`RENTAL_EXPENSE_RATIO`），沒有開放使用者自訂
- 邊際稅率是下拉選單：0%（不併入）/5%/12%/20%/30%/40%，因為工具無法得知使用者的總所得，只能讓使用者自己選

## 已知的簡化/假設（不是bug，除非使用者要求才動）
1. 自住6年免稅的判定只看「`holdingYears>=6 && monthlyRent===0`」，沒有檢查「本人+配偶+未成年子女僅一屋」「戶籍登記」等實際法規要件
2. 租金所得稅用固定 43% 費用推計，沒有讓使用者輸入實際費用單據
3. IRR 是用「加總現金流」逼近，理論上多重現金流反轉時可能有多重根（multiple IRR）問題，但一般房地產買賣（前期投入、中期租金、後期出場）現金流型態單純，實務上不會遇到

## 開發與測試慣例

- **沒有 build step**，改完 HTML 直接能用，不需要 `npm install`
- **沒有測試框架**。驗證計算邏輯的方式：
  1. 先用 `node --check` 驗證抽出來的 `<script>` 內容語法正確
  2. 用 Playwright 開 headless Chromium 實際載入 HTML、操作表單、讀表格輸出，確認數字合理、無 NaN/console error
  3. Playwright 套件本身不在專案的 npm 依賴內，要用全域安裝的版本：
     ```bash
     export NODE_PATH=/opt/node22/lib/node_modules
     node -e "const { chromium } = require('playwright'); ... executablePath: '/opt/pw-browsers/chromium' ..."
     ```
  4. 改動稅率/公式相關邏輯時，可以先手算或用 node 腳本單獨驗證核心函式（例如 `solveMonthlyIRR`），再整合進 HTML

## Git 工作流程

- 開發分支：`claude/vtool-compare-formula-verify-w5x1lx`
- 目前使用者偏好的合併方式：改動完成、測試過後，直接 fast-forward 合併進 `main` 並 push（`git fetch origin main` 確認沒有分歧 → `git checkout main` → `git merge <feature-branch> --ff-only` → `git push origin main` → 切回 feature branch 繼續工作）
- 這個 repo 目前沒有設定 CI/GitHub Pages，合併進 main 後如果要在瀏覽器看到效果，需要重新整理直接開啟該 html 檔案的頁面（GitHub raw / Pages 若有設定則需注意快取）
