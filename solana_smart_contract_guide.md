# Solana 智能合约开发完全指南

> 适用版本：Solana SDK 2.x / Anchor 0.29+  
> 最后更新：2026-03-13

---

## 一、Solana 核心概念

### 1.1 账户模型（Account Model）

Solana 中**一切皆账户**，与以太坊的合约存储模型截然不同：

| 概念 | 以太坊 | Solana |
|------|--------|--------|
| 代码 | 合约地址 | Program Account |
| 状态 | 合约内 Storage | 独立 Data Account |
| 所有者 | 无 | 每个账户都有 `owner` 字段 |
| 租金 | Gas | Rent（SOL 存押金）|

**账户结构**：

```rust
pub struct Account {
    pub lamports: u64,          // SOL 余额（最小单位 lamports）
    pub data: Vec<u8>,          // 账户数据（程序存储状态）
    pub owner: Pubkey,          // 该账户的所有者程序
    pub executable: bool,       // 是否是可执行程序
    pub rent_epoch: Epoch,      // 租金纪元（新版已废弃）
}
```

### 1.2 程序（Program）

Solana 智能合约称为 **Program**，特点：
- **无状态（Stateless）**：程序本身不存储状态，状态存在独立的 Data Account 中
- **并行执行**：Solana 使用 Sealevel 并行运行时，无数据依赖的交易可同时执行
- **可升级**：默认可升级（通过 upgrade authority），可设置不可变

### 1.3 指令（Instruction）

每笔交易由一个或多个 **Instruction** 组成：

```rust
pub struct Instruction {
    pub program_id: Pubkey,           // 调用哪个程序
    pub accounts: Vec<AccountMeta>,   // 涉及的账户列表
    pub data: Vec<u8>,                // 指令参数（序列化后的数据）
}

pub struct AccountMeta {
    pub pubkey: Pubkey,
    pub is_signer: bool,    // 是否需要签名
    pub is_writable: bool,  // 是否可写
}
```

### 1.4 PDA（Program Derived Address）

PDA 是由程序 ID + seeds 通过哈希派生的**链上账户地址**，无私钥，只有程序自身可签名：

```rust
let (pda, bump) = Pubkey::find_program_address(
    &[b"vault", user.key().as_ref()],
    &program_id,
);
```

**用途**：
- 程序托管账户（金库、锁仓）
- 用户专属状态账户
- 跨程序调用（CPI）的权限控制

---

## 二、原生程序开发（Native Program）

### 2.1 最小程序结构

```
hello_world/
├── src/lib.rs       # 程序入口
└── Cargo.toml
```

**Cargo.toml**：

```toml
[package]
name = "hello_world"
version = "0.1.0"
edition = "2021"

[dependencies]
solana-program = "2.2.0"

[lib]
crate-type = ["cdylib", "lib"]
```

**src/lib.rs**：

```rust
use solana_program::{
    account_info::AccountInfo,
    entrypoint,
    entrypoint::ProgramResult,
    msg,
    pubkey::Pubkey,
};

entrypoint!(process_instruction);

pub fn process_instruction(
    _program_id: &Pubkey,
    _accounts: &[AccountInfo],
    _instruction_data: &[u8],
) -> ProgramResult {
    msg!("Hello, world!");
    Ok(())
}
```

### 2.2 状态管理示例（计数器合约）

```rust
use borsh::{BorshDeserialize, BorshSerialize};
use solana_program::{
    account_info::{next_account_info, AccountInfo},
    entrypoint::ProgramResult,
    program_error::ProgramError,
    pubkey::Pubkey,
};

#[derive(BorshSerialize, BorshDeserialize)]
pub struct CounterAccount {
    pub count: u64,
}

pub fn process_increment(accounts: &[AccountInfo]) -> ProgramResult {
    let account_info_iter = &mut accounts.iter();
    let counter_account = next_account_info(account_info_iter)?;

    // 反序列化
    let mut counter = CounterAccount::try_from_slice(&counter_account.data.borrow())?;
    counter.count += 1;

    // 序列化写回
    counter.serialize(&mut *counter_account.data.borrow_mut())?;
    Ok(())
}
```

---

## 三、Anchor 框架开发

### 3.1 什么是 Anchor

Anchor 是 Solana 上最流行的开发框架，提供：
- **宏自动生成**样板代码（账户校验、序列化、错误处理）
- **IDL（Interface Definition Language）**：自动生成客户端 TypeScript SDK
- **账户约束系统**：`#[account(init, seeds=..., bump, payer=...)]`

### 3.2 Anchor 项目结构

```
my_program/
├── programs/my_program/
│   └── src/lib.rs        # 程序逻辑
├── tests/
│   └── my_program.ts     # TypeScript 测试
├── Anchor.toml           # 项目配置
└── Cargo.toml
```

### 3.3 Anchor 合约示例

