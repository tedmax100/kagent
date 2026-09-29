summary: 在本機 k3d 叢集上，從零學會 kagent 的 Harness / AgentTemplate / Agent、MCP 工具、人工核准、子代理、BYO LangGraph，並用 Grafana LGTM 觀測代理
id: kagent-k3d-grafana
categories: kagent,kubernetes,observability
environments: Web
status: Published
feedback link: https://github.com/kagent-dev/kagent/issues
authors: kagent

# 在 k3d 上學 kagent 並用 Grafana 觀測

## 開始之前
Duration: 0:05:00

### 你會學到什麼

- kagent v1alpha3 的三個核心資源：**Harness**（怎麼跑）、**AgentTemplate**（做什麼）、**Agent**（兩者組合）
- 讓代理透過 **MCP** 使用工具，並對危險操作要求**人工核准**
- 用 **子代理（sub-agent）** 組成值班 (on-call) 主管
- 帶入你自己的 **LangChain / LangGraph** 代理（BYO）
- 用 **Grafana LGTM**（Tempo / Loki / Prometheus）看到每一次代理呼叫的 trace、log、metrics

### 你需要準備

- `docker`、`k3d` ≥ 5.8、`kubectl`、`helm` ≥ 3.14、`jq`、`openssl`、`curl`
- 約 6 GB 可用記憶體，以及足夠的磁碟空間
- 一個模型：`OPENAI_API_KEY`、`ANTHROPIC_API_KEY`、`GOOGLE_API_KEY` 擇一；都沒設時會改用本機 Ollama

Negative
: 使用 1.5B 這類小模型只適合範例 1，工具呼叫不穩定。範例 2–4 請用雲端金鑰或 `qwen2.5:7b` 以上的本機模型。

### 架構總覽

```
                      k3d cluster "kagent" (k8s 1.35)
 ┌──────────────────────────────────────────────────────────────────────┐
 │ ns kagent                              ns ate-system (Substrate)     │
 │  kagent-controller ── A2A :8083 ─────▶  ate-api / atelet / router    │
 │  kagent-ui :8080                        WorkerPool kagent-default    │
 │  kagent-tools (MCP: k8s_*)               └─ gVisor Actors = agents   │
 │  kagent-grafana-mcp (MCP) ──┐                     │ OTLP             │
 │  postgres                   │                     ▼                  │
 │ ns observability            └────────▶  lgtm: Grafana :3000          │
 │  (controller + agents ── OTLP :4317 ─▶        Tempo / Loki / Prom)   │
 │ ns demo: web, api (crashloop), worker (bad image), loadgen           │
 └──────────────────────────────────────────────────────────────────────┘
```

每個代理實際上跑在 Substrate 的 gVisor **Actor** 裡；controller 對外提供 A2A gateway，UI 與 `chat.sh` 都透過它和代理對話。

## 調整系統限制
Duration: 0:03:00

k3s 節點會使用 inotify。Ubuntu 預設值（128）在建立兩三個叢集後就會耗盡，新節點會卡在 `Waiting for containerd startup`（`too many open files`）。

```console
sudo sysctl -w fs.inotify.max_user_instances=1024 fs.inotify.max_user_watches=524288
```

永久生效：

```console
printf 'fs.inotify.max_user_instances=1024\nfs.inotify.max_user_watches=524288\n' \
  | sudo tee /etc/sysctl.d/99-inotify.conf
```

Positive
: k3d 節點看得到主機整顆磁碟。`up.sh` 已把 kubelet 的映像檔 GC 調低，避免磁碟快滿時映像被刪；空間要自己管理（`docker system df`）。

## 啟動叢集
Duration: 0:15:00

在 repo 根目錄執行：

```console
export OPENAI_API_KEY=sk-...           # 選填
./examples/k3d-grafana/up.sh           # 第一次約 10 分鐘（拉映像檔）
```

新開一個終端機，讓 port-forward 保持執行：

```console
./examples/k3d-grafana/forward.sh
```

| URL | 用途 |
| --- | --- |
| http://localhost:8080 | kagent UI |
| http://localhost:8083 | controller：A2A gateway、gRPC-Web、`/mcp` |
| http://localhost:3000 | Grafana（admin / admin） |

### up.sh 幫你繞過的 k3d 問題

