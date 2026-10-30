# 傳統 RAG 與智能代理 RAG（Agentic RAG）技術比較與系統研析報告
## （Traditional RAG vs. Agentic RAG: A Comparative Study of Retrieval-Augmented Systems）

---

### 【摘要 (Abstract)】
檢索增強生成（Retrieval-Augmented Generation, RAG）技術自 Lewis 等人（2020）提出以來，透過將大型語言模型（LLMs）與外部知識檢索系統結合，有效緩解了模型的幻覺（Hallucination）問題並提升了生成事實性。然而，傳統 RAG 採用靜態、單次檢索的線性工作流程（Query → Retrieve → Generate），在面對複雜多跳推理（Multi-hop Reasoning）、動態資訊更新及多工具協同調用時面臨嚴峻瓶頸。近期興起的**智能代理式 RAG（Agentic RAG）**將檢索流程置入具備自主規劃、反思、工具調用與錯誤修正的封閉控制迴路中，並透過多代理協同架構（Multi-Agent Collaboration），大幅拓展了檢索增強系統的應用邊界。本報告系統化梳理傳統 RAG 與 Agentic RAG 的架構演進、核心角色分工、技術特徵對比、評估指標體系、失效模式與安全倫理治理，並提供各垂直領域之實際應用落地分析。

---

## 1. 檢索增強生成（RAG）技術演進背景與核心痛點

### 1.1 傳統 RAG（Traditional RAG）之線性架構與瓶頸
傳統 RAG 系統採用標準的一致性管線（Standard Linear Pipeline）：
1. **查詢編碼（Query Encoding）**：將使用者查詢 $q$ 透過 Transformer 稠密編碼器轉換為 Embedding 向量。
2. **文件檢索（Document Retrieval）**：藉由向量資料庫執行近似最近鄰搜尋（ANN / Top-$k$ Cosine Similarity），檢索相關文件段落。
3. **上下文融合（Context Fusion）**：將檢索所得之段落串接至 Prompt 上下文中。
4. **回應生成（Response Generation）**：輸入至語言模型生成最終回應。

#### 傳統 RAG 的固有缺陷：
- **檢索噪音與敏感性高**：對使用者初始查詢之文字品質高度依賴；若關鍵字模糊或存在歧義，檢索出的無關段落將直接干擾模型判斷。
- **缺乏多跳推理能力（No Multi-hop Reasoning）**：無法將「2024年諾貝爾物理學獎得主的研究機構座落於哪一個國家的首都？」此類需多步連鎖查詢的問題進行拆解。
- **無狀態與單向反應式（Stateless & Reactive）**：生成過程無自我反思或驗證機制，一旦檢索到過時或錯誤資料，必然產生錯誤答案（Garbage In, Garbage Out）。

---

## 2. 智能代理 RAG（Agentic RAG）架構與核心能力

Agentic RAG 核心概念是**將「靜態資料管道」轉變為「具備認知迴路的自主推理代理系統」**。系統不再單純反應，而是具備目標導向之規劃與適應能力。

### 2.1 智能代理 RAG 之六大認知能力
1. **感知（Perception）**：深度理解使用者意圖、任務限制與上下文訊號。
2. **推理（Reasoning）**：將複雜複合式問題拆解為有向無環圖（DAG）或多步子任務。
3. **行動（Action）**：動態調用向量檢索器、REST API、即時網頁搜尋、SQL 資料庫或計算器。
4. **反思（Reflection）**：對中間步驟之結果進行事實性與相關性評估，判斷證據是否充足。
5. **適應（Adaptation）**：若檢索失敗或證據矛盾，主動進行「查詢重構（Query Reformulation）」與策略切換。
6. **協作（Collaboration）**：多個特化代理角色互相傳遞上下文、驗證彼此產出。

### 2.2 典型智能代理角色分工矩陣（Role Taxonomy）

| 代理角色 | 英文術語 | 核心職責與工作機制 |
| :--- | :--- | :--- |
| **規劃者** | Planner Agent | 分析全局目標，利用思維鏈（CoT）或分支限界法分解出可執行的階段子目標。 |
| **檢索者** | Retriever Agent | 根據規劃動態執行混合檢索（Dense + Sparse BM25），支援多輪與多跳查詢追蹤。 |
| **工具代理** | Tool Agent | 透過結構化 Function Calling 調用外部計算器、API、知識圖譜本體查詢。 |
| **驗證者** | Verifier / Critic | 檢驗生成內容與檢索證據的一致性（Entailment），防範引文幻覺與邏輯漏洞。 |
| **記憶代理** | Memory Agent | 維護短期工作記憶（Working Memory）與長期情節記憶（Episodic Memory）。 |
| **問答整合者** | QA / Synthesis Agent | 匯總經由驗證之多源結構化與非結構化證據，產出嚴謹且附帶完整引文的回答。 |

