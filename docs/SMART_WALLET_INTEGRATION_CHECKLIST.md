# Smart Wallet Integration Checklist

## Overview

This checklist ensures your smart wallet integrates correctly and securely with SAFE-HAVEN vault contracts. Use it as a step-by-step guide during implementation and as a pre-deployment verification tool.

**Document Version**: 1.1  
**Last Updated**: 2024  
**Target Audience**: Wallet developers, DevOps teams, security auditors

---

## Pre-Integration: Planning Phase

### Authorization Model Review

- [ ] **Reviewed Soroban `require_auth()` documentation**
  - Understand how Soroban authorizes contract addresses
  - Confirm your wallet uses Soroban's native authorization framework
  
- [ ] **Identified wallet type**
  - [ ] Safe-style multisig wallet
  - [ ] Account abstraction (AA) wallet
  - [ ] EOA or hybrid wallet
  - [ ] Other (specify): _______________

- [ ] **Mapped wallet approval flow**
  - Document how your wallet validates and executes transactions
  - Identify where SAFE-HAVEN calls fit in the approval sequence
  
- [ ] **Confirmed token support**
  - Verify tokens accepted by SAFE-HAVEN work with your wallet
  - Check token approval patterns (if applicable)

### Architecture Decision Recording

- [ ] **Documented integration approach**
  - Record whether you're using: `deposit()`, `deposit_for()`, or `deposit_with_session_key()`
  - Explain the rationale for each choice
  
- [ ] **Design review completed**
  - Have security team review integration architecture
  - Document any deviations from standard patterns

---

## Core Integration: Wallet Configuration

### Smart Wallet Address Setup

- [ ] **Smart wallet deployed as Soroban contract**
  - Wallet exists as a contract address on the target network
  - NOT implemented as an EOA or externally owned account
  
- [ ] **Wallet address derivable and consistent**
  - Address generation is deterministic
  - Address remains consistent across deployments
  - Document address derivation method
  
- [ ] **Confirmed wallet as `depositor` in vault calls**
  - Use wallet contract address, NOT individual signer addresses
  - Use wallet contract address, NOT relayer addresses
  - Verified in production configuration files

### Contract Address as Depositor Verification

Smart wallets use their **contract address** as the `depositor` for all vault operations. The vault relies on Soroban's native `require_auth()` to verify the smart wallet's authorization. Verify the following:

- [ ] **Contract addresses are accepted as `depositor`**
  - Vault accepts a Soroban contract address (not just an EOA account) as the `depositor` parameter
  - No validation rejects contract addresses (e.g., no EOA-only checks)
  - Deposit is recorded under the contract address exactly as passed

- [ ] **`require_auth()` works with contract addresses**
  - `require_auth()` on the contract address succeeds when the smart wallet authorizes the call
  - `require_auth()` on the contract address fails when the smart wallet does NOT authorize the call
  - Authorization is enforced by Soroban's native framework (no custom auth logic in the vault)

- [ ] **Deposit from smart wallet succeeds**
  - Smart wallet calls `vault.deposit(wallet_contract_address, token, amount, unlock_time, penalty_bps)`
  - Deposit succeeds and is recorded under the wallet contract address
  - Token transfer from the smart wallet contract to the vault completes correctly

- [ ] **Withdrawal by smart wallet succeeds**
  - Smart wallet calls `vault.withdraw(wallet_contract_address, deposit_id)`
  - Withdrawal succeeds and tokens are returned to the smart wallet contract
  - Deposit state is updated correctly (removed/marked withdrawn)

- [ ] **No special handling is required for contracts**
  - Vault does NOT contain contract-specific code paths (no `if is_contract(...)` branches)
  - Contract addresses and EOA addresses follow the same code path
  - No registration, allowlisting, or whitelisting of contract addresses is needed
  - No additional configuration or flags are required to support smart wallets

- [ ] **Contract address cannot be impersonated**
  - An EOA cannot authorize on behalf of a contract address
  - Only the smart wallet contract itself can authorize operations for its address
  - Attempts by other addresses to act as the contract address fail authorization

### Authorization Configuration

- [ ] **Wallet implements Soroban `require_auth()` pattern**
  - Wallet correctly handles auth invocations from SAFE-HAVEN
  - Wallet authorization logic is executed before state mutations
  
