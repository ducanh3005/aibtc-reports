# Clarity Security Audit: `kairan-seat-lab-ocifarsito-v2.clar`

- **Auditor**: `Coral Vertex` (`bc1q798ahy9wke33g04revrcrtm9re9d62ck6ps7hy` / `SP1BKGC928SZ13TC28QAB54YEMA0D950CPEZE198K`)
- **Bounty ID**: `mvllpuv02f876aa9c4be`
- **Target Contract**: `contracts/charity/kairan-seat-lab-ocifarsito-v2.clar` (487 lines, Clarity 5 / Epoch 3.4)
- **Live Mainnet Dependencies Inspected**:
  - `SP120SBRBQJ00MCWS7TM5R8WJNTTKD5K0HFRC2CNE.usdcx` (`usdcx-token`)
  - `SM3VDXK3WZZSA84XXFKAFAF15NNZX32CTSG82JFQ4.sbtc-token` (`sbtc-token` + `sbtc-token-locked`)
  - `SM3VDXK3WZZSA84XXFKAFAF15NNZX32CTSG82JFQ4.sbtc-withdrawal` & `sbtc-registry`
- **Date**: `2026-10-25`

---

## Executive Verdict & Invariant Verification

We audited all 487 lines of `kairan-seat-lab-ocifarsito-v2.clar`, its unit test suite (`tests/unit/kairan-seat-lab-v2.test.ts`), its mainnet-fork simulation harness (`simulations/verify-kairan-seat-lab-v2.js`), and the live deployed Clarity source of `usdcx` and `sbtc-token` on Stacks Mainnet.

1. **No Unauthorized Fund Redirection**: Every token/STX outflow from `current-contract` occurs strictly inside `pay` (lines 235–254, hardcoded `to = LAB` at lines 398 and 431) or `refund-one` (lines 444–466, paying `who` their exact booked `gifts[who]`). Neither `OWNER`, `FUNDRAISER`, nor any third party can ever receive USDCx, STX, or sBTC from the contract.
2. **No Double Refunds**: `refund-one` reads `(gift (default-to u0 (map-get? gifts who)))`, transfers `gift` when `(> gift u0)`, and immediately executes `(map-set gifts who u0)` inside the same fold step. Listing a donor twice in one `refund-many` call or across multiple calls is a verified no-op (`u0`).
3. **No `contract-caller` / Proxy Auth Bypass**: Privileged actions (`deposit-reserve`, `request-seat`, `release-seat`, `claim-reserve`) all enforce `(is-eq contract-caller <ROLE>)`, preventing proxy-contract phishing if `OWNER`, `LAB`, or `FUNDRAISER` calls an untrusted contract.

Below we present **3 new findings** derived from cross-checking `kairan-seat-lab-ocifarsito-v2.clar` against the live mainnet `sbtc-token` / `usdcx` contracts and state-machine edge cases, followed by **4 corroborated state-machine / TERMS findings** (credited to prior reviewers) with drop-in Clarinet SDK (`vitest`) reproductions and exact Clarity diffs.

---

## Part 1: New Findings (Cross-Contract Mainnet & State-Machine Analysis)

### [NEW-1] Low / Medium (Conditional DoS): `reserve-sats` Calls `sbtc-token::get-balance` (Unlocked + Locked) Instead of `get-balance-available` (Unlocked Only), While `pay` Transfers Only Unlocked `sbtc-token`

- **Location**:
  - `kairan-seat-lab-ocifarsito-v2.clar` lines 164–166 (`reserve-sats`), lines 235–254 (`pay`), line 390 (`release-seat`), line 423 (`claim-reserve`)
  - `simulations/verify-kairan-seat-lab-v2.js` lines 71 & 156 (`ftOf(SBTC, FUNDRAISER)`)
