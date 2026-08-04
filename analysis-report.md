# SETI 太阳系随机器 —— 深度技术分析报告

> 分析日期：2026-08-04

---

## 目录

- [一、项目概述](#一项目概述)
- [二、技术栈](#二技术栈)
- [三、架构设计](#三架构设计)
- [四、核心实现与机制](#四核心实现与机制)
- [五、数据流总结](#五数据流总结)
- [六、模块清单](#六模块清单)
- [七、技术评价](#七技术评价)

---

## 一、项目概述

SETI 太阳系随机器是一款基于浏览器的策略桌游引擎，支持 1-4 名玩家（含 AI），玩家在太阳系棋盘上通过卡牌、科技、数据和工业公司系统展开竞争。核心玩法包括：

- **卡牌系统**：140+ 张卡牌，含三角标效果（弃牌 / 扫描 / 收入）
- **科技系统**：4 条科技树（蓝 / 黄 / 粉 / 通用），带专属面板和奖励链
- **数据系统**：星云数据放置与收集，触发扫描
- **行动系统**：初探、环绕、登陆、扫描、研究五大主行动 + 快速交易
- **外星人系统**：8 个高度差异化的种族，各具独特面板和机制
- **AI 系统**：五层决策架构，支持 8 种外星人策略
- **终局计分**：多维度计分公式

---

## 二、技术栈

| 层级 | 技术 | 说明 |
|------|------|------|
| **UI 层** | 纯 HTML5 + CSS3 | 单页应用，三栏布局（左：外星人/计分｜中：棋盘/行动｜右：科技/牌区/统计），深色太空主题，CSS 自定义变量驱动全局样式 |
| **脚本语言** | ES5+ 原生 JavaScript | 无 TypeScript，无 Babel，无任何转译 |
| **模块系统** | IIFE（立即执行函数表达式） + 全局命名空间 | 每个 JS 文件以 `(function(root, factory){...})(globalThis\|window, function(root){...})` 包裹，挂载到 `window.Seti*`，同时兼容 CommonJS `module.exports` |
| **依赖管理** | 自定义 DI 容器 | `app/dependencies.js` 维护 44 个模块注册表，通过 `collectDependencies()` 一次性注入到 `app.js` |
| **构建工具** | **无** | 没有 `package.json`、没有 npm、没有 webpack/vite，所有脚本通过 `<script>` 标签按依赖顺序加载 |
| **随机数** | 自研确定性 PRNG | 基于 FNV-1a 哈希 + xorshift/multiply 混合算法，支持种子模式和每日固定种子模式，通过劫持 `Math.random` 实现全局确定性 |
| **图片资源** | WebP + PNG | 519 个静态资源文件，分布在 `assets/` 下按功能模块组织（aliens/cards/core/final/industry/symbol/tech_tile/tokens） |
| **数据配置** | CSV + JSON | 用 CSV 定义卡牌元数据（140 行），JSON 定义行星坐标，配置驱动生成 |

### 加载机制

所有 JS 脚本通过 `index.html` 中的 `<script>` 标签按严格的依赖拓扑顺序引入：

```
game/random.js （全局随机数劫持，必须最先加载）
    │
    ├── solar-system/*.js （棋盘布局）
    ├── game/card-catalog.js → game/cards/*.js
    ├── game/players.js
    ├── game/final-scoring.js → game/end-game-scoring.js
    ├── game/rockets.js → game/planet-reference-layout.js → planet-stats.js
    ├── game/actions/*.js
    ├── game/tech/*.js
    ├── game/data/*.js
    ├── game/history/*.js
    ├── game/abilities/*.js
    ├── game/aliens/*.js （8 种族 + catalog + randomizer + render）
    ├── game/industry/*.js
    ├── game/ai/*.js
    └── app/*.js
            └── app.js （依赖注入，组装所有模块）
```

---

## 三、架构设计

### 3.1 整体分层架构

```
┌─────────────────────────────────────────────┐
│              index.html                     │
│   (DOM 骨架：三栏布局 + 模态对话框 + 调试面板)  │
├─────────────────────────────────────────────┤
│              app.js (主编排器)              │
│   状态初始化 / 回合循环 / 结算流程 / UI 同步   │
├───────────────┬──────────────┬──────────────┤
│  app/  (应用层)│ game/ (核心逻辑) │ solar-system/ │
│  事件绑定      │ 行动/卡牌/科技  │ 棋盘可视化      │
│  UI渲染        │ AI/异族/公司   │ 轨道/扇区      │
│  日志导出      │ 数据放置/历史   │                │
│  调试面板      │ 能力链/终局     │                │
└───────────────┴──────────────┴──────────────┘
```

### 3.2 核心设计模式

| 模式 | 应用场景 | 核心模块 |
|------|---------|---------|
| **命令模式** | 撤销/重做引擎 | `game/history/action-history.js` |
| **依赖注入** | 模块组装与解耦 | `app/dependencies.js` |
| **工厂模式** | 状态对象创建 | `createPlayerState()`, `createBaselineState()` |
| **策略模式** | 8 外星人 / 13 公司差异化行为 | `game/aliens/`, `game/industry/` |
| **观察者模式** | 回合结束结算、收入触发 | `game/abilities/chain.js`（能力链） |
| **状态机** | 回合流程 | `app.js` 核心循环 |

---

## 四、核心实现与机制

### 4.1 确定性随机数系统

基于 FNV-1a 哈希 + xorshift/multiply 混合算法：

```javascript
function hashSeed(seed) {
  let hash = 2166136261;
  for (...) { hash ^= charCodeAt; hash = Math.imul(hash, 16777619); }
  return hash >>> 0;
}

function random() {
  randomState.value = (randomState.value + 0x6D2B79F5) >>> 0;
  let value = randomState.value;
  value = Math.imul(value ^ (value >>> 15), value | 1);
  value ^= value + Math.imul(value ^ (value >>> 7), value | 61);
  return ((value ^ (value >>> 14)) >>> 0) / 4294967296;
}
```

**关键特性**：
- 三种模式：`native`（原生随机）、`seeded`（自定义种子）、`daily`（按日期哈希）
- 通过 `installMathRandom()` 全局劫持 `Math.random`
- 支持快照保存/恢复（`getSnapshot()` / `restoreSnapshot()`）

### 4.2 命令模式撤销引擎

```
ActionHistory 生命周期：

beginSession("playCard", "打出卡牌")
  └── beginStep({ label: "支付费用" })
  │     └── record({ undo: ... })        ← 记录可逆操作
  │   endStep()
  └── beginStep({ label: "触发效果" })
  │     └── record({ undo: ... })
  │   endStep()

undoLastStep()    ← 逆序执行 commands[].undo()
rollbackSession() ← 回滚整个行动
commitSession()   ← 确认不可撤销
```

**关键设计点**：
- Step 粒度：每个 Step 包含多个 Command
- 不可撤销屏障：`undoable: false` + `irreversibleReason`
- 已完成效果流注册表：`createCompletedEffectFlowRegistry()` 防止撤销后重复执行
- 主行动 vs 快速行动：不同粒度的撤销支持

### 4.3 能力链系统（效果流程引擎）

```javascript
// 能力链工作机制
createChain(chainId)
  → insertNode(node)       // 效果节点按类型插入
  → canMergeMovementNodes  // 合并相邻移动效果
  → branch(paths)          // 分支展开形成多路径
  → execute()              // 按优先级排序执行
  → undo()                 // 逆序执行 undo 回调
```

**核心能力**：
- **效果合并**：相邻同类移动效果自动合并点数
- **选择分支**：支持玩家交互选择节点，链暂停等待用户输入
- **多源插入**：同一链可从卡牌、科技、公司被动等多个来源动态插入
- **历史集成**：每个节点标注 `effectFlowId`，与撤销引擎联动

### 4.4 AI 系统：五层决策架构

```
                        ┌──────────────┐
                        │  AI Controller│  (app/ai-controller.js)
                        │  回合调度/超时 │
                        └──────┬───────┘
                               │
            ┌──────────────────┼──────────────────┐
            ▼                  ▼                  ▼
    ┌───────────────┐  ┌───────────────┐  ┌────────────────┐
    │   Goals (目标)│  │ Action Graph  │  │   Battle       │
    │   inferGoals  │  │ 候选动作生成   │  │   Analytics    │
    │   目标推理     │  │ 估值/成本计算  │  │   终局分析      │
    └───────┬───────┘  └───────┬───────┘  └────────┬───────┘
            │                  │                    │
            └──────────────────┼────────────────────┘
                               ▼
                      ┌─────────────────┐
                      │    Planner      │
                      │  束搜索 + 组合   │
                      │  buildTurnPlans  │
                      └────────┬────────┘
                               ▼
                      ┌─────────────────┐
                      │   Evaluator     │
                      │  终局计分投影   │
                      │  收入估值       │
                      └────────┬────────┘
                               ▼
                      ┌─────────────────┐
                      │    Policy       │
                      │  最终动作选择   │
                      └─────────────────┘
```

**各层职责**：

| 层级 | 核心函数 | 职责 |
|------|---------|------|
| **Goals** | `inferGoals()` | 动态推演 6 种目标：首轮冲 25 分、抢占痕迹首位、开局收入积累、终局计分专注 |
| **Action Graph** | `buildActionGraph()` | 生成候选动作，计算 `net = score - cost + bonus`；快速行动 beam=3，主行动 beam=6 |
| **Battle Analytics** | `estimateFinalMarkCashout()` | 根据已标记终局公式预估每分增量的终局价值，考虑剩余轮数衰减 |
| **Planner** | `buildTurnPlans()` | 束搜索 + 组合（quick+main / main+quick），计算链式协同分（synergy = 0.25 × 相关行动分） |
| **Policy** | 最终选择 | 结合目标、估值、对手状态输出最终动作 |

**竞速模型**（`race-model.js`）：分析对手进度，调整激进程度；痕迹首位价值估值（yellow=10, pink=12, blue=12）

### 4.5 外星人种族系统

8 个种族独立实现 + 统一随机分配接口：

| 种族 | 核心差异化机制 |
|------|---------------|
| **九折** | 3×5 痕迹格 + 分数阈值触发免费/付费出牌 + 终局最高威胁度减 10% |
| **半人马** | 3×5 痕迹格（1 号位可无限叠放）+ 能量消耗出牌 + 条件结算 |
| **奥陌陌** | 化石收集 + 专属面板 + 转轮标记 + 登陆/环绕统计奖励 |
| **异常点** | 异常信号检测 + 专属牌区 + 连续异常奖励链 |
| **方舟** | 两组卡牌（普通 + 彩色）+ 队列管理 + 自动解锁机制 |
| **符文族** | 符文符号放置 + 面部符号槽位 + 分支奖励标签 |
| **虫** | 化石收集机制 + 探测奖励 + 两阶段行动 |
| **阿米巴** | 符号标记系统 + 多色痕迹交互 |

### 4.6 工业/公司系统

13 家公司，每家含 1 个主动能力 + 0-N 个被动效果：

| 公司 | 特殊机制 |
|------|---------|
| **赫利昂联合体** | 被动收入加成（`helios_passive_reward_slots`） |
| **原教旨主义** | 禁用打牌行动但强化弃牌角标效果 + 第 2/3/4 轮收入 |
| **星际海盗** | 掠夺科技片封锁（`pirates_raid_markers`） |
| **异星实验室** | 三色面板系统（蓝/黄/粉） |

### 4.7 卡牌系统

**CSV 数据驱动**：
```csv
card_id,card_name,price,card_type_code,set,discard_action_code,scan_action_code,income_code
b_1.webp,观测室女座,2,2,basic,2,2,2
```

- `price`: 0-3 信用点
- `card_type_code`: 0=普通, 1=触发型任务, 2=状态型任务, 3=终局计分
- **三角标系统**：左上（弃牌）→ 右上（扫描）→ 右下（收入）各有效果码

60+ 种效果类型：`SCAN_NEBULA`, `RESEARCH_TECH`, `CARD_ORBIT`, `CARD_LAND`, `INCOME`, `FREE_MOVE`, `DRAW_THEN_SCAN`, `HAND_SCAN` 等 + 7 种异常点专属效果

### 4.8 太阳系棋盘

- 4 个转轮 + 4 个扇区，360° 环形棋盘
- 7 个行星（金木土水火天海），各环绕轨道和登陆位置
- 行星参考贴图通过 JSON 定义 1672×941 分辨率下的像素坐标
- 多色玩家标记（火箭/卫星/登陆器）可拖放定位

### 4.9 调试与可观测性

| 功能 | 说明 |
|------|------|
| 调试面板 | 切换玩家、+20 分、旋转、强制收入、获取卡牌、外星人调试 |
| 状态日志 | 双标签面板：状态日志 / 行动日志，支持导出 CSV/JSON |
| AI 行动简报 | 每轮结束弹出 AI 决策摘要 |
| 兜底控制 | AI 接管、强制跳过、重新开始 |
| 公共 API | `window.SetiRandomizer` 暴露全局调试接口 |

---

## 五、数据流总结

```
┌───────────────────────────────────────┐
│           开始界面 (选择参数)           │
└──────────────┬────────────────────────┘
               ▼
┌───────────────────────────────────────┐
│    随机种子初始化 → Math.random 劫持    │
└──────────────┬────────────────────────┘
               ▼
┌───────────────────────────────────────┐
│  状态创建：玩家 / 外星人 / 科技 /       │
│  数据 / 终局 / 太阳系                  │
└──────────────┬────────────────────────┘
               ▼
┌───────────────────────────────────────┐
│       初始牌/公司选择（结算阶段）        │
└──────────────┬────────────────────────┘
               ▼
┌───────────────────────────────────────┐
│          ╔═══ 回合循环 ═══╗           │
│          ║ 1. 行动选择      ║          │
│          ║   ├─ 人类：UI    ║          │
│          ║   └─ AI：五层决策 ║          │
│          ║ 2. 行动执行      ║          │
│          ║   └─ Command模式 ║          │
│          ║ 3. PASS 判定     ║          │
│          ║ 4. 收入结算      ║          │
│          ║ 5. 5轮 → 终局    ║          │
│          ╚══════════════════╝          │
└──────────────┬────────────────────────┘
               ▼
┌───────────────────────────────────────┐
│         终局计分 → 结果展示             │
└───────────────────────────────────────┘
```

---

## 六、模块清单

### 6.1 核心引擎（game/）

| 模块 | 文件数 | 核心职责 |
|------|--------|---------|
| `random.js` | 1 | 确定性 PRNG + Math.random 劫持 |
| `actions/` | 6 | share / launch / orbit / land / scan-effects / research-tech / quick-trades |
| `tech/` | 9 | catalog → board-state → player-tech → placement → bonuses → resolver → render |
| `data/` | 7 | placement → nebula-placement → state → nebula-state → render → nebula-render |
| `history/` | 3 | action-history → commands → transactions（命令模式撤销引擎） |
| `abilities/` | 6 | rocket → scan → planet → data → tech → chain（能力链引擎） |
| `aliens/` | 11 | catalog + state + 8 种族 + render + randomizer |
| `industry/` | 7 | catalog → abilities → passives → state → placement → render |
| `cards/` | 5 | card-catalog → deck → effects → task-state → basic-cards |
| `ai/` | 8 | valuation → goals → race-model → action-graph → planner → evaluator → policy → battle-analytics |

### 6.2 应用层（app/）

| 模块 | 核心职责 |
|------|---------|
| `dependencies.js` | DI 容器，44 个模块注册 |
| `app.js` | 主编排器：初始化 / 回合循环 / 结算 / UI 同步 |
| `events.js` | DOM 事件绑定 |
| `constants.js` | 全局常量 |
| `dom.js` | DOM 操作工具 |
| `ai-controller.js` | AI 回合调度与超时控制 |

### 6.3 可视化层（solar-system/）

- `core.js`：棋盘渲染核心
- `layout.js`：行星与轨道布局计算

### 6.4 资源层（assets/）

```
assets/
├── cards/       （140+ 卡牌 WebP）
├── aliens/      （8 种族面板图像）
├── core/        （核心 UI 元素）
├── final/       （终局计分图像）
├── industry/    （公司面板图像）
├── symbols/     （符号图标）
├── tech_tile/   （科技片图像）
├── tokens/      （标记图像）
└── cards/card_model.csv （卡牌元数据）
```

---

## 七、技术评价

### 优点

1. **零依赖纯前端**：双击 HTML 即可运行，无构建步骤，无网络依赖
2. **命令模式撤销引擎**：设计精良，支持 Step 级撤销和不可逆屏障
3. **五层 AI 决策架构**：目标推理 → 行动图 → 束搜索规划 → 终局估值 → 策略选择，层次分明
4. **能力链系统**：强大的效果编排引擎，支持合并、分支、多源插入和历史集成
5. **高度差异化种族设计**：8 个外星人 + 13 家公司通过策略模式实现，扩展性良好
6. **确定性随机数**：支持每日种子和存档恢复，保证游戏可复现
7. **丰富的调试工具**：调试面板、双标签日志、CSV/JSON 导出、公共 API

### 可改进之处

1. **`app.js` 单体过重**：超 1.4MB，建议拆分为回合管理、结算、UI 同步等独立模块
2. **全局命名空间冲突风险**：`window.Seti*` 缺乏真正的模块作用域，可引入 ES Module
3. **缺少自动化测试**：无单元测试或集成测试
4. **固定布局**：针对特定屏幕尺寸优化，缺少响应式设计
5. **缺少类型安全**：无 TypeScript 支持，大量对象结构隐式约定
6. **无构建工具链**：虽简单但不利于生产优化（代码压缩、资源打包）

### 技术复杂度评级

| 维度 | 评级 | 说明 |
|------|------|------|
| 代码规模 | ★★★★★ | 150+ JS 文件，app.js 1.4MB+ |
| 架构复杂度 | ★★★★☆ | 多层分层 + DI + 命令模式 + 能力链 |
| AI 复杂度 | ★★★★★ | 五层决策架构 + 束搜索 + 终局投影 |
| 领域复杂度 | ★★★★★ | 8 种族 × 13 公司 × 60+ 效果 × 6 行动 |
| 可维护性 | ★★★☆☆ | 设计模式好但缺乏测试和类型约束 |

---
