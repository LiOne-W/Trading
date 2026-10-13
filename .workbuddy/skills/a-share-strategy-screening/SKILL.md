---
name: a-share-strategy-screening
description: A股多策略选股与候选池生成 + 每日选股日报循环（复盘上期→重跑策略→发飞书群）。用 westock-tool 选股CLI按条件/策略/排行筛选A股，按主板/创业板等板块过滤，生成分策略的候选股文档。当用户要求"按某策略选股""筛出主板XX股""下周可能上涨的股票""按条件选A股""复盘上期选股""今天的选股发到飞书群"时使用。内含该工具链的关键踩坑点（市值单位、均线字段名、编码、Bash不可用、lark-cli全路径）。
agent_created: true
---

# A股多策略选股与候选池生成

> 本文件随仓库共享。**首次使用前请先按 `README.md` 的「快速开始」配好环境**（金融数据插件、连接器授权、命令行路径），并按本文「个人配置替换清单」改写为你自己的参数。

## 何时使用

- 用户要求按指定策略、条件、指标筛选 A 股股票
- 需要限定板块（主板 / 创业板 / 科创板）或市值区间
- 需要生成分策略的候选股清单文档
- 用户要求"复盘上期选股 + 重选今日标的 + 发到群"

## 🔧 个人配置替换清单（换机器必做）

| 配置项 | 本仓库默认 | 怎么改 |
|---|---|---|
| Node 受管版本路径 | `%USERPROFILE%\.workbuddy\binaries\node\versions\<ver>\node.exe` | 先 Glob 查实际版本号，不要写死 |
| westock-tool 路径 | `%USERPROFILE%\.workbuddy\plugins\cache\cb_teams_marketplace\finance-data\<ver>\skills\westock-tool\scripts\index.js` | 先 Glob 确认 `<ver>` |
| westock.exe | `%USERPROFILE%\.local\bin\westock.exe` | 由 `westock-data/scripts/setup.cjs` 安装生成 |
| lark-cli | `%USERPROFILE%\.workbuddy\binaries\node\cli-connector-packages\node_modules\@larksuite\cli\bin\lark-cli.exe` | 随飞书连接器安装，不在 PATH |
| 飞书群 chat_id | 需自行填写（见群设置或用 `im +chat-search` 查） | 发送前必须替换 |
| 收件邮箱 | 需自行填写 | 用 QQ 邮箱连接器 |

## 关键环境约束（每次都会踩，务必先读）

### 1. Bash 工具在本机不可用

Git Bash shim 损坏，报 `dirname: command not found` / `mkdir: command not found`。**所有命令一律用 PowerShell 工具执行。**

### 2. PowerShell 工具不回显 stdout

必须在命令末尾把输出重定向到文件，再用 Read 工具读取内容：

```powershell
$OutputEncoding=[System.Text.Encoding]::UTF8;[Console]::OutputEncoding=[System.Text.Encoding]::UTF8
$n="$env:USERPROFILE\.workbuddy\binaries\node\versions\<ver>\node.exe"
& $n "<westock-tool>\scripts\index.js" filter '...' --limit 30 2>&1 | Out-File "<workdir>\_out.txt" -Encoding utf8
```

- **必须先设 `[Console]::OutputEncoding = [System.Text.Encoding]::UTF8`**，否则中文全部乱码
- `2>&1` 用于捕获 stderr（工具报错会走 stderr）
- Node 用受管版本绝对路径，不要用裸 `node`

## 数据源：westock-tool（选股筛选权威入口）

路径：`%USERPROFILE%\.workbuddy\plugins\cache\cb_teams_marketplace\finance-data\<ver>\skills\westock-tool\scripts\index.js`

先 Glob 确认 `<ver>`（如 `1.6.0`）。

### 六个子命令路由

| 用户意图 | 子命令 | 示例 |
|---|---|---|
| 自定义条件表达式 | `filter "expr"` | `filter 'intersect([PE_TTM>0, PE_TTM<20, ROETTM>15])'` |
| 条件型预设 | `filter --preset <Name>` | `filter --preset HighDividend` |
| 策略信号（MACD金叉等） | `strategy <code>` | `strategy macd_golden` |
| 分类标签 | `label <code>` | `label shareholder_central_state` |
| 事件+时间窗口 | `event <code>` | `event shareunlock_next_90` |
| 按指标排序 | `ranking <indicator>` | `ranking CompScore --limit 10` |

查清单必须执行 `--list`（`strategy --list` / `filter --list-presets` / `ranking --list` / `label --list` / `event --list`），不要凭本文档列举。

### 表达式语法

