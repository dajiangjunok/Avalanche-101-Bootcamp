# dajiangjunok · Avalanche Bootcamp

| 作业 | 提交内容 / 状态 |
| --- | --- |
| [Task1](task1.md) | 已附注册与分享截图 |
| [Task2](task2.md) | ERC-20、Scaffold-ETH、Fuji 地址与截图 |
| [Task3](task3.md) | Fuji 交易对、流动性、DEX 定价购买 |
| [Task4](task4.md) | DeFi 研究回答 |
| [Task5](task5.md) | RWA 业务、合约、测试、Fuji 交互 |
| [Task6](task6.md) | 尚待本人 Kite 注册、交互及释放资金截图 |
| [Task7](task7.md) | 独立 Mini-DEX 已测试及部署；待核对课程仓库 |

## 目录

- `task1.md`–`task7.md`：精简提交文档。
- `public/`：截图、测试日志、[Fuji 回执](public/evidence/fuji/deployment.json)与[提交索引](public/submission.json)。
- `projects/avalanche-lab/`：所有项目、合约、测试及运行脚本。

## 运行

```bash
cd projects/avalanche-lab
npm ci
npm run compile
npm start
```

访问 <http://127.0.0.1:4173>，连接 Fuji 钱包并导入 [wallet-import.json](public/evidence/fuji/wallet-import.json) 可查看已部署合约。[详细项目说明](projects/avalanche-lab/README.md)。

本次 Fuji 交易全部确认，手续费共 **0.0000084348082752 测试 AVAX**；为第二测试钱包拨入 **0.005 测试 AVAX**。部署密钥在项目 `.env.local`，第二测试钱包密钥在 `.env.test-wallet`，两者均已 Git 忽略，未进入公开材料。学员档案和奖励钱包保持原样；尚未提交 PR。
