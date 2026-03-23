# Project Sniper

**High-Frequency Trading Agent for Polymarket Sports Arbitrage**

A Rust-based automated trading system that capitalizes on the time latency between live soccer match events and corresponding price updates on Polymarket's order book.

## Architecture

The system consists of three asynchronous modules:

```
┌─────────────────────────────────────────────────────────────────┐
│                      PROJECT SNIPER                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────────┐  │
│  │  THE ORACLE  │───▶│  THE BRAIN   │───▶│  THE EXECUTIONER │  │
│  │              │    │              │    │                  │  │
│  │ Sports Feed  │    │  Strategy    │    │   Polygon Tx     │  │
│  │  WebSocket   │    │   Engine     │    │    Manager       │  │
│  └──────────────┘    └──────────────┘    └──────────────────┘  │
│         │                   │                     │             │
│         ▼                   ▼                     ▼             │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                     REDIS STATE                           │  │
│  │           (Order Books, Positions, Mappings)              │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Module A: The Oracle (Data Ingestion)
- Real-time sports data via BetsAPI (recommended) or WebSocket providers
- Supports multiple providers: **BetsAPI** (easy onboarding), Sportradar, BetGenius, RunningBall
- Primary triggers: `PENALTY_AWARDED`, `RED_CARD`, `VAR_DECISION`
- BetsAPI polling with sub-second event detection

### Module B: The Brain (Strategy Engine)
- Automatic market mapping (Sports Match ID ↔ Polymarket Contract ID)
- Local order book state management
- Expected Value (EV) calculation
- Risk management and position sizing

### Module C: The Executioner (Transaction Manager)
- Direct Polymarket CLOB interaction via Polygon
- EIP-1559 aggressive gas pricing
- Mempool monitoring for front-run detection
- Auto-exit (take profit / stop loss)

## Quick Start

### Prerequisites

- Rust 1.75+ (install via [rustup](https://rustup.rs/))
- Redis 6.0+
- A Polygon wallet with USDC
- BetsAPI account (free tier available)

### Installation

```bash
# Clone the repository
git clone https://github.com/your-org/hft-agent.git
cd hft-agent

# Copy environment template
cp .env.example .env

# Edit .env with your configuration
# IMPORTANT: Add your PRIVATE_KEY and ORACLE_API_KEY
vim .env

# Build the project
cargo build --release
```

### Configuration

1. **Sports Data Provider** (BetsAPI - Recommended):
   - Sign up at [BetsAPI](https://betsapi.com/) - quick registration, no enterprise contracts
   - Get your API token from the dashboard
   - Set `ORACLE_API_KEY` in `.env`
   - Documentation: https://betsapi.com/docs/

   Alternative enterprise providers (require contracts):
   - [Sportradar](https://sportradar.com/)
   - [Genius Sports](https://geniussports.com/)

2. **Polygon Wallet**:
   - Create a wallet for trading
   - Fund with USDC on Polygon
   - Set `PRIVATE_KEY` in `.env`

3. **Redis**: Start Redis server:
   ```bash
   redis-server
   ```

4. **RPC Node**: For best performance, use a private RPC:
   - [Alchemy](https://www.alchemy.com/)
   - [QuickNode](https://www.quicknode.com/)

### Running

```bash
# Run the main trading agent
cargo run --release --bin sniper

# Run market monitor (discovery/debugging)
cargo run --release --bin market-monitor