---

## 3. 傳統 RAG 與智能代理 RAG 核心維度全方位對比

| 比較維度 | 傳統 RAG (Traditional RAG) | 智能代理 RAG (Agentic RAG) |
| :--- | :--- | :--- |
| **工作流程 (Workflow)** | 靜態線性：查詢 → 檢索 → 生成 | 動態閉環：規劃 → 檢索 → 反思 → 行動 → 迭代 |
| **推理深度 (Reasoning)** | 單步、反應式（Single-step Reactive） | 多跳、反思式、多步鏈式推理（Multi-hop Iterative） |
| **規劃機制 (Planning)** | 無（依賴單次 Prompt） | 具備獨立規劃者代理（Planner Agent / DAG 任務樹） |
| **外部工具 (Tool Use)** | 僅能連接單一向量資料庫檢索器 | 支援多工具鏈（APIs, Python 執行環境, SQL, 知識圖譜） |
| **多代理協作 (Multi-Agent)** | 不支援 | 支援分散式專業代理團隊分工與審查 |
| **自適應能力 (Adaptability)**| 極低（無法動態校正初始查詢） | 極高（支援自適應查詢重構與子查詢擴展） |
| **記憶架構 (Memory)** | 無狀態（Stateless） | 支援工作記憶、情節記憶與實體知識沉澱 |
| **資料來源整合** | 單一靜態文本語料庫 | 多模態、結構化/非結構化及即時串流數據 |
| **錯誤自癒 (Self-healing)** | 無（檢索失敗即導致錯誤輸出） | 支援多輪檢錯、重試與候選路徑回滾 |
| **系統延遲 (Latency)** | 低（通常 1~2 秒內） | 較高（因多步迭代與多次 LLM 調用，約 3~10 秒） |
| **計算與 Token 成本** | 低 | 較高（需權衡準確性與推理成本） |
| **評估複雜度** | 低至中等（標準 BLEU / ROUGE / EM） | 高（需追蹤軌跡、工具調用精確度與規劃忠實度） |

---

## 4. 代表性智能代理 RAG 系統與架構範式

1. **MA-RAG (Multi-Agent RAG)**：
   - 包含 Planner、Step Definer、Extractor 與 QA 代理，以管線化代理分工解決複雜法律與科學問題。
2. **RAG-Gym**：
   - 採用強化學習（RL）之 Actor-Critic 架構，透過評論者代理反饋訓練模型動態決定「何時需要檢索」與「何時直接推理」。
3. **ReasonRAG**：
   - 引入過程層級獎勵模型（Process-level Rewards），在多步推理過程中逐步評估檢索查詢品質，避免長鏈條累積誤差。
4. **Graph-R1**：
   - 將輕量化知識超圖（Knowledge Hypergraph）融入 RAG 迴路，支援實體關係的多跳關聯發現與圖拓撲導航。

---

## 5. 跨領域實務應用場景對比（表 III 深度整理）

| 應用領域 | 傳統 RAG 落地範例 | 智能代理 RAG 落地範例 | 關鍵效能提升 |
| :--- | :--- | :--- | :--- |
| **智慧醫療** | 靜態臨床指南手冊與衛教問答 | 整合電子病歷（EHR）、多模態醫學影像、動態交互用藥禁忌檢查 API | 避免藥物相互衝突，實現多步驟鑑別診斷 |
| **金融與審計** | 檢索季報或年報之靜態文字段落 | 動態串接 Bloomberg API、SEC EDGAR 即時財報、動態本益比計算與詐欺異常偵測 | 結合數值計算與合規多源 cross-check |
| **智慧製造** | 工廠機台操作說明書檢索 | 連線 SCADA/PLC 即時遙測、預測性維護分析、自動派工與備品供應鏈排程 | 縮短平均修復時間 (MTTR) 60% 以上 |
| **法律與合規** | 法規法條與先例關鍵字全文搜尋 | 多跳法律推理、判決先例論證鏈追蹤、管轄權爭議比對與即時引文檢驗 | 杜絕虛構判例編號（防範引文幻覺） |
| **教育學習** | 題庫標準答案與解析單次輸出 | 學習者弱點記憶模型、自適應難度生成、階梯式提示（Socratic Tutoring） | 打造個人化長期伴學導師 |
| **網路資安** | CVE 漏洞庫與攻擊特徵匹配 | 即時防火牆日誌分析、結合沙箱動態分析 API、自動生成橫向移動防禦策略 | 應對 0-day 攻擊之多步驟因果溯源 |

