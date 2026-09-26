# Task 7 作业提交 · MiniDex

<!-- 建议放在仓库的 docs/task7.md；截图放在 docs/images/ 下，文件名和下面的占位一致就能直接显示。
     HTML 注释（像这一段）在渲染后看不到，是给自己补截图时看的提示，交之前可以删掉。 -->

- **项目**：MiniDex —— WAVAX/USDC 订单簿交易所。资金托管在 Avalanche Fuji 上的 Vault 合约里，撮合在链下完成
- **代码仓库**：<https://github.com/steven7289665382-sys/vibeCoding_DEX>（`master` 分支）
- **开发方式**：先写规格、测试先行，AI 辅助编码（必做规格见 `docs/06-trading-layer-spec.md`，进阶规格见 `docs/07-advanced-spec.md`）

**提交物一览**

| 要求                          | 位置          |
| --------------------------- | ----------- |
| 代码仓库或提交记录                   | 下方「提交记录」    |
| npm test 与 forge test 全绿的证明 | 一 · 1、一 · 2 |
| 3 个 Fuji 合约地址               | 一 · 2       |
| deposit 和 withdraw 的交易哈希    | 一 · 2、一 · 3 |
| 登录、余额、两地址成交的截图              | 一 · 3       |
| 进阶功能的代码、测试或演示证明             | 二           |

**提交记录**

| 提交        | 内容                                                                         |
| --------- | -------------------------------------------------------------------------- |
| `d90683b` | 必做部分：合约与部署脚本、后端（登录、链上监听、提现签名、账本、撮合引擎、交易服务、WebSocket）、前端交易界面、交易层规格 |
| `cb5ff20` | 清理：删除误提交的 `minidex-history.bundle` 和不再使用的 `useBalances.ts`，`.gitignore` 增加 `*.bundle` |
| `92b9029` | 进阶规格：IOC / FOK、私有 orders 频道、做市机器人                                           |
| `b3b0371` | 撮合引擎支持 IOC / FOK                                                           |
| `e5fdd4a` | 交易服务支持 IOC / FOK；每个操作返回统一的"变化清单"，供推送使用                                     |
| `9b82a13` | WebSocket 私有 orders 频道                                                       |
| `d4a87d3` | 做市机器人（买卖两侧各挂 3 档）                                                          |
| `648a7a0` | 前端：下单可选 IOC / FOK；委托列表改由私有频道推送                                             |

**整体结构**

```
钱包 ──approve / deposit──▶ Vault 合约 ──Deposit 事件──▶ 后端监听，给交易所账本入账
钱包 ──EIP-712 签名登录、下单、撤单──▶ 后端（撮合引擎 + 账本）──WebSocket──▶ 前端（订单簿、成交、余额）
钱包 ◀──提现签名── 后端；用户拿签名自己调用 Vault.withdraw 取回代币
```

---

## 一、必做部分（60 分）

### 1. 撮合引擎（20 分）

#### npm test 全部通过

```bash
cd server
npm test
```

结果：6 个测试文件、80 个用例全部通过。加入进阶功能后是 7 个文件、114 个用例，同样全部通过（见第二部分）。

![npm test 全部通过](images/01-npm-test.png)

<!-- 截图：终端里 npm test 的结尾，能看到 Test Files 6 passed、Tests 80 passed；导入进阶提交之后再截是 7 passed、114 passed，两种都行 -->

#### 补充「时间优先」测试用例

位置：`server/src/engine/OrderBook.test.ts` →「时间优先：同价先到先成交」

同一价位挂两张卖单，买 1.5 时应先吃满先挂的那张（1），再吃后挂那张的 0.5，后挂那张剩 0.5 留在订单簿上。

```ts
it("时间优先：同价先到先成交", () => {
  const first = limit("alice", "sell", "20", "1");
  const second = limit("carol", "sell", "20", "1");
  book.submit(first);
  book.submit(second);

  const { fills } = book.submit(limit("bob", "buy", "20", "1.5"));

  expect(fills).toHaveLength(2);
  expect(fills[0].makerOrderId).toBe(first.id);
  expect(fills[0].qty).toBe(d("1"));
  expect(fills[1].makerOrderId).toBe(second.id);
  expect(fills[1].qty).toBe(d("0.5"));
  expect(book.snapshot(10).asks).toEqual([[d("20"), d("0.5")]]);
});
```

