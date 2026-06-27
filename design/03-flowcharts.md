# 03 — 流程图 (Mermaid Flowcharts)

## 目录

1. [系统全景图](#1-系统全景图)
2. [阶段一：逆向分析](#2-阶段一逆向分析)
3. [阶段二：需求摄入](#3-阶段二需求摄入)
4. [阶段三：代码生成](#4-阶段三代码生成)
5. [阶段四：验证交付](#5-阶段四验证交付)
6. [阶段五：闭环回写](#6-阶段五闭环回写)
7. [需求状态机](#7-需求状态机)
8. [中断恢复流程](#8-中断恢复流程)
9. [完整序列图](#9-完整序列图)
10. [门禁决策树](#10-门禁决策树)

---

## 1. 系统全景图

```mermaid
graph TB
    subgraph External["外部"]
        EXT_IN[外部需求输入<br/>CLI / API / IDE / Webhook]
        HUMAN[人工决策点<br/>确认 / 跳过 / 放弃]
        EXT_OUT[输出<br/>Git Push / 通知]
    end

    subgraph Stage1["Stage 1: 逆向分析"]
        S1_1[代码扫描] --> S1_2[模块深度分析]
        S1_2 --> S1_3[KB 汇总整合]
        S1_3 --> S1_4[需求反向推断]
        S1_4 --> GATE1{门禁 G1<br/>全部通过?}
    end

    subgraph KB["知识库 Git"]
        KB_STORE[("knowledge-base/<br/>架构/模块/API/数据模型")]
        REQ_BASE[("requirements/baseline/")]
    end

    subgraph Stage2["Stage 2: 需求摄入"]
        EXT_IN --> S2_1[输入解析]
        S2_1 --> S2_2[KB 上下文加载]
        S2_2 --> S2_3[影响分析]
        S2_3 --> S2_4[需求文档生成]
        S2_4 --> GATE2{门禁 G2<br/>全部通过?}
    end

    subgraph REQ["需求 Git"]
        REQ_PROG[("requirements/in-progress/")]
        REQ_DONE[("requirements/completed/")]
    end

    subgraph Stage3["Stage 3: 代码生成"]
        S3_1[自动拉取需求] --> S3_2[优先级排序]
        S3_2 --> S3_3[Comet Design]
        S3_3 --> HUMAN1{🔴 人工确认<br/>设计方案}
        HUMAN1 -->|确认| S3_4[Comet Plan]
        S3_4 --> HUMAN2{🔴 人工确认<br/>实现计划}
        HUMAN2 -->|确认| S3_5[Comet Build<br/>TDD + 双审查]
        S3_5 --> GATE3{门禁 G3<br/>全部通过?}
    end

    subgraph Stage4["Stage 4: 验证交付"]
        S4_1[自动化验证] --> S4_2[代码审查]
        S4_2 --> S4_3{问题?}
        S4_3 -->|有 Critical| S4_4[systematic-debugging]
        S4_4 --> S4_1
        S4_3 -->|无| GATE4{门禁 G4<br/>全部通过?}
        GATE4 -->|通过| S4_5[Git Commit 代码]
        S4_5 --> S4_6[更新需求状态→completed]
        S4_6 --> S4_7[移动需求→completed/]
    end

    subgraph Stage5["Stage 5: 闭环"]
        S4_7 --> S5_1[扫描未完成需求]
        S5_1 --> S5_2{有待处理需求?}
        S5_2 -->|有| S5_3[取最高优先级]
        S5_3 --> S3_1
        S5_2 -->|无| S5_4[监听新需求<br/>Cron / Webhook]
        S5_4 -->|新需求到达| EXT_IN
    end

    GATE1 -->|通过| KB_STORE
    GATE1 -->|通过| REQ_BASE
    GATE2 -->|通过| REQ_PROG
    KB_STORE -.->|加载上下文| S2_2
    KB_STORE -.->|加载上下文| S3_3
    REQ_PROG -.->|拉取| S3_1

    style HUMAN1 fill:#ff6b6b,stroke:#c92a2a,color:#fff
    style HUMAN2 fill:#ff6b6b,stroke:#c92a2a,color:#fff
    style GATE1 fill:#ffd43b,stroke:#fab005
    style GATE2 fill:#ffd43b,stroke:#fab005
    style GATE3 fill:#ffd43b,stroke:#fab005
    style GATE4 fill:#ffd43b,stroke:#fab005
```

---

## 2. 阶段一：逆向分析

```mermaid
flowchart TD
    START([开始: 阶段一触发]) --> CHECK{知识库是否<br/>已存在?}
    
    CHECK -->|首次运行| FULL[全量逆向]
    CHECK -->|已存在| DELTA[增量逆向<br/>只分析变更部分]
    
    FULL --> SCAN[1.1 代码扫描<br/>codegraph_explore]
    DELTA --> DIFF[Git diff 获取变更文件]
    DIFF --> SCAN
    
    SCAN --> MANIFEST[生成模块清单<br/>knowledge-base/.manifest.yaml]
    MANIFEST --> G1_1{门禁 G1.1<br/>模块覆盖率 >= 80%?}
    G1_1 -->|否| ADD_MOD[补充手动指定模块]
    ADD_MOD --> SCAN
    G1_1 -->|是| PARALLEL[1.2 并行模块分析]
    
    PARALLEL --> M1[comet-knowledge<br/>模块 A]
    PARALLEL --> M2[comet-knowledge<br/>模块 B]
    PARALLEL --> M3[comet-knowledge<br/>模块 N...]
    
    M1 --> COLLECT[收集 KB 条目]
    M2 --> COLLECT
    M3 --> COLLECT
    
    COLLECT --> G1_2{门禁 G1.2<br/>KB 条目完整?}
    G1_2 -->|否| RETRY[重试缺失模块<br/>最多3次]
    RETRY --> PARALLEL
    G1_2 -->|是| INTEGRATE[1.3 KB 汇总整合]
    
    INTEGRATE --> ARCH[生成 architecture/]
    INTEGRATE --> APIS[生成 apis/]
    INTEGRATE --> MODELS[生成 data-models/]
    
    ARCH --> G1_3{门禁 G1.3<br/>内部引用一致?}
    APIS --> G1_3
    MODELS --> G1_3
    
    G1_3 -->|否| FIX_REF[修复引用]
    FIX_REF --> INTEGRATE
    G1_3 -->|是| INFER[1.4 需求反向推断<br/>brainstorming]
    
    INFER --> REQ_DOCS[生成基线需求文档<br/>requirements/baseline/]
    REQ_DOCS --> G1_4{门禁 G1.4<br/>需求覆盖率 >= 90%?}
    G1_4 -->|否| LOW_CONF[标记低置信度需求]
    LOW_CONF --> G1_5
    G1_4 -->|是| G1_5{门禁 G1.5<br/>阶段一总门禁}
    
    G1_5 -->|通过| COMMIT1[Git Commit<br/>[devloop] stage:1]
    COMMIT1 --> DONE1([阶段一完成])
    G1_5 -->|失败| REPORT1[生成失败报告]
    REPORT1 --> PAUSE1([暂停: 等待人工处理])
    
    style G1_1 fill:#ffd43b,stroke:#fab005
    style G1_2 fill:#ffd43b,stroke:#fab005
    style G1_3 fill:#ffd43b,stroke:#fab005
    style G1_4 fill:#ffd43b,stroke:#fab005
    style G1_5 fill:#ff6b6b,stroke:#c92a2a,color:#fff
```

---

## 3. 阶段二：需求摄入

```mermaid
flowchart TD
    START([开始: 需求输入到达]) --> PARSE[2.1 输入解析]
    
    PARSE --> FORMAT{输入格式?}
    FORMAT -->|自然语言| NL[解析自然语言]
    FORMAT -->|用户故事模板| US[解析用户故事]
    FORMAT -->|Bug 报告| BUG[解析 Bug 报告]
    FORMAT -->|技术规格| SPEC[解析技术规格]
    
    NL --> G2_1
    US --> G2_1
    BUG --> G2_1
    SPEC --> G2_1
    
    G2_1{门禁 G2.1<br/>输入有效性}
    G2_1 -->|无效| REJECT[拒绝: 返回错误信息]
    REJECT --> START
    G2_1 -->|有效| LOAD_KB[2.2 KB 上下文加载]
    
    LOAD_KB --> MATCH[关键词匹配<br/>需求描述 → KB 模块]
    MATCH --> LOAD[加载匹配模块的 KB 条目<br/>+ 全局 architecture/apis/]
    
    LOAD --> G2_2{门禁 G2.2<br/>至少匹配 1 个<br/>相关模块?}
    G2_2 -->|否| MARK_LOW[标记 confidence: low<br/>可能为新模块]
    MARK_LOW --> IMPACT
    G2_2 -->|是| IMPACT[2.3 影响分析<br/>brainstorming]
    
    IMPACT --> IA_DOC[生成影响分析报告<br/>涉及模块/数据模型变更/API变更/风险]
    IA_DOC --> G2_3{门禁 G2.3<br/>影响分析完整?}
    G2_3 -->|否| COMPLETE_IA[补充分析]
    COMPLETE_IA --> IMPACT
    G2_3 -->|是| GEN_REQ[2.4 需求文档生成]
    
    GEN_REQ --> OPENSEC[comet-open<br/>创建 OpenSpec Change]
    OPENSEC --> REQ_DOC[生成统一需求文档<br/>requirements/in-progress/]
    
    REQ_DOC --> G2_4{门禁 G2.4<br/>需求文档结构正确?}
    G2_4 -->|否| FIX_REQ[修复文档]
    FIX_REQ --> GEN_REQ
    G2_4 -->|是| G2_5{门禁 G2.5<br/>阶段二总门禁}
    
    G2_5 -->|通过| HUMAN_CONFIRM{🔴 人工确认<br/>需求内容}
    HUMAN_CONFIRM -->|确认| COMMIT2[Git Commit<br/>[devloop] stage:2<br/>status: proposed]
    HUMAN_CONFIRM -->|修改| FEEDBACK[收集修改意见]
    FEEDBACK --> IMPACT
    COMMIT2 --> DONE2([阶段二完成<br/>→ 进入阶段三])
    
    G2_5 -->|失败| REPORT2[生成失败报告]
    REPORT2 --> PAUSE2([暂停: 等待人工处理])
    
    style G2_1 fill:#ffd43b,stroke:#fab005
    style G2_2 fill:#ffd43b,stroke:#fab005
    style G2_3 fill:#ffd43b,stroke:#fab005
    style G2_4 fill:#ffd43b,stroke:#fab005
    style G2_5 fill:#ff6b6b,stroke:#c92a2a,color:#fff
    style HUMAN_CONFIRM fill:#ff6b6b,stroke:#c92a2a,color:#fff
```

---

## 4. 阶段三：代码生成

```mermaid
flowchart TD
    START([开始: 阶段三触发]) --> PULL[3.1 Git Pull<br/>同步远程]
    PULL --> SCAN[扫描 requirements/in-progress/<br/>status != completed, blocked]
    
    SCAN --> G3_1{门禁 G3.1<br/>需求可构建?}
    G3_1 -->|否| BLOCK[标记为 blocked<br/>记录原因]
    BLOCK --> SKIP[跳过, 取下一个需求]
    SKIP --> SCAN
    G3_1 -->|是| SORT[3.2 优先级排序]
    
    SORT --> SELECT[选取最高分需求]
    SELECT --> LOAD_CTX[加载需求文档 + KB 上下文]
    
    LOAD_CTX --> DESIGN[3.3 Comet Design]
    DESIGN --> DESIGN_STEPS[comet-handoff → brainstorming → design.md]
    DESIGN_STEPS --> DESIGN_GUARD[comet-guard design --apply]
    
    DESIGN_GUARD --> G3_2{门禁 G3.2<br/>ALL CHECKS PASSED?}
    G3_2 -->|否| FIX_DESIGN[修复设计]
    FIX_DESIGN --> DESIGN
    G3_2 -->|是| HUMAN3{🔴 人工确认<br/>设计方案}
    
    HUMAN3 -->|修改| DESIGN
    HUMAN3 -->|确认| PLAN[3.4 Comet Plan<br/>writing-plans]
    
    PLAN --> PLAN_DOC[生成 plan.md + tasks.md]
    PLAN_DOC --> G3_3{门禁 G3.3<br/>Plan 质量}
    G3_3 -->|否| FIX_PLAN[修复计划]
    FIX_PLAN --> PLAN
    G3_3 -->|是| HUMAN4{🔴 人工确认<br/>实现计划}
    
    HUMAN4 -->|暂停| SAVE_STATE[保存状态<br/>等待恢复]
    HUMAN4 -->|确认| CHOOSE_MODE{选择 Build 模式}
    
    CHOOSE_MODE -->|小需求<br/>< 5 tasks| MODE_A[executing-plans<br/>主会话执行]
    CHOOSE_MODE -->|中大型需求| MODE_B[subagent-driven-development<br/>后台子代理]
    
    MODE_A --> BUILD_LOOP
    MODE_B --> BUILD_LOOP
    
    subgraph BUILD_LOOP["3.5 Build Loop (每个 Task)"]
        TASK_START[取下一个未完成 task] --> IMPL[实现 Task]
        IMPL --> TDD[TDD 验证<br/>test-driven-development]
        TDD --> TDD_PASS{测试通过?}
        TDD_PASS -->|否| FIX_IMPL[修复实现<br/>最多3轮]
        FIX_IMPL --> IMPL
        TDD_PASS -->|是| REVIEW1[Spec Compliance Review]
        REVIEW1 --> REVIEW2[Code Quality Review]
        REVIEW2 --> BOTH_PASS{双审查通过?}
        BOTH_PASS -->|否| FIX_REVIEW[修复审查问题]
        FIX_REVIEW --> IMPL
        BOTH_PASS -->|是| TASK_COMMIT[Git Commit task]
        TASK_COMMIT --> TASK_DONE{还有 task?}
        TASK_DONE -->|是| TASK_START
        TASK_DONE -->|否| BUILD_DONE[所有 task 完成]
    end
    
    BUILD_DONE --> BUILD_GUARD[comet-guard build --apply]
    BUILD_GUARD --> G3_4{门禁 G3.4<br/>ALL CHECKS PASSED?}
    G3_4 -->|否| FIX_BUILD[修复]
    FIX_BUILD --> BUILD_LOOP
    G3_4 -->|是| DONE3([阶段三完成<br/>→ 进入阶段四])
    
    style G3_1 fill:#ffd43b,stroke:#fab005
    style G3_2 fill:#ffd43b,stroke:#fab005
    style G3_3 fill:#ffd43b,stroke:#fab005
    style G3_4 fill:#ff6b6b,stroke:#c92a2a,color:#fff
    style HUMAN3 fill:#ff6b6b,stroke:#c92a2a,color:#fff
    style HUMAN4 fill:#ff6b6b,stroke:#c92a2a,color:#fff
```

---

## 5. 阶段四：验证交付

```mermaid
flowchart TD
    START([开始: 阶段四触发]) --> VERIFY[4.1 自动化验证<br/>comet-verify]
    
    VERIFY --> SCALE[comet-state scale<br/>确定验证级别]
    SCALE --> RUN_TESTS[运行测试套件<br/>+ 构建 + Lint + 类型检查]
    
    RUN_TESTS --> G4_1{门禁 G4.1<br/>全部通过?}
    G4_1 -->|失败| COUNT{失败次数?}
    COUNT -->|< 3| DEBUG[4.3 systematic-debugging<br/>定位根因]
    DEBUG --> FIX[生成修复]
    FIX --> RUN_TESTS
    COUNT -->|>= 3| HUMAN_FAIL{🔴 连续3次失败<br/>人工决策}
    HUMAN_FAIL -->|继续修复| DEBUG
    HUMAN_FAIL -->|接受偏差| ACCEPT[记录偏差<br/>标记 known-issue]
    HUMAN_FAIL -->|放弃| ABORT[标记需求 blocked<br/>回滚变更]
    
    G4_1 -->|通过| REVIEW[4.2 代码审查<br/>code-review]
    ACCEPT --> REVIEW
    
    REVIEW --> G4_2{门禁 G4.2<br/>无 Critical 问题?}
    G4_2 -->|有 Critical| FIX_CRIT[修复 Critical 问题]
    FIX_CRIT --> RUN_TESTS
    G4_2 -->|通过| FINAL_CHECK[4.4 最终门禁]
    
    FINAL_CHECK --> G4_4{门禁 G4.4<br/>阶段四总门禁}
    G4_4 -->|失败| REPORT4[生成失败报告]
    REPORT4 --> PAUSE4([暂停: 等待人工处理])
    
    G4_4 -->|通过| COMMIT_CODE[4.5 Git Commit 代码<br/>[devloop] stage:4]
    COMMIT_CODE --> UPDATE_STATUS[4.6 更新需求状态<br/>status: verifying → completed]
    UPDATE_STATUS --> MOVE_REQ[移动需求文档<br/>in-progress/ → completed/]
    MOVE_REQ --> COMMIT_REQ[Git Commit 需求状态<br/>[devloop] stage:5]
    COMMIT_REQ --> DONE4([阶段四完成<br/>→ 进入阶段五])
    
    style G4_1 fill:#ffd43b,stroke:#fab005
    style G4_2 fill:#ffd43b,stroke:#fab005
    style G4_4 fill:#ff6b6b,stroke:#c92a2a,color:#fff
    style HUMAN_FAIL fill:#ff6b6b,stroke:#c92a2a,color:#fff
```

---

## 6. 阶段五：闭环回写

```mermaid
flowchart TD
    START([阶段四完成]) --> SCAN[5.1 扫描待处理需求<br/>requirements/in-progress/]
    
    SCAN --> FILTER[过滤 status != completed<br/>且 status != blocked]
    FILTER --> G5_1{门禁 G5.1<br/>状态一致性}
    
    G5_1 -->|不一致| FIX_STATE[修复状态不一致<br/>同步 Git 文件与 frontmatter]
    FIX_STATE --> SCAN
    
    G5_1 -->|一致| SORT[5.2 优先级排序]
    SORT --> QUEUE{队列非空?}
    
    QUEUE -->|是| PICK[选取最高优先级需求]
    PICK --> CHECK_STATUS{需求当前状态?}
    
    CHECK_STATUS -->|proposed| TO_STAGE2[→ 触发阶段二<br/>从需求完善开始]
    CHECK_STATUS -->|designed| TO_STAGE3_D[→ 触发阶段三<br/>从 Plan 开始]
    CHECK_STATUS -->|planned| TO_STAGE3_P[→ 触发阶段三<br/>从 Build 开始]
    CHECK_STATUS -->|其他| HUMAN_ANOMALY{🔴 异常状态<br/>人工处理}
    
    TO_STAGE2 --> STAGE2_RUN[执行阶段二]
    TO_STAGE3_D --> STAGE3_RUN[执行阶段三]
    TO_STAGE3_P --> STAGE3_RUN
    
    STAGE2_RUN --> STAGE3_RUN
    STAGE3_RUN --> DONE([需求完成])
    DONE --> SCAN
    
    QUEUE -->|空| WAIT[5.3 监听模式]
    WAIT --> CRON[Cron: 每5分钟扫描]
    WAIT --> WEBHOOK[Webhook: push 事件]
    WAIT --> MANUAL[手动: /devloop start]
    
    CRON --> NEW{新需求?}
    WEBHOOK --> NEW
    MANUAL --> NEW
    
    NEW -->|是| QUEUE
    NEW -->|否| WAIT
    
    style G5_1 fill:#ffd43b,stroke:#fab005
    style HUMAN_ANOMALY fill:#ff6b6b,stroke:#c92a2a,color:#fff
```

---

## 7. 需求状态机

```mermaid
stateDiagram-v2
    [*] --> draft: 外部输入
    
    draft --> proposed: 阶段二完成<br/>G2.5 通过
    draft --> rejected: G2.1 失败(不可恢复)
    
    proposed --> designed: 阶段三 Design 完成<br/>G3.2 通过
    proposed --> blocked: G3.1 失败<br/>(KB上下文丢失等)
    
    designed --> planned: 阶段三 Plan 完成<br/>G3.3 通过
    
    planned --> building: 阶段三 Build 开始
    building --> built: 所有 Task 完成<br/>G3.4 通过
    
    built --> verifying: 阶段四开始
    verifying --> completed: G4.4 通过<br/>Git Commit + 状态更新
    verifying --> fixing: G4.1/4.2 失败<br/>进入修复循环
    
    fixing --> verifying: 修复完成
    fixing --> blocked: 连续3次失败<br/>人工决策放弃
    
    blocked --> proposed: 人工解除阻塞
    blocked --> [*]: 需求废弃
    
    completed --> [*]: 归档
    
    note right of draft: 阶段二
    note right of designed: 阶段三
    note right of verifying: 阶段四
    note right of completed: 阶段五
```

---

## 8. 中断恢复流程

```mermaid
flowchart TD
    INTERRUPT([会话中断/恢复]) --> READ_STATE[读取 .comet/loop-state.yaml]
    
    READ_STATE --> HAS_STATE{存在活跃<br/>流程状态?}
    
    HAS_STATE -->|否| FRESH([正常启动<br/>等待指令])
    
    HAS_STATE -->|是| DISPLAY[显示中断摘要<br/>当前阶段/步骤/需求]
    DISPLAY --> RECOVER[comet-state check<br/>--recover]
    
    RECOVER --> RECOVER_ACTION{Recovery Action?}
    
    RECOVER_ACTION -->|resume_from_step| RESUME[从 checkpoint 继续]
    RECOVER_ACTION -->|replay_missing| REPLAY[重放缺失产物]
    RECOVER_ACTION -->|restart_stage| RESTART[重新开始当前阶段]
    
    RESUME --> CHECK_PENDING{有待处理<br/>确认?}
    CHECK_PENDING -->|是| HANDLE_CONFIRM[优先处理确认<br/>恢复对话上下文]
    HANDLE_CONFIRM --> CONTINUE
    CHECK_PENDING -->|否| CONTINUE[继续执行]
    
    REPLAY --> REBUILD[重建上下文<br/>读取 proposal + KB + memory]
    REBUILD --> CONTINUE
    
    RESTART --> CLEANUP[清理当前阶段产物]
    CLEANUP --> CONTINUE
    
    CONTINUE --> SAVE[更新 loop-state.yaml<br/>保存新 checkpoint]
    
    style INTERRUPT fill:#748ffc,stroke:#4263eb,color:#fff
    style RECOVER fill:#748ffc,stroke:#4263eb,color:#fff
```

### 上下文压缩恢复序列

```mermaid
sequenceDiagram
    actor User
    participant Session as Claude Session
    participant State as loop-state.yaml
    participant Git as Git Repo
    participant KB as Knowledge Base
    participant Memory as Memory Files
    
    Note over Session: 检测到上下文压缩
    
    Session->>State: 读取 loop-state.yaml
    State-->>Session: current_stage=3, current_step=3.4
    
    Session->>Session: comet-state check --recover
    Note over Session: Recovery Action: resume_from_step
    
    Session->>Git: 读取 openspec/changes/<name>/proposal.md
    Git-->>Session: 知道正在做什么需求
    
    Session->>KB: 读取 knowledge-base/modules/<affected>/
    KB-->>Session: 获取代码上下文
    
    Session->>Memory: 读取最近 3 个 memory 文件
    Memory-->>Session: 获取决策历史
    
    Session->>User: "恢复: REQ-010 处于 Plan 阶段<br/>plan.md 已生成，等待确认。<br/>是否继续?"
    User->>Session: 确认继续
    
    Session->>State: 更新 checkpoint，继续执行
```

---

## 9. 完整序列图

```mermaid
sequenceDiagram
    actor User
    participant CLI as DevLoop CLI
    participant S1 as Stage1: Reverse
    participant Git as Git Repository
    participant KB as Knowledge Base
    participant S2 as Stage2: Intake
    participant S3 as Stage3: Build
    participant S4 as Stage4: Verify
    participant S5 as Stage5: Loop
    
    Note over User,S5: === 初始化: 阶段一 ===
    
    User->>CLI: /devloop reverse
    CLI->>S1: 启动逆向分析
    
    S1->>Git: 扫描代码仓库
    Git-->>S1: 模块清单
    
    loop 每个模块 (并行)
        S1->>S1: comet-knowledge 分析
        S1->>KB: 写入 KB 条目
    end
    
    S1->>S1: 汇总整合 + 需求推断
    S1->>S1: 门禁 G1.1-G1.5
    
    S1->>Git: Commit KB + 基线需求
    Git-->>S1: commit: abc111
    
    S1-->>CLI: 阶段一完成
    
    Note over User,S5: === 新需求到达: 阶段二 ===
    
    User->>CLI: "添加 OAuth2 登录"
    CLI->>S2: 启动需求摄入
    
    S2->>S2: 解析输入 → G2.1 通过
    S2->>KB: 加载相关模块 KB
    KB-->>S2: auth-service, api-gateway KB
    
    S2->>S2: brainstorming 影响分析
    S2->>S2: 生成需求文档 → G2.2-G2.5
    
    S2-->>User: 🔴 需求文档待确认
    User->>S2: 确认
    
    S2->>Git: Commit 需求文档 (status: proposed)
    Git-->>S2: commit: def222
    
    S2-->>CLI: 阶段二完成
    
    Note over User,S5: === 自动转入阶段三 ===
    
    CLI->>S3: 启动代码生成
    S3->>Git: Pull + 扫描待处理需求
    Git-->>S3: REQ-010 (priority: high)
    
    S3->>KB: 加载 KB 上下文
    KB-->>S3: 完整模块知识
    
    S3->>S3: Comet Design → G3.2
    S3-->>User: 🔴 设计方案待确认
    User->>S3: 确认
    
    S3->>S3: Comet Plan → G3.3
    S3-->>User: 🔴 实现计划待确认
    User->>S3: 确认
    
    S3->>S3: Comet Build (subagent-driven)
    
    loop 每个 Task (后台 agent)
        S3->>S3: 实现 + TDD + 双审查
        S3->>Git: Commit task (commit: ghi333)
    end
    
    S3->>S3: comet-guard build --apply → G3.4
    S3-->>CLI: 阶段三完成
    
    Note over User,S5: === 自动转入阶段四 ===
    
    CLI->>S4: 启动验证交付
    S4->>S4: 自动化验证 → G4.1
    
    alt 测试失败
        S4->>S4: systematic-debugging
        S4->>S4: 修复 → 重新验证
    else 测试通过
        S4->>S4: 代码审查 → G4.2
    end
    
    S4->>S4: 总门禁 G4.4
    S4->>Git: Commit 代码变更
    S4->>Git: 更新需求状态 → completed
    Git-->>S4: commit: jkl444, mno555
    
    S4-->>CLI: 阶段四完成
    
    Note over User,S5: === 闭环: 阶段五 ===
    
    CLI->>S5: 扫描下一个需求
    S5->>Git: 查找 in-progress/
    Git-->>S5: 队列: [REQ-011, REQ-012]
    
    S5->>S3: 取 REQ-011 → 阶段三
    Note over S3: 循环继续...
```

---

## 10. 门禁决策树

```mermaid
flowchart TD
    GATE_START([门禁检查开始]) --> CHECK{检查类型?}
    
    CHECK -->|统计类| STAT[计算指标<br/>如: 覆盖率, 完整率]
    STAT --> STAT_PASS{指标 >= 阈值?}
    STAT_PASS -->|是| STAT_OK([✅ 通过])
    STAT_PASS -->|否| STAT_FAIL
    
    STAT_FAIL --> STAT_ACTION{on_fail 策略?}
    STAT_ACTION -->|warn_and_continue| STAT_WARN[📋 记录警告<br/>不阻断流程]
    STAT_ACTION -->|block_and_retry| STAT_RETRY[🔄 自动重试<br/>最多3次]
    
    STAT_WARN --> STAT_OK
    STAT_RETRY -->|重试成功| STAT_OK
    STAT_RETRY -->|3次失败| STAT_BLOCK[🚫 阻断流程<br/>等待人工处理]
    
    CHECK -->|结构类| STRUCT[验证结构完整性<br/>如: 必填字段, 引用有效]
    STRUCT --> STRUCT_PASS{结构正确?}
    STRUCT_PASS -->|是| STRUCT_OK([✅ 通过])
    STRUCT_PASS -->|否| STRUCT_FAIL
    
    STRUCT_FAIL --> STRUCT_ACTION{on_fail 策略?}
    STRUCT_ACTION -->|block_and_retry| STRUCT_FIX[🔧 自动修复<br/>补全字段, 修复引用]
    STRUCT_ACTION -->|pause_for_human| STRUCT_PAUSE[⏸️ 暂停<br/>等待人工决策]
    
    STRUCT_FIX -->|修复成功| STRUCT_OK
    STRUCT_FIX -->|无法自动修复| STRUCT_PAUSE
    
    STRUCT_PAUSE --> HUMAN_CHOICE{人工决策}
    HUMAN_CHOICE -->|跳过| STRUCT_OVERRIDE([⚠️ 强制通过<br/>记录覆盖])
    HUMAN_CHOICE -->|放弃| STRUCT_ABORT([❌ 流程终止])
    
    CHECK -->|汇总类| AGG[汇总所有子门禁结果]
    AGG --> AGG_PASS{全部通过?}
    AGG_PASS -->|是| AGG_OK([✅ 阶段门禁通过])
    AGG_PASS -->|否| AGG_FAIL[列出所有失败项]
    AGG_FAIL --> AGG_REPORT[生成门禁报告]
    AGG_REPORT --> AGG_BLOCK([🚫 阶段阻断<br/>必须全部修复])
    
    CHECK -->|前置类| PRE[检查前置条件<br/>如: 需求状态, KB 可用性]
    PRE --> PRE_PASS{前置满足?}
    PRE_PASS -->|是| PRE_OK([✅ 可以继续])
    PRE_PASS -->|否| PRE_SKIP[⏭️ 跳过当前项<br/>取下一个]
    
    style STAT_OK fill:#51cf66,stroke:#2f9e44
    style STRUCT_OK fill:#51cf66,stroke:#2f9e44
    style AGG_OK fill:#51cf66,stroke:#2f9e44
    style PRE_OK fill:#51cf66,stroke:#2f9e44
    style STAT_BLOCK fill:#ff6b6b,stroke:#c92a2a,color:#fff
    style AGG_BLOCK fill:#ff6b6b,stroke:#c92a2a,color:#fff
    style STRUCT_ABORT fill:#ff6b6b,stroke:#c92a2a,color:#fff
```