| 問題 | 症狀 | up.sh 的處理 |
| --- | --- | --- |
| Substrate 需要 beta Kubernetes API | Pod 卡在 `ContainerCreating`（`podcertificate` volume） | k3s 1.35 並在 apiserver、controller-manager、kubelet 開 feature gates |
| 磁碟 >85% 時 kubelet 映像 GC | 拉好的映像消失、一直拉不完 | `image-gc-high-threshold=99` |
| 連 ghcr.io 慢或不穩 | helm `--wait` 逾時 | 用 `ctr` 平行預拉所有映像 |
| atelet 自己抓代理映像，且只試一次 | `GoldenActorCrashed`、atelet log `PROTOCOL_ERROR` | 本機 registry `kagent-registry`（`localhost:5001`）+ `--localhost-registry-replacement` |
| gVisor 無法在節點 overlayfs 上放 overlay | `GoldenActorCrashed`、`not supported as upperdir` | 把 `/var/lib/ateom-gvisor` bind-mount 到 ext4 的 k3s volume |
| sandbox 的 DNS 不認得 `host.k3d.internal` | 模型呼叫 `no such host` | `ollama` Service 指向主機 gateway IP |

Negative
: bind mount 不會撐過節點重啟（`docker restart`、重開機）。重跑 `up.sh` 即可，它是冪等的。

### 設定方便的別名

```console
alias k='kubectl --context k3d-kagent -n kagent'
alias chat=./examples/k3d-grafana/chat.sh
```

Positive
: API group 是 `api.kagent.dev`。如果 kubectl 也認得舊版 `kagent.dev` 的 Agent，請改用完整名稱 `agents.api.kagent.dev`。

## 範例 1：hello，最小的 Agent
Duration: 0:10:00

`up.sh` 已經替你建立好這個範例。先理解三個角色：

- **Harness** = *怎麼*跑：runtime、映像 digest、WorkerPool、snapshot 位置
- **template** = *做什麼*：模型、system prompt、工具
- **Agent** = 一個 template + 一個 harness

### 共用的 Harness

```yaml
apiVersion: api.kagent.dev/v1alpha3
kind: Harness
metadata:
  name: kagent-go
  namespace: kagent
spec:
  kagent: {}
  workload:
    image: localhost:5001/kagent-dev/kagent/golang-adk@sha256:...   # Substrate 要求 digest
  substrate:
    workerPoolRef:
      name: kagent-default
    snapshotPolicy:
      location: s3://ate-snapshots/kagent
```

### 最小的 Agent（inline template）

```yaml
apiVersion: api.kagent.dev/v1alpha3
kind: Agent
metadata:
  name: hello
  namespace: kagent
spec:
  template:
    modelConfig:
      name: default-model-config
    description: A friendly general-purpose assistant.
    systemPrompt: |
      You are a concise, friendly assistant running on kagent inside a local k3d cluster.
  harnessRef:
    name: kagent-go
```

### 動手玩

```console
k get harnesses,agents
k get agent hello -o jsonpath="{.status}" | jq
chat hello "Hi, introduce yourself in one sentence"
```

`chat.sh` 會印出 `CONTEXT=...`。帶上它就是同一段對話：

```console
CONTEXT=<印出的 contextId> chat hello "What did I just ask you?"
```

### 試試看

1. 修改 `systemPrompt` 後再 `k apply`。觀察 status 裡的 `desiredRevision` / `latestSuccessfulRevision`：每次變更都會產生新的不可變 revision（Substrate 啟動 golden actor 並做 snapshot）。既有對話會固定在舊 revision，新對話才用新的。
2. 看 Substrate 背後的物件：

```console
k -n ate-system get actortemplates
k get workerpools
```

## 範例 2：K8s 疑難排解（MCP 工具與核准）
Duration: 0:15:00

### 部署故障工作負載與代理

```console
kubectl --context k3d-kagent apply -f examples/k3d-grafana/manifests/demo/broken-apps.yaml
k apply -f examples/k3d-grafana/manifests/02-k8s-agent.yaml
k get agent k8s-troubleshooter -w          # 等 Ready=True
```

### 工具分兩組綁定

```yaml
tools:
- mcp:                         # 唯讀工具：直接執行
    server: {kind: RemoteMCPServer, name: kagent-tool-server}
    tools: [k8s_get_resources, k8s_describe_resource, k8s_get_events,
            k8s_get_pod_logs, k8s_get_resource_yaml]
- mcp:                         # 會改動叢集：需要人工核准
    server: {kind: RemoteMCPServer, name: kagent-tool-server}
    requireApproval: true
    tools: [k8s_patch_resource, k8s_scale, k8s_rollout, k8s_delete_resource]
```

