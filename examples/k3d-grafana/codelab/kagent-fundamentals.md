summary: 寫給新手的 kagent 入門：什麼是 Agent、整體架構、三大資源、各元件分工，以及從 YAML 到對話的完整生命週期
id: kagent-fundamentals
categories: kagent,kubernetes,architecture
environments: Web
status: Published
feedback link: https://github.com/kagent-dev/kagent/issues
authors: kagent

# kagent 入門：原理、架構與生命週期

## 這份教學在講什麼
Duration: 0:05:00

### 適合誰

你聽過 Kubernetes 和 ChatGPT，但沒有用過 kagent。你不需要先懂 Substrate、A2A、gVisor，這些名詞都會在需要時解釋。

### 讀完你會知道

- 什麼是 AI Agent，kagent 幫你解決哪些問題
- kagent 由哪些元件組成，各自負責什麼
- `Agent`、`AgentTemplate`、`Harness` 為什麼是三個資源
- 從你 `kubectl apply` 一份 YAML，到使用者收到回答，中間發生了什麼
- 對話、暫停、還原、分岔（fork）是怎麼運作的

### 學習地圖

```
① 概念     Agent 是什麼？為什麼需要 kagent？
② 全景     四個層次、資料存在哪裡
③ 資源     Agent / AgentTemplate / Harness
④ 元件     controller、translator、Substrate 與 Actor、gateway...
⑤ 生命週期  YAML → Ready → Session → 對話 → 暫停 → 還原
⑥ 進階     工具(MCP)、子代理、人工核准、checkpoint、sandbox
```

Positive
: 想動手操作，請看另一份「在 k3d 上學 kagent 並用 Grafana 觀測」。這份教學只講「為什麼」與「怎麼運作」。

## 先備概念：AI Agent 與 kagent
Duration: 0:10:00

### AI Agent 是什麼

Agent 不是一個神秘的東西。拆開來只有四個部分：

| 部分 | 白話 | 例子 |
| --- | --- | --- |
| **模型 (Model)** | 負責思考的大型語言模型 | GPT、Claude、Gemini |
| **提示 (System Prompt)** | 告訴它「你是誰、該怎麼做」 | 「你是 Kubernetes 疑難排解專家…」 |
| **工具 (Tools)** | 讓它能對外界做事 | 查 Pod、讀 log、查 Grafana |
| **記憶 (Memory / History)** | 記得剛剛聊了什麼 | 同一段對話的前後文 |

它的運作是一個迴圈：

```
使用者提問 → 模型思考 → 需要工具嗎？
                          ├─ 是 → 呼叫工具 → 拿到結果 → 回到「模型思考」
                          └─ 否 → 回答使用者
```

### 為什麼需要 kagent

寫一個 demo agent 很簡單。要讓它在公司裡「正式上線」，問題才開始：

- 每個團隊各寫各的，怎麼**統一管理**？
- 對話中途 Pod 被換掉，**記憶會不會不見**？
- Agent 要能執行程式碼，怎麼**隔離**才安全？
- 使用者離開後 Agent 還佔著資源，怎麼**省成本**？
- 出問題時，怎麼**追蹤**它做了什麼？
- 多個人、多個 Agent，**誰能看誰的對話**？

kagent 的回答是：**把 Agent 當成 Kubernetes 的一種資源來管理**。你用 YAML 宣告「我要一個這樣的 Agent」，kagent 負責把它編譯、部署、執行、暫停、還原，並保存對話。

### 先認識幾個名詞

| 名詞 | 一句話 |
| --- | --- |
| **CRD** | Kubernetes 的自訂資源。kagent 的 Agent 就是一種 CRD |
| **Controller** | 持續比對「你想要的狀態」和「實際狀態」，並把它們拉齊的程式 |
| **A2A** | Agent2Agent 協定：client 和 Agent 溝通的標準（訊息、任務、串流） |
| **MCP** | Model Context Protocol：把「工具」提供給 Agent 的標準 |
| **gRPC** | 元件之間常用的呼叫協定 |
| **PostgreSQL** | 存對話、任務歷史、生命週期狀態的資料庫 |
| **Substrate** | kagent 用來跑 Agent 的運算後端，裡面的執行單位叫 **Actor** |

## 全景圖：四層架構
Duration: 0:10:00

kagent 可以看成四個層次，由上往下：