- [ ] **Authorization logic executed before token transfer**
  - Wallet checks approval threshold/policy BEFORE calling vault
  - Token approval (if needed) is obtained BEFORE vault call
  
- [ ] **Authorization checks are immutable after deployment**
  - Security-critical authorization logic cannot be disabled
  - Admin functions cannot bypass authorization checks

### Signature Verification (Multisig Wallets)

- [ ] **Multisig threshold correctly implemented**
  - Minimum number of signers required is enforced
  - Threshold is configurable but immutable during a transaction
  
- [ ] **Signature aggregation implemented**
  - Contract correctly verifies all signatures
  - Duplicated signatures are rejected
  - Out-of-order signatures do not bypass verification
  
- [ ] **Signature verification uses safe libraries**
  - Cryptographic functions from established libraries (e.g., `soroban-auth`, Stellar SDK)
  - No custom cryptography implementations
  - Code reviewed by security team

### Replay Protection

- [ ] **Transaction nonce or sequence tracking implemented**
  - Each transaction has a unique identifier to prevent replays
  - Nonce is incremented after each successful operation
  
- [ ] **Nonce validation enforced before authorization**
  - Old/replayed transactions are rejected early
  - Nonce state is updated atomically with operation
  
- [ ] **Chain ID included in signed data (if applicable)**
  - Prevents cross-chain replay attacks
  - Current network is verified during authorization

---

## Integration Pattern: Direct Deposit

### Standard Deposit Flow

Use this when the wallet directly funds its own deposit.

- [ ] **Wallet calls `vault.deposit(depositor, token, amount, unlock_time, penalty_bps)`**
  - `depositor` is the wallet contract address
  - `token` is a valid Stellar asset contract address
  - `amount` is positive and wallet has sufficient balance
  - `unlock_time` is in the future (Unix timestamp)
  - `penalty_bps` is 0-10000 (0-100%)
  
- [ ] **Wallet authorizes the deposit call**
  - Authorization context includes this contract call
  - Soroban sees wallet address as `require_auth()` caller
  
- [ ] **Token transfer authorized by wallet**
  - If token requires approval, wallet initiates approval first
  - Token transfer amount matches vault call
  - Token transfer goes directly from wallet to vault
  
- [ ] **Deposit recorded under wallet address**
  - Vault records ownership as `depositor` (wallet address)
  - Transaction signer irrelevant; ownership is wallet address
  
- [ ] **Event monitoring by wallet address**
  - Monitor `deposit` events filtered by wallet `depositor`
  - Do NOT filter by transaction signer or relayer

### Implementation Validation

- [ ] **Test direct deposit on local network**
  - Deposit creates entry with correct ID (starts at 0)
  - Deposit amount matches vault state
  - Token transfer executed correctly
  
- [ ] **Test multiple deposits per wallet**
  - IDs increment correctly (0, 1, 2, ...)
  - Each deposit is independent
  - Withdrawing one does not affect others
  
- [ ] **Test authorization failure scenarios**
  - Unauthorized address cannot create deposit
  - Operation reverted without state mutation
  - No partial state updates

---

## Integration Pattern: Third-Party Funding (Deposit For)

### Relayer-Funded Deposit Flow

Use this when a relayer or payer funds the deposit for another address (beneficiary).

- [ ] **Payer calls `vault.deposit_for(payer, depositor, token, amount, unlock_time, penalty_bps)`**
  - `payer` is the relayer/funder address (wallet or EOA)
  - `depositor` is the beneficiary (wallet or EOA that owns the deposit)
  - `token` is a valid Stellar asset
  - `amount` must be funded by `payer`
  
- [ ] **Payer address is authorized by Soroban**
  - `require_auth()` validates payer is authorized to spend
  - Payer must authorize the vault call
  
- [ ] **Token transfer from payer to vault**
  - Payer has sufficient balance
  - Token transfer amount matches vault call
  - Token approval (if needed) is from payer
  
- [ ] **Deposit ownership is `depositor`, not payer**
  - Only `depositor` can withdraw the deposit
  - Payer has no further claims on the deposit
  - Multiple payers can fund the same beneficiary
  
- [ ] **Verify payer cannot withdraw beneficiary's deposit**
  - Payer tries to withdraw beneficiary's ID → authorization fails
  - Beneficiary withdraws successfully

### Implementation Validation

- [ ] **Test payer-funded workflow on local network**
  - Deposit ID created correctly
  - Ownership is beneficiary (depositor parameter)
  - Tokens transferred from payer account
  