Positive
: `AgentTemplate` 是獨立資源，Agent 用 `templateRef` 引用。這也讓範例 4 可以把它當子代理重用。

### 提問

```console
chat k8s-troubleshooter "Which pods in namespace demo are unhealthy, and why?"
```

### 體驗人工核准

在 **UI**（http://localhost:8080）向 `k8s-troubleshooter` 說：

> Fix the worker deployment's image tag to nginx:1.27-alpine

代理會在 patch 之前停下（A2A 狀態 `input-required`），等你按 **Approve** 或 **Reject**。

### 查看 MCP server 提供的所有工具

```console
k get remotemcpservers kagent-tool-server -o jsonpath='{.status.discoveredTools[*].name}'
```

## 範例 3：Grafana 分析師
Duration: 0:10:00

`up.sh` 已建立 Grafana service account token，並讓 chart 的 `kagent-grafana-mcp` 指向 LGTM。`broken-apps.yaml` 內的 `loadgen` 會持續送出 OTLP traces、logs、metrics。

```console
k apply -f examples/k3d-grafana/manifests/03-grafana-agent.yaml
chat grafana-analyst "Which Prometheus and Loki datasources are available?"
chat grafana-analyst "How many error logs did service checkout emit in the last 10 minutes? What does a typical one say?"
chat grafana-analyst "What is the p95 latency of the spans for service checkout? Show the PromQL you used."
```

Positive
: 若最後一題找不到 metrics，請它先列出名稱含 `traces_span` 或 `duration` 的 metric。這些名稱取決於 LGTM 從 span 產生的 metric，自己摸索出來本身就是很好的練習。

## 範例 4：值班主管與子代理
Duration: 0:10:00

```console
k apply -f examples/k3d-grafana/manifests/04-oncall-agent.yaml
chat oncall "Users report checkout is failing. Triage namespace demo and the checkout service."
```

`oncall` 不直接碰工具，而是委派：

```yaml
tools:
- subAgent:
    name: kubernetes
    description: Inspects pods, events and logs in the cluster.
    templateRef: {name: k8s-troubleshooter}
- subAgent:
    name: observability
    description: Queries Prometheus metrics and Loki logs through Grafana.
    templateRef: {name: grafana-analyst}
```

- `subAgent.templateRef` 重用範例 2、3 的 template。
- 它們以 Shared 子代理的身分跑在 oncall 的 actor 內，不需要額外的 Agent。
- 到 Grafana → Explore → Tempo 可以看到父層 `invoke_agent` span，底下每次委派各有一個子 `invoke_agent` span。

## 範例 5：BYO LangGraph 代理
Duration: 0:20:00