```rust
use anchor_lang::prelude::*;

declare_id!("Fg6PaFpoGXkYsidMpWxTWqzMgNR9g1MxaKXB4GFxhB5");

#[program]
mod my_program {
    use super::*;

    pub fn initialize(ctx: Context<Initialize>, initial_count: u64) -> Result<()> {
        let counter = &mut ctx.accounts.counter;
        counter.authority = ctx.accounts.user.key();
        counter.count = initial_count;
        Ok(())
    }

    pub fn increment(ctx: Context<Increment>) -> Result<()> {
        let counter = &mut ctx.accounts.counter;
        counter.count += 1;
        msg!("Counter: {}", counter.count);
        Ok(())
    }
}

// ====== 账户结构定义 ======

#[derive(Accounts)]
pub struct Initialize<'info> {
    // init: 创建并初始化账户
    // payer: 由 user 支付租金
    // space: 账户大小（8 字节判别器 + 数据）
    #[account(
        init,
        payer = user,
        space = 8 + CounterState::INIT_SPACE,
        seeds = [b"counter", user.key().as_ref()],
        bump
    )]
    pub counter: Account<'info, CounterState>,

    #[account(mut)]
    pub user: Signer<'info>,
    pub system_program: Program<'info, System>,
}

#[derive(Accounts)]
pub struct Increment<'info> {
    #[account(
        mut,
        seeds = [b"counter", authority.key().as_ref()],
        bump,
        has_one = authority  // 验证 authority 字段匹配
    )]
    pub counter: Account<'info, CounterState>,
    pub authority: Signer<'info>,
}

// ====== 状态账户数据结构 ======

#[account]
#[derive(InitSpace)]
pub struct CounterState {
    pub authority: Pubkey, // 32 bytes
    pub count: u64,        // 8 bytes
}

// ====== 自定义错误码 ======

#[error_code]
pub enum CounterError {
    #[msg("Overflow detected")]
    Overflow,
    #[msg("Unauthorized")]
    Unauthorized,
}
```

### 3.4 常用账户约束

| 约束 | 说明 |
|------|------|
| `#[account(init, ...)]` | 创建并初始化新账户 |
| `#[account(mut)]` | 账户可写 |
| `#[account(signer)]` | 账户必须签名 |
| `#[account(seeds=[...], bump)]` | PDA 账户验证 |
| `#[account(has_one = field)]` | 验证账户字段与传入账户一致 |
| `#[account(constraint = expr)]` | 自定义约束表达式 |
| `#[account(close = dest)]` | 关闭账户并归还租金 |
| `#[account(owner = program_id)]` | 验证账户所有者 |
| `#[account(address = pubkey)]` | 验证账户地址 |

---

## 四、跨程序调用（CPI）

### 4.1 普通 CPI

```rust
use anchor_lang::solana_program::program::invoke;

// 调用系统程序转账
invoke(
    &system_instruction::transfer(&from.key(), &to.key(), lamports),
    &[from.to_account_info(), to.to_account_info()],
)?;
```

### 4.2 带 PDA 签名的 CPI

```rust
use anchor_lang::solana_program::program::invoke_signed;

let seeds = &[b"vault", user.key.as_ref(), &[bump]];
let signer_seeds = &[&seeds[..]];

invoke_signed(
    &system_instruction::transfer(&vault.key(), &recipient.key(), amount),
    &[vault.to_account_info(), recipient.to_account_info()],
    signer_seeds,
)?;
```

### 4.3 Anchor CPI 写法

```rust
// 调用其他 Anchor 程序
token::transfer(
    CpiContext::new_with_signer(
        ctx.accounts.token_program.to_account_info(),
        token::Transfer {
            from: ctx.accounts.vault.to_account_info(),
            to: ctx.accounts.user_token.to_account_info(),
            authority: ctx.accounts.vault_authority.to_account_info(),
        },
        &[&[b"vault", &[bump]]],
    ),
    amount,
)?;
```

---

## 五、Token 操作（SPL Token）

### 5.1 创建 Token Mint

```rust
#[derive(Accounts)]
pub struct CreateMint<'info> {
    #[account(
        init,
        payer = payer,
        mint::decimals = 6,
        mint::authority = payer,
    )]
    pub mint: Account<'info, Mint>,
    #[account(mut)]
    pub payer: Signer<'info>,
    pub system_program: Program<'info, System>,
    pub token_program: Program<'info, Token>,
}
```

### 5.2 创建 Token Account（ATA）

```rust
#[account(
    init_if_needed,
    payer = payer,
    associated_token::mint = mint,
    associated_token::authority = user,
)]
pub user_token_account: Account<'info, TokenAccount>,
```

---

## 六、测试

### 6.1 litesvm（Rust 单元测试）

```rust
#[cfg(test)]
mod tests {
    use litesvm::LiteSVM;
    use solana_sdk::{
        instruction::Instruction,
        message::Message,
        signature::{Keypair, Signer},
        transaction::Transaction,
    };

    #[test]
    fn test_program() {
        let mut svm = LiteSVM::new();
        let payer = Keypair::new();
        svm.airdrop(&payer.pubkey(), 1_000_000_000).unwrap();

        // 加载程序
        let program_kp = Keypair::new();
        svm.add_program_from_file(program_kp.pubkey(), "target/deploy/my_program.so")
           .unwrap();

        let ix = Instruction {
            program_id: program_kp.pubkey(),
            accounts: vec![],
            data: vec![],
        };
        let msg = Message::new(&[ix], Some(&payer.pubkey()));
        let tx = Transaction::new(&[&payer], msg, svm.latest_blockhash());
        let result = svm.send_transaction(tx);
        assert!(result.is_ok());
    }
}
```

