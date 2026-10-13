# A股多策略选股日报（Quant）

基于公开行情数据的 **A 股多策略选股 + 每日复盘日报** 工作流：用 7 套实战验证过的策略筛选沪深主板候选股，每个交易日收盘后做「复盘上期 → 重跑策略 → 推送日报」的闭环。

> ⚠️ 本项目仅用于策略研究与数据整理，**不构成任何投资建议**。详见文末免责声明。

---

## 目录结构

```
├── 选股/                     # 每日选股成果（核心产出）
│   └── 2026-09-29/
│       ├── 00-总览.md        # 当日 7 策略速查表 + 大盘环境 + 操作要点
│       ├── 00上期选股复盘/    # 上期候选股复盘（00 前缀保证排序第一）
│       ├── 三道筛选法/        # 每个策略一份选股清单.md
│       ├── 均线多头极简选股法/
│       ├── 低位首板龙头战法/
│       ├── 小市值轮动策略/
│       ├── 红利低波策略/
│       ├── 双低可转债轮动策略/
│       ├── 二八轮动策略/
│       └── 节后错杀反弹筛选/
├── 选股策略/                  # 7 套策略的完整说明（逻辑、亮点、不足、适用环境）
│   ├── 00-总览.md
│   └── 01~07-*.md
└── .workbuddy/skills/         # 项目级技能：把整套工作流交给 AI 助手自动执行
    └── a-share-strategy-screening/SKILL.md
```

---

## 快速开始（clone 后要跑起来，照着做）

### 前置条件

| 依赖 | 说明 | 必需 |
|---|---|---|
| WorkBuddy 客户端 | 本项目的执行环境（AI 助手 + 技能 + 连接器） | ✅ |
| **金融数据插件 finance-data** | 提供 `westock-tool`（选股筛选）与 `westock-data`（行情快照）两个 CLI，需在 WorkBuddy 插件/连接器市场中搜索安装并授权 | ✅ |
| Node.js 受管运行时 | WorkBuddy 自带，路径 `%USERPROFILE%\.workbuddy\binaries\node\versions\<ver>\node.exe` | ✅ |
| 飞书连接器 | 用于把日报发到群（lark-cli），需各自扫码授权 | 可选 |
| QQ 邮箱连接器 | 用于把选股结果发邮箱 | 可选 |

### 三步上手

**第 1 步：拉代码并用 WorkBuddy 打开该目录**

```bash
git clone https://github.com/LiOne-W/Trading.git
```

用 WorkBuddy 打开 clone 下来的目录作为**工作目录**。此时 `.workbuddy/skills/a-share-strategy-screening/SKILL.md` 会自动作为**项目级技能**加载——整套选股工作流（含 7 个策略的筛选表达式、踩坑点、发群流程）就绪，无需手工安装。

> 如果你的环境没有自动加载项目级技能，把该 SKILL.md 复制到自己的用户级技能目录 `%USERPROFILE%\.workbuddy\skills\` 即可。

**第 2 步：初始化行情 CLI（一次性）**

```powershell
$OutputEncoding=[System.Text.Encoding]::UTF8;[Console]::OutputEncoding=[System.Text.Encoding]::UTF8
$n="$env:USERPROFILE\.workbuddy\binaries\node\versions\<你的node版本>\node.exe"
Set-Location "$env:USERPROFILE\.workbuddy\plugins\cache\cb_teams_marketplace\finance-data\<ver>\skills\westock-data"
& $n scripts/setup.cjs      # 安装 westock.exe 到 %USERPROFILE%\.local\bin\
```

验证：

```powershell
& "$env:USERPROFILE\.local\bin\westock.exe" quote sh600019 | Out-File "$env:TEMP\_q.txt" -Encoding utf8
```

**第 3 步：告诉助手今天要做什么**

直接用自然语言下指令即可，例如：

- 「复盘昨天选股，重选今天的，发飞书群」
- 「按三道筛选法筛主板股，每策略 3 只」
- 「分析被节前情绪错杀的优质股，筛节后反弹候选，发群」

助手会按技能里固化的流程执行：拉大盘环境 → 跑 `westock-tool filter` → 剔除非主板 → `westock quote` 核对 → 生成 `选股/<日期>/` 文档 → 确认后发群。

---

## 每人必须替换的个人配置

换机器/换人后，以下几项是**私有的，仓库里的默认值不适用**，请改成自己的：

| 配置项 | 位置 | 怎么获取 |
|---|---|---|
| 飞书群 `chat_id` | 技能 SKILL.md「个人配置替换清单」 | 群设置，或用 `lark-cli im +chat-search` 搜索群名 |
| 收件邮箱 | 发给邮箱时指定 | 用你自己的邮箱连接器 |
| 飞书授权 | `lark-cli auth login --scope im:message.send_as_user` | 各自扫码，**不要共享 token** |
| Node / 插件路径中的版本号 | 技能 SKILL.md 顶部清单 | 先 Glob 查实际版本，不要照抄 |

---

## 7 套策略一览

| 策略 | 核心逻辑 | 适配环境 |
|---|---|---|
| 三道筛选法 | 基本面 + 估值 + 趋势 + 资金四关全过 | 全环境，普跌日信号最少 |
| 均线多头极简 | 收盘 > MA5 > MA10 > MA20 > MA60 + 量能资金 | 趋势市；缩量震荡市易失效 |
| 低位首板龙头战法 | 低位首个涨停 + 次日溢价 | 情绪升温期；晋级率 <50% 时停用 |
| 小市值轮动 | 20~80 亿微盘 + 低估值 + 资金流入 | 小盘占优期 |
| 红利低波 | 高股息 + 低估值 + 低波动 | 普跌市/防守期（实测最稳） |
| 双低可转债映射 | 低 PE + 低 PB + 高股息蓝筹底仓 | 过节/防守底仓 |
| 二八轮动 | 中证500 vs 沪深300 的 20 日强弱 | 需两个指数同向为正才启用 |

各策略的完整逻辑、亮点与不足见 `选股策略/`。

---

## 常见坑（已固化在技能里）

1. **市值字段单位是「元」**：`TotalMV < 60` 会返回空，必须写 `TotalMV < 6000000000`（60 亿）
2. **均线字段名带下划线**：`MA_20`，写成 `MA20` 静默返回空数据
3. **PE/PB 必须排除负值**：`intersect([PE_TTM>0, ...])`
4. **PowerShell 不回显 stdout**：命令末尾 `Out-File xxx.txt` 再用 Read 读
5. **飞书多行消息不能用 `$'...\n...'`**：PowerShell 不解析 `\n`，必须用 here-string `@'...'@` 写真实换行

---

## 免责声明

本项目所有内容基于公开数据和量化规则整理，**仅供策略研究与学习参考，不构成投资建议**。市场有风险，投资需谨慎。过往表现不预示未来收益。任何策略都无法预测个股次日涨跌，输出定位为"当前状态下符合策略特征的候选池"。