```
┌─────────────────────────────────────────────────────────────┐
│ ① 宣告層  你寫 YAML：Agent / AgentTemplate / Harness          │
│           儲存在 Kubernetes                                   │
├─────────────────────────────────────────────────────────────┤
│ ② 編譯層  controller + translator                            │
│           把 YAML 翻成「不可變的 revision」並交給 Substrate     │
├─────────────────────────────────────────────────────────────┤
│ ③ 執行層  Substrate Actor（真的在跑 Agent 的行程）              │
│           可暫停、可還原、可替換                                 │
├─────────────────────────────────────────────────────────────┤
│ ④ 對話層  A2A gateway + Session + PostgreSQL                 │
│           client 從這裡進來，對話歷史存在這裡                     │
└─────────────────────────────────────────────────────────────┘
```

### 一張圖看完整流程

```
        你                                        使用者 / UI / 其他 Agent
        │ kubectl apply                                   │ A2A / MCP / gRPC
        ▼                                                 ▼
  ┌───────────┐  編譯   ┌─────────────┐          ┌─────────────────┐
  │  Agent 等  │───────▶│ controller  │          │ 公開 A2A gateway │
  │ (Kubernetes)│        │ + translator│          │  (port 8083)    │
  └───────────┘        └──────┬──────┘          └───────┬─────────┘
                              │ revision                │ 解析 Session、驗證權限
                              ▼                         ▼
                        ┌───────────┐  建立/還原   ┌───────────────┐
                        │  ate-api   │◀───────────│ Session 服務    │
                        │ (Substrate)│            └───────┬───────┘
                        └─────┬─────┘                     │
                              ▼                            ▼
                        ┌───────────────────────────────────────┐
                        │ Substrate Actor（跑 Agent 的行程）       │
                        │ /data = DurableDir（私有狀態，可跨替換）  │
                        └───────────────┬───────────────────────┘
                                        │ 私有 gRPC：TaskStore
                                        ▼
                                  ┌────────────┐
                                  │ PostgreSQL │ 任務、事件、Session、checkpoint
                                  └────────────┘
```

### 資料存在哪裡？

這是理解 kagent 最重要的一張表：**誰是這份資料的「唯一真相」**。

| 東西 | 存在哪 | 白話 |
| --- | --- | --- |
| `Agent`、`Harness`、`AgentTemplate`、`SandboxTemplate` | **Kubernetes** | 你「想要什麼」 |
| 編譯好的 revision | PostgreSQL + ate-api | 「已經做好、不可修改」的版本 |
| `Session` | **PostgreSQL** | 一段對話的身分與狀態。**不是** Kubernetes 資源 |
| A2A 的 context、task、event | **PostgreSQL** | 對話與任務的完整歷史 |
| checkpoint | PostgreSQL + Substrate snapshot | 命名的「存檔點」 |
| Actor 與 DurableDir | **Substrate** | 行程本身，與它私有的檔案 |

Negative
: 常見誤解：「Session 是 CRD，可以 `kubectl get session`」。不行。Session 在資料庫裡，用 gRPC / A2A / MCP 操作。

### 四個對外入口

| 入口 | 用來做什麼 |
| --- | --- |
| **Kubernetes API** | 撰寫 Agent、Harness、AgentTemplate、模型、MCP server |
| **gRPC / gRPC-Web** | 管理 Session、分享、checkpoint、sandbox（UI 用這個） |
| **A2A** | 和 Agent 對話、管理任務與串流 |
| **MCP** | 讓外部 client 列出/呼叫 Session、checkpoint、fork、sandbox |

一條設計原則貫穿全部：**kagent 不自己發明第二套 session 或 task 模型**，對話與任務的語意完全交給上游 A2A。

## 三大資源：Agent、AgentTemplate、Harness
Duration: 0:15:00

三個資源都是 `api.kagent.dev/v1alpha3`。

### 用餐廳來理解

| 資源 | 餐廳比喻 | 回答的問題 |
| --- | --- | --- |
| **AgentTemplate** | 食譜 | 這個 Agent「**做什麼**」？ |
| **Harness** | 廚房與廚師 | 它「**怎麼跑**」？ |
| **Agent** | 一位實際上工的廚師 | 一份食譜 + 一個廚房 |

好處：同一份食譜（行為）可以換不同廚房（runtime）；同一個廚房也能讓很多份食譜共用。

### AgentTemplate：做什麼

包含：

