# Cosmos SDK Quick Start: Practical Dependency Map

A beginner's guide to understanding how all the pieces connect and work together.

---

## 🗺️ The Big Picture: From User Command to Running Chain

```
User types:  minid start
      ↓
main.go executes
      ↓
root.go creates CLI commands
      ↓
commands.go registers "start" and other commands
      ↓
"start" calls newApp()
      ↓
app.go initializes MiniApp
      ↓
app.yaml defines module order
      ↓
Blockchain runs!
```

---

## 📚 Reading Order: How to Understand the Code

### **Level 1: Entry Points (START HERE)**

#### 1.1 `cmd/minid/main.go` (5 lines of logic)

**What it does:** Starts everything

```go
func main() {
    params.SetAddressPrefixes()  // Set prefix to "mini"
    rootCmd := cmd.NewRootCmd()  // Create CLI
    svrcmd.Execute(rootCmd, ...)  // Run CLI
}
```

**What to understand:**

-   Entry point of entire application
-   Calls `SetAddressPrefixes()` from config
-   Creates root CLI command from `root.go`

**Questions to ask:**

-   What does `SetAddressPrefixes()` do? → Go to `config.go`
-   Where does `NewRootCmd()` come from? → Go to `root.go`

---

#### 1.2 `cmd/minid/root.go` (The CLI Builder)

**What it does:** Creates the command tree

```
NewRootCmd()
    ↓
    ├─ depinject.Inject() → Loads app.yaml, creates modules
    ├─ Creates rootCmd (minid)
    └─ initRootCmd() → Registers all subcommands
         ├─ start, query, tx, keys, etc.
```

**Key line:**

```go
autoCliOpts.EnhanceRootCommand(rootCmd)
```

This auto-generates query/tx commands from app.yaml!

**What to understand:**

-   `depinject.Inject()` uses app.yaml to initialize the app
-   `PersistentPreRunE` runs BEFORE every command (load config, set gas price, etc.)
-   Default gas price set to "0mini" (no fees required)
-   CometBFT timeout set to 3 seconds

**Key config here:**

```go
srvCfg.MinGasPrices = "0mini"  // ← Anyone can send tx without fees
cmtCfg.Consensus.TimeoutCommit = 3 * time.Second  // ← Blocks every 3 sec
```

---

#### 1.3 `cmd/minid/commands.go` (The Command Registry)

**What it does:** Registers all CLI commands

```
initRootCmd()
    ├─ genutilcli.InitCmd()      → minid init
    ├─ genutilcli.Commands()     → minid genesis, gentx
    ├─ queryCommand()            → minid query
    ├─ txCommand()               → minid tx
    ├─ keys.Commands()           → minid keys
    └─ server.AddCommands()      → minid start, export, etc.
```

**Two important functions here:**

**a) `newApp()` - Creates the blockchain application**

```go
func newApp(logger, db, ...) -> MiniApp {
    app.NewMiniApp(...)  // ← Calls app.go
}
```

Called by `minid start` to initialize blockchain

**b) `appExport()` - Exports blockchain state**

```go
func appExport(...) -> ExportedApp {
    app.NewMiniApp(...)
    miniApp.ExportAppStateAndValidators(...)  // ← Calls export.go
}
```

Called to backup chain state (for upgrades)

---

### **Level 2: Configuration & Setup**

#### 2.1 `app/params/config.go` (Address & Token Setup)

**What it does:** Global configuration for the chain

```go
const (
    CoinUnit = "mini"              // Token name
    Bech32PrefixAccAddr = "mini"   // Address prefix
)

func SetAddressPrefixes() {
    // Set "mini1..." addresses
    // Set "minivaloper1..." validator addresses
    // Set "minivalcons1..." consensus node addresses
    // Set address validator function (20 or 32 bytes only)
}

func RegisterDenoms() {
    // Register "mini" as token denomination
}
```

**Key concept - Address Types:**

| Type               | Prefix                           | Example              | Used For                |
| ------------------ | -------------------------------- | -------------------- | ----------------------- |
| Account            | `mini`                           | `mini1abc...`        | Regular users           |
| Validator Operator | `minivaloper`                    | `minivaloper1abc...` | Validator admin address |
| Consensus Node     | `minivalcons`                    | `minivalcons1abc...` | Blockchain validator    |
| Public Keys        | `minipub`, `minivaloperpub`, etc | -                    | Key management          |

**Address validation rule:**

-   Must be 20 or 32 bytes (no other sizes allowed)
-   Prevents typos creating invalid addresses

---

#### 2.2 `app/params/encoding.go` (Codec Configuration)

**What it does:** Defines how data is serialized

```go
type EncodingConfig struct {
    InterfaceRegistry  // Register message types
    Codec              // Protobuf encoder/decoder
    TxConfig           // How to encode transactions
    Amino              // Legacy amino codec
}
```

**Simply put:** This defines "how to convert Go objects ↔ bytes"

---

### **Level 3: Application Core**

#### 3.1 `app/app.go` (The Blockchain State Machine)

**We already covered this deeply, but reminder:**