- [ ] **Test payer cannot access beneficiary funds**
  - Payer authorized to pay, not to withdraw
  - Only beneficiary address can withdraw
  
- [ ] **Test multiple payers funding same beneficiary**
  - Each payer creates separate deposit
  - IDs are independent and sequential
  - Beneficiary can manage all deposits

### Security Considerations

- [ ] **Payer balance verified before transfer**
  - Transaction reverts if payer has insufficient balance
  - No partial deposits created
  
- [ ] **Depositor address cannot be zero address**
  - Invalid depositor addresses rejected
  - Prevents accidental fund locks
  
- [ ] **No reward to payer**
  - Payer receives no interest or vault proceeds
  - Payer investment is one-way transfer

---

## Integration Pattern: Session Keys

### Session Key Authorization Workflow

Use this for time-bounded delegated funding (gasless or relayed deposits).

#### Setup Phase

- [ ] **Wallet calls `vault.authorize_session_key(wallet, session_key, expires_at)`**
  - Called by wallet (requires `require_auth()` for wallet address)
  - `session_key` is the delegated address (can be EOA or contract)
  - `expires_at` is a future Unix timestamp
  - Expiry is explicit and immutable
  
- [ ] **Session key scope is single wallet**
  - Each session key is authorized for one wallet only
  - Same session key cannot be used with multiple wallets
  - Prevents cross-wallet delegation attacks

#### Usage Phase

- [ ] **Session key calls `vault.deposit_with_session_key(session_key, wallet, token, amount, unlock_time, penalty_bps)`**
  - `session_key` must match registered key (authorization checked)
  - `wallet` must match wallet session key is scoped to
  - Must be called before expiry timestamp
  
- [ ] **Expiry validation enforced before operation**
  - Check: `expires_at > env.ledger().timestamp()`
  - Expired keys rejected with `SessionKeyExpired` error
  - No state mutation if expired
  
- [ ] **Token transferred from session key, not wallet**
  - Session key must have sufficient balance
  - Wallet does NOT fund the deposit
  - Reduces load on wallet (gasless pattern possible)
  
- [ ] **Deposit ownership is wallet, not session key**
  - Vault records `depositor` as wallet address
  - Session key cannot withdraw this deposit
  - Only wallet can revoke or manage session keys

#### Revocation Phase

- [ ] **Wallet calls `vault.revoke_session_key(wallet, session_key)` when needed**
  - Only wallet that authorized key can revoke
  - Revocation is immediate
  
- [ ] **Revoked keys cannot be reused**
  - After revocation, key is permanently disabled for this wallet
  - Cannot re-register revoked key (if business logic prevents it)
  
- [ ] **Revocation does not affect existing deposits**
  - Deposits created by session key remain owned by wallet
  - Wallet can manage deposits normally
  
- [ ] **Session key expiry acts as automatic revocation**
  - Expired keys are automatically invalid
  - No manual revocation needed if you wait for expiry
  - Explicit revocation recommended for security

### Session Key Expiry Management

- [ ] **Expiry timestamp is explicit and documented**
  - Do NOT use vague durations like "1 hour from now"
  - Calculate: `current_timestamp + session_duration`
  - Document session duration in configuration
  
- [ ] **Multiple session keys per wallet allowed**
  - Wallet can authorize multiple concurrent session keys
  - Each key has independent expiry
  - Useful for: different relayers, seasonal delegation, key rotation
  
- [ ] **Expiry is checked at operation time**
  - Session key authorized at time T with expiry T+3600
  - If current time > expiry, operation fails
  - No grace period
  
- [ ] **Clock skew tolerance considered**
  - Ledger timestamp is authoritative (not wall-clock)
  - Stellar network closure time ≈ 5 seconds
  - Set expiry with buffer: `now + 3600 + buffer`
  - Document buffer (e.g., 60 seconds) in config

### Ledger Time Integration

- [ ] **Ledger timestamp used for all time checks**
  - NOT wall-clock time
  - NOT block number
  - Use `env.ledger().timestamp()` in contracts
  
- [ ] **Expiry semantics well-defined**
  - `expires_at > ledger_timestamp()` means NOT YET EXPIRED
  - `expires_at <= ledger_timestamp()` means EXPIRED
  - Document this explicitly in code comments

