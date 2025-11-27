# GraphQL：讓 API 查詢更精準的查詢語言

## 為什麼寫這篇

最近在接一個專案，後端 API 設計遇到瓶頸。

一個商品列表頁面，手機版只需要顯示商品名稱和價格，但網頁版要顯示完整資訊（評價、庫存、描述等）。用 REST API 的話，要嘛回傳全部資料讓手機浪費流量，要嘛做兩個不同的 endpoint 增加維護成本。

還有更麻煩的：前端要顯示「訂單詳情」，需要呼叫 3 個 API：
- `/orders/{id}` 取得訂單基本資訊
- `/users/{userId}` 取得客戶資料  
- `/products/{productId}` 取得每個商品詳情

這就是傳說中的 **N+1 查詢問題**。

後來突然想起之前有聽過 GraphQL，說它可以「想要什麼就拿什麼」。研究後發現這東西確實解決了 REST 的很多痛點，但也帶來新的複雜度。這篇筆記記錄了 GraphQL 的核心概念和實際應用。

---

## 如果只用 REST API 會怎樣？

想像你去餐廳點餐。REST API 就像套餐——廚師決定套餐有什麼，你不能說「我不要沙拉，多給我一點薯條」。要嘛全部接受，要嘛換別的套餐。

### REST 的典型問題

**過度取得 (Over-fetching)**

就像點了一個套餐，結果送來一堆你不想吃的配菜。API 回傳了 30 個欄位，但手機版只需要顯示 3 個。剩下的 27 個欄位白白浪費了使用者的網路流量。在 4G/5G 吃到飽的時代可能還好，但如果使用者在國外漫遊，或是在網路不穩的環境，這就是個大問題。

**不足取得 (Under-fetching)**

這更麻煩。就像你點了漢堡，發現沒有薯條，要再點一份。吃到一半發現沒飲料，又要再點。每次點餐都要排隊等待。

在程式的世界，這代表你需要發送多次請求才能組合出一個完整的頁面。每次請求都有網路延遲，使用者體驗就像在看投影片，一張一張慢慢載入。

**版本管理地獄**

REST API 更新版本是個大工程。你有 `/api/v1/users`，後來發現要加新欄位，但不能直接改，因為舊版 App 還在用。於是你做了 `/api/v2/users`。一年後又要改，變成 `/api/v3/users`。

最後你的後端要同時維護三個版本的 API，每次改 bug 要改三個地方。名副其實的版本地獄QQ。

---

## GraphQL 是什麼？

GraphQL 不是資料庫，也不是程式語言，而是一種「查詢語言」。

把它想像成去自助餐廳。你拿著盤子，想要什麼就夾什麼。要兩塊雞排？可以。不要青菜？沒問題。這就是 GraphQL 的精神：**客戶端決定要什麼資料**。

### 從一個簡單的例子開始

假設你要顯示使用者的個人頁面。用 REST 的話：

```
GET /api/users/123

回傳一大包資料：
{
  "id": 123,
  "name": "Moofon",
  "email": "moofon0222@gmail.com",
  "phone": "0912345678",
  "address": "...",
  "created_at": "2025-10-14",
  "updated_at": "2024-11-14",
  "last_login": "...",
  // ... 還有 20 個欄位
}
```

但如果你只需要顯示名字和 email，那其他欄位都是浪費。

GraphQL 的做法：

```graphql
{
  user(id: 123) {
    name
    email
  }
}
```

回傳的資料：

```json
{
  "data": {
    "user": {
      "name": "Moofon",
      "email": "moofon0222@gmail.com"
    }
  }
}
```

看到差異了嗎？你要什麼，就給你什麼，不多不少。

### 更複雜的例子：關聯資料

這才是 GraphQL 真正強大的地方。假設你要顯示一個部落格文章，包含作者資訊和留言。

REST 的做法需要多次請求：

```
1. GET /posts/456          → 取得文章
2. GET /users/789          → 取得作者資料
3. GET /posts/456/comments → 取得留言列表
4. GET /users/111          → 取得留言者 1 的資料
5. GET /users/222          → 取得留言者 2 的資料
...
```

五個請求！如果有 10 則留言，可能要發 13 個請求。

GraphQL 一次搞定：

```graphql
{
  post(id: 456) {      
    title              # 1. 我要編號 456 的文章
    content            # 2. 給我文章的標題
    author {           
      name             # 3. 給我作者的名字
      avatar           # 4. 作者的頭像
    }
    comments {         
      content          # 5. 每則留言的內容
      createdAt        # 6. 留言建立時間
      author {         
        name           # 7. 留言者的名字
      }
    }
  }
}
```

一個請求，拿到所有需要的資料。而且你注意到了嗎？文章作者我們要 `name` 和 `avatar`，但留言作者只要 `name`。這種細緻的控制，REST 做不到。

