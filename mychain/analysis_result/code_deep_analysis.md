# Code Deep Analysis

## File1: app/params/encoding.go

**Location:** `app/params/encoding.go`  
**Package:** `params`  
**Purpose:** Define encoding configuration structure for the blockchain application

---

## Package Declaration

```go
package params
```

This file shares scope with all other files in the `params` directory (e.g., `config.go`). They can directly access each other's public elements without imports.

---

## Imports

```go
import (
 "github.com/cosmos/cosmos-sdk/client"
 "github.com/cosmos/cosmos-sdk/codec"
 "github.com/cosmos/cosmos-sdk/codec/types"
)
```

### Import Details

1. **`github.com/cosmos/cosmos-sdk/client`** - Client Package

    - Handles client-side communication with blockchain nodes
    - Provides: CLI commands, RPC requests, transaction handling
    - Used here for: `client.TxConfig`

2. **`github.com/cosmos/cosmos-sdk/codec`** - Codec Encoding Library

    - Reference: <https://pkg.go.dev/github.com/cosmos/cosmos-sdk@v0.50.13/codec>
    - Encodes/decodes data between Go structs and binary/JSON formats
    - Provides: `Codec` and `LegacyAmino`

3. **`github.com/cosmos/cosmos-sdk/codec/types`** - Codec Types Package
    - Defines types and interfaces for codec operations
    - Used here for: `types.InterfaceRegistry`

---

## EncodingConfig Structure

```go
type EncodingConfig struct {
 InterfaceRegistry types.InterfaceRegistry
 Codec             codec.Codec
 TxConfig          client.TxConfig
 Amino             *codec.LegacyAmino
}
```

### Field Breakdown

| Field               | Type                      | Purpose                                                                                                                                                                                            |
| ------------------- | ------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `InterfaceRegistry` | `types.InterfaceRegistry` | Registry mapping message types to their implementations. Enables deserialization of unknown message types.                                                                                         |
| `Codec`             | `codec.Codec`             | Modern codec for encoding/decoding. Default protobuf-based. Used for all new code.                                                                                                                 |
| `TxConfig`          | `client.TxConfig`         | Transaction configuration. Handles how transactions are built, signed, and serialized.                                                                                                             |
| `Amino`             | `*codec.LegacyAmino`      | **DEPRECATED** - Reference: <https://pkg.go.dev/github.com/cosmos/cosmos-sdk@v0.50.13/codec#LegacyAmino>. Legacy amino codec maintained for backward compatibility with older blockchain versions. |

### Field Access

All fields are **capitalized** (public). Any other package importing `params` can access them:

```go
import "mychain/app/params"

// Usage example:
config := params.EncodingConfig{}
config.Codec           // ✓ Accessible
config.InterfaceRegistry  // ✓ Accessible
config.TxConfig        // ✓ Accessible
config.Amino           // ✓ Accessible (deprecated, but still accessible)
```

---

## Documentation Comment

```go
// EncodingConfig specifies the concrete encoding types to use for a given app.
// This is provided for compatibility between protobuf and amino implementations.
```

**Interpretation:**

-   The structure encapsulates all encoding-related components needed by the application
-   Supports both modern protobuf encoding and legacy amino encoding
-   Enables smooth transition from amino to protobuf without breaking existing functionality

---

## Role in Cosmos SDK Application

The `EncodingConfig` is a configuration container that:

1. **Handles Message Serialization** - `Codec` and `InterfaceRegistry` work together to serialize/deserialize blockchain messages
2. **Manages Transactions** - `TxConfig` configures how transactions are constructed and signed
3. **Maintains Backward Compatibility** - `Amino` ensures old data can still be read and processed

In practice, this structure is instantiated in `app.go` during `MiniApp` initialization and used throughout the application's lifetime.

---

## Key Concepts

### InterfaceRegistry

When Alice sends 100 mini to Bob, the transaction is encoded as binary. The `InterfaceRegistry` maps the message type (`bank.MsgSend`) to its implementation, allowing the decoder to reconstruct the message from bytes.

### Codec vs LegacyAmino

