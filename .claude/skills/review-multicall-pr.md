# Multicall3 PR Review Skill

Use this skill when reviewing PRs that add new chain deployments to the multicall repository.

## Validation Requirements

Every PR adding a new chain must pass ALL of these checks:

### 1. Chain ID Uniqueness
- Extract chain IDs from the PR diff
- Verify each chain ID is NOT already in `deployments.json`
- Check for duplicates within the PR itself

### 2. Chainlist Verification
```bash
curl -s https://chainlist.org/rpcs.json | jq '.[] | select(.chainId == CHAIN_ID)'
```
- Confirm the chain exists on chainlist.org
- Verify the chain name matches (approximately)
- Get RPC URL for bytecode verification

### 3. Bytecode Verification
The deployed contract MUST match Multicall3 on Ethereum mainnet.

**Reference (Ethereum Mainnet Multicall3):**
- Address: `0xcA11bde05977b3631167028862bE2a173976CA11`
- Size: 7,619 bytes
- SHA256: `e23dba1cd18a22e11ad869e35e5cd7f8923063be694ddbeb96e6efc47b204fce`

**Verification command:**
```bash
cast code 0xcA11bde05977b3631167028862bE2a173976CA11 --rpc-url RPC_URL > /tmp/chain_bytecode.txt
echo "Bytes: $(wc -c < /tmp/chain_bytecode.txt)"
echo "SHA256: $(shasum -a 256 /tmp/chain_bytecode.txt | cut -d' ' -f1)"
```

**Acceptable results:**
- **EXACT MATCH**: SHA256 matches mainnet - approved
- **METADATA MISMATCH**: Same size, different hash - check source code matches (different compiler version is OK)
- **DIFFERENT SIZE/CODE**: Reject unless source code is verified identical

### 4. JSON Schema Validation
Each entry must have:
```json
{
  "name": "Chain Name",
  "chainId": 12345,
  "url": "https://explorer.example.com/address/0xcA11bde05977b3631167028862bE2a173976CA11"
}
```

- Use `chainId` (not `chainID`)
- Use 2-space indentation (not tabs)
- Optional `address` field only for non-standard addresses (zkSync, Tron, etc.)

## Workflow

1. **Get PR diff:**
   ```bash
   gh pr diff PR_NUMBER
   ```

2. **Extract chains to validate** from the diff

3. **For each chain, run validation:**
   ```bash
   # Check chainlist
   curl -s https://chainlist.org/rpcs.json | jq '.[] | select(.chainId == CHAIN_ID)'

   # Check uniqueness
   cat deployments.json | jq '.[] | select(.chainId == CHAIN_ID)'

   # Verify bytecode
   cast code 0xcA11bde05977b3631167028862bE2a173976CA11 --rpc-url RPC_URL > /tmp/bytecode.txt
   shasum -a 256 /tmp/bytecode.txt
   ```

4. **Report results** in table format:
   | Chain | Chain ID | Chainlist | Bytecode | Status |
   |-------|----------|-----------|----------|--------|

5. **If changes needed:**
   - Checkout PR: `gh pr checkout PR_NUMBER`
   - Reset to main: `git reset --hard origin/main`
   - Add valid entries manually
   - Commit and push: `git push --force`

## Common Issues

- **Duplicate chain ID**: Remove the duplicate entry
- **Wrong `chainID` vs `chainId`**: Fix the casing
- **Tabs instead of spaces**: Convert to 2-space indent
- **RPC not responding**: Cannot verify - request working RPC from contributor
- **Bytecode mismatch**: Check if source matches (metadata diff OK), otherwise reject
- **Deprecated testnet** (e.g., Goerli): Consider removing

## Quick Validation Script

```bash
# Validate a single chain
CHAIN_ID=12345
RPC_URL="https://rpc.example.com"

echo "=== Validating chain $CHAIN_ID ==="
curl -s https://chainlist.org/rpcs.json | jq ".[] | select(.chainId == $CHAIN_ID) | {name, chainId}"
cast code 0xcA11bde05977b3631167028862bE2a173976CA11 --rpc-url $RPC_URL > /tmp/check.txt
echo "Size: $(wc -c < /tmp/check.txt) bytes (expect 7619)"
echo "SHA256: $(shasum -a 256 /tmp/check.txt | cut -d' ' -f1)"
echo "Expected: e23dba1cd18a22e11ad869e35e5cd7f8923063be694ddbeb96e6efc47b204fce"
```