实现方式：每个价位是一个先进先出队列，新单追加到队尾，成交从队头开始吃。

#### 补充「拒绝 self-trade（自成交）」测试用例

位置：同一文件 →「OrderBook 自成交防护（preventSelfTrade）」，共 3 个用例：

| 用例                      | 验证什么                                  |
| ----------------------- | ------------------------------------- |
| 默认关闭：自己能吃自己的单           | 对照组，证明开关确实起作用                         |
| 开启后：会碰到自己挂单的订单整单拒绝，簿子不变 | 限价单、市价单都抛出 `SelfTradeError`，订单簿前后完全一致 |
| 开启后：吃饱前碰不到自己的单就正常成交     | 不误伤：只吃到别人的单，或限价没越过自己的挂单时正常成交          |

```ts
it("开启后：会碰到自己挂单的订单整单拒绝，簿子不变", () => {
  const book = new OrderBook({ preventSelfTrade: true });
  book.submit(limit("carol", "sell", "20", "1"));
  book.submit(limit("alice", "sell", "21", "1"));
  const before = book.snapshot(10);

  // 先吃 carol 的 20，还剩 1 会碰到 alice 自己的 21 → 整单拒绝
  expect(() => book.submit(limit("alice", "buy", "21", "2"))).toThrow(SelfTradeError);
  expect(() => book.submit(market("alice", "buy", "2"))).toThrow(SelfTradeError);
  expect(book.snapshot(10)).toEqual(before);
});
```

规则：撮合前先按真实顺序"演练"一遍。如果下单方在吃饱之前会碰到同一地址的挂单，就整单拒绝，不产生任何成交、也不挂单。选择整单拒绝、而不是"吃到自己为止"，是为了让结果原子化，便于理解和对账。

服务端的交易服务开启了这个开关，另外还有两层测试：

- `server/src/exchange.test.ts`「自成交：拒绝，冻结原样退回，账本和订单簿不变」
- `server/src/routes.test.ts`：自成交下单返回 HTTP 400

### 2. 合约部署到 Fuji 测试网（20 分）

#### forge test 全部通过

```bash
cd contracts
forge test
```

结果：11 个用例全部通过。覆盖测试币铸造与小数位、充值（转账、事件、代币白名单、零金额），提现（后端签名成功、nonce 重放、签名过期、签名者不对、别人冒用签名），EIP-712 摘要对拍，以及 owner 权限。

![forge test 全部通过](images/02-forge-test.png)

<!-- 截图：forge test 的输出，能看到 11 passed; 0 failed -->

#### 3 个已部署的合约地址

网络：Avalanche Fuji（chainId 43113），部署账号 `0x99D30eE672866C8aB6490c53512347B68fDba0f5`