---

## GraphQL 的三大操作

GraphQL 不只是查詢，它有三種操作類型，對應不同的使用情境。

### 1. Query（查詢）

Query 就像 REST 的 GET，用來取得資料。但它更靈活，可以精確指定要什麼欄位，還可以一次查詢多個資源。

想像你在建立一個儀表板頁面，需要顯示多種資訊：當前使用者、今日訂單、低庫存商品。用 REST 你要發三個請求，但 GraphQL 可以這樣：

```graphql
query DashboardData {
  currentUser {
    name
    unreadNotifications
  }
  todayOrders {
    count
    totalRevenue
  }
  lowStockProducts {
    name
    remaining
  }
}
```

一個查詢，三種資料，一次到位。

### 2. Mutation（變更）

Mutation 對應 REST 的 POST、PUT、DELETE，用來修改資料。名字取得很好——"mutation" 就是「變化」的意思（順便學一下英文xd）。

創建新使用者的例子：

```graphql
mutation CreateUser($name: String!, $email: String!) {
  createUser(name: $name, email: $email) {
    id
    name
    email
    createdAt
  }
}
```

注意到了嗎？即使是修改操作，你還是可以指定要回傳什麼資料。這在 REST 中通常要再發一個 GET 請求才能拿到。

### 3. Subscription（訂閱）

這是 REST 做不到的：即時更新。

想像你在做一個拍賣網站，使用者需要即時看到出價更新。用 REST 你只能不斷輪詢（每秒問一次「有新出價嗎？」），但 GraphQL Subscription 可以這樣：

```graphql
subscription OnNewBid($auctionId: ID!) {
  bidPlaced(auctionId: $auctionId) {
    amount
    bidder {
      name
    }
    timestamp
  }
}
```

當有新出價時，伺服器會主動推送資料給客戶端。就像訂閱 YouTube 頻道，有新影片會自動通知你。

**補充一下:** 這點我在寫的時候其實發現與Webhook滿像的，概念上都是「事件發生 → 通知」，但機制不太一樣：

**GraphQL Subscription**：客戶端主動透過 WebSocket 長連線與伺服器保持連線，伺服器有事件時直接推送到該連線。適用於即時的畫面更新（如聊天室、拍賣即時報價）。

**Webhook**：伺服器主動向外部服務發送一個 HTTP POST 請求來通知事件。外部服務不需要與伺服器保持連線。

| 特性 | GraphQL Subscription | Webhook |
| :--- | :--- | :--- |
| **發起方** | **客戶端**主動建立連線 | **伺服器**主動向外部 URL 發送請求 |
| **連線協定** | **WebSocket** | **HTTP POST** |
| **連線類型** | **長連線** (保持開啟，等待 Server 推送) | **單次請求** (事件發生時發送) |
| **資料流向** | 伺服器 **推送** 至已連線的客戶端 | 伺服器 **呼叫** 外部服務的 API |
| **適用情境** | 網頁/App 等**即時畫面**更新 | 服務之間**後台通知** (支付完成、版本發佈) |

