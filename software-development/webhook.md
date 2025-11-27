# 從 LINE Bot 實作理解 Webhook 機制

## 為什麼寫這篇

最近在做 LINE Bot,發現 Webhook 這個東西雖然概念簡單,但實際串接時還是踩了不少坑。

一開始照著文件設定,Bot 就是不回應。檢查 log 發現 LINE 有發請求過來,但訊息就是發不出去。後來才發現是 `replyToken` 過期了——因為我處理太慢,超過 5 秒才回 200,LINE 就放棄重試了。

還有一次是使用者反應「為什麼傳一次訊息,機器人回三次?」查了半天才知道是網路抖動造成 LINE 重送 Webhook,而我沒做冪等性處理 (同一個操作執行多次,結果都跟執行一次一樣)。

這些坑讓我重新思考 Webhook 的設計邏輯:為什麼要快速回應?為什麼要驗證簽名?為什麼要處理重複請求?這篇筆記記錄了實作過程中的一些心得。

---

## 如果不用 Webhook 會怎樣?

假設 LINE 不提供 Webhook,我們只能用輪詢 (Polling) 的方式:

```python
# 輪詢方式 - 持續詢問
while True:
    response = requests.get('https://api.line.me/v2/messages')
    
    if response.json()['has_new_message']:
        messages = response.json()['messages']
        for msg in messages:
            process_message(msg)
    
    time.sleep(1)  # 每秒問一次
```

### 輪詢的問題

| 問題 | 說明 |
|------|------|
| **資源浪費** | 即使沒訊息也要一直發請求,99% 的請求都是白費的 |
| **延遲** | 輪詢間隔 1 秒,訊息最多延遲 1 秒才收到 |
| **API 限制** | 高頻率輪詢容易被 rate limit |
| **伺服器負擔** | 如果有 1000 個 Bot,LINE 每秒要處理 1000 次查詢 |

### Webhook 的優勢

```python
# Webhook - 事件驅動
@app.post('/webhook')
async def webhook(request: Request):
    event = await request.json()
    process_message(event)
    return {'status': 'ok'}
```

| 優勢 | 說明 |
|------|------|
| **即時** | 訊息一到立刻通知,零延遲 |
| **省資源** | 只在有事件時才發請求 |
| **可擴展** | 不管多少 Bot,LINE 只在需要時才發通知 |

簡單說就是:**Pull = 我問你,Push = 你通知我**

---

## LINE Bot 的 Webhook 機制

### 架構圖

```
使用者 → LINE Platform → Webhook → 你的 Bot 後端 → LINE Messaging API
```

整個流程:

1. 使用者傳訊息到 LINE
2. LINE Platform 發送 POST 請求到你設定的 Webhook URL
3. 你的 Bot 處理訊息並回傳 200 OK
4. Bot 透過 LINE Messaging API 回覆訊息

---

## 實作細節與陷阱

### 1. 簽名驗證

LINE 會在 Header 加上 `X-Line-Signature`,用來驗證請求確實來自 LINE Platform。

```python
def validate_signature(body: bytes, signature: str) -> bool:
    """驗證 LINE Webhook 簽名"""
    hash_value = hmac.new(
        CHANNEL_SECRET.encode('utf-8'),
        body,
        hashlib.sha256
    ).digest()
    
    return base64.b64encode(hash_value).decode('utf-8') == signature
```

為什麼需要驗證?  
任何知道你 Webhook URL 的人都能發送偽造請求。不驗證簽名的話,攻擊者可以假冒 LINE 發送惡意資料。

### 2. 快速回覆與非同步處理

LINE 要求在 **5 秒內回覆 200 OK**,否則會判定 Webhook 失敗並重試。

```python
# ❌ 錯誤:同步處理可能超時
@app.post('/webhook')
async def callback(request: Request):
    body = await request.body()
    events = json.loads(body)['events']
    
    for event in events:
        process_complex_task(event)  # 可能超過 5 秒
    
    return 'OK'  # 太慢了

# ✅ 正確:先回覆,再處理
@app.post('/webhook')
async def callback(request: Request):
    body = await request.body()
    
    # 立即回覆
    asyncio.create_task(process_events(body))
    return 'OK'

async def process_events(body: bytes):
    events = json.loads(body)['events']
    for event in events:
        await process_complex_task(event)
```

或者使用訊息佇列:

```python
import redis
from rq import Queue

q = Queue(connection=redis.Redis())

@app.post('/webhook')
async def callback(request: Request):
    body = await request.body()
    
    # 丟到背景處理
    q.enqueue(process_events, body)
    
    return 'OK'
```

### 3. 冪等性處理

網路不穩定時,LINE 可能重送相同的 Webhook。必須確保重複處理不會產生副作用。

```python
import redis

redis_client = redis.Redis()

@handler.add(MessageEvent, message=TextMessage)
def handle_message(event):
    message_id = event.message.id
    
    # 檢查是否已處理
    if redis_client.exists(f'processed:{message_id}'):
        return
    
    # 標記為已處理 (TTL 24 小時)
    redis_client.setex(f'processed:{message_id}', 86400, '1')
    
    # 實際處理邏輯
    process_message(event)
```

或使用 `replyToken` 的特性:每個 token 只能用一次,LINE 會自動處理重複。

### 4. Reply vs Push

LINE Bot 有兩種發訊息方式:

**Reply (使用 replyToken)**:
```python
line_bot_api.reply_message(
    event.reply_token,
    TextSendMessage(text='回覆訊息')
)
```
- 只能在收到 Webhook 後立即使用
- 每個 replyToken 只能用一次
- 不計費

**Push (使用 userId)**:
```python
line_bot_api.push_message(
    user_id,
    TextSendMessage(text='主動推播')
)
```
- 可以隨時主動發送
- 需要 user_id
- 計費(免費額度外)

---

## 開發環境設定

### 本地開發:使用 ngrok

LINE Webhook 必須是公開的 HTTPS 網址,開發時可用 ngrok:

```bash
# 啟動 Bot
uvicorn main:app --reload --port 8000

# 另一個 terminal
ngrok http 8000
```

ngrok 是一個反向代理服務，可以將本地開發環境下的服務臨時公開到網際網路，提供一個公開的網址讓外部連線存取。
會提供類似 `https://abc123.ngrok.io` 的網址,填入 LINE Developers Console。

---

## 總結

Webhook 的核心就是**事件驅動的 HTTP 回調**:

1. 註冊公開的 HTTPS 端點
2. 對方在事件發生時發送 POST 請求
3. 快速回覆 200 OK
4. 非同步處理實際邏輯

理解這個模式後,不只是 LINE Bot,許多第三方服務整合(GitHub、Stripe、Discord)都是類似的概念。

重點在於:
- **安全性**:驗證請求來源
- **可靠性**:處理重試與冪等性
- **效能**:先回覆,再處理

---

## 參考資源

- https://medium.com/@justinlee_78563/line-bot-%E7%B3%BB%E5%88%97%E6%96%87-%E4%BB%80%E9%BA%BC%E6%98%AF-webhook-d0ab0bb192be
- https://ithelp.ithome.com.tw/m/articles/10193212
- [LINE Messaging API 文件](https://developers.line.biz/en/docs/messaging-api/)
- [ngrok](https://ngrok.com/)