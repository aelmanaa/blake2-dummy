# @noble/hashes/blake2 Import Error Reproduction

This repository reproduces a dependency resolution issue with `@chainlink/ccip-sdk` when used alongside `ethers` v5.

## Problem

The SDK imports `@noble/hashes/blake2`, a subpath that only exists in `@noble/hashes` v1.7.0+. When npm nests an older version under the SDK, the import fails.

## Root Cause

`@chainlink/ccip-sdk@0.93.0` depends on `ethers@6.16.0`, which depends on `@noble/hashes@1.3.2`.

When your project uses ethers v5:

- npm places ethers v5 at top level (your direct dependency)
- npm nests ethers v6 under the SDK
- `@noble/hashes@1.3.2` follows ethers v6 and gets nested under the SDK
- SDK imports resolve to the nested v1.3.2, which lacks `./blake2`

When your project uses ethers v6:

- npm shares ethers v6 at top level
- `@noble/hashes@1.3.2` nests under ethers only
- SDK imports resolve to top-level `@noble/hashes@1.8.0`

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
