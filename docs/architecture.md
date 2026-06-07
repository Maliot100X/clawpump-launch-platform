# ClawPump Launch Platform — Full Architecture

## 1. Research Summary

### Virtuals Protocol (How They Do It)
- Quote Currency: $VIRTUAL (not ETH)
- Bonding Curve: AMM paired with $VIRTUAL liquidity
- Graduation: 42,000 $VIRTUAL threshold
- Auto-migration to Uniswap V2
- Trading Fee: 1% (70% creator, 30% treasury)
- LP Lock: 10 years
- Anti-Sniper: Dynamic buy-tax 99% → 1%

### pump.fun (Solana Baseline)
- Quote Currency: SOL
- Bonding Curve: Constant Product AMM (x*y=k)
- Graduation: 85 SOL → Raydium
- Trading Fee: 1% (100% creator during curve)
- Initial Supply: 1 billion tokens
- Curve Allocation: 793.1M tokens

### ClawPump (Current Platform)
- Wraps pump.fun API
- Adds agent automation
- 65% creator fee
- Gasless option (empty)
- Self-funded option

## 2. Our Platform Design

### Core Concept
Launch tokens where:
- Quote currency = ClawPump tokens (not SOL)
- Hold ClawPump to launch
- Trade with ClawPump for fee discounts
- Hold 1M+ for APY allocation

### Token Requirements
- Hold X ClawPump tokens → Launch access
- Hold Y ClawPump tokens → Fee discounts
- Hold Z ClawPump tokens → APY allocation

### Bonding Curve
- Uses ClawPump tokens as quote
- Constant Product AMM (x*y=k)
- Graduation threshold in ClawPump tokens
- Migrates to Raydium

## 3. Technical Architecture

### Smart Contract (Solana/Anchor)
```rust
// Custom bonding curve program
pub fn initialize_pool(
    ctx: Context<InitializePool>,
    new_token_mint: Pubkey,
    quote_token_mint: Pubkey, // ClawPump token
    initial_price: u64,
    graduation_threshold: u64,
) -> Result<()>
```

### Key Components
1. Pool initialization with ClawPump as quote
2. Buy/sell logic using ClawPump tokens
3. Graduation mechanism to Raydium
4. Anti-sniper protection
5. Fee distribution

### Integration Points
- ClawPump API for agent launches
- Solana for on-chain transactions
- Raydium for DEX migration

## 4. Revenue Model
- Platform fee: 1-2% of trading volume
- Graduation fees: Fixed per migration
- Premium features: Subscription

## 5. Timeline
- Smart contract: 2-4 weeks
- Frontend: 2-3 weeks
- AI integration: 1-2 weeks
- Testing: 1-2 weeks
- Total: 6-10 weeks
