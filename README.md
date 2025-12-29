# 遊戲邏輯與計分機制說明文件：Coffee Baum 758

本文件旨在為開發工程師提供《高章洙的咖啡危機》經營模擬遊戲的核心邏輯說明，包含數據結構、計分演算法、事件系統及結局判定機制。

## 1. 專案概述
*   **遊戲類型**：經營模擬決策 (Management Simulation)。
*   **技術架構**：原生 JavaScript (Vanilla JS), HTML5, CSS3。
*   **遊戲結構**：固定進行 4 回合（春、夏、秋、冬），每回合玩家需從三個維度進行決策。

---

## 2. 核心數據結構 (Core Data Structures)

### 2.1 決策維度 (Categories & Options)
每回合玩家必須從以下三個維度各選取一個選項（三選一）：
1.  **裝潢 (Decoration)**：選項 A, B, C
2.  **菜單 (Menu)**：選項 A, B, C
3.  **品質控管 (Quality Control)**：選項 A, B, C

### 2.2 遊戲狀態物件 (GameState)
```javascript
let gameState = {
    currentRound: 0,        // 整數 (1-4)
    cumulativeScore: 0,     // 累計總分
    selectedOptions: {      // 存儲玩家當前選擇
        '裝潢': null,
        '菜單': null,
        '品管': null
    },
    roundScores: [],        // 各回合得分歷史紀錄
    eventOccurred: false,   // 該回合是否觸發事件
    currentEvent: null,     // 當前事件物件
    eventModifiers: {}      // 存儲事件對特定選項的加成/減分值
};
```

---

## 3. 計分演算法 (Scoring Algorithm)

### 3.1 核心公式
單回合得分為三個決策選項的「基礎分」與「事件修正分」之總和：

$$RoundScore = \sum_{n=1}^{3} (BaseScore_{n} + Modifier_{n})$$

*   $n$: 三個維度（裝潢、菜單、品管）。
*   $BaseScore$: 該季節該選項的原始權重值（預設值域：-1, 1, 3）。
*   $Modifier$: 隨機事件對該選項產生的偏移量（若無事件影響則為 0）。

### 3.2 基礎計分矩陣 (BASE_SCORES)

| 季節 | 決策維度 | 選項 A | 選項 B | 選項 C |
| :--- | :--- | :---: | :---: | :---: |
| **春季** | 裝潢 / 菜單 / 品管 | 3 / 3 / 1 | 1 / 1 / -1 | -1 / -1 / 3 |
| **夏季** | 裝潢 / 菜單 / 品管 | 1 / 3 / 3 | 3 / 1 / 1 | -1 / -1 / -1 |
| **秋季** | 裝潢 / 菜單 / 品管 | -1 / 1 / 3 | 3 / 1 / 1 | 1 / 3 / -1 |
| **冬季** | 裝潢 / 菜單 / 品管 | 1 / 3 / 3 | -1 / 1 / 1 | 3 / -1 / -1 |

---

## 4. 隨機事件與修正系統 (Event System)

### 4.1 觸發機制
*   每回合開始時，有 **50% 機率** (`Math.random() < 0.5`) 觸發事件。
*   從 `EVENTS` 陣列中篩選尚未使用的事件 (`used: false`) 並隨機抽取。

### 4.2 事件修正清單 (Modifiers)
事件會根據特定季節，對特定選項注入 `delta` 值：

| 事件名稱 | 影響季節 | 影響對象 (維度-選項) | 修正值 (Delta) |
| :--- | :--- | :--- | :---: |
| **咖啡豆價格飆升** | 春季 | 品管-C / 菜單-B | +2 / -2 |
| **網紅探店衝擊** | 春季/秋季 | 裝潢-A | +2 |
| **政府工時稽查** | 夏季/秋季 | 品管-A | +2 |
| **連鎖店極限促銷** | 夏季/冬季 | 菜單-B / 菜單-A | -3 / +2 |
| **在地文化媒體專訪** | 秋季 | 裝潢-B / 裝潢-A | +2 / -2 |
| **競爭者設計雷同** | 秋季 | 菜單-C / 菜單-A | +2 / -2 |
| **極端氣候衝擊** | 冬季 | 裝潢-A / 裝潢-C | +2 / -2 |
| **顧客關係危機** | 冬季 | 裝潢-C / 品管-B | +2 / -2 |

---

## 5. 流程控制邏輯 (Control Flow)

### 5.1 回合結算流程
1.  **計算分數**：調用 `getOptionScore()` 取得各項總分。
2.  **更新狀態**：將 `RoundScore` 累加至 `cumulativeScore`。
3.  **判定單季成敗**：
    *   **成功 (Success)**: $RoundScore \ge 7$。
    *   **虧損 (Failure)**: $RoundScore < 7$。
4.  **動態反饋**：
    *   檢索該回合「得分貢獻最高」的維度類型。
    *   根據「成敗結果」從 `ROUND_FEEDBACK` 字典調用對應的紐約時報文案。

### 5.2 遊戲結局判定 (Endings)
於第四回合結束後，根據最終 `cumulativeScore` (S) 決定結局路徑：

| 分數門檻 (S) | 結局編號 | 標題 | 經營結果 |
| :--- | :--- | :--- | :--- |
| **$S \le 4$** | `ending0` | **絕望的退出者** | 經營失敗，現金流斷裂，租約到期後結束營業。 |
| **$5 \le S \le 18$** | `ending1` | **苦撐的生存者** | 勉強收支平衡，維持低利潤的長工時勞動。 |
| **$S > 18$** | `ending2` | **成功的弄潮兒** | 掌握社群與趨勢，成功在市場突圍並獲利。 |

---

## 6. 前端實作參考 (Technical Implementation)

### 6.1 UI 切換
*   **方法**：透過操作 DOM 的 `classList` (添加/移除 `.hidden`) 來切換 `startScreen`, `gameScreen`, `resultScreen`, `endScreen`。

### 6.2 交互約束
*   **選項選擇**：點擊選項按鈕時更新 `gameState.selectedOptions` 並高亮對應 DOM。
*   **提交鎖定**：使用 `Array.values(gameState.selectedOptions).every(val => val !== null)` 檢查是否三項皆已選擇，否則禁用 `submitBtn`。

### 6.3 動態渲染
*   `generateOptions(season)`：根據當前季節從 `OPTIONS_TEXT` 抓取文案，並在渲染時檢查 `eventModifiers` 是否存在，若有則動態顯示分數變化。

---

## 7. 擴展與維護建議
*   **平衡調整**：若需調整遊戲難度，可直接修改 `BASE_SCORES` 矩陣或 `RoundScore >= 7` 的判定標準。
*   **數據擴充**：`EVENTS` 陣列具備良好的擴展性，新增事件時僅需定義 `modifiers` 對象即可。
*   **多語言支持**：所有 UI 文字皆存儲於 `OPTIONS_TEXT` 與 `SEASON_DESCRIPTIONS` 中，易於進行 i18n 遷移。