- 模型設定 (`modelConfig`)
- 描述與 system prompt
- 工具綁定（MCP 工具）
- skills、plugins
- 子代理 (`subAgent`)

```yaml
apiVersion: api.kagent.dev/v1alpha3
kind: AgentTemplate
metadata:
  name: k8s-helper
  namespace: kagent
spec:
  modelConfig:
    name: default-model-config
  description: 協助檢查 Kubernetes 工作負載
  systemPrompt: |
    你是 Kubernetes 疑難排解專家。先看 Pod 狀態，再看事件與 log，
    最後用一兩句話說明根本原因。
  tools:
  - mcp:
      server: {kind: RemoteMCPServer, name: kagent-tool-server}
      tools: [k8s_get_resources, k8s_get_events, k8s_get_pod_logs]
```

Positive
: AgentTemplate 沒有 runtime status。它只是「可重用的行為描述」。

### Harness：怎麼跑

選擇**恰好一種** runtime，並描述執行環境：

| Runtime | 說明 |
| --- | --- |
| `kagent` | kagent 自己的 ADK（Go 或 Python） |
| `codex` | OpenAI Codex |
| `claude` | Claude |
| `byo` | Bring Your Own：你自己的映像檔（例如 LangGraph） |

Harness 還包含：映像檔（必須用 digest 固定）、command / args、環境變數與憑證參照、WorkerPool、snapshot 存放位置。

```yaml
apiVersion: api.kagent.dev/v1alpha3
kind: Harness
metadata:
  name: kagent-go
  namespace: kagent
spec:
  kagent: {}
  workload:
    image: example.com/runtime@sha256:0000...   # 必須是 digest
  substrate:
    workerPoolRef:
      name: kagent-default
    snapshotPolicy:
      location: s3://snapshots/kagent/
```

### Agent：把兩者組起來

`Agent` 把一個 template 和一個 harness 配成一對。兩邊各自可以選「內嵌 (inline)」或「參照 (ref)」，**各自恰好選一個**：

```yaml
# 寫法一：兩邊都參照（最常用，方便重用）
spec:
  templateRef: {name: k8s-helper}
  harnessRef:  {name: kagent-go}
```

```yaml
# 寫法二：template 內嵌、harness 參照（最小的 Agent）
spec:
  template:
    modelConfig: {name: default-model-config}
    systemPrompt: 你是一個簡潔友善的助理。
  harnessRef: {name: kagent-go}
```

Negative
: 內嵌的 spec 是「完整的值」，不是「覆蓋 (override)」。`AgentTemplate` 與 `Harness` 彼此沒有綁定關係，只有 `Agent` 把它們湊在一起。

### Agent 自己有什麼

Agent 擁有 **status**：是否 Ready、警告、`desiredRevision`（想要的版本）、`latestSuccessfulRevision`（最近成功的版本）。

### 其他你會遇到的資源

| 資源 | 用途 |
| --- | --- |
| `ModelConfig` | 用哪家模型、金鑰在哪 |
| `ModelProviderConfig` | 模型供應商設定 |
| `RemoteMCPServer` | 一個遠端 MCP 工具伺服器 |
| `SandboxTemplate` | 獨立 sandbox 的工具環境（後面會講） |

### 為什麼 API 不暴露 Kubernetes 細節

公開 API 只描述「Agent 的**行為**」，不會讓你寫 service account、排程限制、任意容器。這是刻意的：基礎設施怎麼跑是 kagent 的事，這樣底層可以演進，你的 YAML 不用跟著改。

## 元件導覽：誰負責什麼
Duration: 0:15:00

原則：**每個元件只做一件事**。

### 控制面（在 kagent 命名空間）

| 元件 | 職責 |
| --- | --- |
| **controller** | 監看 Agent 等資源，驅動編譯與部署；同時是 gateway 所在處 |
| **translator** | 解析引用、把設定「編譯」成 revision（只編譯，不套用） |
| **harness compiler** | 每種 runtime 一個（kagent / Codex / Claude / BYO），把輸入翻成該 runtime 的格式 |
| **Session 服務** | 建立、暫停、還原、刪除 Session；權限檢查 |
| **A2A gateway** | 對外的 A2A 入口，把請求送到正確的 Actor |
| **TaskStore（私有 gRPC）** | 讓 Actor 把任務進度寫進 PostgreSQL |
| **PostgreSQL** | 保存 Session、任務事件、revision、checkpoint |
| **UI** | 網頁介面，透過 gRPC-Web 與 A2A 使用上面的功能 |
| **MCP endpoint** (`/mcp`) | 讓其他 client 把 kagent 當工具用 |