```go
type MiniApp struct {
    *runtime.App  // Base framework

    // Keepers (state managers)
    AccountKeeper
    BankKeeper
    StakingKeeper
    DistrKeeper
    ConsensusParamsKeeper
}

func NewMiniApp(...) -> *MiniApp {
    // Step 1: depinject.Inject() loads app.yaml
    // Step 2: Auto-creates all Keepers
    // Step 3: Build the app with database
    // Step 4: Load latest state (or create new)
}
```

**This is where everything comes together.**

---

#### 3.2 `app.yaml` (The Blueprint)

**Already covered, but quick reference:**

```yaml
modules:
    - name: runtime
      begin_blockers: [distribution, staking] # Run at block start
      end_blockers: [staking] # Run at block end
      init_genesis: [auth, bank, distribution, staking, genutil] # Init order

    - name: auth
      config:
          bech32_prefix: mini # ← Comes from config.go

    - name: bank
      blocked_module_accounts_override: [...] # Can't send to these
```

**The ORDER IS CRITICAL:**

-   `init_genesis` order ensures accounts exist before tokens, tokens before staking
-   `begin_blockers` order ensures rewards calculated before validator changes
-   Reversing breaks everything!

---

#### 3.3 `app/export.go` (Export Chain State)

**What it does:** Creates a backup of chain state

```go
func (app *MiniApp) ExportAppStateAndValidators(...) {
    // Get current state
    // Convert to JSON (genesis format)
    // Export validator set
    // Return for backup/upgrade
}

func (app *MiniApp) prepForZeroHeightGenesis(...) {
    // Special case: Export to height 0
    // Withdraw all rewards
    // Jail unauthorized validators
    // Reset all heights to 0
}
```

**When is this used?**

-   Backing up the chain
-   Upgrading to a new version
-   Testing with exported state

---

#### 3.4 `scripts/init.sh` (Initialize Chain)

**What it does:** First-time setup

```bash
# 1. Remove old chain data
rm -rf $HOME/.minid

# 2. Set configuration
minid config set client chain-id demo
minid config set client keyring-backend test

# 3. Create test accounts
minid keys add alice
minid keys add bob

# 4. Create genesis
minid init test --chain-id demo

# 5. Add accounts to genesis
minid genesis add-genesis-account alice 10000000mini
minid genesis add-genesis-account bob 1000mini

# 6. Create validator
minid genesis gentx alice 1000000mini --chain-id demo

# 7. Finalize genesis
minid genesis collect-gentxs
```

**What each step does:**

| Step | Command                             | Result                                            |
| ---- | ----------------------------------- | ------------------------------------------------- |
| 3    | `minid keys add alice`              | Creates key pair, saves in keyring-backend test   |
| 4    | `minid init test`                   | Creates ~/.minid directory, genesis.json template |
| 5    | `minid genesis add-genesis-account` | Adds alice with 10M mini tokens in genesis        |
| 6    | `minid genesis gentx alice`         | alice creates validator tx, bonds 1M mini         |
| 7    | `minid genesis collect-gentxs`      | Collects all gentx, creates final genesis         |

---

## 🔄 Complete Execution Flow

### **When you run: `make install && minid start`**

```
1. Makefile (make install)
   ├─ go mod verify              # Check dependencies
   └─ go install ./cmd/minid     # Compile binary
                                 Result: minid binary installed

2. Shell (minid start)
   └─ main.go executes
       ├─ params.SetAddressPrefixes()
       │  └─ Sets bech32_prefix = "mini"
       ├─ cmd.NewRootCmd()
       │  ├─ root.go NewRootCmd()
       │  │  ├─ depinject.Inject(app.AppConfig())
       │  │  │  └─ Loads app.yaml
       │  │  └─ initRootCmd()
       │  │     └─ commands.go registers all commands
       │  └─ Returns rootCmd
       └─ svrcmd.Execute(rootCmd, "start")

3. "start" command executes
   ├─ commands.go newApp()
   │  └─ app.go NewMiniApp()
   │     ├─ depinject.Inject(AppConfig())
   │     │  └─ Auto-creates:
   │     │     ├─ AccountKeeper
   │     │     ├─ BankKeeper
   │     │     ├─ StakingKeeper
   │     │     ├─ DistrKeeper
   │     │     └─ ConsensusParamsKeeper
   │     └─ app.Load(loadLatest)
   │        ├─ If ~/.minid exists: Load latest state
   │        └─ If new: Initialize from genesis
   │
   └─ CometBFT starts
      ├─ Loads genesis.json
      ├─ Starts producing blocks
      └─ Calls BeginBlock, ExecuteTx, EndBlock, Commit

4. Running chain
   ├─ Every block:
   │  ├─ distribution.BeginBlock()   ← distribution Keeper
   │  ├─ staking.BeginBlock()        ← staking Keeper
   │  ├─ Execute all transactions
   │  ├─ staking.EndBlock()          ← staking Keeper
   │  └─ Commit state
   │
   └─ User can query/transact via minid query/tx
```

---

## 📊 Dependency Diagram

