# Solana 应用开发文档库

本目录收录 Solana 链上程序开发相关的技术文档与学习资料。

---

## 文档列表

| 文件 | 说明 |
|------|------|
| [solana_smart_contract_guide.md](./solana_smart_contract_guide.md) | Solana 智能合约开发完全指南（入门 → 进阶） |
| [solana_senior_engineer_guide.md](./solana_senior_engineer_guide.md) | Solana 智能合约高级工程师指南（底层机制 / 安全 / 架构设计） |

---

## solana_smart_contract_guide.md 内容概览

一份面向 Rust 开发者的 Solana 合约开发完整参考手册，涵盖：

1. **核心概念** — 账户模型、Program、Instruction、PDA
2. **原生程序开发** — 最小程序结构、状态管理（计数器示例）
3. **Anchor 框架** — 完整合约示例、账户约束速查表
4. **跨程序调用（CPI）** — 普通 CPI / PDA 签名 CPI / Anchor CPI
5. **Token 操作** — SPL Token Mint、关联代币账户（ATA）
6. **测试** — litesvm Rust 单元测试 + Anchor TypeScript 测试
7. **部署流程** — 本地验证节点 / Devnet / Mainnet
8. **安全最佳实践** — 常见漏洞 + 审计工具推荐
9. **Compute Unit 优化** — zero_copy、批量借用等技巧
10. **参考资源** — 官方文档、工具链汇总

---

## 相关项目

- [First_Solana_Application_Hello_World](https://github.com/luhuimao/First_Solana_Application_Hello_World) — Hello World 链上程序示例（含 litesvm 测试 + RPC 客户端）
- [klend](https://github.com/luhuimao/klend) — Kamino Finance 借贷协议源码研究