### Implementation Validation

- [ ] **Test session key authorization on local network**
  - Key authorized successfully
  - Authorized key can create deposit
  - Deposit recorded under wallet
  
- [ ] **Test session key expiry enforcement**
  - Valid key (before expiry) works
  - Expired key (after expiry) fails
  - Test boundary: `expires_at == current_timestamp`
  
- [ ] **Test session key revocation**
  - After revocation, key cannot create deposits
  - Deposits already created remain valid
  - Can re-authorize same key with new expiry
  
- [ ] **Test session key scope**
  - Key authorized for wallet A works with wallet A
  - Same key with wallet B fails (if scope enforced)
  - Prevents cross-wallet attacks

### Security Best Practices: Session Keys

- [ ] **Every session key has explicit expiry**
  - NEVER use unbounded session keys
  - Expiry must be configurable per use case
  
- [ ] **Session key is NOT the wallet private key**
  - Use distinct keypairs for delegation
  - Session key compromise does not expose wallet
  - Reduces blast radius of key theft
  
- [ ] **Session key has limited balance**
  - Fund session key only with needed amount
  - Wallet retains control of primary funds
  - Session key cannot spend wallet's entire balance
  
- [ ] **Revoke session keys when no longer needed**
  - Revoke immediately after intended session ends
  - Revoke before expiry if key compromised
  - Implement revocation monitoring/alerts
  
- [ ] **Monitor session key usage**
  - Track which keys created which deposits
  - Alert on unexpected session key usage
  - Audit session key lifecycle events
  
- [ ] **Session key credentials stored securely**
  - Use secure key management (e.g., AWS KMS, HashiCorp Vault)
  - Rotate keys periodically
  - Never commit session keys to version control

---

## Integration Pattern: Multisig Wallets

### Multisig Wallet Requirements

- [ ] **Wallet implements Safe-style or similar multisig pattern**
  - Multiple signers required for approvals
  - Threshold of signatures enforced
  - Signature aggregation logic implemented
  
- [ ] **Multisig wallet is deployed as Soroban contract**
  - Wallet address is a contract address
  - NOT deployed as individual signer EOAs
  
- [ ] **Approval threshold enforced before vault call**
  - Multisig checks approval count before calling SAFE-HAVEN
  - Approved transactions produce wallet-signed data
  - Wallet calls vault contract after threshold met

### Multisig Authorization Flow

- [ ] **Individual signers do NOT directly authorize vault call**
  - Signers authorize the multisig wallet operation
  - Multisig wallet (after approval) authorizes vault call
  - Vault sees multisig contract address as `require_auth()` caller
  
- [ ] **SAFE-HAVEN vault does NOT inspect individual signers**
  - Vault trusts multisig contract authorization
  - Vault does NOT validate signature count
  - Vault does NOT reverse-engineer signer identities
  
- [ ] **Deposit is recorded under multisig wallet address**
  - `depositor` is multisig contract address
  - Individual signers have no direct ownership claim
  - Signers can only access via multisig authorization

### Multisig Implementation Validation

- [ ] **Test multisig wallet deposit on local network**
  - Configure M-of-N signers (e.g., 2-of-3)
  - Gather M signatures
  - Multisig wallet calls vault
  - Deposit recorded under multisig address
  
- [ ] **Test multisig rejection below threshold**
  - Configure 2-of-3 threshold
  - Provide only 1 signature
  - Multisig rejects operation
  - Vault call never reached
  
- [ ] **Test individual signer cannot bypass multisig**
  - Signer A tries to call vault directly (not via multisig)
  - Authorization check fails
  - Signer cannot withdraw multisig-owned deposit
  
- [ ] **Test multisig withdrawal**
  - Multisig-owned deposit created
  - Multisig provides approval for withdrawal
  - Withdrawal succeeds
  
- [ ] **Test multisig cannot withdraw single-signer deposit**
  - Single-signer address owns deposit (not multisig)
  - Multisig tries to withdraw
  - Authorization fails

### Multisig Security Considerations

- [ ] **Signature order does not matter**
  - Signatures can be sorted or aggregated in any order
  - Signature verification is order-independent
  
- [ ] **Duplicate signatures rejected**
  - Same signature cannot be used twice
  - Prevents threshold bypass
  
- [ ] **Signature is over the correct operation**
  - Data includes: contract address, method, parameters
  - Signature binds to this specific operation
  - Cannot be reused for different operations
  