### 運算面（Substrate，在 ate-system 命名空間）

| 元件 | 職責 |
| --- | --- |
| **ate-api** | Substrate 的 API：管理 ActorTemplate、Actor、snapshot |
| **Actor** | 真的在跑 Agent 的隔離行程（預設 gVisor sandbox） |
| **ActorTemplate** | Actor 的模板，由編譯好的 revision 產生 |
| **WorkerPool** | 一批能跑 Actor 的 worker，決定沙箱類別（`gvisor` 或 `microvm`） |
| **DurableDir** | 掛在 Actor 的 `/data`，放要跨 Actor 替換而保留的私有狀態 |
| **snapshot** | Actor 的記憶體/狀態存檔，用來快速還原 |

### 重要的埠

| 埠 | 用途 |
| --- | --- |
| `8083` | 對外：gRPC、gRPC-Web、A2A、MCP、health |
| Actor `:80` | **私有**：Actor 的 A2A gRPC |
| Actor `:8081` | **私有**：`/readyz` 就緒檢查 |

Positive
: client 永遠拿不到 Actor 的位址。gateway 透過私有網路自己找到 Actor。

### 邊界規則（為什麼這樣切）

- 傳輸層（gRPC / HTTP handler）只做格式轉換，不做流程。
- Service 與 workflow 決定「順序」，每一步交給擁有它的元件。
- 編譯器**只翻譯**，套用與就緒檢查由 controller 和 Substrate adapter 負責。
- Store **只存資料**、保證交易一致；它不啟動 Actor，也不做網路呼叫。
- 絕不在資料庫交易或鎖之中做網路呼叫。

## 深入理解：Substrate 與 Actor
Duration: 0:12:00

前面一直提到 Substrate 和 Actor。這一步把它們講清楚，因為後面的生命週期都建立在它們之上。

### 一句話

**Substrate 是一個「跑隔離小行程，並且能隨時把它們存檔、叫醒」的平台。** kagent 自己不啟動容器來跑 Agent，而是把這件事交給 Substrate。**Actor** 是 Substrate 裡的一個執行單位，也就是一個正在跑的 Agent。

### 為什麼不直接用 Pod？

Agent 的使用模式是「講一下、停很久、再講一下」。如果每個對話都開一個 Pod：

| 問題 | 一般 Pod | Substrate Actor |
| --- | --- | --- |
| 使用者去吃飯了，Agent 閒著 | Pod 一直佔著記憶體 | 可以**暫停**，幾乎不佔資源 |
| 使用者回來了 | 冷啟動，要等很久 | 從存檔**快速還原** |
| Agent 會執行 AI 產生的程式碼 | 和其他 Pod 共用核心，風險高 | 預設跑在 **gVisor 沙箱**裡 |
| Pod 被換掉 | 行程內的狀態全部消失 | 狀態可以存檔，也能跨替換保留 |
| 一千個閒置對話 | 一千個 Pod | 一千個暫停中的 Actor |

### 用「遊戲存檔」來理解

```
ActorTemplate   =  遊戲光碟（一個固定的版本）
golden snapshot =  剛開機、就緒時拍下的「初始存檔」
Actor           =  一個正在玩的存檔槽
pause / suspend =  離開遊戲，把進度存起來
resume          =  讀檔，從剛剛的地方繼續
```

### 名詞對照

| 名詞 | 白話 | 在 kagent 裡對應什麼 |
| --- | --- | --- |
| **ActorTemplate** | Actor 的模板 | 由編譯好的 **revision** 產生，一個 revision 對應一個 |
| **golden snapshot** | 模板啟動一次、就緒後拍的初始存檔 | 新對話都從這裡還原，所以不必重新開機 |
| **Actor** | 依模板生出來的實際行程 | 一個 **Session** 對應一個 Actor |
| **WorkerPool** | 一批能承載 Actor 的機器 | Harness 的 `workerPoolRef` 指向它 |
| **sandbox class** | Actor 的隔離強度 | `gvisor`（預設）或 `microvm` |
| **DurableDir** | 掛在 Actor `/data` 的持久目錄 | 放要跨 Actor 替換而保留的私有狀態 |
| **ate-api** | Substrate 的 API 入口 | controller 透過它建立 ActorTemplate 與 Actor |

