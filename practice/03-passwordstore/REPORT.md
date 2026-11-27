# PasswordStore: practice pre-audit report

> ⚠️ **Practice report · open-source educational contract · not a client engagement.**
> The contract is Cyfrin's [3-passwordstore-audit](https://github.com/Cyfrin/3-passwordstore-audit), a public training codebase. This report shows the SENTRYX pre-audit format.

| Report ID | Commit | Scope | Window |
|---|---|---|---|
| SNTX-PA-PRACTICE-01 | [`63541e5`](https://github.com/Cyfrin/3-passwordstore-audit/tree/63541e54586cca26e0a69b3ddc6b8fed150e7d2d) | `PasswordStore.sol`, 20 nSLOC | Practice review (time-boxed, 2 hours) |

## 1. Summary

PasswordStore lets one owner store a "private" password and update it later. We found two High issues. The first lets anyone overwrite the password. The second lets anyone read it, because contract storage is public. We also found one informational issue. Both High findings are proven with a Foundry test, included below.

| ID | Title | Severity | Status |
|---|---|---|---|
| H-01 | `setPassword()` has no access control, so anyone can overwrite the owner's password | High | Fix verified (local) |
| H-02 | The "private" password is readable by anyone from contract storage | High | Open · design change needed |
| I-01 | Incorrect NatSpec `@param` on `getPassword()` | Info | Open |

## 2. Scope and method

**In scope:** `src/PasswordStore.sol` at commit `63541e5` (20 nSLOC). There are no external dependencies and no external calls.
**Out of scope:** deployment scripts, the frontend and anything off-chain.

**Method:** a manual pass on access control and state against the README and NatSpec, a second pass on data visibility and storage layout, and a Foundry test for each High candidate. A candidate we could not prove with a test would have been downgraded or dropped. We did not run static analysis tools on this practice review. Severity: impact × likelihood.

## 3. Findings

### High · H-01 · `setPassword()` has no access control, so anyone can overwrite the owner's password

**Location:** `PasswordStore.sol#L26-L29`

The NatSpec says "only the owner can set a new password", and `getPassword()` enforces that rule. `setPassword()` does not check `msg.sender` at all.

```solidity
function setPassword(string memory newPassword) external {   // no owner check
    s_password = newPassword;
    emit SetNewPassword();
}
```

**Impact:** any address can replace the stored password at any time. The owner then reads back, and may rely on, a value an attacker chose. In our test, an attacker overwrites the password in one call and the owner reads `"hacked"`.

**Proof of concept** (passes on commit `63541e5`, fails after the fix):

```solidity
function test_H01_anyoneCanSetPassword() public {
    vm.prank(attacker);
    store.setPassword("hacked");               // attacker is not the owner

    vm.prank(owner);
    assertEq(store.getPassword(), "hacked");   // owner reads attacker's value
}
```

**Fix:** add the same owner check that `getPassword()` uses. Also add a test asserting that a non-owner call reverts.

```solidity
function setPassword(string memory newPassword) external {
    if (msg.sender != s_owner) revert PasswordStore__NotOwner();
    s_password = newPassword;
    emit SetNewPassword();
}
```

### High · H-02 · The "private" password is readable by anyone from contract storage

**Location:** `PasswordStore.sol`, `s_password` (storage slot 1)

`private` in Solidity only stops other contracts from reading a variable through Solidity. All contract storage is public on the blockchain. `s_owner` sits in slot 0 and `s_password` in slot 1, and anyone can read slot 1 directly with `eth_getStorageAt`. The owner check in `getPassword()` does not help, because the attacker never calls that function.

**Impact:** the password is exposed to everyone, which defeats the contract's stated purpose ("others won't be able to see"). In our test, a non-owner reads slot 1 and decodes `"myPassword"`.

**Proof of concept** (passes on commit `63541e5`):

```solidity
function test_H02_passwordReadableFromStorage() public {
    bytes32 raw = vm.load(address(store), bytes32(uint256(1)));
    uint256 len = uint8(raw[31]) / 2;          // short string: last byte = length * 2
    bytes memory b = new bytes(len);
    for (uint256 i = 0; i < len; i++) {
        b[i] = raw[i];
    }
    assertEq(string(b), "myPassword");
}
```

The same read works from outside with `cast storage <address> 1 --rpc-url <rpc>`.

**Fix:** never store secrets in plaintext on-chain. Keep the password off-chain. If something must be stored, store only an encrypted value or a hash, with the key kept off-chain. This is a design flaw, so there is no one-line code fix. Fixing H-01 does not change this result: the H-02 test still passes after the H-01 fix.

### Info · I-01 · Incorrect NatSpec on `getPassword()`

**Location:** `PasswordStore.sol#L33`

`getPassword()` documents `@param newPassword`, but it takes no parameters. Remove the `@param` line.

## 4. What we checked and ruled out

- **Reentrancy:** the contract makes no external calls and does not send ETH. Not exploitable.
- **Access control on reads:** `getPassword()` reverts for non-owners with `PasswordStore__NotOwner()`. The check itself is correct, but H-02 bypasses it.
- **Owner change:** `s_owner` is set once in the constructor and has no setter, so the owner cannot be changed or hijacked.
- **Arithmetic and rounding:** there is no arithmetic in the contract.
- **Compiler:** `pragma solidity 0.8.18` is pinned. No issue.

## 5. Fix verification

We verified H-01 on a local copy by applying the owner check above. The H-01 test then fails as expected with `PasswordStore__NotOwner()`, so the attack no longer works. The H-02 test still passes after the fix, confirming that H-02 needs a design change and cannot be fixed with an access-control check. This was a local check on our own patch, not a fix commit from a project team.

Test results on the unfixed commit (Forge 1.7.1, Solc 0.8.18):

```
[PASS] test_H01_anyoneCanSetPassword() (gas: 26593)
[PASS] test_H02_passwordReadableFromStorage() (gas: 10074)
Suite result: ok. 2 passed; 0 failed
```

## 6. Limits of this review

This was a time-boxed practice review of one commit and one file. It reduces risk but does not guarantee there are no bugs. Contracts that will hold significant value should also get a full audit, invariant tests and a bug bounty.

---

SENTRYX · [sentryxaudit.com](https://sentryxaudit.com) · audits@sentryxaudit.com