- [ ] **Signature algorithm uses established standards**
  - Stellar uses Ed25519
  - Signature verification uses standard Stellar SDK
  - No custom cryptography

---

## Integration Pattern: Account Abstraction (AA) Wallets

### AA Wallet Architecture

- [ ] **Wallet deployed as Soroban contract**
  - AA wallet is a smart contract (not EOA)
  - Custom authorization logic implemented in wallet contract
  
- [ ] **Entry point contract for relay operations (optional)**
  - Entry point can relay calls to AA wallet
  - Entry point calls AA wallet, which authorizes vault call
  - Enables gasless patterns
  
- [ ] **Authorization logic customizable per wallet**
  - SAFE-HAVEN does NOT enforce AA wallet structure
  - Wallet implements its own approval/verification
  - Vault trusts Soroban `require_auth()` mechanism

### AA Wallet Authorization Flow

- [ ] **AA wallet authorizes vault call via Soroban auth**
  - Wallet implements custom authorization logic
  - Authorization is checked by Soroban (not SAFE-HAVEN)
  - Vault sees wallet contract address as caller
  
- [ ] **Entry point (if used) calls AA wallet first**
  - Entry point receives relayed transaction
  - Entry point calls AA wallet to validate
  - AA wallet authorizes entry point call
  - Entry point then calls vault
  - Vault sees entry point address, which is authorized by AA wallet
  
- [ ] **Deposit is recorded under AA wallet address**
  - If AA wallet directly calls vault: `depositor` is AA wallet
  - If entry point relays: `depositor` is still AA wallet (not entry point)
  - Wallet maintains full control and ownership

### AA Wallet Implementation Validation

- [ ] **Test AA wallet direct deposit**
  - AA wallet calls vault directly
  - Deposit recorded under AA wallet address
  - AA wallet can withdraw own deposit
  
- [ ] **Test AA wallet with entry point relay**
  - Entry point receives relayed transaction
  - Entry point calls AA wallet for authorization
  - AA wallet authorizes entry point call
  - Entry point calls vault
  - Deposit recorded under AA wallet address
  
- [ ] **Test AA wallet with session key delegation**
  - AA wallet authorizes session key with `authorize_session_key()`
  - Session key calls `deposit_with_session_key()`
  - Deposit recorded under AA wallet
  - AA wallet can revoke session key anytime
  
- [ ] **Test multi-operation sequence**
  - Create multiple deposits
  - Transfer tokens between deposits (if applicable)
  - AA wallet maintains authorization throughout

### AA Wallet Security Considerations

- [ ] **Entry point does NOT own deposits**
  - Depositor is AA wallet, NOT entry point
  - Entry point is purely relaying mechanism
  - Wallet retains full control
  
- [ ] **Entry point cannot be substituted mid-operation**
  - AA wallet verifies entry point identity
  - Only authorized entry point can relay
  - Different entry points require re-authorization
  
- [ ] **AA wallet maintains recovery procedures**
  - Wallet implements key recovery (if needed)
  - Recovery does NOT affect vault deposits
  - Recovery changes wallet authorization, not depositor
  
- [ ] **AA wallet can implement custom spending limits**
  - Wallet can limit per-transaction amounts
  - Wallet can limit daily/weekly aggregates
  - SAFE-HAVEN respects wallet's authorization decision

---

## Smart Wallet Security Model

This section documents the security model that governs how SAFE-HAVEN vaults interact with smart wallets. Understanding this model is essential for integrators and auditors.

### Trust Boundaries

- **Vault trusts Soroban `require_auth()`**: The vault does not implement its own authorization scheme. It relies entirely on Soroban's native `require_auth()` to verify that the address passed as `depositor` (or `payer`) has authorized the call.

- **Contract addresses are first-class principals**: Soroban treats contract addresses and account addresses uniformly for authorization purposes. A contract address can authorize a call exactly as an account can, provided the contract itself invokes the authorization.

- **Vault does not inspect wallet internals**: The vault has no knowledge of a smart wallet's signer set, threshold, session keys, or recovery mechanism. It only observes whether the contract address authorized the call.

### Authorization Guarantees

- **Only the contract can authorize for its own address**: An EOA cannot authorize on behalf of a contract address. Soroban's authorization framework binds authorization to the actual caller. This prevents impersonation of smart wallets.