### 6.2 Anchor TypeScript 测试

```typescript
import * as anchor from "@coral-xyz/anchor";
import { Program } from "@coral-xyz/anchor";
import { MyProgram } from "../target/types/my_program";
import { expect } from "chai";

describe("my_program", () => {
  const provider = anchor.AnchorProvider.env();
  anchor.setProvider(provider);
  const program = anchor.workspace.MyProgram as Program<MyProgram>;

  it("Initializes counter", async () => {
    const [counterPda] = anchor.web3.PublicKey.findProgramAddressSync(
      [Buffer.from("counter"), provider.wallet.publicKey.toBuffer()],
      program.programId
    );

    await program.methods
      .initialize(new anchor.BN(0))
      .accounts({ counter: counterPda })
      .rpc();

    const counter = await program.account.counterState.fetch(counterPda);
    expect(counter.count.toNumber()).to.equal(0);
  });
});
```

---

## 七、部署流程

### 7.1 本地开发网

```bash
# 启动本地验证节点
solana-test-validator

# 切换到本地网络
solana config set -ul

# Airdrop SOL
solana airdrop 2

# 编译（SBF 格式）
cargo build-sbf

# 部署
solana program deploy ./target/deploy/my_program.so
```

### 7.2 Devnet / Mainnet

```bash
# 切换到 Devnet
solana config set -ud

# 部署到 Devnet（需要 SOL）
solana program deploy ./target/deploy/my_program.so \
  --keypair ~/.config/solana/devnet-keypair.json

# 查看程序信息
solana program show <PROGRAM_ID>

# 程序升级
solana program upgrade ./target/deploy/my_program.so <PROGRAM_ID>

# 关闭升级权限（设为不可变）
solana program set-upgrade-authority <PROGRAM_ID> --final
```

### 7.3 Anchor 部署

```bash
# 构建
anchor build

# 部署
anchor deploy

# 一键测试（本地）
anchor test
```

---

## 八、安全最佳实践

### 8.1 账户验证必查清单

- ✅ 验证 `owner`：账户所有者必须是预期的程序
- ✅ 验证 `is_signer`：涉及权限操作的账户必须签名
- ✅ 验证 PDA seeds + bump：防止账户替换攻击
- ✅ 避免整型溢出：使用 `checked_add / checked_mul`
- ✅ 避免重入：Solana 架构天然防重入，但 CPI 顺序仍需注意

### 8.2 常见漏洞

| 漏洞类型 | 描述 | 防御 |
|----------|------|------|
| 账户替换攻击 | 传入伪造的同类型账户 | 验证 PDA seeds + owner |
| 缺少签名校验 | 任何人都能调用权限接口 | `has_one`、`constraint` |
| 整数溢出 | u64 加减乘出现溢出 | `checked_*` 系列方法 |
| 重复初始化 | 已初始化账户被再次初始化 | 检查 discriminator 或标志位 |
| 闪电贷攻击 | 同笔交易内借还操纵价格 | TWAP 预言机 + 指令内验证 |

### 8.3 推荐工具

| 工具 | 用途 |
|------|------|
| [Sec3 X-Ray](https://sec3.dev) | 自动漏洞扫描 |
| [Soteria](https://soteria.dev) | 静态分析 |
| [Trident](https://github.com/Ackee-Blockchain/trident) | Fuzz 测试框架 |
| [Anchor IDL Audit](https://docs.audit.anchor-lang.com) | IDL 合规性检查 |

---

## 九、gas（Compute Unit）优化

```rust
// 1. 在指令开头请求更多 CU（如需要）
use solana_program::compute_budget::ComputeBudgetInstruction;
let ix = ComputeBudgetInstruction::set_compute_unit_limit(400_000);

// 2. 使用 zero_copy 减少反序列化开销（大状态账户）
#[account(zero_copy)]
pub struct LargeState {
    pub data: [u64; 1000],
}

// 3. 避免在循环中反复借用 account.data
// 不推荐：
for i in 0..100 { let val = account.data.borrow()[i]; }
// 推荐：
let data = account.data.borrow();
for i in 0..100 { let val = data[i]; }
```

---

## 十、参考资源

| 资源 | 链接 |
|------|------|
| Solana 官方文档 | [docs.solana.com](https://docs.solana.com) |
| Anchor 文档 | [anchor-lang.com](https://www.anchor-lang.com) |
| Solana Cookbook | [solanacookbook.com](https://solanacookbook.com) |
| SolDev 教程 | [soldev.app](https://www.soldev.app) |
| litesvm | [github.com/LiteSVM/litesvm](https://github.com/LiteSVM/litesvm) |
| Helius RPC | [helius.dev](https://www.helius.dev) |
| Kamino Finance klend 源码 | [github.com/luhuimao/klend](https://github.com/luhuimao/klend) |