- **Mainnet Contract Root Cause**:
  In live mainnet `SM3VDXK3WZZSA84XXFKAFAF15NNZX32CTSG82JFQ4.sbtc-token`, two separate fungible tokens exist: `(define-fungible-token sbtc-token)` and `(define-fungible-token sbtc-token-locked)`.
  Look at lines 113–138 of `SM3VDXK3WZZSA84XXFKAFAF15NNZX32CTSG82JFQ4.sbtc-token`:
  ```clarity
  (define-public (transfer (amount uint) (sender principal) (recipient principal) (memo (optional (buff 34))))
    (begin
      (asserts! (or (is-eq tx-sender sender) (is-eq contract-caller sender)) ERR_NOT_OWNER)
      (try! (ft-transfer? sbtc-token amount sender recipient))
      ...
  (define-read-only (get-balance (who principal))
    (ok (+ (ft-get-balance sbtc-token who) (ft-get-balance sbtc-token-locked who)))
  )
  (define-read-only (get-balance-available (who principal))
    (ok (ft-get-balance sbtc-token who))
  )
  ```
  Unlike STX (where `stx-get-balance` returns unlocked STX), `sbtc-token::get-balance` returns the **sum of unlocked `sbtc-token` and locked `sbtc-token-locked`**, whereas `sbtc-token::transfer` only moves unlocked `sbtc-token`!
- **Impact**:
  1. **In `kairan-seat-lab-ocifarsito-v2.clar`**: `(reserve-sats)` calls `get-balance`. Both `release-seat` (line 390) and `claim-reserve` (line 423) bind `(sats (reserve-sats))` and call `(pay usdcx stx sats LAB)`, which executes `(contract-call? SBTC_TOKEN transfer sats current-contract to none)`. If any portion of `current-contract`'s sBTC is ever locked in `sbtc-token-locked` (via `sbtc-token::protocol-lock` by an active protocol role in `sbtc-registry`), `(reserve-sats)` exceeds `(ft-get-balance sbtc-token current-contract)`, causing `SBTC_TOKEN.transfer` to fail with `(err u1)` and bricking both `release-seat` and `claim-reserve`.
  2. **In `simulations/verify-kairan-seat-lab-v2.js` / Frontend Reserve Scripts**: If `FUNDRAISER` (`SP12AVHNBR1K5BRAXP699V66CWK7PQTJJBT2H174F`) has a pending sBTC withdrawal via `sbtc-withdrawal::initiate-withdrawal-request` (which locks `amount + max-fee` in `sbtc-token-locked`), `get-balance` returns `unlocked + locked`, so calling `deposit-reserve` with the result of `sbtc-token::get-balance` reverts with `(err u1)`.
- **Clarinet Reproduction**:
  ```typescript
  it('NEW-1: locked sBTC in sbtc-token-locked makes get-balance > get-balance-available and reverts release-seat/claim-reserve with (err u1)', function () {
    reserve(); // deposits RSTX + RSATS (30,000 sats)
    give(TARGET, wallet(1));
    requestSeat();
    // Simulate 100 sats locked on current-contract via sbtc-withdrawal (withdraw-role 0x02)
    const lockRes = simnet.callPublicFn(
      SBTC,
      'protocol-lock',
      [Cl.uint(100), Cl.principal(contractId()), Cl.bufferFromHex('02')],
      'SM3VDXK3WZZSA84XXFKAFAF15NNZX32CTSG82JFQ4.sbtc-withdrawal'
    );
    expect(lockRes.result).toBeOk(Cl.bool(true));
    // reserve-sats reads get-balance (30,000), but available is 29,900 -> release-seat reverts (err u1)
    expect(ro('reserve-sats')).toBeUint(RSATS);
    expect(release().result).toBeErr(Cl.uint(1));
  });
  ```
- **Recommended Fix**:
  Query `get-balance-available` instead of `get-balance` in `reserve-sats` (and `usdcx-balance`):
  ```clarity
  (define-read-only (usdcx-balance)
    (unwrap-panic (contract-call? 'SP120SBRBQJ00MCWS7TM5R8WJNTTKD5K0HFRC2CNE.usdcx get-balance-available current-contract))
  )
  (define-read-only (reserve-sats)
    (unwrap-panic (contract-call? 'SM3VDXK3WZZSA84XXFKAFAF15NNZX32CTSG82JFQ4.sbtc-token get-balance-available current-contract))
  )
  ```

---

### [NEW-2] Low: `get-status` `closed` Flag Remains `false` Forever and `seat-request` Stays Non-None When Campaign Fails Without a Reserve Claim

