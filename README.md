# GyroSnipe — MQTT over WebSocket 體感槍戰

手機當槍（陀螺儀瞄準）、大螢幕當戰場的多人即時對戰。單一 HTML 檔，無後端，
所有遊戲邏輯跑在房主瀏覽器，玩家端只送輸入。

---

## 怎麼跑

**本機快速測試（不需要 HTTPS）**

直接用瀏覽器開 `gyrosnipe.html`：一個分頁按「開房」，另外開幾個分頁輸入房間碼加入。
沒有陀螺儀時會自動改用滑鼠／觸控拖曳瞄準，遊戲邏輯完全一樣。

**正式展示（手機陀螺儀）**

陀螺儀 API 需要 **HTTPS**，所以要先部署：

```bash
# GitHub Pages
git init && git add gyrosnipe.html && git commit -m "init"
# 推上 GitHub → Settings → Pages → 選 main branch → 得到 https://<你>.github.io/<repo>/gyrosnipe.html
```

Cloudflare Pages 直接拖資料夾上去也可以。部署後：

1. 筆電開網址 → 按「開房（大螢幕）」→ 投影出去
2. 同學掃 QR（或輸入 5 碼房間碼）
3. 手機朝向大螢幕中央 → 按「校準」
4. 房主按「開始遊戲」，三分鐘比分數

iOS 需要使用者手動觸發授權，「校準」按鈕已經包含 `requestPermission()` 呼叫。

---

## MQTT Topic 設計

命名空間 `ntub-iot-gyrosnipe/{房間碼}/`

| Topic | QoS | Retained | 說明 |
|---|:---:|:---:|---|
| `up/join/{id}` | 1 | ✗ | 玩家報到（兼 5 秒心跳，重連自動補報到） |
| `up/aim/{id}` | **0** | ✗ | 瞄準向量 `"x,y"`，20Hz |
| `up/fire/{id}` | **1** | ✗ | 開火事件 |
| `up/ping/{id}` | 0 | ✗ | RTT 量測 |
| `up/leave/{id}` | 1 | ✗ | **LWT** — 手機斷線時房主立刻知道 |
| `dn/host` | 1 | ✓ | **LWT** — 房主 online / offline |
| `dn/meta` | 1 | ✓ | 階段、剩餘時間、玩家名單，1Hz |
| `dn/hud` | **0** | ✗ | HP／彈藥／分數，10Hz |
| `dn/event` | 1 | ✗ | 命中、擊倒、開始、結束 |
| `dn/pong/{id}` | 0 | ✗ | RTT 回應 |

房主用單一 wildcard 訂閱 `up/#` 接收所有玩家輸入。

---

## 報告可以講的技術點

**1. QoS 不是越高越好**

瞄準向量用 QoS 0：每 50ms 一包，掉一包下一包立刻蓋過去，重送反而讓畫面倒退。
開火用 QoS 1：一槍就是一槍，不能掉也不該重複。同一個系統裡兩種需求，是解釋
QoS 取捨最直觀的例子。

**2. Retained message 解決「中途加入」**

`dn/meta` 是 retained。手機晚進場或斷線重連，一訂閱就立刻拿到當前階段與剩餘秒數，
不用等下一秒的廣播。

**3. Last Will 做離線偵測**

手機和房主都設了 LWT。實測拔網路後，房主端在 1 秒內就把該玩家移除（比 8 秒的
無訊號逾時快得多），玩家端也會立刻看到「房主離線」。

**4. 房主權威 + 客戶端預測**

玩家端只送「我朝這個座標開了一槍」，命中判定、彈藥、分數全部在房主端算。
手機端會先預測性地扣掉一發子彈讓手感即時，100ms 後房主的 HUD 廣播會修正回來——
這正是網路遊戲的標準做法，可以拿來對比「如果讓客戶端自己報命中會怎樣」。

**5. 頻寬實測**

畫面右下角的統計面板是實跑數據。三個玩家時約 **50~60 msg/s、4 KB/s**。
瞄準向量刻意用 `"0.4213,0.5518"` 這種緊湊字串而不是 JSON，可以現場改成 JSON
做前後對比，展示 payload 設計對頻寬的影響。

**6. WebSocket 與原生 MQTT**

瀏覽器只能走 WebSocket（8083 / 8084 wss），ESP32 之類的裝置走原生 1883。
兩者在同一個 broker 上是互通的——如果要做延伸，接一顆 ESP32 實體按鈕當扳機，
就能同場證明這件事。

---

## 參數調整

程式最上方的常數區：

```js
const MAG        = 6;      // 彈匣容量
const RELOAD_MS  = 1500;   // 換彈時間
const FIRE_CD    = 220;    // 開火冷卻（防連點）
const MAX_HP     = 3;
const RESPAWN_MS = 3000;
const ROUND_S    = 180;    // 一局長度
const HIT_R      = 0.052;  // 命中半徑（螢幕寬度比例，調大比較好命中）
```

人多的時候建議把 `HIT_R` 調小一點（0.04）、`ROUND_S` 縮短到 120 秒。

---

## 已知限制

- 預設連公用 broker `broker.emqx.io`，**任何人都能連**。正式展示請自架
  Mosquitto／EMQX 並開啟 WebSocket listener，或至少換一組不好猜的命名空間。
- QR code 圖片來自 `api.qrserver.com`。校園網路擋掉的話圖不會出現，
  但房間碼與網址仍會顯示，手動輸入即可。
- 陀螺儀的 alpha（方位角）在室內會緩慢漂移，長時間玩需要重新按「校準」。
  這點本身也可以寫進報告的「限制與改進」。
