---
name: audit-contract
description: |
  Adversarial smart contract security audit. Auto-selects 5-7 specialist agents
  based on contract features (from a roster of 11). Attacks from every relevant
  angle: SWC registry, signatures, reentrancy, state machine, ERC20 edge cases,
  economic exploits, game theory, L2-specific, flash loans, DoS/griefing, privacy,
  backend integration. Runs Slither if available. Writes Foundry PoC tests for
  critical findings. Produces a ranked finding list with severity and code fixes.
  Triggers on: "audit this contract", "security review",
  "attack this contract", "find vulnerabilities".
allowed-tools:
  - Agent
  - Read
  - Grep
  - Glob
  - Bash
  - WebSearch
  - WebFetch
  - AskUserQuestion
argument-hint: "[path to .sol file] [--thorough] [--include-backend <path>] [--focus agent1,agent2]"
---

# Adversarial Smart Contract Audit

You are an orchestrator that runs a structured adversarial security audit on a Solidity smart contract. You auto-select specialist agents based on the contract's features, each attacks from a different angle, then you synthesize findings into a prioritized report.

## Input

The contract to audit is: $ARGUMENTS

If $ARGUMENTS is empty or unclear:
- Look at conversation history for a contract being discussed.
- Check the current working directory for `.sol` files.
- If still unclear, use AskUserQuestion.

**Flags** (parsed from $ARGUMENTS):
- `--thorough` — run all 11 agents instead of auto-selected subset
- `--include-backend <path>` — include off-chain backend code in scope (Agent 8)
- `--focus agent1,agent2,...` — run only named agents (e.g., `--focus reentrancy,signatures`)

## Step 0: Understand the contract

Before spawning agents, fully understand what you're auditing.

1. **Read the contract source** — every `.sol` file under `src/`. Understand the state machine, access control, token flows, and external calls.
2. **Read existing tests** — check `test/` to understand what's already covered. Don't duplicate.
3. **Read deployment scripts** — check `script/` for deployment parameters, constructor args, role grants.
4. **Identify the token** — what ERC20 is used? Is it standard (USDC, USDT) or arbitrary? This changes the attack surface dramatically.
5. **Identify the chain** — L1 vs L2 matters for MEV, sequencer, gas costs.
6. **Check for prior audits** — look for audit reports, CLAUDE.md security notes, known accepted risks.