# Run execution test (dry run)
cargo run --release --bin test-execution
```

## Trading Strategy

### Penalty Arbitrage Example

1. System monitors Real Madrid vs Barcelona
2. Current Polymarket price for Real Madrid win: $0.40
3. Oracle receives: `event: PENALTY_AWARDED, team: Real Madrid`
4. Brain calculates: Penalty increases win probability to ~70%
5. Brain checks order book: Offers at $0.41, $0.42, $0.45
6. Executioner buys all shares up to $0.55 (200ms elapsed)
7. Market catches up 10 seconds later, price jumps to $0.70
8. Executioner sells at $0.69 for profit

### Supported Events

| Event | Impact | Priority |
|-------|--------|----------|
| `PENALTY_AWARDED` | +20% probability shift | Critical |
| `RED_CARD` | +15% probability shift | Critical |
| `VAR_DECISION` | Variable | High |
| `GOAL` | +25% shift (market freeze risk) | High |

## Risk Management

### Built-in Protections

- **Max Slippage**: Configurable limit (default 5%)
- **Max Position Size**: Per-trade limit (default $1000)
- **Stop Loss**: Auto-exit on loss (default -10%)
- **Take Profit**: Auto-exit on profit (default +20%)
- **Gas Cap**: Maximum gas price (default 500 gwei)
- **Front-run Detection**: Mempool monitoring

### Key Risks

1. **Block Time Risk**: Polygon ~2.2s block time means transactions wait in mempool
2. **Slippage**: Low liquidity can cause significant price impact
3. **Market Freeze**: Markets may freeze on high-impact events
4. **API Rate Limits**: Excessive queries can trigger bans

## Development

### Project Structure

```
src/
├── main.rs              # Main orchestrator
├── lib.rs               # Library exports
├── config.rs            # Configuration management
├── error.rs             # Error types
├── types.rs             # Core data types
├── oracle/              # Sports data ingestion
│   ├── mod.rs
│   ├── feed.rs          # WebSocket connection
│   ├── parser.rs        # Event parsing
│   └── providers.rs     # Provider implementations
├── brain/               # Strategy engine
│   ├── mod.rs
│   ├── market.rs        # Polymarket client
│   ├── orderbook.rs     # Order book management
│   ├── strategy.rs      # Trading strategies
│   └── signals.rs       # Signal generation
├── executioner/         # Transaction management
│   ├── mod.rs
│   ├── tx_manager.rs    # Transaction handling
│   ├── gas.rs           # Gas optimization
│   ├── mempool.rs       # Front-run detection
│   ├── polymarket.rs    # Exchange integration
│   └── position.rs      # Position management
├── state/               # State management
│   ├── mod.rs
│   ├── cache.rs         # In-memory cache
│   └── persistence.rs   # Redis persistence
└── utils/               # Utilities
    ├── mod.rs
    ├── logging.rs
    ├── metrics.rs
    └── retry.rs
```

### Testing

```bash
# Run unit tests
cargo test

# Run with logging
RUST_LOG=debug cargo test

# Run specific test
cargo test test_penalty_strategy
```

### Benchmarking

```bash
# Run benchmarks
cargo bench
```

## Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `ORACLE_WEBSOCKET_URL` | BetsAPI base URL | `https://api.b365api.com/v3` |
| `ORACLE_API_KEY` | BetsAPI token | Required |
| `ORACLE_PROVIDER` | Provider type | `betsapi` |
| `POLYMARKET_API_URL` | Polymarket CLOB API | `https://clob.polymarket.com` |
| `POLYGON_RPC_URL` | Polygon RPC endpoint | `https://polygon-rpc.com` |
| `PRIVATE_KEY` | Wallet private key | Required |
| `REDIS_URL` | Redis connection URL | `redis://127.0.0.1:6379` |
| `MAX_POSITION_SIZE_USD` | Max trade size | `1000` |
| `MAX_SLIPPAGE_PERCENT` | Max allowed slippage | `5` |
| `LOG_LEVEL` | Logging level | `info` |

See `.env.example` for complete list.

## Deployment

### Server Recommendations

- **Location**: Co-locate near Polygon RPC nodes (US East, EU West)
- **Specs**: 4+ CPU cores, 8GB+ RAM
- **Network**: Low-latency connection to sports provider

### Production Checklist

- [ ] Use private RPC node (not public)
- [ ] Secure private key storage
- [ ] Set up monitoring/alerting
- [ ] Configure appropriate position limits
- [ ] Test with small amounts first

## License

MIT License - See [LICENSE](LICENSE) for details.

## Disclaimer

This software is for educational purposes. Trading involves significant risk. Past performance does not guarantee future results. Use at your own risk.