- **Authorization is checked before state mutation**: All state-mutating functions call `require_auth()` on the relevant address before any state changes. Failed authorization reverts the entire operation with no partial state updates.

- **No special handling for contracts**: The vault contains no contract-specific code paths. Contract addresses and EOA addresses follow identical logic. This eliminates a class of bugs where contract handling diverges from account handling.

### Ownership Model

- **Depositor is the owner**: The `depositor` parameter recorded at deposit time is the sole owner of the deposit. For smart wallets, this is the wallet's contract address.

- **Signers are not owners**: Individual signers of a multisig or AA wallet have no direct claim on deposits. They can only act through the wallet contract, which must authorize the call.

- **Relayers are not owners**: In `deposit_for()` flows, the payer funds the deposit but the `depositor` owns it. In session key flows, the session key funds the deposit but the wallet owns it.

### Threat Model

- **Compromised signer**: A single compromised signer cannot move funds if the wallet's threshold is not met. The vault defers to the wallet's authorization decision.

- **Compromised session key**: A compromised session key can only create deposits (funded by the session key's own balance) up to its expiry. It cannot withdraw wallet-owned deposits or spend wallet funds. Revoke the key to stop further use.

- **Malicious relayer**: A relayer in `deposit_for()` or entry-point flows cannot withdraw deposits it funded. Ownership is bound to the `depositor` address, not the payer.

- **Impersonation attempt**: An attacker cannot authorize as a contract address they do not control. Soroban's authorization framework rejects such attempts.

### Integration Responsibilities

- **Wallet is responsible for its own authorization logic**: The vault does not validate the wallet's internal approval mechanism. The wallet must correctly implement threshold checks, signature verification, replay protection, and any other controls before authorizing a vault call.

- **Wallet must authorize before calling the vault**: The wallet should complete its internal approval flow and then invoke the vault call within the same authorization context, so Soroban sees the contract address as the authorizing principal.

- **Integrators must test both success and failure paths**: Verify that authorized calls succeed and unauthorized calls fail, for both deposits and withdrawals.

---

## Security Audit Checklist

### Authorization & Access Control

- [ ] **`require_auth()` called before state mutation**
  - Authorization check is first operation in state-mutating functions
  - No state changes before auth validation
  
- [ ] **Authorization scope is correct**
  - Wallet address is authorized for wallet-owned operations
  - Session key is authorized for delegated operations
  - Payer is authorized for `deposit_for()` operations
  
- [ ] **Unauthorized addresses cannot execute operations**
  - Non-wallet address cannot create/withdraw wallet deposits
  - Non-session-key address cannot use session key
  - Non-payer address cannot fund `deposit_for()`
  
- [ ] **Authorization is idempotent**
  - Same operation succeeds if authorized, fails if not
  - No state corruption from repeated auth checks
  
- [ ] **Authorization cannot be bypassed**
  - No alternate code paths skip auth
  - No admin override of authorization
  - No emergency mode that disables auth

### Contract Address Depositor Verification

- [ ] **Contract addresses accepted as `depositor`**
  - Vault accepts contract addresses without rejection
  - No EOA-only validation blocks contract addresses
  - Deposit recorded under the exact contract address passed

- [ ] **`require_auth()` accepts contract addresses**
  - Authorized contract address passes `require_auth()`
  - Unauthorized contract address fails `require_auth()`
  - Authorization uses Soroban native framework only

- [ ] **Deposit from smart wallet succeeds**
  - Smart wallet contract calls `deposit()` and succeeds
  - Tokens transferred from contract to vault
  - Deposit recorded under contract address

- [ ] **Withdrawal by smart wallet succeeds**
  - Smart wallet contract calls `withdraw()` and succeeds
  - Tokens returned to contract address
  - Deposit state updated correctly

- [ ] **No special handling required for contracts**
  - No contract-specific branches in vault code
  - No allowlist/registration of contract addresses
  - Contract and EOA addresses follow identical code paths

- [ ] **Contract address cannot be impersonated**
  - EOA cannot authorize on behalf of a contract address
  - Only the contract itself can authorize its operations
  - Impersonation attempts fail authorization

### Session Key Security

- [ ] **Session key expiry enforced every use**
  - Expiry checked before ANY operation with session key
  - No state mutation if expired
  - Expiry value is immutable after authorization
  
- [ ] **Session key scope is enforced**
  - Key authorized for wallet A cannot be used with wallet B
  - Wallet information is part of the authorization check
  - Cross-wallet usage fails fast
  
- [ ] **Session key revocation is immediate**
  - After revocation, key cannot create any operations
  - Revocation is effective immediately (no grace period)
  - Revoked key cannot be re-used (if business logic prevents it)
  
- [ ] **Session key does NOT inherit wallet privileges**
  - Session key can create deposits but cannot withdraw them
  - Session key cannot transfer wallet funds outside vault
  - Wallet remains authorization point for sensitive operations
  
- [ ] **Multiple session keys do NOT interfere**
  - Each key is tracked independently
  - Revocation of key A does not affect key B
  - Expiry of key A does not affect key B
  
- [ ] **Session key cannot spend wallet's balance**
  - Only session key's own balance is used
  - Wallet funds remain untouched by session key
  - Wallet can be empty while session key operates

### Input Validation

- [ ] **Token address is validated**
  - Token contract exists on the network
  - Token address is not zero address
  - Token implements standard interface (transfer, mint, etc.)
  
- [ ] **Amount is validated**
  - Amount > 0 (no zero deposits)
  - Amount does not overflow (checked against max uint256)
  - Sender has sufficient balance
  
- [ ] **Unlock time is validated**
  - Unlock time is in the future
  - Unlock time is reasonable (e.g., not 1000 years away)
  - Unlock time is before platform end-of-life date
  
- [ ] **Penalty basis points are validated**
  - Penalty is 0-10000 (0-100%)
  - Penalty is not negative
  - Penalty is reasonable for vault use case
  
- [ ] **Session key address is validated**
  - Session key is not zero address
  - Session key is not same as wallet address
  - Session key is not same as another registered key
  
- [ ] **Depositor address is validated**
  - Depositor is not zero address
  - Depositor address format is valid
  - Contract addresses are accepted (not rejected as invalid)

### State Management

- [ ] **Deposit IDs are assigned sequentially**
  - First deposit for address is ID 0
  - IDs increment by 1
  - IDs never reused even after withdrawal
  
- [ ] **Deposit ownership is immutable**
  - Once deposit created under address, owner cannot change
  - Cannot transfer deposit to different owner
  - Cannot delegate ownership to another address
  
- [ ] **Deposits are isolated**
  - Withdrawing deposit X does not affect deposit Y
  - Canceling deposit X does not affect deposit Y
  - State mutations are atomic per operation
  
- [ ] **No duplicate session keys per wallet**
  - Same session key cannot be authorized twice for same wallet
  - Re-authorization of same key is rejected or replaces old entry
  - Prevents confusion and tracking bugs

### Error Handling

- [ ] **Error codes are specific and actionable**
  - `NotAuthorized` = authorization failed (not just missing auth)
  - `SessionKeyExpired` = key expired (not generic error)
  - `FundsStillLocked` = unlock time not reached
  - `SessionKeyNotAuthorized` = key not registered
  
- [ ] **Errors include minimal information**
  - Error message does NOT leak sensitive state
  - Error message does NOT leak timing information
  - Error message is sufficient for debugging

### Replay & Nonce Protection

- [ ] **Wallet implements replay protection**
  - Nonce or sequence tracking prevents old transactions from replaying
  - Nonce is checked before wallet authorization
  
- [ ] **Cross-chain replay is prevented**
  - Chain ID or network identifier included in signed data
  - Different networks require different signatures
  
- [ ] **Session key replays are prevented**
  - Session key operations include timestamp
  - Old operations cannot be replayed on new network state
  - Nonce incremented per operation

### Cryptographic Security

- [ ] **Signatures use Stellar Ed25519 standard**
  - All signatures are Ed25519 (Soroban standard)
  - No custom signature algorithms
  - Signature verification uses established libraries
  
- [ ] **Signature verification is constant-time**
  - Signature check does NOT leak timing information
  - Verification time does NOT depend on signature validity
  
- [ ] **Private key handling is secure**
  - Private keys never logged or exposed
  - Private keys stored in secure key management system
  - Key rotation periodic and documented

### Testing Coverage

- [ ] **Authorization tests exist**
  - Test wallet authorizes operations
  - Test non-wallet cannot authorize operations
  - Test authorization failure reverts state
  
- [ ] **Contract address depositor tests exist**
  - Test contract address accepted as `depositor`
  - Test `require_auth()` succeeds for authorized contract address
  - Test `require_auth()` fails for unauthorized contract address
  - Test deposit from smart wallet succeeds
  - Test withdrawal by smart wallet succeeds
  - Test no special handling required (same code path as EOA)
  
- [ ] **Session key tests exist**
  - Test key authorization and usage
  - Test key expiry enforcement
  - Test key revocation
  - Test scope (wallet-specific)
  
- [ ] **Multisig tests exist**
  - Test threshold enforcement
  - Test signature aggregation
  - Test duplicate signature rejection
  - Test invalid signature rejection
  
- [ ] **Error case tests exist**
  - Test each error condition
  - Test error specificity
  - Test no state mutation on error
  
- [ ] **Edge case tests exist**
  - Test boundary conditions (expiry at exact time)
  - Test maximum values (large amounts, max penalty)
  - Test zero values (zero amount, zero penalty)
  
- [ ] **Integration tests exist**
  - Test full workflow from user perspective
  - Test multiple wallets interacting
  - Test session key lifecycle

---

## Pre-Deployment Verification

### Code Review Checklist

- [ ] **Authorization logic reviewed by security team**
  - Code review completed with sign-off
  - No outstanding security concerns
  - Authorization checks are in correct order
  
- [ ] **Signature verification reviewed by cryptography expert**
  - Signature scheme is appropriate
  - Verification logic is correct
  - No timing side-channels
  
- [ ] **State management reviewed**
  - Deposits are correctly stored and retrieved
  - State mutations are atomic
  - No race conditions
  
- [ ] **Error handling reviewed**
  - All error paths are correct
  - No unhandled exceptions
  - Error messages are appropriate

### Static Analysis

- [ ] **Solidity/Rust linter passes**
  - No compiler warnings
  - Code style is consistent
  - All formatting rules applied
  
- [ ] **Security static analysis tool run**
  - Tool: _________________ (e.g., Slither, Mythril)
  - No high/critical issues
  - Medium issues reviewed and documented
  
- [ ] **Dependency audit completed**
  - No known vulnerabilities in dependencies
  - All dependencies up-to-date
  - Dependency changes documented

### Testnet Deployment

- [ ] **Deployed to public testnet**
  - Network: ________________________ (e.g., Stellar Testnet)
  - Deployment date: _________________
  - Wallet address: ___________________
  
- [ ] **Testnet tests passed**
  - All integration tests pass on testnet
  - No network-specific issues
  - Performance acceptable
  
- [ ] **Testnet monitoring active**
  - Deployed for min. 1 week
  - No critical issues found
  - User feedback reviewed (if applicable)

### Documentation Readiness

- [ ] **Integration guide written**
  - Step-by-step instructions for developers
  - Example code provided
  - Common pitfalls documented
  
- [ ] **Security guide written**
  - Authorization model explained
  - Smart wallet security model documented
  - Session key best practices documented
  - Revocation procedures explained
  
- [ ] **API documentation complete**
  - All functions documented
  - Parameters documented
  - Return values documented
  - Error codes documented
  
- [ ] **FAQ updated**
  - Common integration questions answered
  - Troubleshooting section provided
  - Links to security resources

### Operational Readiness

- [ ] **Monitoring configured**
  - Deposit events logged
  - Authorization failures logged
  - Session key operations logged
  - Alerts configured for failures
  
- [ ] **Key management procedures documented**
  - Session key generation procedure
  - Key storage location
  - Key rotation schedule
  - Key revocation procedure
  
- [ ] **Incident response plan drafted**
  - Process for revoking compromised keys
  - Process for suspending wallet (if applicable)
  - Escalation contacts
  - Communication templates
  
- [ ] **Runbook prepared**
  - Step-by-step deployment instructions
  - Rollback procedures
  - Verification steps
  - Contact list for escalation

---

## Post-Deployment: Operations

### Day-1 Operations

- [ ] **Deployed to mainnet**
  - Network: ________________________
  - Deployment date: _________________
  - Wallet address: ___________________
  - Deployment confirmed by 2 team members
  
- [ ] **Monitoring verified**
  - Event logs being collected
  - Alerts configured and tested
  - Dashboard accessible
  
- [ ] **On-call contact established**
  - Primary contact: ___________________
  - Backup contact: ____________________
  - Escalation chain documented