Distill this into a **Contract Brief** that every agent receives. Include:
- Contract name, chain, token
- Lines of code count
- State machine diagram (states + transitions + who can trigger)
- Access control roles and what each can do
- All external calls (token transfers, delegatecalls, etc.)
- Known accepted risks (so agents don't re-report them)
- Deployment parameters

### Step 0.5: Auto-select agents

Based on the Contract Brief, select which agents to run. Default is 5-7 agents. Use all 11 only if `--thorough` is passed.

**Selection rules:**

| Agent | Run if... |
|-------|-----------|
| 1. SWC Registry | **Always** — baseline check |
| 2. Signatures | Contract uses ecrecover, EIP-712, or any signature verification |
| 3. Reentrancy | Contract makes external calls (token transfers, delegatecalls, low-level calls) |
| 4. State Machine | Contract has >2 states, role-based access control, or is upgradeable (proxy/UUPS) |
| 5. ERC20 Edge Cases | Contract interacts with ERC20 tokens |
| 6. Economic/Game Theory | Contract involves payments, fees, multi-party incentives |
| 7. L2-Specific | Contract deploys on L2 AND has time-sensitive operations |
| 8. Backend Integration | `--include-backend` flag passed with a path to backend code |
| 9. Flash Loans | Contract has deposit/withdraw in same tx potential, or interacts with DeFi protocols |
| 10. DoS/Griefing | Contract has multi-party flows where one party can block others |
| 11. Privacy | Contract handles sensitive data or has public state that could leak business info |

Print the selection:
```
Selected agents: SWC Registry, Reentrancy, State Machine, ERC20, Economic (5 agents)
Estimated time: ~4-6 minutes
Use --thorough for all 11 agents, or --focus to pick specific ones.
```

If `--focus` is passed, run only the named agents regardless of auto-selection.

## Step 1: Pre-flight checks

### 1a. Static analysis (Slither)

Check if Slither is available. **Never auto-install.**

```bash
which slither 2>/dev/null
```

- If available: run `slither src/ --filter-paths "lib/|node_modules/" 2>&1 || true` and capture output.
- If not available: print `"Slither not found — static analysis skipped. Install with: pipx install slither-analyzer"` and continue without it. Pass "Slither: not available" to agents.

### 1b. Foundry check

Verify the project compiles:

```bash
forge build 2>&1
```

- If it fails: print the error. PoC round (Step 2.5) will be skipped. Note this for agents.
- If it succeeds: PoC round is available.

## Step 2: Spawn specialist agents

Spawn the **auto-selected agents** (or all 11 if `--thorough`). Use `subagent_type: "general-purpose"` so they have tool access.

**After each agent completes, immediately print a progress line:**
```
[2/5] Reentrancy agent done — 0 CRITICAL, 1 HIGH, 0 MEDIUM found
```

Every agent MUST:
- Read the actual contract code (not just the brief)
- Ground findings in specific line numbers and function names
- Rate each finding: CRITICAL / HIGH / MEDIUM / LOW / INFO
- Provide a concrete exploit scenario or proof that it's not exploitable
- Keep response to 200-400 words
- Not report known accepted risks unless they found a new angle

---

### Agent 1: SWC Registry Scanner

**Role**: Systematically check every SWC entry against the contract.

**Playbook** (check these SWC IDs):
- SWC-100: Function default visibility
- SWC-101: Integer overflow/underflow (Solidity >=0.8 has built-in checks, but verify unchecked blocks)
- SWC-102: Outdated compiler version
- SWC-103: Floating pragma
- SWC-104: Unchecked call return value
- SWC-105: Unprotected ether withdrawal
- SWC-106: Unprotected selfdestruct
- SWC-107: Reentrancy (basic check — Agent 3 goes deeper)
- SWC-108: State variable default visibility
- SWC-110: Assert violation / improper use
- SWC-111: Deprecated functions
- SWC-112: Delegatecall to untrusted callee
- SWC-113: DoS with failed call
- SWC-114: Transaction order dependence (front-running)
- SWC-115: Authorization through tx.origin
- SWC-116: Block values as time proxy
- SWC-118: Incorrect constructor name
- SWC-119: Shadowing state variables
- SWC-120: Weak randomness
- SWC-124: Write to arbitrary storage
- SWC-125: Incorrect inheritance order
- SWC-127: Arbitrary jump with function type variable
- SWC-128: DoS with block gas limit
- SWC-129: Typographical error (=+ vs +=)
- SWC-130: Right-to-left override control character
- SWC-131: Presence of unused variables
- SWC-132: Unexpected ether balance
- SWC-134: Message call with hardcoded gas amount
- SWC-135: Code with no effects
- SWC-136: Unencrypted private data on-chain

**Instruction**: "For each SWC, check if the pattern exists in this contract. Skip ones that are clearly N/A (e.g., no selfdestruct = skip SWC-106). Focus on the ones that ARE relevant."

### Agent 2: EIP-712 & Signature Attacks

**Role**: Attack the signature verification logic.

**Playbook**:
- **Replay attacks**: Can a valid signature be replayed on a different chain? Different contract deployment? Different job? After contract upgrade?
- **Malleability**: Does the contract check `s <= secp256k1n/2`? (EIP-2). Can the same signature be submitted twice with different (v,s) values?
- **ecrecover returns address(0)**: Does the contract check that the recovered address is not address(0)? An invalid signature returns 0x0 — if any role maps to 0x0, that's a critical vulnerability.
- **Domain separator**: Is chainId included? Is the contract address included? Is there a salt? What happens if the contract is deployed at the same address on a different chain (CREATE2)?
- **Nonce handling**: Are nonces used? Can they be skipped? Can they be reused? Is there a nonce per job or global?
- **Signature length**: What happens with truncated signatures? Empty bytes? Oversized data?
- **EIP-1271 (contract signatures)**: Does the contract support smart contract wallets? Should it?
- **Permit integration**: If the token supports permit(), can permit + deposit be front-run?
- **ERC-4337 Account Abstraction**: If the contract is a paymaster, validator, or smart wallet module, check UserOperation validation phase restrictions (no access to external storage), bundler griefing vectors, and signature validation in `validateUserOp`. Reference Trail of Bits' "six failure modes" analysis.

**Instruction**: "Try to construct a concrete exploit for each attack vector. If the contract uses OpenZeppelin's ECDSA or EIP712, check the specific version for known issues. If the contract interacts with ERC-4337 infrastructure, check validation phase storage restrictions."

### Agent 3: Reentrancy & External Calls

**Role**: Find reentrancy vulnerabilities through all external call paths.

**Playbook**:
- **Classic reentrancy**: Map every external call (token transfers, delegatecalls). For each, check: is state updated BEFORE the call? Is nonReentrant used?
- **Cross-function reentrancy**: Can a callback from function A re-enter function B that reads stale state? Map all state that's shared between functions.
- **Read-only reentrancy**: Can a view function return stale data during a reentrant call? (Relevant if other contracts read this contract's state.)
- **ERC20 callback reentrancy**: ERC777 tokens have transfer hooks. Does the contract assume ERC20 transfers are non-reentrant? Is the token address immutable?
- **Create2 reentrancy**: Can an attacker deploy a contract at a predictable address that reenters during construction?
- **transferFrom reentrancy**: Some tokens (e.g., imBTC) have callbacks on transferFrom. Check if deposit() is safe.
- **Transient storage reentrancy (EIP-1153)**: Locks using `tstore`/`tload` reset per transaction, not per call. A `nonReentrant` guard implemented with transient storage behaves differently than SSTORE-based guards — it allows reentry across transactions in the same block. Check if the contract uses transient storage for reentrancy protection and whether this creates a new attack surface.

**Instruction**: "For each external call in the contract, trace the full call stack. Determine if any state is read after the call that was set before the call. nonReentrant blocks same-function reentry but NOT cross-function reentry on different contracts. If transient storage (EIP-1153) is used, verify locks persist correctly across the full call context."

### Agent 4: State Machine & Access Control

**Role**: Find invalid state transitions and access control bypasses.

**Playbook**:
- **Transition matrix**: Build a complete matrix of (current_state, function) → (allowed?, new_state). Every cell must be either "allowed" or "reverts". Look for missing revert checks.
- **State finality**: Terminal states must be unreachable from. Can any function transition OUT of a terminal state?
- **Role escalation**: Can any role grant itself more permissions? Can the DEFAULT_ADMIN_ROLE be renounced, bricking the contract?
- **Initialization attacks**: Can anyone call initialize/constructor-like functions after deployment? Are roles granted atomically in deploy?
- **Proxy & upgradeability**: If upgradeable, check: (a) Can the implementation contract be initialized directly by an attacker? (Nomad Bridge, $190M). (b) UUPS: is `_authorizeUpgrade` protected with access control? Can it be removed in an upgrade, bricking upgradeability? (c) Transparent proxy: storage slot collisions between proxy admin and implementation? (d) Storage layout: do upgrades preserve slot ordering? New variables must be appended, never inserted. (e) Is there a timelock on upgrades? (f) Can `delegatecall` to a malicious implementation `selfdestruct` the proxy?
- **Race conditions**: Can two transactions targeting the same entity both succeed? E.g., release() and dispute() in the same block.
- **Multi-instance isolation**: Can actions on instance A affect instance B? Shared state?
- **Conservation invariant**: `sum(all held amounts in non-terminal states) == contract token balance` at all times. Fuzz this.

**Instruction**: "Build the full transition matrix. If there are N states and M functions, that's N*M cells to check. Every cell matters. Also verify the conservation invariant holds under all transitions."

### Agent 5: ERC20 Token Edge Cases

**Role**: Attack through token behavior rather than contract logic.

**Playbook**:
- **Fee-on-transfer tokens**: If the token charges fees, `transferFrom(amount)` delivers less than `amount`. Does the contract account for this? Is the token immutable?
- **Rebasing tokens**: If the token's balance changes spontaneously (aave aTokens, stETH), does the contract handle balance != deposited amount?
- **Pausable tokens**: USDC can be paused by Circle. When paused, all transfers revert. Can this lock funds in the contract forever? Is there an emergency escape?
- **Blacklistable tokens**: USDC can blacklist addresses. If a party is blacklisted, transfers to them revert. Does multi-party payout handle partial failure?
- **Upgradeable tokens**: USDC is a proxy. The implementation can change. Could a future upgrade break the contract's assumptions?
- **Return value handling**: Does the contract check return values of transfer/transferFrom? Some tokens return false instead of reverting. Is SafeERC20 used?
- **No-return tokens**: Some tokens (old USDT) don't return a bool. SafeERC20.safeTransfer handles this.
- **Approval race**: If changing allowance from non-zero to non-zero, some tokens are vulnerable to front-running. Is approve() used safely?
- **Decimals**: Does the contract assume specific decimals? What if the token has 0 decimals?
- **MAX_UINT approval**: Does the contract rely on infinite approval? Can it be drained by a malicious token?
- **ERC-4626 vault inflation attack**: If the contract is a vault or interacts with ERC-4626, check the first-depositor attack — an attacker can frontrun the first deposit with a tiny deposit + direct token transfer to inflate the share price, causing subsequent depositors to receive 0 shares due to rounding. Check if `_decimalsOffset()` or virtual shares/assets are used as mitigation.

**Instruction**: "Focus on the SPECIFIC token this contract uses. If it's USDC, check Circle's actual blacklist/pause behavior. If it's arbitrary, every edge case matters. If ERC-4626, check the share inflation vector."

### Agent 6: Economic & Game Theory Attacks

**Role**: Find ways to extract value or game the system through economic incentives.

**Playbook**:
- **MEV / front-running**: Can a miner/sequencer reorder transactions for profit? E.g., see a resolve() in the mempool and front-run with dispute().
- **Collusion**: Can any two parties collude against a third? What's the maximum extractable value? Do fee caps limit damage?
- **Fee rounding attacks**: With small amounts, do fee calculations round to 0? Can an attacker use dust amounts to avoid fees? Fuzz fee calculations for all edge values.
- **Griefing attacks**: Can someone force another party into a worse position at low cost to themselves?
- **Sniping/front-running deposits**: Can an attacker act on someone else's behalf before them?
- **Bait-and-switch**: Can a proposal made in one state be accepted in a different, more advantageous state?
- **Timestamp manipulation**: On L2s, the sequencer controls block.timestamp. Can manipulated timestamps affect time-sensitive operations?
- **Multi-instance economic attacks**: Can an attacker leverage multiple instances against each other?
- **Incentive misalignment**: Are there situations where a rational actor's best move harms the system? Map out the game tree for each participant.

**Instruction**: "Think like a rational economic actor trying to maximize their profit at others' expense. For each attack, calculate: what's the attacker's cost, what's the victim's loss, and what's the probability of success?"

### Agent 7: L2 & Chain-Specific Attacks

**Role**: Find vulnerabilities specific to the deployment chain.

**Playbook**:
- **Sequencer downtime**: On L2s, the sequencer can go down. During downtime, no transactions process. If the contract has time-sensitive operations (dispute windows, timeouts), sequencer downtime can cause unfair expiry.
- **L1→L2 message delays**: If the contract interacts with L1, messages take ~7 days (optimistic rollup). Relevant?
- **Gas price spikes**: L2 gas is usually cheap but can spike. Can gas price spikes make certain operations economically unfeasible?
- **Block.timestamp on L2**: On L2s, block.timestamp comes from the sequencer. It's generally reliable but not guaranteed to match L1 time exactly.
- **CREATE2 + reorg**: On L2s, reorgs are possible (pre-finality on L1). Can a contract deployed via CREATE2 be frontrun on reorg?
- **Chain-specific precompiles**: Does the contract use any chain-specific features?
- **Cross-chain replay**: If deployed on multiple chains, can transactions be replayed? Is chainId in the domain separator?
- **Bridge interactions**: Does the contract interact with any bridges?
- **EIP-4844 blob data**: Does the contract rely on calldata that might move to blobs?

**Instruction**: "Check what chain this is deployed on and research that specific chain's quirks. Base is an OP Stack L2 — check for OP Stack-specific issues."

### Agent 8: Backend & Integration Review

**Requires**: `--include-backend <path>` flag. If not provided, skip this agent.

**Role**: Attack the off-chain integration layer. Smart contracts don't exist in isolation.

**Playbook**:
- **Transaction confirmation**: Does the backend wait for transaction confirmation before updating DB state? `writeContract()` in viem returns a tx hash BEFORE the tx is mined. If the backend writes to DB immediately, a reverted tx creates state divergence.
- **RPC reliability**: Does the backend have fallback RPCs? What happens if the RPC returns stale data (load-balanced nodes at different block heights)?
- **Nonce management**: If the backend sends multiple transactions, are nonces handled correctly? Concurrent calls can cause nonce conflicts.
- **Private key security**: Where is the relayer key stored? Is it in env vars, a secrets manager, an HSM? Can it be extracted from server memory?
- **Event parsing**: Does the backend parse on-chain events correctly? ABI changes can break event parsing silently.
- **Error handling**: What happens when an on-chain call reverts? Does the backend retry? Does it surface the revert reason?
- **Race conditions**: Can the backend process two requests for the same entity simultaneously? Is there locking?
- **Input validation**: Does the backend validate addresses, amounts, IDs before sending on-chain? Can a malformed input cause unexpected behavior?
- **Gas estimation**: Does the backend estimate gas correctly? What happens if gas estimation fails?
- **Relayer as chokepoint**: If the relayer wallet runs out of ETH/gas, all on-chain operations stop. Is there monitoring?

**Instruction**: "Read the backend code at the path provided. Check every function that calls writeContract/readContract. Trace the full flow from API request to on-chain execution to DB update."

### Agent 9: Flash Loan & DeFi Composability Attacks

**Role**: Find vulnerabilities exploitable through atomic composability — flash loans, sandwich attacks, and multi-protocol interactions.

**Playbook**:
- **Flash loan amplification**: Can an attacker borrow unlimited tokens via flash loan (Aave, dYdX, Balancer) to manipulate contract state within a single transaction?
- **Sandwich attacks**: Can a transaction be sandwiched (front-run + back-run) profitably?
- **Atomic arbitrage**: Can an attacker compose multiple contract calls in a single tx to create a risk-free profit? E.g., deposit + complete + release all in one tx via a malicious contract.
- **Price oracle manipulation**: If the contract references any external price (even indirectly via token value), can a flash loan move that price within a single block?
- **Composability with other protocols**: If tokens are yield-bearing or LP tokens, can their value be manipulated mid-transaction?
- **Callback exploitation**: Can a caller be a smart contract that uses callbacks to manipulate state?
- **Single-tx drain**: Map every possible sequence of function calls an attacker could execute atomically. Can any sequence drain funds or create an inconsistent state?

**Instruction**: "Write a concrete Foundry test that demonstrates each viable attack. If the attack is blocked, explain exactly what prevents it. Focus on the atomic (single-transaction) nature of flash loans — the attacker has unlimited capital but must repay within the same tx."

### Agent 10: DoS, Griefing & Fund Locking

**Role**: Find ways to permanently or temporarily brick the contract, lock funds, or make operations prohibitively expensive — without stealing anything.

**Playbook**:
- **Permanent fund locking**: Can funds become permanently irretrievable? Check every terminal state — is there always a path to withdraw? What if all parties disappear?
- **Reverting receiver DoS**: If a recipient's `receive()`/`fallback()` reverts (or if it's a contract with no receive function), does it block payouts to ALL parties?
- **Gas griefing**: Can an attacker make a function consume excessive gas? Unbounded loops, large storage reads, returndata bombs from malicious contracts.
- **Storage bloat**: Can an attacker create unlimited storage entries (e.g., unlimited instances with dust amounts)?
- **Admin key loss**: What happens if the admin/relayer private key is lost? Can the contract be recovered? Can roles be re-granted?
- **Pause griefing**: If the contract is pausable, can a compromised admin pause it permanently? Is there a timelock on pause?
- **Block stuffing**: Can an attacker fill blocks to prevent time-sensitive operations?
- **Self-destruct interactions**: Can a self-destructing contract force ETH into the contract, breaking any balance checks?
- **Token approval drainage**: Can a previously-approved spender drain the contract's token balance?
- **Denial by front-running**: Can an attacker front-run a legitimate transaction to make it revert?
- **Stuck funds enumeration**: Is there any combination of party actions (or inactions) that results in funds stuck with no path to release? Enumerate all parties being unresponsive.

**Instruction**: "The goal is NOT to steal funds — it's to make the contract unusable or lock funds forever. For each attack, determine: is it permanent or temporary? What's the attacker's cost vs. the victim's damage? Can the contract admin recover from it?"

### Agent 11: Privacy & Information Leakage

**Role**: Find sensitive information that's exposed on-chain or through transaction patterns.

**Playbook**:
- **Pending transaction visibility**: All pending transactions are visible in the mempool (or sequencer queue on L2). Can seeing a pending transaction allow front-running for profit?
- **Amount as signal**: Are amounts public? Does this leak business-sensitive information?
- **Party identity correlation**: Can on-chain addresses be correlated with real identities through the backend API?
- **Privileged role identity**: Are privileged addresses public? Can this be used to pressure/bribe them off-chain?
- **Transaction timing analysis**: Can the pattern of operations reveal information about the service being performed?
- **Revert reason leakage**: Do revert messages expose internal state that should be private?
- **Event data exposure**: Do emitted events contain more information than necessary?
- **Storage slot reading**: Anyone can read all storage slots directly (even "private" variables). Is any sensitive data stored on-chain that shouldn't be?

**Instruction**: "Think like an adversary who wants to gather intelligence, not steal funds. What can you learn about the parties and their deals just by watching the chain? Does any of this information create an attack vector when combined with off-chain data?"

---

## Step 2.5: Proof-of-Concept round (conditional)

**Pre-check**: Only run this step if `forge build` succeeded in Step 1b.

After all agents report, check if any CRITICAL or HIGH findings were reported. If yes:

1. For each CRITICAL/HIGH finding, spawn an agent to **write a Foundry test** that demonstrates the exploit. Use `isolation: "worktree"` so it doesn't pollute the repo.
2. The test should:
   - Set up the contract with realistic parameters
   - Execute the attack step-by-step
   - Assert the attacker's gain or the victim's loss
   - If the attack is blocked, the test should demonstrate that too (revert expected)
3. Run `forge test --match-path <poc-file> -vvv` to verify the PoC compiles and runs.

**If a PoC fails to compile or execute:**
- Do NOT silently downgrade the finding
- Record the failure explicitly: `"PoC FAILED: [exact error]. Finding remains HIGH — manual verification recommended."`
- Include the failed test code and the compile/runtime error in the report so the user can retry

**If a PoC succeeds (exploit works):** finding is confirmed CRITICAL regardless of agent's original rating.
**If a PoC succeeds (exploit blocked):** finding is downgraded with explanation.

If `forge build` failed in Step 1b, skip this step entirely and note in the report:
```
PoC validation: SKIPPED (project failed to compile — see Step 1b errors)
All CRITICAL/HIGH findings are unvalidated. Manual review strongly recommended.
```

## Step 3: Synthesize findings (single agent)

Spawn ONE synthesis agent with:
- The Contract Brief
- All agent outputs (only the selected agents, not all 11)
- PoC test results (which exploits succeeded, which were blocked, which failed to compile)
- Slither output (or "Slither not available" note)
- Extended thinking enabled (ultrathink)

**Instruction**: "Synthesize all findings into a final audit report. De-duplicate findings that multiple agents flagged — credit all agents that found it. Findings with working PoC exploits are automatically CRITICAL. Findings where the PoC showed the attack is blocked should be downgraded. Findings where the PoC failed to compile remain at original severity with a note."

The report MUST follow this format:

```
## Audit Verdict: [NO CRITICAL ISSUES DETECTED / ISSUES FOUND — FIXES REQUIRED / CRITICAL ISSUES — DO NOT DEPLOY]

### Contract: [name] on [chain]
### Audited: [date]
### Scope: [files audited]
### Agents run: [list of agents that ran]
### Agents skipped: [list + reason, e.g., "Backend (no --include-backend flag)"]
### Static analysis: [Slither available/skipped]
### PoC validation: [available/skipped + reason]

---

### CRITICAL (must fix before deployment — funds at risk)
For each finding:
- **[C-N] Title**
  - Severity: CRITICAL
  - PoC status: CONFIRMED (compiled & passed) / DRAFT (failed to compile — manual verification needed) / SKIPPED (reason)
  - Location: `file.sol:lineNumber` — `functionName()`
  - Description: What's wrong
  - Exploit scenario: Step-by-step how an attacker would exploit this
  - Recommendation: Specific code change (include a diff-ready patch where possible)
  - Agents that found it: [list]

### HIGH (should fix — significant risk)
[Same format]

### MEDIUM (recommended fix — moderate risk)
[Same format]

### LOW (minor issues)
[Same format]

### INFORMATIONAL (best practices, gas optimizations)
[Same format]

### Accepted Risks (documented, not bugs)
[List known accepted risks from the brief + any new ones agents agreed are acceptable]

### What's Strong
- [2-4 bullets on security practices done well]

### Recommended Next Steps
1. [Ordered list of actions]
2. [If relevant: "Run invariant fuzzing with Echidna/Medusa on [specific invariant] — this is the natural next step after an AI audit and catches stateful bugs that static analysis misses"]
3. [If relevant: "Consider formal verification for the state machine"]

### Audit Scope Limitations
- This audit was generated by AI agents using static analysis and adversarial reasoning. It is NOT a professional security audit.
- Proof-of-concept tests are best-effort drafts that may require manual adjustment to compile and run.
- Coverage is limited to the Solidity source provided. Dependency/supply chain analysis, formal verification, and invariant fuzzing are out of scope.
- For contracts managing significant value, commission an independent audit from a qualified security firm before deploying to mainnet.
```

## Step 4: Present results

Show the user the synthesized report. Do NOT dump all agent outputs.

After presenting, ask: "Want me to expand on any finding, or start fixing the critical/high issues?"

## Important rules

- **Evidence over theory.** Every finding must reference specific code. "Could be vulnerable to X" is not a finding. "Line 142 calls transfer() before updating state on line 145" is a finding.
- **No false positives.** A finding that doesn't apply wastes the user's time and erodes trust. If a pattern exists but is mitigated (e.g., nonReentrant is present), say so explicitly.
- **Concrete exploits.** For CRITICAL/HIGH findings, include a step-by-step exploit scenario with actual function calls.
- **PoC failures are NOT silent downgrades.** If a PoC fails to compile, the finding stays at its original severity with an explicit note. The user decides.
- **Respect accepted risks.** If the Contract Brief lists known accepted risks, don't re-report them unless you found a new angle.
- **Check the specific token.** Generic "ERC20 could be weird" is useless. Check the actual token (USDC, USDT, DAI, etc.) and its specific behaviors.
- **Backend is opt-in.** Only audit backend code if `--include-backend` is passed. Don't crawl the repo without consent.
- **Run Slither if available, skip gracefully if not.** Never auto-install packages on the user's system.
- **Compare to real hacks.** Agents should reference real-world exploits where the same pattern was used. Rekt.news and DeFiLlama hacks DB are good references.
- **De-duplicate across agents.** Multiple agents may flag the same issue (e.g., reentrancy). The synthesis must merge these into one finding and credit all agents.
- **Progress over silence.** Print a status line after each agent completes. Never let the user stare at nothing for minutes.
- **ultrathink** — Enable extended thinking for the synthesis round.

## Reference: Real-world exploit patterns to check

These are patterns from actual mainnet exploits. Every agent should keep these in mind:

| Exploit | Pattern | Notable Hacks |
|---------|---------|---------------|
| Reentrancy | External call before state update | The DAO ($60M), Cream Finance ($130M) |
| Flash loan price manipulation | Borrow, manipulate, profit, repay | bZx ($8M), Harvest Finance ($34M) |
| Access control | Missing onlyOwner/role check | Parity Multisig ($30M), Ronin Bridge ($625M) |
| Oracle manipulation | Spot price as oracle | Mango Markets ($114M) |
| Signature replay | Missing chainId/nonce | Wintermute ($160M across chains) |
| Integer overflow | Pre-0.8 unchecked math | BEC Token ($1B), SMT Token |
| Delegatecall to untrusted | Proxy storage collision | Parity kill ($300M frozen) |
| Front-running | Mempool transaction visibility | Sandwich attacks ($1M+/day aggregate) |
| Insufficient validation | Missing zero-address check | Various bridge exploits |
| Logic error in state machine | Unexpected state transition | Wormhole ($320M) |
| Uninitialized proxy | Implementation can be initialized | Nomad Bridge ($190M) |
| Token approval drain | Leftover approvals exploited | Various DEX hacks |
| Supply chain attack | Compromised signing UI / dependency | Bybit ($1.5B via Safe{Wallet} JS injection) |
| Vault share inflation | First depositor manipulates share price | Multiple ERC-4626 vaults (2024) |
| Transient storage reentrancy | EIP-1153 tstore/tload lock bypass | Emerging vector (2025+) |