### 它怎麼和 kagent 接在一起

```
你 apply Agent
   ↓ 編譯
revision（不可變的設定包）
   ↓ controller 呼叫 ate-api
建立 ActorTemplate → 啟動一次 → 拍 golden snapshot     ← Agent 變成 Ready
   ↓
使用者開新對話 = 建立 Session
   ↓
Substrate 建一個 Actor（起始是 suspended，來自 golden snapshot）
   ↓
使用者送訊息 → Substrate 入口把 Actor 叫醒 → Agent 開始跑
   ↓
回答完 → 背景 worker 把 Actor suspend，記下 snapshot
   ↓
下次來訊息 → 再叫醒
```

同一個 revision 的第二個、第三個 Actor，都是從同一份 golden snapshot 長出來的，所以開新對話很快。

### pause 和 suspend 的差別

| 動作 | 什麼時候 | 說明 |
| --- | --- | --- |
| **pause** | 任務在等人輸入（例如等你按核准） | 留在原節點暫停，因為很快就會回來 |
| **suspend** | 任務結束了 | 拍下確切的 snapshot 再暫停，之後可以在別的節點還原 |

### 容易搞混：Actor 不是「真相的所在」

| 資料 | 存哪 | 說明 |
| --- | --- | --- |
| 對話歷史、任務狀態 | **PostgreSQL** | Actor 壞了或被換掉，它還在 |
| Actor 行程的記憶體狀態 | **snapshot** | 用於暫停與還原 |
| 框架私有檔案、工作區、下載的資源 | **DurableDir**（`/data`） | runtime 私有，不是公開歷史 |

所以 **Actor 是可替換的算力**，重要的東西存在別處。這也是為什麼說 Session 是穩定的身分，Actor 只是它目前用的那台機器。

Positive
: 換個角度想：Kubernetes 的 Deployment 管的是「一直開著的服務」，Substrate 管的是「隨時可以睡著、又能秒醒的對話」。

Negative
: Substrate 是獨立的專案。kagent 文件只描述它與 kagent 的接點（ate-api、ActorTemplate、WorkerPool、DurableDir、snapshot）。它的內部實作請看 Substrate 自己的文件。

## 生命週期一：從 YAML 到 Ready
Duration: 0:15:00

你執行 `kubectl apply -f agent.yaml` 之後：

```
Agent
 ├─ template / templateRef ─┐
 └─ harness  / harnessRef  ─┴─▶ ① 解析（把引用與子代理樹全部展開）
                                   │
                                   ▼
                               ② 建立「明確輸入」（與 runtime 無關）
                                   │
                                   ▼
                               ③ 依 runtime 選 compiler
                                  kagent / Codex / Claude / BYO
                                   │
                                   ▼
                               ④ 產生不可變 revision
                                  （workload、設定、Agent Card、容量、snapshot、出處、egress）
                                   │
                                   ▼
                               ⑤ 選 WorkerPool 沙箱類別、計算 digest、
                                  交給 ate-api 建立 ActorTemplate
                                   │
                                   ▼
                               ⑥ 等 golden snapshot 就緒
                                   │
                                   ▼
                               ⑦ 記錄 revision，更新 latestSuccessfulRevision → Ready
```

### 關鍵字：revision 與 golden snapshot

- **revision**：某一時刻「編譯完成、不可再改」的 Agent 版本。改 prompt、換模型、改 Harness 的 command，都會產生**新的** revision。
- **golden snapshot**：用這個 revision 先啟動一個 Actor、等它就緒後拍下的存檔。之後新對話直接從這份存檔還原，**不用重新開機**，所以很快。

### 失敗時會怎樣

| 情況 | 結果 |
| --- | --- |
| 編譯失敗（引用不存在、有循環…） | status 顯示原因，`desiredRevision` 為空；**前一個成功版本仍可使用** |
| Substrate 或資料庫暫時失敗 | `Ready=False`，原因 `RuntimePreparationFailed`；controller 以 1 秒到 30 秒的指數退避不斷重試 |
| 找不到 WorkerPool | `WorkerPoolNotFound` |
| 不支援的沙箱類別 | `RevisionInvalid` |
| 設定太大（單一環境變數超過 32,768 字元） | `Compatible=False`，`ActorTemplateInvalid` |

Positive
: 重點：**編譯或套用失敗，不會弄壞正在跑的對話**。舊版本一直保留。

### 為什麼要「不可變」