-   **Codec** (protobuf): Faster, smaller message size, modern standard
-   **LegacyAmino** (deprecated): Slower, larger messages, maintained only for compatibility with pre-2020 blockchain versions

### TxConfig

Determines how `minid tx bank send ...` commands are processed—how signatures are applied, how gas is calculated, how messages are packed into transactions.

---

## Summary

`encoding.go` defines a single structure `EncodingConfig` that bundles together all encoding/decoding/transaction configuration components. It's a convenience container that makes passing encoding-related configuration throughout the application simple and organized.

---

## File2: app/params/config.go

**Location:** `app/params/config.go`  
**Package:** `params`  
**Purpose:** Define blockchain configuration constants and setup functions for address prefixes and coin denomination

---

## Imports

```go
import (
 "cosmossdk.io/errors"
 "cosmossdk.io/math"

 sdk "github.com/cosmos/cosmos-sdk/types"
 "github.com/cosmos/cosmos-sdk/types/address"
 sdkerrors "github.com/cosmos/cosmos-sdk/types/errors"
)
```

### Import Details

1. **`cosmossdk.io/errors`** - Error handling utilities
2. **`cosmossdk.io/math`** - Math operations (used for decimal values)
3. **`sdk "github.com/cosmos/cosmos-sdk/types"`** - SDK types (alias: `sdk`)
    - Core blockchain types and configurations
4. **`github.com/cosmos/cosmos-sdk/types/address`** - Address validation rules
5. **`sdkerrors "github.com/cosmos/cosmos-sdk/types/errors"`** - SDK error definitions (alias: `sdkerrors`)

---

## Constants Definition

```go
const (
 CoinUnit = "mini"
 DefaultBondDenom = CoinUnit
 Bech32PrefixAccAddr = "mini"
)
```

| Constant              | Value                | Purpose                                                                                   |
| --------------------- | -------------------- | ----------------------------------------------------------------------------------------- |
| `CoinUnit`            | `"mini"`             | The blockchain's native coin name. Like "BTC" for Bitcoin or "ETH" for Ethereum.          |
| `DefaultBondDenom`    | `CoinUnit` (="mini") | The coin type used for staking/bonding. Set to `CoinUnit` so both refer to the same coin. |
| `Bech32PrefixAccAddr` | `"mini"`             | Address prefix for regular user accounts. Example: `mini1abc123...`                       |

---

## Variables Definition

```go
var (
 Bech32PrefixAccPub = Bech32PrefixAccAddr + "pub"
 Bech32PrefixValAddr = Bech32PrefixAccAddr + "valoper"
 Bech32PrefixValPub = Bech32PrefixAccAddr + "valoperpub"
 Bech32PrefixConsAddr = Bech32PrefixAccAddr + "valcons"
 Bech32PrefixConsPub = Bech32PrefixAccAddr + "valconspub"
)
```

**Why use `var` instead of `const`?**  
These values are computed by concatenating `Bech32PrefixAccAddr` with suffixes. Go's `const` only allows simple literals, not expressions. So `var` is required.

| Variable               | Value              | Purpose                                                                                      |
| ---------------------- | ------------------ | -------------------------------------------------------------------------------------------- |
| `Bech32PrefixAccPub`   | `"minipub"`        | Public key prefix for regular accounts. Example: `minipub1abc123...`                         |
| `Bech32PrefixValAddr`  | `"minivaloper"`    | Address prefix for validators (block producers). Example: `minivaloper1abc123...`            |
| `Bech32PrefixValPub`   | `"minivaloperpub"` | Public key prefix for validators. Example: `minivaloperpub1abc123...`                        |
| `Bech32PrefixConsAddr` | `"minivalcons"`    | Address prefix for consensus nodes (participate in voting). Example: `minivalcons1abc123...` |
| `Bech32PrefixConsPub`  | `"minivalconspub"` | Public key prefix for consensus nodes. Example: `minivalconspub1abc123...`                   |

**Bech32 Prefixes Explained:**

-   Bech32 is the address format standard for blockchains
-   Different roles (accounts, validators, consensus nodes) have different prefixes
-   One glance at an address tells you what role it belongs to
-   All prefixes in this chain start with "mini" to identify the blockchain

