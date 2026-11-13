# 天才交易员 · 宏观交易模拟盘

一个**单文件、纯前端**的宏观经济交易模拟器：用宏观基本面、利率、预期差、多空博弈驱动五个板块的价格，你在有限的交易回合里判断方向与仓位，最后结算评级。

> ⚠️ **免责声明（请务必先读）**
> 本项目是**编程与宏观经济教学演示工具**。页面内所有行情、K 线、宏观数据、利率、新闻与事件**均由浏览器端程序随机生成**，与真实市场、真实机构、真实标的无关；程序**不接入任何行情源、交易接口或资金账户**，**不涉及任何资金**，**不构成任何投资建议**，也不承诺任何收益。请勿将模拟结果用于现实投资决策。
> 模拟盘**不含融资与杠杆功能**（做多只用现金，做空为无保证金的反向下注，不会欠钱、不会爆仓）。现实中的融资融券存在维持担保比例、强平线与利息，规则与此完全不同，**切勿照此操作**。
> 存档仅保存在用户浏览器本地（`localStorage`），不上传任何服务器。

---

**[在线体验GeniusTrader（Github Pages）](<https://wolfheimkehr.github.io/GeniusTrader/>)**

## 一、怎么跑起来

没有任何构建步骤、没有依赖、没有后端。

```text
方式 1：双击 index.html，用浏览器打开（推荐 Chrome / Edge）
方式 2：拖进任意静态托管 —— GitHub Pages / Gitee Pages / serv00 / 自己的 Nginx
```

GitHub Pages 部署：

1. 新建仓库（Public），把本目录下**所有文件**（`index.html`、`README.md`、`LICENSE`、`.nojekyll`）上传到仓库根目录；
2. 仓库 `Settings → Pages`，Source 选 `Deploy from a branch`，Branch 选 `main` / 根目录 `/ (root)`，保存；
3. 等 1–2 分钟，访问 `https://<你的用户名>.github.io/<仓库名>/`。

> `.nojekyll` 是给 GitHub Pages 看的，作用是让 Pages 关闭 Jekyll 处理，直接原样发布文件，别删。

---

## 二、玩法概览

| 要素 | 说明 |
| --- | --- |
| 起始资金 | 10,000 |
| 目标笔数 | 默认 20 笔（对局设置里可改） |
| 板块 | 科技 / 消费 / 金融 / 能源 / 债券（债券走反向逻辑，是最好的对冲） |
| 交易动作 | 做多、做空、平仓，以及加仓 / 减仓一半 / 全部平仓 |
| 仓位比例 | 轻 10% / 半 30% / 重 50%，吸底条与挂单浮层**共享同一状态**，随手可换 |
| 挂单 | 限价挂单，最多同时 6 笔；到价以挂单价成交，**挂单不占资金**，成交时才扣款 |
| 设价 | 可直接点 K 线图上某个价位，自动填入挂单价 |
| 节奏 | 「每日拍数」5 / 10 / 20 / 30；暂停、1× / 2× / 4× 调速 |
| 开局预演 | 先自动跑 40 / 80 / 120 个交易日的"历史行情"，跑完解锁交易 |
| 存档 | 每 5 拍自动存档，刷新不丢；可清档重开 |

**图表可切换**：分时（逐拍曲线）/ 日 K / 周 K / 月 K，叠加估值中枢、MA5、MA20、布林带、成交量、成本线，每项都有独立开关。

**附加功能**：分享战绩、导出 CSV 对局日志、欢迎页教学说明。

---

## 三、市场模型（改参数前先看这段）

价格不是随机游走，而是几股力量叠加：

```
板块价格 = 初始价 × 趋势漂移(个股慢随机游走) × 宏观因子 × 情绪
         + 回归项(向 fairValue 动态中枢回归) ± 事件冲击 ± 噪声(按日 K 尺度标定)
```

- **宏观**：双频正弦（长波 0.017 + 短波 0.046）叠加一条 `regime` 随机游走，避免"周期永远 25 天"的机械感；
- **板块**：每个板块有独立的 `idioPhase` / `idioPeriod`（52–132 拍）与 `trend` 慢游走、`volLevel` 波动聚集，所以五个板块**不再同涨同跌**（实测日收益相关性约 −0.07 ~ +0.10）；
- **回归锚**：不是固定基准价，而是随宏观变化的 `fairValue(s)`，回归系数 0.012，偏离 28% 才加强，避免价格被钉死在基准上；
- **量纲**：全部按"日 K 尺度"标定（`volatility 0.004` 左右），日均波动约 2%，单日极值 7–10%。

改参数集中在文件顶部的 `CONFIG`：

```js
startBalance, totalTrades, tickMs, volatility, bondVol, fee,
maxPoints, chartRange, maxBars, newsMax, eventChance, corpEventChance
```

---

## 四、代码结构导览（给下一个接手的人 / AI）

单文件，全部逻辑在 `index.html` 里，按顺序分三段：`<style>` → DOM 骨架 → `<script>`。

**状态**（改功能基本都在动这里）

```js
state = {cash, sectors, current, macro, rates, expect, pos, ratio:0.3, trades, streak,
         best, finished, timer, paused, speed, activeEvents, newsFeed, chartRange,
         log:[], tickNo, preSim, preDaysTarget, orders:[], chartMap:{lo,hi,plotH,padT,padB}}
state.ui = {rounds, showPower, fair, ma, band, vol, cost, barsPerDay:10,
            period:'day', preDays:40}
```

**关键函数**

| 函数 | 职责 |
| --- | --- |
| `marketStep()` | 纯推进一拍市场（不改 UI，可被预演与实盘复用） |
| `tick()` | 一拍 = `marketStep()` + `checkOrders()` + 每 5 拍存档 + `render()` |
| `startPreSim / finishPreSim / applyLockState / canTrade` | 开局预演、进度条与交易锁定 |
| `openPos / addPos / reducePos / closeSec / closeAll` | 开仓与平仓（含做空） |
| `placeOrder / cancelOrder / checkOrders / executeOrder` | 限价挂单全流程 |
| `openOrderPanel / closeOrderPanel / syncSheetToKeyboard / setRatio` | 挂单浮层、键盘避让、比例共享 |
| `drawChart / drawTickChart / buildView` | 日/周/月聚合与绘制、分时绘制 |
| `fairValue / prevClose / portfolio / grade` | 估值中枢、昨收、账户汇总、评级 |
| `render / renderDock / renderOrders / renderPos` | 各区域刷新 |
| `saveGame / loadGame / reset` | `localStorage` 存档（key：`tt_save_v7`） |
| `showWelcome / hideWelcome / exportLog / shareResult` | 引导、导出、分享 |

**响应式断点**（想调布局看这里，CSS 里都带注释）

```
单栏        < 480px
极窄双档    480–559px
紧凑双档    560–767px
双栏        480–1179px（内容 max-width 1024）
三栏        ≥ 1180px（内容 max-width 1440）
超宽        ≥ 1600px
```

**已知坑（都是踩过的）**

- `file://` 下 `navigator.clipboard` 可能不可用 → 代码里已有 `execCommand` 兜底；
- 浏览器对同名本地文件缓存很顽固 → 所以版本号写进文件名，并在顶栏放一个 `vN · 视口 xxpx · 单栏/双栏` 徽章，白屏时先看徽章判断是不是旧文件；
- 无头浏览器最小窗口约 496px，更窄的截图是裁切假象，不是布局坏了；
- K 线图设价依赖 `state.chartMap`（每次绘制时记录绘图区上下边界），改绘图区尺寸时记得同步更新它。

---

## 五、版本历史

| 版本 | 主要变化 |
| --- | --- |
| v3 | 自定义回合数、多空博弈条开关、五类叠加线（估值 / 均线 / 布林带 / 成交量 / 成本线） |
| v4 | 每日拍数（5/10/20/30）、真 K 线（红涨绿跌）、暂停与倍速 |
| v5 | 日 / 周 / 月视图聚合 |
| v6 | 分时（逐拍）曲线 |
| v7 | 本地自动存档、分享战绩、CSV 导出、欢迎页 |
| v8 | 开局预演 40/80/120 天 + 进度条 |
| v9 | 市场模型重构：宏观双频、板块独立相位、动态估值中枢、日 K 尺度重标 |
| v9.1 | 吸底交易条、限价挂单、点 K 线设价 |
| v9.2 | 去掉页面中部重复的交易按钮，交易入口只留吸底条 |
| v9.3 | 挂单改为底部浮层，跟随键盘（`visualViewport`）上移，输入框不再被遮挡 |
| v9.4 | 仓位比例从持仓卡移入吸底条 + 挂单浮层，共享状态、随手可调 |
| v9.5 | 四处置入风险提示与免责（欢迎页 / 结算弹窗 / 页脚 / 源码注释），分享文案带声明 |
| v9.6 | 修复PC端鼠标单击板块按钮不切换 |
| v9.7 | 修复债强股弱与数据不符的问题 |


---

## 六、License

[MIT](./LICENSE) —— 可自由使用、修改、分发、商用，保留版权声明即可。

如果你基于本项目做二次分发或教学使用，**请保留页面内的免责声明**，并继续标注"模拟盘 / 教学演示"，不要包装成实盘工具或投资建议。