```
main.go
   ↓
root.go (NewRootCmd)
   ├─ depinject.Inject(app.AppConfig())
   │  ├─ Loads app.yaml
   │  └─ Loads app/params/config.go
   │     ├─ SetAddressPrefixes()
   │     └─ RegisterDenoms()
   ├─ Calls initRootCmd() from commands.go
   └─ Returns rootCmd with all commands

When "start" command runs:
   ↓
commands.go newApp()
   ↓
app.go NewMiniApp()
   ├─ depinject.Inject(AppConfig())
   │  ├─ Loads app.yaml (modules, order)
   │  ├─ Auto-creates Keepers:
   │  │  ├─ AccountKeeper (from auth module)
   │  │  ├─ BankKeeper (from bank module)
   │  │  ├─ StakingKeeper (from staking module)
   │  │  ├─ DistrKeeper (from distribution module)
   │  │  └─ ConsensusParamsKeeper (from consensus module)
   │  └─ Creates module manager
   │
   ├─ app.Load(loadLatest)
   │  ├─ Loads state from ~/.minid/data
   │  └─ Or initializes from genesis
   │
   └─ Returns MiniApp ready to run
      ├─ Can receive BeginBlock, ExecuteTx, EndBlock, Commit
      └─ Can respond to queries/transactions
```

---

## 🎯 Key Files Quick Reference

| File          | Purpose              | What to Look For             |
| ------------- | -------------------- | ---------------------------- |
| `main.go`     | Entry point          | Where everything starts      |
| `root.go`     | CLI builder          | Command-line interface setup |
| `commands.go` | Command registration | How commands are wired       |
| `app.go`      | Blockchain app       | Module registration, Keepers |
| `app.yaml`    | Module blueprint     | Module order, config         |
| `config.go`   | Global config        | Address prefix, token name   |
| `encoding.go` | Codec setup          | How data serializes          |
| `export.go`   | State backup         | Export for upgrades          |
| `init.sh`     | First-time setup     | Creating test accounts       |

---

## 🚀 Beginner Challenges

### **Challenge 1: Trace a Transaction**

1. User runs: `minid tx bank send alice bob 100mini`
2. Find the code path:
    - `main.go` → `root.go` → `commands.go` (where does "tx" come from?)
    - What Keeper handles the send?
    - What does `app.yaml` say about when it executes?

### **Challenge 2: Understanding Address Format**

1. Look at `config.go`
2. Why are there 6 different Bech32 prefixes?
3. Generate an address: `minid keys add testkey`
4. Try sending to a validator address instead (will fail, why?)

### **Challenge 3: Init a Chain from Scratch**

1. Run `scripts/init.sh`
2. Before each `minid` command, think about which file it comes from
3. Check `~/.minid/config/genesis.json` after each step
4. See how genesis.json changes

### **Challenge 4: Change Configuration**

1. Edit `app.yaml`: Change `begin_blockers` order
2. Try to compile and run
3. See what breaks (understanding why teaches you!)
4. Revert and compare

---

## 💡 Common Confusions Cleared

**Q: Where do "minid query" and "minid tx" commands come from?**

A:

-   `root.go` line: `autoCliOpts.EnhanceRootCommand(rootCmd)` auto-generates them from app.yaml
-   Modules declare their query services and message types
-   SDK creates CLI commands automatically

**Q: How does minid know about "mini1..." addresses?**

A:

-   `main.go` calls `params.SetAddressPrefixes()`
-   `config.go SetAddressPrefixes()` sets SDK-wide prefix
-   All addresses created after this use "mini" prefix

**Q: What happens when you run `minid start`?**

A:

1. `root.go` finds "start" command
2. Calls `commands.go newApp()` which calls `app.go NewMiniApp()`
3. `NewMiniApp()` uses `depinject.Inject(app.AppConfig())`
4. That loads `app.yaml` and creates Keepers
5. CometBFT starts and begins producing blocks

**Q: Why is the order in app.yaml init_genesis so important?**

A:

-   `auth` must run first (create accounts)
-   `bank` must run after auth (assign balances to accounts)
-   `staking` must run after bank (delegate tokens)
-   `distribution` must run after staking (calculate rewards)
-   Changing order = invalid state

---

## 📖 Reading Path Summary

**For Understanding the Flow:**

1. Start: `main.go` (5 lines, understand entry)
2. Next: `root.go` (understand CLI creation)
3. Next: `commands.go` (understand command registration)
4. Then: `app.go` (understand module initialization)
5. Reference: `app.yaml` (understand execution order)
6. Details: `config.go`, `encoding.go` (understand configuration)

**For Practical Setup:**

1. Run: `scripts/init.sh` (understand chain creation)
2. Check: `~/.minid/config/genesis.json` (see what was created)
3. Modify: Change addresses, balances in genesis
4. Restart: Run `minid start` (see results)

**For Deep Understanding:**

1. Read: `export.go` (understand state management)
2. Modify: `app.yaml` (change module order, observe breakage)
3. Add: New Keeper to app.go (understand dependency injection)
4. Debug: Use `minid debug` commands to trace state