因為 **Session 會釘住 (pin) 某一個 revision**：

- 你改了 prompt 並 apply → 產生新 revision。
- **已經開始的對話**繼續用舊 revision，不會突然「性格大變」。
- **新開的對話**使用最新成功的 revision。

### 除錯入口

```console
kubectl get agent <name> -o yaml        # 看 status.conditions、revision
kubectl -n ate-system get actortemplates
kubectl get workerpools
kubectl logs deploy/kagent-controller
```

## 生命週期二：Session
Duration: 0:10:00

### Session 是什麼

**Session = 一段對話的穩定身分**。特點：

- 存在 PostgreSQL，不是 Kubernetes 資源。
- `Session.id` 就是公開 A2A 的 `contextId`。
- 釘住**一個** revision，並對應**一個** Substrate Actor。
- Actor 可以被換掉，Session（與歷史）不會變。

用旅館來比喻：Session 是你的「訂房紀錄」，Actor 是「你住的那間房」。房間可以換，訂房紀錄與行李（歷史）仍在。

### 建立 Session

1. 選擇該 Agent 的 **latest successful revision**。
2. 建立一個**決定性命名**的 Actor，起始狀態是 **suspended（暫停）**。
3. Substrate 接受後，Session 標成 **ready**。

注意：建立 Session **不會**特地把 Actor 叫醒來檢查健康，因為映像的就緒性在準備 ActorTemplate 時就已確認。

### 顯式操作

| 操作 | 效果 |
| --- | --- |
| **Suspend** | 邏輯上暫停 |
| **Resume** | 恢復 |
| **Delete** | 先關閉新任務、停止並刪除 Actor，最後留下墓碑 (tombstone) |

已在目標狀態時再叫一次 Suspend/Resume，視為成功（idempotent）。

### 客戶端要記得重試

Session 生命週期操作跨越「資料庫」與「Substrate」兩個系統。kagent 用「持久化的階段 + 可重試 + 補償清理」處理。若遇到暫時性錯誤：

- **重送同一個變更請求**（`Get` 只能觀察，不會推進它）。
- 如果 client 不再重試，這個操作會一直保持 pending，即使 API 重啟。

### 閒置到期

預設 **7 天**沒有任務活動就會自動刪除 Session（`controller.sessionIdleTTL`，`0` 代表關閉）。

- 執行中、或在等人輸入 (`INPUT_REQUIRED` / `AUTH_REQUIRED`) 的任務**永不**過期。
- 過期只刪 Session 的 Actor 與存取權；A2A 歷史與明確建立的 checkpoint **仍留在資料庫**。

## 生命週期三：一次對話的旅程
Duration: 0:15:00

### 步驟總覽

```
Client ──(1) 送出訊息──▶ Gateway
                          │ (2) 建立或找到 Session、預留「派送」
                          ▼
                       Actor  ◀── 必要時從 snapshot 還原
                          │ (3) 執行：思考、呼叫工具
                          │ (4) 每一步都寫進 TaskStore
                          ▼
                       PostgreSQL
                          │ (5) 完成後「原子地」發布任務狀態與歷史
                          ▼
Client ◀──(6) 串流/最終結果── Gateway
                          
       （client 收到答案後，背景的 lifecycle worker 才處理暫停）
```

### (1) 路由

公開端點就是一個具名 Agent：`/agents/{namespace}/{name}`。

- 訊息**沒帶** context ID → 建立一段新對話（新 Session）。
- 帶 `contextId` → 繼續那段對話。
- 只帶 task ID → 由它反查 Session。
- 兩個都帶，必須一致。

每次操作都檢查：這個 Session 屬於此 Agent，且呼叫者是擁有者或被分享者。

### (2) 派送預留

Gateway 先向 Session 服務取得「派送預留 (dispatch reservation)」，才把訊息轉給 Actor。預留期間，閒置的暫停、checkpoint、顯式生命週期操作都被擋住，直到 Actor 存下「進行中」的任務。每次預留兩分鐘後過期。

### (3)(4) 執行與持久化

- 一個 Actor **同一時間只跑一個任務**。
- Actor 每產生一個更新，先透過 **TaskStore** 寫進資料庫，**再**串流給 client。
- 使用版本檢查（樂觀鎖），重試不會產生重複歷史。

### A2A 任務狀態