---

## init() Function

```go
func init() {
 SetAddressPrefixes()
 RegisterDenoms()
}
```

**What is `init()`?**  
Special function in Go that runs automatically when the package is loaded (before any other code in the package executes).

**What happens:**

1. `SetAddressPrefixes()` - Configures all address prefixes globally
2. `RegisterDenoms()` - Registers the coin denomination

This ensures the blockchain configuration is set up immediately when the program starts.

---

## RegisterDenoms() Function

```go
func RegisterDenoms() {
 err := sdk.RegisterDenom(CoinUnit, math.LegacyOneDec())
 if err != nil {
  panic(err)
 }
}
```

**Purpose:**  
Register the coin "mini" with the SDK's global denomination registry.

**Parameters:**

-   `CoinUnit` ("mini") - The coin name
-   `math.LegacyOneDec()` - Decimal value (1.0 = one whole coin)

**Error Handling:**

-   If registration fails, `panic(err)` stops the program immediately
-   This ensures the blockchain won't start without proper coin configuration

---

## SetAddressPrefixes() Function

```go
func SetAddressPrefixes() {
 config := sdk.GetConfig()
 config.SetBech32PrefixForAccount(Bech32PrefixAccAddr, Bech32PrefixAccPub)
 config.SetBech32PrefixForValidator(Bech32PrefixValAddr, Bech32PrefixValPub)
 config.SetBech32PrefixForConsensusNode(Bech32PrefixConsAddr, Bech32PrefixConsPub)
 config.SetAddressVerifier(func(bytes []byte) error { ... })
}
```

### Step-by-Step Breakdown

**Line 1:** `config := sdk.GetConfig()`

-   Gets the global SDK configuration object
-   `:=` declares a new variable and assigns value

**Lines 2-4:** Set prefixes for different roles

-   `SetBech32PrefixForAccount()` - Sets "mini" and "minipub" for regular users
-   `SetBech32PrefixForValidator()` - Sets "minivaloper" and "minivaloperpub" for validators
-   `SetBech32PrefixForConsensusNode()` - Sets "minivalcons" and "minivalconspub" for consensus nodes

**Lines 5-17:** `SetAddressVerifier()` - Custom address validation
Validates that any address on this chain meets requirements:

-   **Not empty** - addresses cannot be zero bytes
-   **Max length** - cannot exceed `address.MaxAddrLen` bytes
-   **Length must be 20 or 32 bytes** - these are valid Cosmos SDK address lengths

```go
if len(bytes) == 0 {
 return errors.Wrap(sdkerrors.ErrUnknownAddress, "addresses cannot be empty")
}
```

This returns an error if address is empty.

```go
if len(bytes) > address.MaxAddrLen {
 return errors.Wrapf(sdkerrors.ErrUnknownAddress, "address max length is %d, got %d", address.MaxAddrLen, len(bytes))
}
```

This returns an error if address is too long, showing what max length is allowed and what was received.

```go
if len(bytes) != 20 && len(bytes) != 32 {
 return errors.Wrapf(sdkerrors.ErrUnknownAddress, "address length must be 20 or 32 bytes, got %d", len(bytes))
}
```

This ensures address is exactly 20 or 32 bytes. No other lengths are accepted.

---

## Configuration Workflow

When the program starts:

```
1. Package imports config.go
   ↓
2. init() executes automatically
   ├─ SetAddressPrefixes()
   │  └─ Sets all Bech32 prefixes globally
   └─ RegisterDenoms()
      └─ Registers "mini" coin
   ↓
3. Blockchain is configured and ready
   ├─ Addresses like "mini1abc123..."
   ├─ Validators like "minivaloper1abc123..."
   └─ Coin "mini" is recognized
```

---

## Summary

`config.go` is the blockchain's identity card. It defines:

-   **Coin name:** "mini" (like Bitcoin's "BTC")
-   **Address prefixes:** Different roles get different prefixes for identification
-   **Validation rules:** What constitutes a valid address on this chain

These settings are enforced throughout the entire blockchain lifecycle. Once `SetAddressPrefixes()` is called, the config is sealed and cannot be changed, ensuring consistency.