[`byo-langgraph/`](https://github.com/kagent-dev/kagent/tree/main/examples/k3d-grafana/byo-langgraph) 是一個普通的 LangGraph ReAct 代理（`agent.py` 內有三個 Python 工具），由 `server.py` 包裝成 Substrate 能執行的形式。Harness 使用 `byo: {}` 取代 kagent runtime；AgentTemplate 仍提供模型與 system prompt，所以這兩者可以用 `kubectl apply` 修改，工具則是程式碼。

```console
./examples/k3d-grafana/byo-langgraph/build.sh     # build、推到 kagent-registry、固定 digest
k apply -f examples/k3d-grafana/manifests/05-byo-langgraph.yaml
chat langgraph-helper "現在台北幾點？幫我算 (1234 * 5.5) / 3，再擲兩顆骰子"
CONTEXT=<印出的 id> chat langgraph-helper "剛剛骰子加起來是多少？"
```

第二題能答對，代表記憶在 actor 暫停 (suspend) 後仍然存在。

### BYO 映像在 Substrate 上必須做到

| 需求 | 原因 | 位置 |
| --- | --- | --- |
| 在 `:80` 提供 A2A over gRPC | gateway 會連到這裡 | `server.py` `grpc_lifespan` |
| `:8081` 提供 `GET /readyz`，且必須在 `:80` 開始監聽之後 | Substrate 一收到回應就做 snapshot | `server.py` `readiness_lifespan` |
| 用 `KAgentTaskStore`（`KAGENT_API_URL`）存 task 狀態 | 歷史、核准、暫停都需要 | `server.py` `build_app` |
| Harness 要有 `workload.command` | 沒有的話 API 會拒絕 BYO Harness | `05-byo-langgraph.yaml` |
| bookworm + uv 管理的 CPython | 其他 Python 在 gVisor restore 會 crash | `Dockerfile` |
| async checkpointer（`InMemorySaver`） | executor 用 `astream`；`SqliteSaver` 只支援同步 | `agent.py` |

Negative
: 1.0.0-alpha5 的粗糙處：`kagent-langgraph` 的 `KAgentApp` 只提供 HTTP JSON-RPC（沒有 gRPC、沒有 `/readyz`），所以 `server.py` 參考 `kagent-adk` 自建 listener；PyPI 上的 `kagent-core==1.0.0a5` 依賴不在 PyPI 的 `kagent-proto`，所以 `build.sh` 從此 checkout 建三個 SDK wheel；`import kagent.core` 需要 `openai` 但沒有宣告。

## 在 Grafana 觀測代理
Duration: 0:15:00

`up.sh` 已開啟 kagent 的 OTel 匯出（`otel.traces/metrics/logs.enabled`），並設定 `otel.capture.messageContent=true`，因此 prompt 與回應都會出現在 span 裡。

Negative
: 本機示範可以這樣做，正式環境請不要開啟 message content 擷取。

打開 Grafana（http://localhost:3000，admin / admin）→ **Explore**：

| 資料源 | 查詢 | 你會看到 |
| --- | --- | --- |
| Tempo | `{ resource.service.name = "hello" }` | 單一代理的 trace：`a2a.request` → `invoke_agent` → 模型 / 工具呼叫 |
| Tempo | `{ span.gen_ai.operation.name = "invoke_agent" }` | 所有代理呼叫。打開 oncall 的 trace 看父 → 子代理樹 |
| Tempo | `{ resource.service.name = "kagent-controller" }` | controller 端：gateway、gRPC |
| Tempo | `{ span.gen_ai.usage.input_tokens > 0 }` | 有 token 用量的推論 span |
| Loki | `{service_name="kagent-controller"}` | controller 的 JSON log，內含 `trace_id`，可直接跳到 Tempo |
| Prometheus | `histogram_quantile(0.95, sum by (le, rpc_method) (rate(rpc_server_call_duration_seconds_bucket[5m])))` | controller gRPC 延遲 |

### 練習

在 Grafana 用 TraceQL metrics 建一個小 dashboard，顯示每個代理的 `invoke_agent` 次數與平均時間：

```
{ span.gen_ai.operation.name = "invoke_agent" } | rate() by (resource.service.name)
```

然後請 `grafana-analyst` 用 `search_dashboards` 找到它。

Positive
: span 屬性的定義見 `docs/architecture/telemetry.md` 與 `telemetry-contract.md`。

## 疑難排解
Duration: 0:05:00

| 症狀 | 檢查 |
| --- | --- |
| Agent 一直 `Ready=False` | `k get agent X -o yaml` 看 conditions；`k logs deploy/kagent-controller`；`k -n ate-system get actortemplates` |
| 對話卡住後 504 | actor 一直沒 ready：`k -n ate-system logs ds/atelet`；`k get workerpools` |
| 工具呼叫失敗 | `k get remotemcpservers` 看 Accepted condition；`k logs deploy/kagent-grafana-mcp` 或 `deploy/kagent-tools` |
| 模型呼叫失敗（Ollama） | 在主機執行 `curl localhost:11434/api/tags`，確認模型已 pull |
| Grafana MCP 401 | token 產生在已重啟的 LGTM pod（emptyDir）。重跑 `up.sh` |

## 清理與下一步
Duration: 0:02:00

```console
./examples/k3d-grafana/down.sh
```

這會刪除整個叢集。

### 你完成了

- 用 Harness + AgentTemplate + Agent 描述代理
- 用 MCP 工具與 `requireApproval` 控制風險
- 用子代理組合專家
- 帶入自己的 LangGraph 代理
- 在 Grafana 追蹤整條呼叫鏈

### 延伸閱讀

- [kagent 架構文件](https://github.com/kagent-dev/kagent/tree/main/docs/architecture)
- [Grafana LGTM](https://github.com/grafana/docker-otel-lgtm)