| 合约                | 地址                                                                                                                              | 部署交易                                                                                                                        |
| ----------------- | ------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| Vault             | [`0x78f603fcD2F42AED08f6e095bdA26de5E2d1141A`](https://testnet.snowtrace.io/address/0x78f603fcD2F42AED08f6e095bdA26de5E2d1141A) | [`0x43d0e430…5b89bfd`](https://testnet.snowtrace.io/tx/0x43d0e43085d5ac8f1687035f7913fd965d754cf84a9e17317e689043d5b89bfd)  |
| MockUSDC（6 位小数）   | [`0x98e85ec670C49e961758c81704F242BF3421e251`](https://testnet.snowtrace.io/address/0x98e85ec670C49e961758c81704F242BF3421e251) | [`0xf6e9f8a5…650acb4b`](https://testnet.snowtrace.io/tx/0xf6e9f8a5a30a9c7aa83df352ee12bcbc9fc647cfe1a26fe474b8bcb9650acb4b) |
| MockWAVAX（18 位小数） | [`0x2388BF421373970d20d60940024628728E1eb6f0`](https://testnet.snowtrace.io/address/0x2388BF421373970d20d60940024628728E1eb6f0) | [`0xf6308564…c4c32c3d`](https://testnet.snowtrace.io/tx/0xf630856462feba79a9a0c75ebbf05092ccd58957f1c2b0dcf28f9ebcc4c32c3d) |

#### 真实的 deposit 交易

- tx hash：0xa64269714435ff404de1e2a9ad8685acd20e7bccc8562f66008a5214b40f3b2e
- Snowtrace：https://testnet.snowtrace.io/tx/0xa64269714435ff404de1e2a9ad8685acd20e7bccc8562f66008a5214b40f3b2e
- 内容：地址 `0x99D30eE672866C8aB6490c53512347B68fDba0f5` 调用 `Vault.deposit(token, amount)` 充值 100 WAVAX（token 为 MockWAVAX `0x2388BF421373970d20d60940024628728E1eb6f0`，amount = 100 × 10¹⁸）。交易日志里有 MockWAVAX 的 `Transfer`（本地址 → Vault，100 WAVAX）和 Vault 的 `Deposit` 事件，后端监听到后给该地址的交易所余额入账

![deposit 交易](images/03-deposit-tx.png)

<!-- 截图：Snowtrace 上这笔交易的页面（Status: Success，To 是 Vault 地址）；或前端充值表单显示「✅ 已入账」 -->

### 3. 端到端演示（20 分）

运行方式：

```bash
cd server && npm start      # 在线模式，连接 Fuji
cd web && npm run dev       # 浏览器打开 http://localhost:5173
```

#### 登录成功

流程：MetaMask 连接 → 切换到 Avalanche Fuji → EIP-712 签名登录。后端校验签名后签发 JWT。

![登录成功](images/04-login.png)

<!-- 截图：顶栏能看到「Avalanche Fuji」、钱包地址、绿色「已登录」 -->

#### 余额显示

「资产与充提」里同时显示交易所余额（可用 / 冻结，由 WebSocket 实时推送）和钱包的链上余额。

![余额显示](images/05-balance.png)

<!-- 截图：充值之后的「资产与充提」，交易所余额不为 0 -->

#### 两个不同地址完成成交

| 角色   | 地址                                           | 操作                              |
| ---- | -------------------------------------------- | ------------------------------- |
| 卖方 A | `0x99D30eE672866C8aB6490c53512347B68fDba0f5` | 限价卖出 `1` WAVAX @ `7.5` USDC（挂单）       |
| 买方 B | `0xBa05969a75765b2cCeA803620CaD1511ffF88925` | 买入 `1` WAVAX（主动吃单，成交价 7.5 USDC） |

后端成交日志（买卖双方是两个不同地址）：

```
[trade] #1 1 WAVAX @ 7.5 USDC buyer=0xba05969a75765b2ccea803620cad1511fff88925 seller=0x99d30ee672866c8ab6490c53512347b68fdba0f5 taker=buy
```

![卖方 A：历史委托显示已成交](images/06-trade-seller.png)

<!-- 截图：地址 A 的页面，「历史委托」里卖单「已成交」，最近成交里有这一笔 -->

![买方 B：历史委托显示已成交](images/07-trade-buyer.png)

<!-- 截图：地址 B 的页面，「历史委托」里买单「已成交」，余额里多了 WAVAX -->

#### withdraw 交易

流程：前端请求 `POST /withdraw` → 后端先从账本扣款，再用签名私钥做 EIP-712 签名（10 分钟有效）→ 用户自己调用 `Vault.withdraw(token, amount, nonce, deadline, signature)` 取回代币。合约校验签名、nonce 和截止时间，并要求签名里的用户就是调用者本人。

- tx hash：`0x66e7db7c46cb4d417487ad984a48c7a406dd5c5b8f42f0d0b2491b4eb6706e56`
- Snowtrace：https://testnet.snowtrace.io/tx/0x66e7db7c46cb4d417487ad984a48c7a406dd5c5b8f42f0d0b2491b4eb6706e56
- 内容：地址 `0x99D30eE672866C8aB6490c53512347B68fDba0f5`（卖方 A）调用 `Vault.withdraw` 提现 10 WAVAX，区块 58754413，状态 Success。交易日志里有 MockWAVAX 的 `Transfer`（Vault → 本地址，10 WAVAX）和 Vault 的 `Withdraw` 事件

![withdraw 交易](images/08-withdraw-tx.png)

<!-- 截图：Snowtrace 上 withdraw 交易的页面；或前端「✅ 提现成功，代币已到钱包」 -->

---

## 二、进阶部分（本次提交 4 项，共 34 分）

| #   | 进阶项                   | 分值 | 主要代码                                                                                  | 证明                 |
| --- | --------------------- | -- | ------------------------------------------------------------------------------------- | ------------------ |
| 1   | 做市机器人，买卖两侧各挂 3 档      | 10 | `server/src/marketmaker.ts`                                                           | 15 个测试用例 + 截图       |
| 2   | 支持 IOC / FOK 订单        | 8  | `server/src/engine/OrderBook.ts`、`server/src/exchange.ts`、`web/src/OrderForm.tsx`     | 14 个测试用例 + 截图       |
| 3   | WebSocket 私有 orders 频道 | 8  | `server/src/ws.ts`、`server/src/app.ts`、`web/src/useMarket.ts`                         | 5 个测试用例 + 抓到的真实推送消息 |
| 4   | AI 安全审查报告 + 修复一个真实问题   | 8  | `server/.env.example`、`.gitignore`                                                    | 审查报告 + 验证命令        |

进阶功能新增 34 个测试用例（必做阶段是 6 个文件、80 个用例），现在全部通过：

```
$ cd server && npm test
 Test Files  7 passed (7)
      Tests  114 passed (114)
```

另外做过变异检查：在实现里故意改坏 7 处关键逻辑，每一处都会让测试失败，说明这些测试能抓到真实的错误。

下面两张截图是在后端离线模式下截的：资金用水龙头发放，做市参考价固定为 7.5（`MM_PRICE_SOURCE=fixed`），每 1 秒调整一次（`MM_INTERVAL_MS=1000`），方便复现。离线和在线模式只是入金方式不同，下单、撮合、推送、做市用的是同一份代码。

### 1. 做市机器人：买卖两侧各挂 3 档（10 分）

**开启**：在 `server/.env` 里设 `MARKET_MAKER=1`，重启后端。默认关闭，其余参数见 `server/.env.example` 里的 `MM_*`。

**报价**：参考价取 Binance AVAXUSDT 的最新成交价。取不到时用上一次取到的价格；一次都没取到过，就用 `MM_BASE_PRICE`（默认 7.5）。默认参数下，参考价为 7.5 时挂出：

| 档位 | 离参考价 | 买价     | 卖价     | 数量（每边）   |
| -- | ---- | ------ | ------ | -------- |
| 1  | 0.1% | 7.4925 | 7.5075 | 5 WAVAX  |
| 2  | 0.2% | 7.4850 | 7.5150 | 10 WAVAX |
| 3  | 0.3% | 7.4775 | 7.5225 | 15 WAVAX |

**调整**：默认每 3 秒一轮，只做增量调整：

- 参考价偏离当前报价超过 0.05% 才整体重挂，避免价格每跳一下就撤单重挂
- 某一档被吃得只剩不到一半，就撤掉重新挂满；其余档位不动
- 先撤后挂，机器人自己的买单和卖单不会互相成交
- 余额不够挂某一档就跳过，不报错也不中断
- 一轮里的所有变化合并成一次 WebSocket 推送

**测试**：新文件 `server/src/marketmaker.test.ts`，15 个用例，覆盖以下情况：

- 3 + 3 档的价格和数量
- 价格不变时，第二轮什么都不动
- 价格变动超过阈值时整体重挂
- 被吃掉后补档
- 取价失败时的兜底
- 余额不够时跳过
- 和两个真人随机交易 300 步后，账本守恒，冻结额与挂单一致

**演示**：未登录的访客打开页面，就能看到机器人挂出的 3 + 3 档。

![做市机器人：买卖两侧各 3 档](images/09-mm-orderbook.png)

**风险说明**：做市账户是一个内部地址（默认 `0x…aa`），没有私钥，不能登录，也不能提现。启动时只在账本里给它虚拟注资，没有链上资金对应。用户和它成交得到的币，提现时实际是从 Vault 里其他用户的充值中支出。所以它默认关闭，只用于演示；生产环境的做市账户必须使用真实充值。这一条也记入了下面安全审查报告的第 5 项。

### 2. 支持 IOC / FOK 订单（8 分）

下单时可以选有效方式（`timeInForce`）。它只用于限价单，默认是 GTC：

| 有效方式    | 能立即成交的部分  | 剩余部分                  |
| ------- | --------- | --------------------- |
| GTC（默认） | 成交        | 挂在订单簿上                |
| IOC     | 成交        | 撤销，冻结的钱退回             |
| FOK     | 能全部成交时才成交 | 不能全部立即成交就整单撤销，一笔都不成交  |

**实现**：

- 撮合引擎（`server/src/engine/OrderBook.ts`）：
  - FOK 先只读地演练一遍，算出限价以内能立即成交多少。不够就直接返回，订单簿不动
  - IOC 正常撮合，剩余部分不挂单
  - 自成交检查排在前面，照旧整单拒绝
- 交易服务（`server/src/exchange.ts`）：
  - 冻结规则和普通限价单一样
  - 剩余部分被撤销时解冻：买单解冻"限价 × 剩余数量"，卖单解冻剩余数量
  - IOC、FOK 都返回 201，订单状态是 `filled` 或 `cancelled`，靠已成交数量区分"部分成交"和"一笔没成交"
  - 市价单带 `timeInForce` 返回 400
- 前端（`web/src/OrderForm.tsx`）：限价单可以选 GTC / IOC / FOK。历史委托的状态会写明结果，例如"部分成交，剩余撤销""未能全部成交，整单撤销"

**测试**：新增 14 个用例：

- 撮合引擎 9 个（`OrderBook.test.ts`「OrderBook IOC / FOK」）
- 交易服务 4 个（`exchange.test.ts`「Exchange IOC / FOK」）
- 接口 1 个（`routes.test.ts`）

原有的随机守恒测试也混入了 IOC / FOK：随机下单、撤单 600 步，每一步检查每种资产的总额不变、冻结额等于挂单应冻结的金额，并断言 IOC / FOK 订单从不留在订单簿上。

**演示**：对着上面做市机器人的卖盘（卖一 7.5075 × 5，卖二 7.5150 × 10），用同一个账户连下三单：

| 下单               | 结果                                                                              |
| ---------------- | ------------------------------------------------------------------------------- |
| IOC 限价买 7.51 × 8 | 吃掉卖一的 5 个，均价 7.5075；剩余 3 个撤销，冻结退回。USDC 余额 1000 → 962.4625（= 1000 − 5 × 7.5075） |
| FOK 限价买 7.51 × 8 | 7.51 以内只有 5 个，不够 8 个，整单撤销，一笔不成交，余额不变                                            |
| FOK 限价买 7.52 × 8 | 卖一 5 个 + 卖二 3 个，全部成交，均价 7.5103                                                  |

第一单吃空卖一之后，做市机器人在 1 秒内把这一档补回了 5 个，所以第二、三单面对的还是同样的卖盘。

![IOC / FOK：历史委托里的三笔订单](images/10-ioc-fok.png)

### 3. WebSocket 私有 orders 频道（8 分）

登录后的 WebSocket 连接（`/ws?token=JWT`）除了公共的订单簿、成交和自己的余额，还会收到自己的订单。别的用户和匿名连接收不到。

```jsonc
// 连接建立时推一次：最近 100 条订单，新的在前
{ "type": "orders", "data": { "snapshot": true,  "orders": [ ... ] } }
// 之后订单一有变化就推：只含变化了的订单（最新的完整状态）
{ "type": "orders", "data": { "snapshot": false, "orders": [ ... ] } }
```

**什么时候推**：

- 下单
- 成交：吃单方和被吃到的挂单方都推
- 撤单
- IOC / FOK 的剩余被撤销
- 做市机器人调整挂单

**实现**：

- 交易服务的每个操作（下单、撤单、做市调整）都返回同一种"变化清单"，包括：这次的成交、余额变了的用户、变了的订单及其主人（`server/src/exchange.ts`）
- `server/src/app.ts` 里的 `publish` 统一负责推送：订单簿和成交广播给所有人；余额和订单按主人分组，只推给本人。HTTP 下单撤单和做市机器人都走它
- `server/src/ws.ts`：带有效 JWT 的连接按地址分组；token 无效直接拒绝连接（401）；不带 token 只能收公共行情
- 前端 `web/src/useMarket.ts`：收到快照直接替换，之后按订单编号合并。委托列表完全用推送来的数据，不再请求 `GET /orders`

**测试**：

- `routes.test.ts`「私有 orders 频道」：起真实的 WebSocket 连接，验证以下几点：
  - 连接时收到自己的订单快照
  - 下单、成交、撤单都推给订单主人
  - 另一个用户和匿名连接收不到
- `exchange.test.ts`「Exchange 变化清单（推送用）」4 个用例：验证推送依据的变化清单。例如一次成交里，吃单方和每个被吃到的挂单方都在清单里，主人正确；多次操作合并时，同一张订单只留最新状态

**演示**：Alice 挂买单 1 WAVAX @ 7.5（订单 #11），随后 Bob 卖出 0.4 WAVAX，吃掉其中一部分。Alice 什么都没做，她的连接收到了下面这条推送，委托列表里的已成交数量随之变成 0.40：

```json
{"type":"orders","data":{"snapshot":false,"orders":[{"id":"11","side":"buy","type":"limit","timeInForce":"GTC","price":"7.5","qty":"1","filledQty":"0.4","filledQuote":"3","status":"open","createdAt":1790448982417}]}}
```

同一时间，Bob 和匿名访客的连接都没有收到 Alice 的订单；前端全程没有请求过 `GET /orders`。

### 4. AI 安全审查报告 + 修复一个真实问题（8 分）

- **审查范围**：合约 `contracts/src/Vault.sol`、`MockERC20.sol`；后端 `server/src`（登录、链上监听、提现签名、账本、交易服务、撮合引擎、WebSocket、做市机器人）；前端 `web/src`；配置文件 `.env.example`、`.gitignore`
- **审查方式**：由 AI（Claude）逐文件阅读代码，沿着资金流检查：充值 → 入账 → 冻结 → 成交 → 提现签名 → 链上提现。重点看私钥与签名、重放、精度、账本一致性。修复方案经本人确认后实施

#### 发现汇总

| #   | 严重程度 | 问题                                                                                                                  | 位置                                 | 状态                         |
| --- | ---- | ------------------------------------------------------------------------------------------------------------------- | ---------------------------------- | -------------------------- |
| 1   | 严重   | 后端提现签名私钥写在 `server/.env.example` 里，而 `.gitignore` 只忽略 `.env`，第一次提交就会把私钥带进仓库。拿到这把私钥的人可以给自己签任意金额的提现票据，把 Vault 里的币全部取走 | `server/.env.example`              | ✅ 已修复                      |
| 2   | 高    | 账本只存在内存里，启动时也不回放链上的 Deposit 事件：后端一重启，所有人的交易所余额清零，币留在 Vault 里提不出来；后端停机期间的充值也永远不会入账                                   | `server/src/ledger.ts`、`chain.ts`  | 未修复（对应进阶项「数据持久化」）          |
| 3   | 中    | 提现签名没有落库、也没有对账：后端先扣账本再签名，用户拿到签名不上链，钱会一直扣着；也没有监听 Withdraw 事件标记完成或在过期后退回                                              | `server/src/routes.ts` `/withdraw` | 未修复（代码注释已标注）               |
| 4   | 中    | 下单、申请登录 nonce 都没有限流，订单和 nonce 都存在内存里，恶意刷请求可以把内存撑满                                                                   | `server/src/routes.ts`、`auth.ts`   | 未修复                        |
| 5   | 中    | 做市机器人只在账本里虚拟注资，没有链上资金对应：用户和它成交得到的币，提现时实际是从 Vault 里其他用户的充值中支出 | `server/src/marketmaker.ts`、`index.ts` | 设计取舍：仅演示用，默认关闭；生产环境做市账户必须用真实充值 |
| 6   | 低    | JWT 存在 localStorage，页面一旦出现 XSS 就能被读走；WebSocket 的 token 放在 URL 参数里，可能出现在代理日志中                                        | `web/src/api.ts`、`useMarket.ts`    | 未修复（教学简化）                  |
| 7   | 低    | Vault 的 owner 可以随时更换 signer，owner 私钥相当于 Vault 的总钥匙                                                                  | `Vault.sol` `setSigner`            | 设计取舍；生产应改用多签 + 时间锁         |
| 8   | —    | 充值入账不等确认数                                                                                                           | `server/src/chain.ts`              | 已评估：Avalanche 出块即最终确认，不需要修 |

做得对的地方（审查中确认过）：

- 提现签名把用户绑定为 `msg.sender`，别人拿到签名也提不走
- EIP-712 域包含 chainId 和合约地址，不能跨链、跨合约重放
- nonce 全局唯一，并有截止时间
- 先改状态再转账，并加了重入保护，用 SafeERC20 转账
- 登录 nonce 一次性使用，校验前先删除
- 后端启动时核对两个代币的小数位，防止地址填反导致金额错 10^12 倍
- 撮合开启了自成交防护
- 私有 orders 频道只推给订单主人；WebSocket 带无效 token 直接拒绝连接

#### 修复的真实问题：签名私钥差点被提交进仓库

**问题**：`server/.env.example` 里放的是真实配置，包括 `BACKEND_SIGNER_PRIVATE_KEY`（Vault 的提现签名私钥）。`server/.gitignore` 只有 `node_modules/` 和 `.env` 两行，所以 `.env.example` 会被提交。

**影响**：`Vault.withdraw` 只校验两件事：签名来自 signer，以及签名里的用户就是调用者。攻击者拿到 signer 私钥后，可以给自己的地址签一张金额等于 Vault 全部余额的票据，再调用 `withdraw`，把所有用户的 USDC 和 WAVAX 取走。合约不会去问后端账本。

**修复**（包含在仓库第一个提交 `d90683b` 里：私钥在提交之前就换成了占位符，所以从未进入 git 历史）：

```diff
- VAULT_ADDRESS=0x78f6…141A
- USDC_ADDRESS=0x98e8…e251
- WAVAX_ADDRESS=0x2388…b6f0
- # 后端签名钱包私钥，对应部署时的 SIGNER_ADDRESS。不要提交到 git
- BACKEND_SIGNER_PRIVATE_KEY=0x<真实私钥，此处打码>
+ # 部署脚本最后三行输出的地址
+ VAULT_ADDRESS=
+ USDC_ADDRESS=
+ WAVAX_ADDRESS=
+ # 后端签名钱包私钥，对应部署时的 SIGNER_ADDRESS。真实值只填在 .env 里，绝不能写进这个文件
+ BACKEND_SIGNER_PRIVATE_KEY=
```

（地址已缩写。）

同时在仓库根目录新增 `.gitignore`（`node_modules/`、`dist/`、`.env`），`web/` 目录也被覆盖到。真实配置只留在被忽略的 `server/.env` 里。

**验证**：

```powershell
git log --all -p -- server/.env.example | Select-String "PRIVATE_KEY=0x"   # 没有输出 = 私钥从未进入任何提交
git check-ignore -v server/.env                                              # 能看到命中的 .gitignore 规则
```

**后续建议**：

- 明文存放过的私钥，在生产环境应视为已泄露：用 owner 调用 `Vault.setSigner(新地址)` 换一把签名私钥，并把新私钥交给密钥管理服务
- 在提交前加密钥扫描（如 gitleaks），防止同类问题再次出现