| 狀態 | 意思 |
| --- | --- |
| `submitted` / `working` | 已接收 / 執行中 |
| `input-required` | 需要人類輸入（例如核准或回答問題） |
| `auth-required` | 需要授權 |
| `completed` | 完成 |
| `failed` / `canceled` | 失敗 / 被取消 |

### (5) 原子發布

完成時，Actor 先做完本地清理，再確認最後一次儲存；資料庫在**同一個交易**內發布任務狀態與歷史。所以 client 一定能讀到完成的結果，即使後面的暫停很慢或失敗。

### 自動休眠（Quiescence）：省成本的關鍵

回答完之後，一個獨立的 **Session lifecycle worker** 在背景處理：

| 任務狀態 | 動作 |
| --- | --- |
| 在等人輸入 (`input-required` / `auth-required`) | **pause**：留在原節點暫停 |
| 已結束 (terminal) | **suspend**：記下確切的 snapshot 後暫停 |

- Session 邏輯上仍是 **READY**。
- 下次有授權的請求進來，Substrate 入口會**自動把 Actor 叫醒**。
- 若新的一輪在 worker 認領之前就到，新的一輪會**優先**，休眠工作作廢。

### (6) 重連

- `GetTask` / `ListTasks` 只讀資料庫，**不會叫醒 Actor**。
- `SubscribeToTask`：已完成的任務直接回傳儲存結果；進行中的接上即時串流。
- **沒有**事件游標，也不保證重播每個 token。重連後以「目前任務」取代畫面即可。

Negative
: 若送出訊息後連線中斷、結果不明，**不要盲目重送**。已知 context / task ID 時，請用 `GetTask` 或 `SubscribeToTask` 查結果。建立新對話的第一則訊息，用相同 message ID 重試是安全的。

## 工具、子代理與人工核准
Duration: 0:10:00

### 工具：MCP

Agent 靠 **MCP** 取得工具。你在 AgentTemplate 用 `RemoteMCPServer` 引用一個工具伺服器，再列出允許使用的工具名稱：

```yaml
tools:
- mcp:
    server: {kind: RemoteMCPServer, name: kagent-tool-server}
    tools: [k8s_get_resources, k8s_get_events]
```

只列出你需要的工具，是最簡單的最小權限做法。

kagent 自己也提供一個 **MCP endpoint**（`/mcp`，埠 8083），讓外部 client 能列出與呼叫 Session、建立 checkpoint、fork，以及使用 sandbox。

### 子代理 (Shared sub-agent)

一個 template 可以把另一個 template 當成「工具」使用：

```yaml
tools:
- subAgent:
    name: reviewer
    description: 審查提議的變更
    templateRef: {name: review-context}
```

- 子代理**共用父代理的 runtime 與 Harness**，在同一個 Actor 內以行程內方式執行。
- 編譯時會檢查缺少的引用與循環。
- 另有 `agentRef`（Dedicated，獨立 Agent、獨立 Session），**目前尚未支援**。

### 人工核准（Human in the Loop）

Agent 可以在做危險操作前暫停，等人類決定：

1. Agent 送出 `input-required`，訊息中帶著核准請求（哪個工具、什麼參數）。
2. Gateway 把這個狀態存起來，Actor 在該邊界休眠。
3. 人類（UI 上按 Approve / Reject）送出決定，回到**同一個** task。
4. Runtime 驗證決定，從**原本暫停的位置**繼續。

Positive
: 這不是「把問題重新丟給模型」。恢復的是被暫停的那個精確操作。沒有第二套任務模型。

Negative
: 哪些工具會觸發核准，取決於 runtime 與工具本身，以及你安裝版本的 CRD 欄位。請以 `docs/architecture/human-in-the-loop.md` 與實際 CRD 為準。

## Checkpoint 與 Fork
Duration: 0:08:00

### 概念

- **Checkpoint**：對話的一個「命名存檔點」= 資料庫的歷史邊界 + Substrate snapshot。
- **Fork**：從某個 checkpoint **長出一段新的對話**。

```
對話 A:  ──●──●──●──●──●──▶
                 │
                 └─ checkpoint "before-fix"
                       │
對話 B (fork):          └──●──●──▶   （新的 session、context、history）
```

### 規則

- 只有**已完成的終態任務**可以建立 checkpoint；在等人輸入的任務不能。
- 建立時必須帶 `expected_head_task_id`，確認你要存的正是那個終態任務。
- snapshot 是背景記錄的，所以剛完成時可能收到 `FailedPrecondition`（`KAGENT_CHECKPOINT_SNAPSHOT_PENDING`），**用同一個請求重試**即可。
- Fork 得到全新的 session / context / history ID，並複製到 checkpoint 為止的歷史。
- 來源 Session 過期後，保留的 checkpoint 仍可 fork。