- 单条件：`字段 比较符 值`
- **AND 必须用 `intersect([条件1, 条件2])`** —— 不支持 `&` / `&&` / `AND`
- OR 用 `union([...])`
- **只有表达式里写到的字段才会出现在返回列中**。想多看某字段，就把它加进条件里，例如想同时拿到流通市值：`intersect([..., NegotiableMV > 0])`

### 🔴 三个致命踩坑

1. **`TotalMV` / `NegotiableMV` 的筛选单位是「元」，不是亿元**
   - 输出表格里显示 "1254.65"（亿元），但条件里必须写 `TotalMV > 100000000000`（1000亿）
   - 写 `TotalMV < 60` 会直接返回"数据为空"
   - 换算：1亿元 = 100000000

2. **均线字段名带下划线**：`MA_5` / `MA_10` / `MA_20` / `MA_30` / `MA_60` / `MA_120` / `MA_250`
   - 写成 `MA5` 会返回空数据，不报错，极易误判为"没有符合的股票"

3. **PE/PB 必须排除负值**：`intersect([PE_TTM > 0, PE_TTM < 20, ...])`

### 已验证可用的字段（沪深 A 股）

- 行情：`ClosePrice` `OpenPrice` `HighPrice` `LowPrice` `ChangePCT` `RangePCT` `TurnoverValue` `TurnoverVolume` `TurnoverRate` `VolumeRatio` `Chg5D` `Chg10D` `Chg20D` `Chg60D` `ChgYtd`
- 市值：`TotalMV` `NegotiableMV` `TotalShares` `FloatShares`
- 估值：`PE_TTM` `PE_Fwd` `PE_Lyr` `PB` `PS_TTM` `DividendRatioTTM`（股息率 TTM %）
- 财务：`ROETTM` `ROA` `TORGrowRate`（营收同比 %）`NPParentCompanyYOY` `DebtAssetsRatio` `NetOperateCashFlow` `GrossIncomeRatio`
- 技术：`MA_5/10/20/30/60/120/250` `DIF` `DEA` `MACD` `KDJ_K/D/J` `RSI_6/12/24` `BOLL_UPPER/MID/LOWER`
- 资金：`MainNetFlow`（主力净流入）`JumboNetFlow` `BlockNetFlow` `MainNetFlow5D/10D/20D` `MainInflowRank`
- 筹码：`ChipProfitRate` `ChipConcentration90/70`

## 板块过滤：代码前缀

| 板块 | 前缀 |
|---|---|
| 沪市主板 | 600 / 601 / 603 / 605 |
| 深市主板 | 000 / 001 / 002 / 003 |
| 创业板 | 300 / 301 |
| 科创板 | 688 / 689 |
| 北交所 | 8xx / 4xx |

**注意**：工具不支持按板块直接过滤，做法是拉足够多的行（`--limit 60~80`），再按 code 前缀人工剔除。港股要加 `--market hk`，美股 `--market us`。

## 数据源：westock-data（个股价量快照）

CLI 需先安装（一次性）：

```powershell
Set-Location "<finance-data>\skills\westock-data"
& $n scripts/setup.cjs          # 安装到 %USERPROFILE%\.local\bin\westock.exe
```

之后批量取行情（逗号分隔代码，一次可传 15~35 只）：

```powershell
& "$env:USERPROFILE\.local\bin\westock.exe" quote sh600019,sz000651,sh603986 | Out-File ... -Encoding utf8
```

返回含：price / change_percent / turnover_rate / volume_ratio / pe_ratio / pb_ratio / dividend_ratio_ttm / total_market_cap / circulating_market_cap / chg_5d / chg_10d / chg_20d / chg_60d / chg_ytd / high_52week / low_52week 等。

## 7 个策略的筛选表达式（可直接复用）

| 策略 | filter 表达式 |
|---|---|
| 三道筛选法 | `intersect([ROETTM>12, TORGrowRate>10, DebtAssetsRatio<60, PE_TTM>0, PE_TTM<25, ClosePrice>MA_20, MainNetFlow>0])` |
| 均线多头极简 | `intersect([ClosePrice>MA_5, MA_5>MA_10, MA_10>MA_20, MA_20>MA_60, TurnoverValue>500000000, MainNetFlow>0])` |
| 小市值轮动 | `intersect([TotalMV>2000000000, TotalMV<8000000000, PE_TTM>0, PE_TTM<30, ROETTM>5, ClosePrice>MA_20, MainNetFlow>0])` `--orderby TotalMV --asc` |
| 红利低波 | `intersect([DividendRatioTTM>4, PE_TTM>0, PE_TTM<15, ROETTM>8, PB<2, ClosePrice>MA_20, MainNetFlow>0])` |
| 双低映射 | `intersect([PE_TTM>0, PE_TTM<15, PB<1.5, DividendRatioTTM>2, ClosePrice>MA_20, MainNetFlow>0])` |
| 二八轮动（预案） | `intersect([TotalMV>10000000000, TotalMV<50000000000, PE_TTM>0, ClosePrice>MA_20, MainNetFlow>0])` `--orderby MainNetFlow --desc` |
| 低位首板 | 无固定表达式，从当日涨停榜筛低位主板首板（需人工核对位置与题材）；仅在连板晋级率 >50% 且出现 4 板以上空间板时启用 |
| 错杀反弹（附加） | A 组 `intersect([Chg20D<-12, ROETTM>12, TORGrowRate>10, DebtAssetsRatio<60, PE_TTM>0, PE_TTM<30, RSI_6<35])`；B 组 `intersect([Chg20D<-10, MainNetFlow5D>0, ROETTM>12, PE_TTM>0, PE_TTM<30])` |