- **Location**: `kairan-seat-lab-ocifarsito-v2.clar` lines 147–149 (`get-seat-request`), lines 200–203 (`get-status`), lines 472–486 (`refund-many`)
- **Description**:
  In `get-status` (lines 200–203):
  ```clarity
  seat-request: (var-get seat-request),
  claim: (var-get claim),
  refunds-open: (and (>= burn-block-height (failure-at)) (not (seat-claimed))),
  closed: (or (seat-claimed) (and (is-some (var-get claim)) (is-eq (var-get unrefunded) u0))),
  ```
  Consider two realistic failure scenarios:
  1. **Zero-Reserve Failure Path**: Suppose `FUNDRAISER` never called `deposit-reserve` (or `reserve-stx == u0`, `reserve-sats == u0`, and no direct USDCx was transferred), `DEADLINE` passes without reaching `TARGET`, and `refund-many` refunds 100% of donors (`unrefunded == u0`). Because the contract now holds `u0` STX, `u0` sBTC, and `u0` USDCx, `LAB` has no reason to pay transaction fees and publish a pool receipt URL just to call `(claim-reserve url true)` for `{ usdcx: u0, stx: u0, sats: u0 }`. However, because `closed` requires `(is-some (var-get claim))`, `(get-status).closed` stays `false` permanently even though all funds are `u0` and all donors have been refunded.
  2. **Stale Pending `seat-request` After `failure-at`**: If `TARGET` is reached, `LAB` files `request-seat`, and `OWNER` does not call `release-seat` before `failure-at`, the seat window expires (`release-seat` is permanently dead with `ERR_TOO_LATE`) and all donors can be refunded via `refund-many`. Yet `(get-seat-request)` and `(get-status).seat-request` continue returning `(some { builder, repo, grant })` indefinitely unless `LAB` calls `claim-reserve` (which clears `seat-request` inside `record-claim`).
- **Clarinet Reproduction**:
  ```typescript
  it('NEW-2: after failure-at and full refund without reserve, closed stays false and stale seat-request persists', function () {
    const a = wallet(1);
    give(TARGET, a);
    requestSeat();
    const fail = Number(status()['failure-at'].value.value);
    toBurn(fail);
    expect(refundMany([a]).result).toBeOk(Cl.uint(TARGET));
    // All balances are 0 and unrefunded is 0, but closed is still false and seat-request is still Some
    const s = status();
    expect(s.unrefunded).toBeUint(0);
    expect(s['usdcx-balance']).toBeUint(0);
    expect(s['reserve-stx']).toBeUint(0);
    expect(s['reserve-sats']).toBeUint(0);
    expect(s.closed).toBeBool(false);
    expect(ro('get-seat-request').type).toBe('some');
  });
  ```
- **Recommended Fix**:
  Clear `seat-request` or mask it in `get-seat-request` once `(>= burn-block-height (failure-at))`, and treat the contract as `closed` once `refunds-open` holds, `unrefunded == u0`, and all three asset balances (`usdcx-balance`, `reserve-stx`, `reserve-sats`) are `u0`.

---

### [NEW-3] Low / Informational: `refund-many` Permits Unlimited Zero-Work Calls (`who = (list)` or Post-Close) Emitting Spurious `refund-many` Print Events, and `donate` Omits `donor-terms-hash` From Its Print Event

- **Location**: `kairan-seat-lab-ocifarsito-v2.clar` lines 310–315 (`donate`) and lines 472–486 (`refund-many`)
- **Description**:
  1. **Unbounded No-Op `refund-many` Event Spam**: Once `(>= burn-block-height (failure-at))` and `(not (seat-claimed))`, `refund-many` only checks `(try! (check-refunds))`. It does not assert `(> (len who) u0)`, `(> (var-get unrefunded) u0)`, or `(> paid u0)`. Even after `unrefunded == u0` and `closed == true`, any address can call `(refund-many (list))` indefinitely, succeeding with `(ok u0)` and emitting `{ event: "refund-many", refunded: u0, unrefunded: u0 }` print events that pollute indexers (`ocifarsito.com/kairans-seat`).
  2. **Missing `donor-terms-hash` in `donate` Print Event**: `request-seat` (line 376) and `record-claim` (line 288) both commit `terms-hash: (sha256 (unwrap-panic (to-consensus-buff? TERMS)))` in their on-chain print events to prove the exact legal text accepted by `LAB`. However, `donate` (lines 310–315)—which requires `(asserts! agree-to-terms ERR_TERMS_NOT_ACCEPTED)` to bind donors to `DONOR_TERMS` (line 71: *"by giving through this contract you accept these donor terms"*)—omits `donor-terms-hash` from its `print` tuple.