### 用途

「這一步之前很好，想試另一種問法」、實驗不同 prompt、除錯重現。

## Sandbox：獨立的暫存工作區
Duration: 0:05:00

### 是什麼

**Sandbox** 是一個個人專屬的暫存環境，可以執行指令、讀寫檔案，到期或刪除即消失。

| 設定 | 執行期資源 | 介面 |
| --- | --- | --- |
| `Agent`（AgentTemplate + Harness） | `Session` | A2A 訊息與任務 |
| `SandboxTemplate` | `Sandbox` | 行程與檔案操作 |

### 重要觀念

- Session **不會**跑 sandbox；兩者生命週期各自獨立。
- Agent 想用 sandbox，是透過 **MCP** 建立一個獨立的 Sandbox（人類用同一套服務）。
- 建立 `SandboxTemplate` 只是準備可重用的執行環境，**不會**配置實際的 Sandbox。
- 映像檔必須以 digest 固定。

## 觀測與名詞表
Duration: 0:07:00

### 觀測 (Telemetry)

kagent 可匯出 **OpenTelemetry** traces，並以 `gen_ai.*`、`a2a.*`、`kagent.*` 屬性描述每次呼叫，例如：

- `gen_ai.operation.name = invoke_agent`：一次 Agent 呼叫
- `gen_ai.conversation.id`、`a2a.task.id`：對話與任務
- `gen_ai.usage.*`：token 用量

Negative
: 開啟 `otel.capture.messageContent` 會把 prompt 與回應寫進 span，適合本機除錯，正式環境請審慎。

### 名詞表

| 名詞 | 意思 |
| --- | --- |
| Agent | 一個 AgentTemplate + 一個 Harness 的組合 |
| AgentTemplate | Agent 的行為（模型、prompt、工具） |
| Harness | Agent 的執行方式（runtime、映像、資源） |
| Revision | 編譯完成、不可變的 Agent 版本 |
| Golden snapshot | 依 revision 預先啟動並拍下的存檔，加速新對話 |
| Substrate | 運算後端 |
| Actor | 跑 Agent 的隔離行程 |
| WorkerPool | 能跑 Actor 的 worker 集合 |
| DurableDir | Actor 的 `/data`，跨 Actor 替換保留 |
| Session | 一段對話的穩定身分（= A2A contextId） |
| Task | A2A 中的一次工作單位 |
| TaskStore | Actor 把任務進度寫進資料庫的私有通道 |
| Quiescence | 閒置時自動 pause / suspend |
| Checkpoint | 對話的命名存檔點 |
| Fork | 從 checkpoint 長出新對話 |
| HITL | 人工介入（核准、回答問題） |
| MCP | 工具協定 |
| A2A | Agent 對話與任務協定 |

## 常見問題與下一步
Duration: 0:05:00

### 常見問題

**改了 prompt，為什麼舊對話沒變？**
因為 Session 釘住舊 revision。開新對話才會用新版本。

**Agent 一直 `Ready=False`？**
先看 `kubectl get agent X -o yaml` 的 conditions，再看 controller log 與 ActorTemplate。

**對話中途 Pod 被重建，記憶會不見嗎？**
不會。任務歷史在 PostgreSQL；Actor 私有狀態在 DurableDir 與 snapshot。

**為什麼查歷史不會叫醒 Actor？**
`GetTask` / `ListTasks` 只讀資料庫，設計上就是如此。

**我可以自己寫 Agent 嗎？**
可以，用 BYO Harness，但映像必須實作私有 A2A gRPC、`/readyz`，並透過 TaskStore 保存任務。

**目前還沒做完的？**
Dedicated 子代理（`agentRef`）、有政策管控的跨 Session 委派、checkpoint 分享、多副本 gateway 協調。

### 下一步

- 動手：「在 k3d 上學 kagent 並用 Grafana 觀測」
- 深入閱讀 `docs/architecture/`：
  - `configuration-and-compilation.md`
  - `runtime-and-lifecycle.md`
  - `a2a-gateway.md`
  - `persistence-checkpoints-and-forks.md`
  - `human-in-the-loop.md`
  - `telemetry.md`