---

## 6. 系統評估指標體系（Evaluation Metrics）

### 6.1 傳統 RAG 指標（聚焦結果文字比對）
- **BLEU / ROUGE**：評估生成字串與標準答案之 n-gram 精確度與召回率。
- **Exact Match (EM)**：嚴格字串吻合度，適用於簡短事實題。
- **FactScore**：計算答案中可查證事實原子句的正確比例。

### 6.2 智能代理 RAG 指標（聚焦推理軌跡與行為評估）
- **Plan Fidelity（計畫忠實度）**：執行步驟是否精確遵循規劃者所拆解的邏輯架構。
- **Tool Success Rate（工具調用成功率）**：API 參數格式正確率、資料庫查詢語法有效性與回傳值解析度。
- **Reasoning Traceability（推理可追溯性）**：決策軌跡（Thought-Action-Observation）之透明度與可解釋性。
- **Task Completion Rate（端到端任務完成率）**：在限制輪次與資源預算內成功解決複雜問題之比例。

---

## 7. 常見失效模式與緩解對策（Failure Modes & Mitigations）

| 失效模式 | 現象描述 | 核心緩解對策 (Mitigation Strategies) |
| :--- | :--- | :--- |
| **檢索噪音 (Retrieval Noise)** | 引入不相關文字造成上下文污染與誤導 | 混合 BM25+Dense 檢索、重排序（Re-ranking）、新穎性懲罰與動態片段過濾。 |
| **時間漂移 (Temporal Drift)** | 檢索出過期資訊（如舊版法規或已修訂醫療指南） | 時間感知索引（Temporal-aware Indexing）、時間約束過濾、最新版本動態驗證。 |
| **工具鏈故障 (Tool Failures)** | API 調用超時、格式解析錯誤或授權失敗 | 結構化 JSON Schema 約束、多重備援降級方案、自動重試與異常回退機制。 |
| **規劃偏離 (Planning Drift)** | 任務拆解方向錯誤或陷入無限遞迴迴圈 | 計畫驗證機制、分支限界搜索（Branch-and-Bound）、設定最大跳數與邊際效用停止規則。 |
| **引文幻覺 (Citation Hallucination)**| 模型憑空捏造論文 DOI、法條編號或來源網址 | 強制「檢索後約束生成」、專屬引用檢驗代理（Citation Verifier Agent）。 |
| **資料安全與隱私洩漏** | 日誌或 Prompt 包含個人身分識別資訊（PII） | 本地端邊緣檢索、敏感欄位自動動態遮罩（PII Masking）、嚴格權限存取控制（ACL）。 |

---

## 8. 倫理、安全與治理規範

1. **最小權限原則（Principle of Least Privilege）**：各工具代理僅授予最小必要讀寫權限，防止未授權操作。
2. **證據不足拒答機制（Refusal on Insufficient Evidence）**：當檢索品質或置信度低於安全閾值時，主動拒絕回答或提示不確定性。
3. **標準化模型卡（Model Cards）**：詳實記錄代理能力範圍、訓練資料截止期、已知局限性與錯誤偏見。
4. **人類在環（Human-in-the-Loop, HITL）**：在高風險醫療、金融交易與法律判決等場景中，強制將關鍵行動交由人類專家複核授權。

---

## 9. 總結與展望
傳統 RAG 提供了一種低延遲、高吞吐的快速事實查證機制；而 Agentic RAG 則透過自主規劃、動態多工具整合與反思迭代，克服了知識密集型複雜任務的推理瓶頸。未來，兩者將與**知識圖譜（Knowledge Graphs）**、**多模態感知（Multimodal Sensing）**及**物聯網代理（Agentic IoT）**深度整合，推動自主智慧系統邁向全新的「代理人物聯網（Internet of Agents, IoA）」時代。