## 标准工作流

1. **市场环境**：先用 `agentic_search` 取当日大盘状态、主线板块、涨跌停家数（这是金融类请求的强制前置）
2. **逐策略构建 filter 表达式**：把策略的文字规则翻译成字段条件；先跑一次小规模探针验证表达式不返回空
3. **`--limit 60` 拉候选池**，`--orderby <字段> --desc/--asc` 排序
4. **按代码前缀剔除**非目标板块
5. **对候选股 `westock quote` 批量取快照**，核对流通市值、位置（Chg60D）、换手率、量比
6. **生成文档**：按 `选股/<YYYY-MM-DD>/<策略名>/选股清单.md` 组织；每份含筛选逻辑+表达式、候选表格、逐只入选理由、风险提示、使用建议
7. **所有含买卖判断的输出必须附固定免责声明**，禁止改写

## 输出纪律（金融场景红线）

- 禁止编造数据；数据缺失就说"数据源未覆盖"
- 所有关键数字标注来源+时点（如"腾讯自选股接口，截至 2026-09-18 收盘"）
- 必须明确说明：**没有任何策略能预测个股次日涨跌**，输出定位为"当前状态下符合策略特征的候选池"，不是确定性预测
- 含具体投资建议的输出必须附固定免责声明，不可省略或改写

## 每日选股日报循环（复盘 → 重选 → 发群）

固定节奏：每个交易日收盘后复盘上期选股、用当日数据重跑策略、把结果发到飞书群。

### 第 1 步：复盘上期选股

1. 从 `选股/<上期日期>/` 读取上期候选股名单（00-总览.md + 各策略清单）
2. `agentic_search` 取当日大盘：指数涨跌、成交额、涨跌停家数、领涨领跌板块、主线是否切换
3. `westock.exe quote` 批量拉上期全部候选股当日行情（一次 15~35 只，分批）
4. 逐策略统计平均涨幅 vs 上证指数，回答三个问题：
   - 与预期的差别（哪些超预期/低于预期，为什么——通常归因到"板块是否当日主线"）
   - 上期预判哪些被验证/证伪
   - **持仓处置建议**：趋势完好→持有；二连板→卖一半兑现；跌破关键均线/放量转弱→离场；写入逐只说明
5. 输出到 `选股/<今日日期>/00上期选股复盘/上期选股复盘.md`（**文件夹名加 `00` 前缀保证排序第一**）

### 第 2 步：重跑策略选股

- 复用上方「7 个策略的筛选表达式」，只换日期
- 每策略选 3 只，存入 `选股/<今日日期>/<策略名>/选股清单.md` + `00-总览.md`
- 二八轮动信号检查：比较中证500 与沪深300 的 Chg20D，两者均为负→观望/持债（仅列预案）；中证500 转正→翻多

### 第 3 步：发飞书群

- lark-cli **不在 PATH**，必须用全路径（见「个人配置替换清单」）
- 以用户身份发送（`--as user`），scope `im:message.send_as_user` 需各自授权；若报权限错误走 `auth login --scope ... --no-wait` + QR 流程
- 多行 markdown **必须用 PowerShell here-string（`@'...'@`）写真实换行**。🔴 坑：bash 风格的 `$'...\n...'` 在 PowerShell 中**不解析 `\n`**，会原样发出字面 `\n` 文本
- **输出必须重定向到文件再 Read 验证回执**（PowerShell 工具不回显 stdout；文件上传时 stderr 的 "uploading file" 进度会伪装成红色错误，只要 JSON 回执 `ok: true` 即成功）
- 附件用 `--file "选股/<日期>/00-总览.md"`
- 发送前用 AskUserQuestion 让用户确认消息内容（群内他人可见）
- 发完清理临时 txt，写工作日志

## 收尾

- 清理工作区临时 txt 文件（`Remove-Item -LiteralPath` 逐个删，不要用通配符）
- 写项目工作日志到 `.workbuddy/memory/YYYY-MM-DD.md`