有興趣也可以參考我 [Webhook](https://hackmd.io/@Moofon/ByyIJnbgWg)  的文章

---

## Schema：GraphQL 的規則書

Schema 是 GraphQL 的核心，它定義了「遊戲規則」：有什麼資料可以查、資料長什麼樣子、資料之間的關係。

### 為什麼 Schema 這麼重要？

想像你去圖書館借書。REST API 就像沒有目錄的圖書館——你要自己去書架上找，或者問管理員（看文件）。而且文件可能過期了，管理員（後端工程師）可能也不確定某本書在哪裡。

GraphQL 的 Schema 就像一個永遠準確的電子目錄。你可以查詢：
- 有哪些書（資料類型）
- 每本書的資訊（欄位）
- 書在哪個書架（關聯）
- 能不能借出（權限）

而且這個目錄是**自動更新**的。Schema 改了，文件就自動更新，不會有文件和程式不一致的問題。

### Schema 的基本結構

```graphql
# 定義一個使用者類型
type User {
  id: ID!           # ! 表示必填，一定會有值
  name: String!     
  email: String     # 沒有 ! 表示可能是 null
  posts: [Post!]!   # 文章陣列，陣列和內容都不能是 null
}

# 定義文章類型
type Post {
  id: ID!
  title: String!
  content: String!
  author: User!     # 每篇文章都有作者
  published: Boolean!
}

# 定義可以查詢什麼
type Query {
  user(id: ID!): User
  users: [User!]!
  post(id: ID!): Post
}

# 定義可以修改什麼
type Mutation {
  createPost(title: String!, content: String!): Post!
  deletePost(id: ID!): Boolean!
}
```

Schema 就像合約，前後端都要遵守。前端知道可以查什麼、會拿到什麼格式的資料。後端知道要提供什麼資料、接受什麼參數。

---

## GraphQL 解決了什麼問題？

### 1. 不同裝置，不同需求

現代應用要支援各種裝置：手機、平板、電腦、智慧手錶。每種裝置的螢幕大小不同，需要的資料量也不同。

想像你在做一個新聞網站：
- **智慧手錶**：只要標題
- **手機**：標題 + 摘要 + 一張圖
- **平板**：標題 + 摘要 + 內容預覽 + 圖片
- **電腦**：完整內容 + 相關新聞 + 留言

用 REST，你有兩個選擇：
1. 做四個不同的 API（維護地獄）
2. 一個 API 回傳所有資料（浪費頻寬）

用 GraphQL，同一個 API，不同的查詢，各取所需。

### 2. 前後端並行開發

傳統開發流程是這樣的：
1. 後端定義 API
2. 後端實作
3. 前端才能開始串接
4. 發現資料不夠，後端再改
5. 無限循環...

有了 GraphQL Schema，流程變成：
1. 前後端一起定義 Schema
2. 前端用 Mock 資料（假資料）開發
3. 後端實作 Schema
4. 自然串接

前後端可以並行開發，不用互相等待。Schema 就是雙方的合約。

### 3. API 演進不再痛苦

REST API 的演進是個難題。你不能隨便改變回傳格式，因為可能會弄壞正在使用的 App。所以才會有 v1、v2、v3...

GraphQL 的做法優雅很多：
- **新增欄位**：直接加，舊的客戶端不查詢就不會收到
- **刪除欄位**：先標記為 `@deprecated`，給開發者時間遷移
- **改變邏輯**：可以新增欄位，讓新舊邏輯並存

再複習一下:
假設你的豆漿原本是甜的，你想改成無糖。
👉 如果你直接改掉，習慣喝甜的客人會崩潰 [如果是RestAPI 就得再發新版本(新菜單）]。
所以你會這樣做：
原本的「甜豆漿」還是留著（舊欄位）
新增「無糖豆漿」給新客人（新欄位）
GraphQL 做法：
```graphql
sweetSoyMilk: String @deprecated
soyMilk: String
}
```
兩種版本並存，讓舊客人不會壞。

再來看看下面的例子：

```graphql
type User {
  id: ID!
  name: String!
  
  # 2024 年新增的欄位，舊 App 不會查詢
  avatarUrl: String
  
  # 準備棄用的欄位
  username: String @deprecated(reason: "請使用 name 欄位")
}
```

---

## 實務上的挑戰

GraphQL 不是萬靈丹，它也有自己的問題。

### 1. 後端複雜度增加

REST API 的實作相對單純：一個 endpoint 對應一個功能，你清楚知道要回傳什麼資料。

GraphQL 不一樣。同一個查詢，客戶端可能要不同的欄位組合。這代表你要處理各種可能性。

舉個例子，查詢使用者的文章：

```graphql
# 查詢 A：只要文章標題
{ user(id: 1) { posts { title } } }

# 查詢 B：要文章和所有留言
{ user(id: 1) { posts { title content comments { text } } } }

# 查詢 C：要文章、留言、留言者資訊
{ user(id: 1) { 
  posts { 
    title 
    comments { 
      text 
      author { name email }
    } 
  } 
} }
```

三個完全不同的查詢，後端要能聰明地處理，不能傻傻地把所有資料都撈出來。

### 2. N+1 查詢問題

這是 GraphQL 最惡名昭彰的問題。

假設你要顯示 10 篇文章的作者：

```graphql
{
  posts {      # 1 次資料庫查詢
    title
    author {   # 每篇文章查一次作者 = 10 次查詢
      name
    }
  }
}
```

總共 11 次資料庫查詢！如果是 100 篇文章，就是 101 次。

解決方法是使用 DataLoader 或類似的批次載入機制。把 10 個作者 ID 收集起來，一次查詢，而不是查 10 次。但這又增加了實作的複雜度。

### 3. 安全性考量

GraphQL 讓客戶端有很大的自由度，但自由度太大可能造成問題。

**查詢深度攻擊**

惡意使用者可能送出超深的查詢：

```graphql
{
  user {
    posts {
      author {
        posts {
          author {
            posts {
              # 繼續往下 100 層...
            }
          }
        }
      }
    }
  }
}
```

這種查詢可能讓你的伺服器當機。

**查詢廣度攻擊**

或者要求大量資料：

```graphql
{
  users(limit: 10000) {
    posts(limit: 10000) {
      comments(limit: 10000) {
        # 10000 × 10000 × 10000 = 爆炸
      }
    }
  }
}
```

所以生產環境的 GraphQL 需要加上各種保護：
- 查詢深度限制
- 查詢複雜度計算
- Rate limiting
- 查詢白名單（預先定義可用查詢）

### 4. 快取不容易

REST API 的快取很簡單。`GET /users/123` 的結果可以直接被瀏覽器、CDN、反向代理快取。

GraphQL 的查詢都是 POST 請求到 `/graphql`，每個查詢都可能不同，傳統的 HTTP 快取機制都失效了。

你需要在應用層做快取，或者使用支援 GraphQL 的 CDN。這又是額外的複雜度。

---

## 什麼時候該用 GraphQL？

### 適合的場景

**1. 客戶端有不同設備**

如果你的 API 要服務各種不同的客戶端（iOS、Android、Web、智慧電視、手錶），而且每個客戶端需要的資料差異很大，GraphQL 是好選擇。

Facebook 就是因為這個原因發明了 GraphQL。他們要服務各種裝置，從功能手機到高階電腦，每個裝置的需求都不同。

**2. 快速迭代的產品**

新創公司或是快速成長的產品，需求經常變動。GraphQL 讓前端可以自由調整查詢，不用等後端改 API。

**3. 資料關聯複雜**

如果你的資料像蜘蛛網一樣互相關聯（社群網站、知識圖譜），GraphQL 的圖狀查詢特別合適。

想像 LinkedIn 的人脈網路：你認識 A，A 認識 B，B 認識 C。用 REST 查詢三層人脈關係會很痛苦，但 GraphQL 很自然。

**4. 微服務架構**

如果後端是微服務架構，GraphQL 可以作為 API Gateway，對外提供統一的介面，對內整合各個微服務。

Netflix 就是這樣用。他們有幾百個微服務，但對外只露出一個 GraphQL API。

### 不適合的場景

**1. 簡單的 CRUD 應用**

如果你只是做一個簡單的待辦事項 App，REST 就夠了。GraphQL 會增加不必要的複雜度。

：能用簡單方法解決的問題，就不要用複雜的方法。

**2. 檔案上傳下載**

GraphQL 不擅長處理二進位資料。如果你的應用主要是處理圖片、影片、檔案，還是用專門的服務比較好。

---

## 實際案例分享

### GitHub 的 GraphQL API

GitHub 在 2016 年推出 GraphQL API，他們分享了幾個有趣的數據：

使用 REST API 時，顯示一個 Pull Request 頁面需要呼叫超過 10 個 endpoint。改用 GraphQL 後，一個查詢搞定。

但他們也遇到挑戰。有使用者送出極度複雜的查詢，消耗大量資源。他們的解法是計算每個查詢的「成本」，超過限制就拒絕。

### Shopify 的經驗

Shopify 是電商平台，他們的 GraphQL API 要服務數十萬個網路商店。

他們發現最大的好處是**減少往返次數**。以前顯示一個商品頁面，平均要 5-6 個 API 呼叫。用 GraphQL 後降到 1-2 個。

但他們也強調，GraphQL 不是萬能的。對於簡單的操作（like 查詢庫存），REST 反而更簡單直接。

---

## 開始使用 GraphQL

如果你決定嘗試 GraphQL，這裡有一些建議：

### １. Schema 優先

花時間好好設計 Schema。Schema 是 GraphQL 的基石，設計不好後面會很痛苦。

幾個原則：
- 命名要一致（統一用 `userId` 或 `user_id`）
- 善用型別系統（別什麼都用 String）
- 考慮未來擴展（但不要過度設計）

### ２. 工具很重要

GraphQL 有很棒的工具生態系：
- **GraphQL Playground**：互動式的查詢介面
- **Apollo Client**：強大的前端框架
- **GraphQL Code Generator**：自動產生型別定義

善用工具可以大幅提升開發效率。

### ３. 監控和優化

上線後要密切監控：
- 查詢效能（哪些查詢特別慢？）
- 錯誤率（哪些查詢容易出錯？）
- 使用模式（使用者實際上查詢什麼？）

根據實際使用情況持續優化。

---

## 總結

最後來 Wrap up 一下這個酷東西吧，GraphQL 的核心價值是**讓前端掌控資料需求**。

它解決了 REST 的一些痛點：
- Over-fetching 和 Under-fetching
- 版本管理困難
- 多次往返的效能問題

但也帶來新的挑戰：
- 後端實作複雜度增加
- N+1 查詢問題
- 快取和安全性考量

**GraphQL 不是要取代 REST**，就像 NoSQL也沒有取代 SQL一樣，他們各有適用的場景。

如果你正在建構複雜的、資料密集的應用，需要服務多種客戶端，GraphQL 還是推薦 👍。

---

## 參考資料
 - https://graphql.org/learn/
 - https://ithelp.ithome.com.tw/m/articles/10200678
 - https://aws.amazon.com/tw/compare/the-difference-between-graphql-and-rest/
 - https://zh.wikipedia.org/zh-tw/GraphQL