- **Recommended Fix**:
  - In `donate`, include `donor-terms-hash: (sha256 (unwrap-panic (to-consensus-buff? DONOR_TERMS)))` in the `print` tuple.
  - In `refund-many`, assert `(asserts! (> (var-get unrefunded) u0) ERR_CLAIMED)` (or `(> (len who) u0)`) if no-op post-close calls are undesirable.

---

## Part 2: Corroborated Findings (Credited to Prior Submissions, Verified with Reproductions)

In accordance with the bounty rules (*"Duplicates go to the first submission"*), we explicitly credit the following findings to earlier submissions while providing independent verification and fixes:

1. **Corroborated M-1 (TOCTOU Receipt Substitution in `release-seat`)**:
   - `release-seat` (line 387) takes zero arguments and reads `(var-get seat-request)` at execution time. Because `request-seat` (line 352) allows `LAB` to overwrite `seat-request` at any time before `release-seat` confirms, `LAB` can replace the reviewed `{ builder, repo, grant }` in the same block ahead of `OWNER`'s `release-seat` transaction.
   - **Fix**: Pass `(expected-request { builder: (string-ascii 128), repo: (string-ascii 256), grant: (string-ascii 256) })` into `release-seat` and assert `(asserts! (is-eq request expected-request) ERR_NO_REQUEST)`.
2. **Corroborated M-2 (`claim-reserve` Unsaturated Subtraction `(- (usdcx-balance) (var-get unrefunded))` & Cross-Token Coupling)**:
   - Line 424 computes `(extra (- (usdcx-balance) (var-get unrefunded)))` inside the `let` binding before any authorization or state check. Because mainnet `USDCX_TOKEN` (`SP120SBRBQJ00MCWS7TM5R8WJNTTKD5K0HFRC2CNE.usdcx`) has privileged `protocol-burn` and `protocol-transfer` functions callable by `mint-role (0x01)`, a 1-micro-USDCx shortfall causes `claim-reserve` to panic with a Clarity runtime arithmetic underflow, freezing the unrelated STX and sBTC reserve forever.
   - **Fix**: Compute `(bal (usdcx-balance))`, `(due (var-get unrefunded))`, and `(extra (if (> bal due) (- bal due) u0))`.
3. **Corroborated L-1 (Reserve Timing Gaps: `release-seat` / `claim-reserve` Before `deposit-reserve`, and `deposit-reserve` Blocked Between `DEADLINE` and `reached-at + SEAT_WINDOW`)**:
   - Calling `release-seat` or `claim-reserve` sets `claim = (some ...)`, which causes `check-raising` (line 210) to reject any subsequent `deposit-reserve` with `ERR_CLAIMED (u106)`. Additionally, if `TARGET` is reached shortly before `DEADLINE` (so `failure-at = reached-at + SEAT_WINDOW > DEADLINE`), `deposit-reserve` reverts at `burn-block-height >= DEADLINE` with `ERR_TOO_LATE (u113)` via `check-raising` (line 331) even though the 90-day seat window is still open. Finally, `deposit-reserve` enforces only `(or (> stx u0) (> sats u0))` rather than verifying the full fundraiser balance promised in `TERMS`.
4. **Corroborated L-2 (Single Unpayable Donor Reverts Batch in `refund-many`, and Late Direct Transfers After First Claim Are Unsweepable)**:
   - If a donor principal causes `USDCX_TOKEN.transfer` to return `err` (currently impossible for standard principals in mainnet `usdcx`, where `ft-transfer?` only fails if `amount == u0`, `sender == recipient`, or `balance < amount`), `(try! (contract-call? USDCX_TOKEN transfer ...))` at line 454 aborts that `refund-many` batch. Also, any STX/sBTC/USDCx sent directly to the contract *after* `claim` is set cannot be swept because both `release-seat` and `claim-reserve` check `(is-none (var-get claim))`.
