
# Adrena Protocol - High-Level Overview

## Repository Structure

The Adrena repository is a comprehensive decentralized trading protocol built on Solana, implementing leveraged trading with staking and governance features.

### Core Components

#### **1. Programs Directory ([/programs/adrena/](cci:7://file:///Users/aamir/Desktop/work/adrena/programs/adrena:0:0-0:0))**
The main smart contract implementation written in Rust using the Anchor framework:

- **[lib.rs](cci:7://file:///Users/aamir/Desktop/work/adrena/programs/adrena/src/lib.rs:0:0-0:0)**: Main program entry point with 100+ instruction handlers
- **[state/](cci:7://file:///Users/aamir/Desktop/work/adrena/programs/adrena/src/state:0:0-0:0)**: Core data structures and account management
  - [cortex.rs](cci:7://file:///Users/aamir/Desktop/work/adrena/programs/adrena/src/state/cortex.rs:0:0-0:0): Central protocol logic and constants
  - [pool.rs](cci:7://file:///Users/aamir/Desktop/work/adrena/programs/adrena/src/state/pool.rs:0:0-0:0): Liquidity pool management
  - [custody.rs](cci:7://file:///Users/aamir/Desktop/work/adrena/programs/adrena/src/state/custody.rs:0:0-0:0): Asset custody and token management
  - [position.rs](cci:7://file:///Users/aamir/Desktop/work/adrena/programs/adrena/src/state/position.rs:0:0-0:0): Trading position handling
  - [staking.rs](cci:7://file:///Users/aamir/Desktop/work/adrena/programs/adrena/src/state/staking.rs:0:0-0:0): Staking system logic
  - [user_staking.rs](cci:7://file:///Users/aamir/Desktop/work/adrena/programs/adrena/src/state/user_staking.rs:0:0-0:0): User-specific staking data
  - [user_profile.rs](cci:7://file:///Users/aamir/Desktop/work/adrena/programs/adrena/src/state/user_profile.rs:0:0-0:0): User profile and achievement system
  - [oracle.rs](cci:7://file:///Users/aamir/Desktop/work/adrena/programs/adrena/src/state/oracle.rs:0:0-0:0): Price feed integration
- **[instructions/](cci:7://file:///Users/aamir/Desktop/work/adrena/programs/adrena/src/instructions:0:0-0:0)**: Protocol operations split into:
  - [admin/](cci:7://file:///Users/aamir/Desktop/work/adrena/programs/adrena/src/instructions/admin:0:0-0:0): Administrative functions (47 files)
  - [public/](cci:7://file:///Users/aamir/Desktop/work/adrena/programs/adrena/src/instructions/public:0:0-0:0): User-facing operations (73 files)
- **[adapters/](cci:7://file:///Users/aamir/Desktop/work/adrena/programs/adrena/src/adapters:0:0-0:0)**: External protocol integrations (SPL Governance, Token Metadata)

#### **2. CLI Tools ([/cli/](cci:7://file:///Users/aamir/Desktop/work/adrena/cli:0:0-0:0))**
TypeScript-based command-line interface for protocol management:

- **[cli.ts](cci:7://file:///Users/aamir/Desktop/work/adrena/cli/src/cli.ts:0:0-0:0)**: Main CLI with 50+ commands for deployment and management
- **[client.ts](cci:7://file:///Users/aamir/Desktop/work/adrena/cli/src/client.ts:0:0-0:0)**: Core client library for interacting with the protocol
- **[types.ts](cci:7://file:///Users/aamir/Desktop/work/adrena/cli/src/types.ts:0:0-0:0)**: TypeScript type definitions
- Comprehensive deployment guides for devnet/mainnet

#### **3. Configuration & Build**
- **[Anchor.toml](cci:7://file:///Users/aamir/Desktop/work/adrena/Anchor.toml:0:0-0:0)**: Anchor framework configuration
- **[Cargo.toml](cci:7://file:///Users/aamir/Desktop/work/adrena/Cargo.toml:0:0-0:0)**: Rust workspace setup with patched dependencies
- **[rust-toolchain.toml](cci:7://file:///Users/aamir/Desktop/work/adrena/rust-toolchain.toml:0:0-0:0)**: Toolchain specifications
- Custom Solana fork for compatibility patches

## Key Protocol Features

### **Trading System**
- **Leverage Trading**: Long/short positions with up to 10x leverage
- **Multiple Assets**: Support for various tokenized assets
- **Limit Orders**: Advanced order book functionality
- **Liquidations**: Automated position liquidation system
- **Fee Management**: Dynamic fee calculation and distribution

### **Staking & Governance**
- **Dual Staking Types**: 
  - LM (Liquidity Mining) tokens with voting rights
  - LP (Liquidity Provider) tokens
- **Lock Periods**: 0-540 days with multiplier rewards (1x-4x)
- **Early Exit**: Penalty system with 5%-40% fees
- **Governance**: SPL Governance integration for protocol decisions

### **Liquidity Management**
- **Multi-Asset Pools**: Support for diverse asset custody
- **Dynamic AUM**: Asset Under Management tracking
- **Oracle Integration**: Real-time price feeds from multiple sources
- **Swap Functionality**: Built-in DEX capabilities

### **User Features**
- **Profile System**: Achievement tracking and referral system
- **Vesting**: Token vesting schedules for team/investors
- **Rewards**: Multi-tier reward distribution
- **Genesis Programs**: Special early adopter benefits

## Technical Architecture

### **Smart Contract Design**
- **Zero-Copy Accounts**: Optimized for Solana's memory model
- **PDAs (Program Derived Addresses)**: Efficient account derivation
- **Security Audits**: Audited by Offside Labs and OtterSec
- **Comprehensive Testing**: Extensive test suite with integration tests

### **Build System**
- **Anchor Framework**: v0.29.0 for Solana development
- **Custom Solana Fork**: Patched for compatibility
- **Optimized Builds**: LTO and fat binary optimization
- **CI/CD Ready**: Pre-commit hooks and formatting standards

### **Development Tools**
- **TypeScript SDK**: Full client library for frontend integration
- **CLI Tools**: Complete deployment and management suite
- **Local Development**: Localnet testing with Solana test validator
- **Documentation**: Comprehensive guides and API documentation

## Protocol Economy

The protocol implements a sophisticated economic model with:
- **Trading Fees**: Revenue from position management and swaps
- **Staking Rewards**: Incentive alignment through token emissions
- **Governance Power**: Voting rights based on staked LM tokens
- **Referral System**: User acquisition incentives
- **Achievement System**: Gamification elements for engagement

This architecture enables Adrena to function as a complete decentralized derivatives exchange with sophisticated risk management, user incentives, and governance mechanisms.

The comprehensive overview of the Adrena repository has been completed. The analysis covered all major components including the smart contract programs, CLI tools, configuration files, and overall protocol architecture. The repository implements a sophisticated decentralized trading protocol on Solana with leveraged trading, staking, governance, and liquidity management features.
