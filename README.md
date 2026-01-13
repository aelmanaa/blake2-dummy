# @noble/hashes/blake2 Import Error Reproduction

This repository reproduces a dependency resolution issue with `@chainlink/ccip-sdk` when used alongside `ethers` v5.

## Problem

The SDK imports `@noble/hashes/blake2`, a subpath that only exists in `@noble/hashes` v1.7.0+. When npm nests an older version under the SDK, the import fails.

## Root Cause

`@chainlink/ccip-sdk@0.93.0` has two relevant dependencies:
- `@noble/hashes@^1.8.0` (direct) — for the `blake2` import
- `ethers@6.16.0` — which transitively depends on `@noble/hashes@1.3.2`

### Why npm hoists differently

**npm can only share packages at the top level when versions are compatible.** When versions conflict, npm nests the incompatible version inside the package that requires it.

**When your project uses ethers v5:**

1. Your `ethers@5.x` goes to top level (direct dependency wins)
2. SDK's `ethers@6.16.0` **conflicts** → npm nests it under `@chainlink/ccip-sdk/node_modules/`
3. The nested `ethers@6.16.0` brings its own `@noble/hashes@1.3.2`, also nested
4. When SDK imports `@noble/hashes/blake2`, Node.js resolves **from the SDK's directory upward**
5. It finds the nested `@noble/hashes@1.3.2` first → **fails** (no `./blake2` export)

**When your project uses ethers v6:**

1. Your `ethers@6.x` and SDK's `ethers@6.16.0` are **compatible** → shared at top level
2. SDK's direct `@noble/hashes@^1.8.0` goes to top level
3. ethers' `@noble/hashes@1.3.2` is nested under `ethers/node_modules/` only
4. When SDK imports `@noble/hashes/blake2`, it resolves to top-level `@noble/hashes@1.8.0` → **works**

## Expected node_modules Structure

### ethers v5 project (FAILS)

```
node_modules/
├── @chainlink/ccip-sdk/
│   └── node_modules/
│       ├── @noble/hashes@1.3.2    ← SDK resolves HERE (lacks ./blake2) ❌
│       └── ethers@6.16.0
├── @noble/hashes@1.8.0            ← has ./blake2, but SDK can't see it
└── ethers@5.8.0                   ← your direct dependency
```

### ethers v6 project (WORKS)

```
node_modules/
├── @chainlink/ccip-sdk/           ← no nested node_modules
├── @noble/hashes@1.8.0            ← SDK resolves HERE (has ./blake2) ✅
└── ethers@6.16.0                  ← shared with SDK
```

## Reproduction Steps

### ethers v5 (fails)

```bash
cd blake2-npm-ethersv5
npm install
npx tsx test.ts
```

**Expected error:**

```
Error [ERR_PACKAGE_PATH_NOT_EXPORTED]: Package subpath './blake2' is not defined by "exports"
in node_modules/@chainlink/ccip-sdk/node_modules/@noble/hashes/package.json
```

### ethers v6 (works)

```bash
cd blake2-npm-ethersv6
npm install
npx tsx test.ts
```

**Expected output:**

```
EVMChain: function
```

## Verify node_modules Structure

### ethers v5

```bash
cd blake2-npm-ethersv5

# SDK has nested @noble/hashes
cat node_modules/@chainlink/ccip-sdk/node_modules/@noble/hashes/package.json | jq .version
# Output: "1.3.2"

# SDK has nested ethers v6
cat node_modules/@chainlink/ccip-sdk/node_modules/ethers/package.json | jq .version
# Output: "6.16.0"

# Top-level ethers is v5
cat node_modules/ethers/package.json | jq .version
# Output: "5.8.0"
```

### ethers v6

```bash
cd blake2-npm-ethersv6

# SDK has NO nested @noble/hashes
ls node_modules/@chainlink/ccip-sdk/node_modules/@noble/hashes 2>&1
# Output: No such file or directory

# Top-level @noble/hashes is v1.8.0
cat node_modules/@noble/hashes/package.json | jq .version
# Output: "1.8.0"
```

## Why `.js` Extension Fix Doesn't Work

SDK v0.94.0 changed the import from `@noble/hashes/blake2` to `@noble/hashes/blake2.js`. This does not fix the issue because `@noble/hashes@1.3.2` exports neither `./blake2` nor `./blake2.js`.

| @noble/hashes version | `./blake2` | `./blake2.js` | `./blake2b`      |
| --------------------- | ---------- | ------------- | ---------------- |
| 1.3.2                 | No         | No            | Yes              |
| 1.8.0                 | Yes        | Yes           | Yes (deprecated) |

## Suggested Fix

Use `@noble/hashes/blake2b` instead of `@noble/hashes/blake2`. The `./blake2b` subpath exists in all versions.
