# Assignment-9-Capstone-

==============================================================================
 mZAR REMITTANCE PROTOTYPE  -  README
 Notebook: A9_mZAR_Remittance_Prototype.ipynb
 Module:   BLCH9X2 (Blockchain) - Capstone, Part A
==============================================================================

CONTENTS
  1.  What this prototype does
  2.  Requirements
  3.  How to run it
  4.  Cell-by-cell run explanation (Sections 1-13)
  5.  API endpoint reference
  6.  Security and payment controls
  7.  Expected final state
  8.  What changes between runs
  9.  Troubleshooting
  10. Scope, limitations and attribution


------------------------------------------------------------------------------
1. WHAT THIS PROTOTYPE DOES
------------------------------------------------------------------------------
A worker in South Africa sends value to a family member in Zimbabwe using a
rand-backed stablecoin called mZAR.

The payment journey is:

    DEPOSIT 1,000 local units  ->  TRANSFER 750 mZAR  ->  REDEEM 750 local units
       (on-ramp / mint)            (signed, mined)        (off-ramp / burn)

Reserve rule: every 1 mZAR in circulation is backed by 1 simulated rand held
in the reserve vault. A "Reserve agent" (the issuer) signs every mint and every
payout receipt. All money, reserves and payouts are SIMULATED.

The prototype combines:
  - A proof-of-work blockchain (SHA-256, difficulty 2)
  - ECDSA (secp256k1) wallets that sign payment instructions
  - A mempool for pending transfers
  - A reserve ledger enforcing vault_fiat == circulating == sum(balances)
  - A Flask REST API and an HTML block explorer


------------------------------------------------------------------------------
2. REQUIREMENTS
------------------------------------------------------------------------------
  - Python 3.10 or later (developed on Python 3.13.9, Anaconda base env)
  - Jupyter Notebook or JupyterLab
  - Packages: flask, ecdsa
    (installed automatically by the first code cell: %pip install flask ecdsa)
  - Standard library only otherwise: json, hashlib, time, secrets, copy,
    threading, html, urllib
  - Free local port 5000 (only needed for the optional Section 13)


------------------------------------------------------------------------------
3. HOW TO RUN IT
------------------------------------------------------------------------------
  1. Open A9_mZAR_Remittance_Prototype.ipynb in Jupyter.
  2. Choose Kernel -> Restart Kernel and Run All Cells
     (or run each cell top to bottom with Shift+Enter).
  3. Cells MUST run in order. Later cells depend on objects created earlier
     (wallets, the platform, the Flask test client, the payment instruction).
  4. Every demo cell contains assert statements. If any assert fails, the cell
     stops with an AssertionError - this means the run did not behave as
     expected. A clean run raises no errors.
  5. Section 13 starts a real local web server. Open
     http://127.0.0.1:5000/explorer while it runs, then run the final cell to
     stop the server.

  If you rerun individual cells out of order, restart the kernel and run all
  cells again. The demo depends on exact state (e.g. block numbers, nonces).


------------------------------------------------------------------------------
4. CELL-BY-CELL RUN EXPLANATION
------------------------------------------------------------------------------

--- Install cell --------------------------------------------------------------
  %pip install flask ecdsa
  Installs (or confirms) Flask and the ecdsa library. Output says
  "Requirement already satisfied" if they are already present.

--- SECTION 1: Setup ----------------------------------------------------------
  Imports libraries and defines the global settings:

    ASSET                  = "mZAR"
    DIFFICULTY             = 2     (block hash must start with "00")
    REQUIRED_CONFIRMATIONS = 2     (blocks needed before funds can be redeemed)
    GENESIS_PREV           = 64 zeros (previous hash of the genesis block)

  Helper functions:
    canonical(value)   - JSON with sorted keys and no spaces, so the same data
                         always produces identical bytes (needed for hashing
                         and signing).
    digest(value)      - SHA-256 of the canonical JSON; used as tx_id.
    sha256_hex(data)   - SHA-256 of a plain string; used for block hashes.
    positive_amount(v) - rejects anything that is not a positive int
                         (blocks 0, -1, 1.5, True and "100").
    valid_address(v) / require_address(v)
                       - an address must be exactly 40 lowercase hex chars.

  Expected output: "Setup ready", the asset, difficulty 2, confirmations 2,
  and GENESIS_PREV printed as 64 zeros.

--- SECTION 2: Wallets and authorisation --------------------------------------
  Wallet class:
    - Generates a secp256k1 private (signing) key.
    - address = first 40 hex characters of SHA-256(public key).
    - sign()        signs bytes deterministically with SHA-256.
    - sign_record() signs a dictionary as canonical JSON (used by the issuer).
    - instruction() builds a signed payment instruction containing:
        kind, asset, sender, recipient, amount, timestamp, nonce, public_key
      plus "signature" (ECDSA over those fields) and "tx_id" (their digest).

  verify_instruction(tx) accepts an instruction only if:
    - amount is a positive integer, asset is mZAR, kind is transfer or burn
    - nonce is a non-negative int and timestamp is positive
    - the public key hashes to the stated sender address
      (so nobody can sign for someone else's address)
    - tx_id equals the digest of the signed fields
    - the ECDSA signature is valid

  verify_record(record, issuer_key) checks issuer-signed records (mints and
  payout receipts): the ID matches the digest and the issuer signature is
  valid.

  Three wallets are created: Sender, Recipient and Reserve agent (issuer).
  Expected output: their three 40-character addresses.

--- SECTION 3: Blockchain and mempool ------------------------------------------
  Block       - index, timestamp, transactions, previous_hash, nonce, hash.
                compute_hash() hashes every field except the stored hash.
  create_genesis_block()  - block 0, no transactions, previous_hash = 64 zeros.
  create_linked_block()   - next block pointing at the previous block's hash.
  check_genesis_previous_hash() - confirms genesis points to exactly 64 zeros.
  is_hash_valid()         - stored hash matches recomputed hash (+ PoW prefix).
  Blockchain  - holds the chain; append_block() rejects a wrong index, wrong
                previous hash, wrong hash or unmet difficulty; verify_chain()
                re-checks every block from genesis.
  mine_block()- tries nonce 0, 1, 2, ... until the hash starts with "00".
                Returns the block, number of attempts and time taken.
  Mempool     - holds only valid, signed TRANSFER instructions waiting to be
                mined (adapted from Assignment 3).

  Demonstration in this cell:
    - Prints the genesis block and confirms previous_hash is 64 zeros.
    - Builds and mines a worked-example "Block 1" linked to genesis.
      NOTE: this demo block is NOT part of the payment chain.
    - Shows that a tampered genesis (previous_hash ending in 1) fails the
      check (prints False).

--- SECTION 4: Reserve ledger and payment controls -----------------------------
  RemittancePlatform is the core of the system. It holds:
    blockchain, pool (mempool), ledger, receipts, events and a thread lock.

  The ledger contains:
    vault_fiat  - simulated rand held in reserve
    circulating - total mZAR issued
    balances    - mZAR per address
    nonces      - next expected nonce per address (replay protection)

  invariant(): vault_fiat == circulating == sum(balances), nothing negative.
  This is the 1:1 reserve rule from Lecture 10 and is checked after EVERY
  transaction.

  Key methods:
    apply(state, tx)  Applies one transaction to a ledger copy:
                       mint     - must carry a valid Reserve agent signature;
                                  raises vault_fiat, circulating and balance.
                       transfer - checks signature, nonce, balance and
                                  recipient address; moves the balance.
                       burn     - checks signature, nonce, balance and that the
                                  destination is FIAT_VAULT; lowers balance,
                                  vault_fiat and circulating.
                       Then increments the sender's nonce and re-checks the
                       invariant.
    commit(txs)       Applies txs to a COPY of the ledger, mines a block, and
                       only then replaces the real ledger. A failure leaves the
                       ledger untouched (all-or-nothing).
    onramp()          Simulated deposit. The Reserve agent signs a mint record,
                       which is committed in its own block. Refused while
                       transfers are pending.
    submit()          Accepts a transfer into the mempool after: duplicate
                       check (HTTP 409), and a projected replay of all pending
                       transfers plus the new one (prevents spending the same
                       funds twice while unmined).
    mine_pending()    Mines all pending transfers into one block (an empty
                       block if none are pending - used for confirmations).
    lookup(tx_id)     Returns status pending / mined / confirmed:
                       confirmations = chain length - block index
                       "confirmed" once confirmations >= 2.
    eligible_balance()Funds that can be redeemed: replays only blocks with
                       at least 2 confirmations, requires that the wallet has
                       no pending outgoing transfer and no unconfirmed outgoing
                       activity, and returns the lower of confirmed and current
                       balance.
    offramp(tx)       Redemption: needs a signed burn instruction and enough
                       eligible balance. Commits the burn, then the Reserve
                       agent signs a payout receipt listing the funding
                       transfer IDs, the block, and reserves after payout.
    verify_chain()    Full audit: checks every block's index, link, hash and
                       PoW, rejects duplicate tx IDs, replays every transaction
                       from an empty ledger, and requires the replayed state to
                       equal the live ledger.
    status()          Ledger + invariant_ok + chain_ok + block and pending
                       counts.

  Expected output: "Genesis ready", the issuer address, and True for both
  blockchain verification and verify_chain().

--- SECTION 5: Payment API and explorer ----------------------------------------
  Builds the Flask app (see Section 5 of this README for every endpoint) and
  an HTML explorer template that lists blocks newest-first, with readable
  names (Sender, Recipient, Reserve agent, Reserve vault).

  Important behaviour:
    - Every request takes the platform lock, so requests are processed one
      at a time.
    - DuplicateTransaction -> HTTP 409; ValueError / KeyError / TypeError
      -> HTTP 400 with an "error" message.

  A FRESH platform is created here (replacing the one from Section 4) and the
  Flask test client is set up. The helper call(method, path, body, expected)
  sends a request, prints the status code and JSON, and asserts the expected
  status code.

  Expected output: "Fresh payment API ready" and "All deposits and payouts
  are simulated".

--- SECTION 6: On-ramp - issue 1,000 mZAR --------------------------------------
  POST /onramp {address: sender, amount: 1000}  -> HTTP 201
    - Reserve agent signs a mint record; mined into BLOCK 1.
    - vault_fiat = circulating = 1000.
  GET /issuer -> issuer label, address and public key.
  The mint record is fetched from the chain and its issuer signature is
  verified against the published public key.

  Expected output: "Mint signed by the Reserve agent: True",
  "Sender mZAR: 1000".

--- SECTION 7: Sign and submit 750 mZAR ----------------------------------------
  The sender signs a transfer of 750 mZAR to the recipient (nonce 0).
  POST /transactions/new -> HTTP 201, status "pending", block_index 2.
  GET /transactions/<tx_id> -> status "pending", 0 confirmations.
  The recipient balance is still 0 because nothing is mined yet.

  Expected output: "Public-key signature verified: True" and
  "Recipient confirmed balance before mining: 0".

--- SECTION 8: Mine and verify the payment -------------------------------------
  Step 1  POST /mine -> transfer mined into BLOCK 2. The response shows the
          proof-of-work effort (difficulty, nonce, attempts, seconds).
  Step 2  Lookup: 1 confirmation, status "mined". Recipient now holds 750.
  Step 3  The recipient signs a burn of 750 to FIAT_VAULT and tries to redeem.
          POST /offramp -> HTTP 400 "Funds have not reached required
          confirmations". This is deliberate: only 1 of 2 confirmations.
          vault_fiat is still 1000.
  Step 4  POST /mine with no pending transfers -> empty confirmation BLOCK 3.
  Step 5  Lookup: 2 confirmations, status "confirmed".

--- SECTION 9: Off-ramp - redeem 750 mZAR --------------------------------------
  POST /offramp with the same signed burn -> HTTP 201.
    - Burn mined into BLOCK 4.
    - vault_fiat and circulating fall from 1000 to 250.
    - Recipient balance becomes 0.
    - A Reserve-agent-signed receipt is returned with: burn_tx_id,
      paid_local 750, payout "SIMULATED", block and block hash,
      funding_tx_ids (the 750 transfer), reserves after payout, receipt_id.
  GET /receipts/<receipt_id> returns the identical receipt, and its signature
  verifies against the issuer key.

  Expected output: "Recipient simulated payout: 750 local units",
  "Receipt signed by the Reserve agent: True", and the funding tx ID.

--- SECTION 10: Final balances and rejected payments ---------------------------
  GET /status shows: sender 250, recipient 0, vault 250, circulating 250,
  5 blocks, 0 pending, invariant_ok true, chain_ok true.

  Three attacks are then attempted - all are rejected:
    a) Insufficient funds: sender tries to send 300 (only has 250)
       -> HTTP 400 "Insufficient balance".
    b) Tampered signature: a signed 10 mZAR instruction is edited to 20
       -> HTTP 400 "Invalid signature or transaction ID".
    c) Replay: the already-mined 750 payment is resubmitted
       -> HTTP 409 "Duplicate transaction".

  The cell finally asserts that status is exactly the same as before the
  attacks: "All rejected attempts left balances and reserves unchanged."

--- SECTION 11: Explorer and final verification --------------------------------
  - GET /verify -> verify_chain true, invariant_ok true, length 5,
    difficulty 2.
  - Prints a payment summary for the 750 transfer (tx ID, block 2, block
    hash, 3 confirmations, sender, recipient, amount, status, receipt ID).
  - Renders the HTML explorer inline in the notebook, showing blocks 4 -> 0:
        Block 4  burn      Recipient -> Reserve vault (payout)   750
        Block 3  (empty confirmation block)
        Block 2  transfer  Sender -> Recipient                    750
        Block 1  mint      Reserve agent -> Sender               1000
        Block 0  (genesis)
  - Tamper test: a COPY of the chain has the payment amount changed to 751.
    verify_chain() on that copy returns False ->
    "Altered-chain test: rejected".

  Expected final line: "LIVE DEMONSTRATION COMPLETE".

--- SECTION 12: Additional checks ----------------------------------------------
  Runs 14 extra tests on a SEPARATE test platform, so the presentation state
  is not disturbed:
     1. Invalid deposit amounts (0, -1, 1.5, True, "100") return HTTP 400 and
        change nothing.
     2. Pending transfers reserve funds: after a pending 70 of 100, a second
        40 is rejected.
     3. A signature cannot authorise another wallet's address.
     4. A block re-hashed without valid proof of work is rejected.
     5. An empty chain is invalid.
     6. Uppercase addresses are rejected for transfers and deposits.
     7. Forged mints (unsigned, or signed by the wrong key) are rejected -
        even when inserted into a fully re-mined copy of the chain.
     8. A wallet can redeem while a DIFFERENT wallet has a pending transfer.
     9. A receipt with an edited payout amount fails verification.
    10. append_block() rejects a block that does not link to the tip.
    11. The timestamp is signed: changing it breaks the signature.
    12. Submitting the same pending instruction twice returns HTTP 409.
    13. Hashing is deterministic (same contents -> same hash).
    14. The main demonstration state is unchanged.

  Expected output: "Additional checks passed" and
  "Presentation state preserved".

--- SECTION 13: Optional localhost explorer ------------------------------------
  With RUN_LOCAL_SERVER = True, starts the same Flask app on a background
  thread at 127.0.0.1:5000 (localhost only, never 0.0.0.0, per Lecture 08).
  If the cell is rerun, the earlier server is shut down first.
  It calls /status to confirm the server works.

  Expected output: "Local server running.", chain verification True, reserve
  invariant True, and the link http://127.0.0.1:5000/explorer.
  A log line such as "GET /status HTTP/1.1 200" is normal.

  Set RUN_LOCAL_SERVER = False to skip this step.

--- Stop server cell -----------------------------------------------------------
  Shuts down the local server and frees port 5000.
  Expected output: "Local server stopped." (or "No local server is running.")


------------------------------------------------------------------------------
5. API ENDPOINT REFERENCE
------------------------------------------------------------------------------
  METHOD  PATH                      PURPOSE                           SUCCESS
  POST    /onramp                   Simulated deposit -> signed mint  201
  POST    /transactions/new         Queue a signed transfer           201
  POST    /transfer                 Alias of /transactions/new        201
  POST    /mine                     Mine pending transfers            200
  POST    /offramp                  Redeem via signed burn + receipt  201
  GET     /receipts/<receipt_id>    Fetch a payout receipt            200/404
  GET     /issuer                   Reserve agent public key          200
  GET     /verify                   Chain + invariant check           200
  GET     /status                   Full ledger and health            200
  GET     /chain                    Full chain as JSON                200
  GET     /transactions/<tx_id>     Transaction status lookup         200/404
  GET     /explorer/rows            Latest 10 blocks (summary rows)   200
  GET     /explorer                 HTML block explorer               200

  Errors: 400 for invalid input / failed checks, 409 for duplicates,
  404 for unknown receipts or transactions.


------------------------------------------------------------------------------
6. SECURITY AND PAYMENT CONTROLS
------------------------------------------------------------------------------
  Control                       Where it is enforced
  ----------------------------  ----------------------------------------------
  ECDSA signatures              verify_instruction() for transfers and burns
  Address bound to public key   verify_instruction()
  Issuer-only minting           apply() -> verify_record() with issuer key
  Signed payout receipts        offramp() -> issuer.sign_record()
  Replay protection (nonces)    apply() requires nonce == next expected nonce
  Duplicate rejection (409)     submit() checks mempool and chain
  No double-spend while pending submit() replays pending transfers first
  Confirmation depth (2)        eligible_balance() before any redemption
  1:1 reserve invariant         apply() after every transaction
  Atomic commits                commit() works on a ledger copy
  Tamper evidence               verify_chain() replays the full ledger
  Proof of work                 mine_block(), append_block(), verify_chain()
  Strict amounts / addresses    positive_amount(), require_address()
  Serialised API requests       Flask before_request / teardown_request lock


------------------------------------------------------------------------------
7. EXPECTED FINAL STATE (main demonstration)
------------------------------------------------------------------------------
  Blocks:        5 (genesis, mint, transfer, empty confirmation, burn)
  Sender:        250 mZAR   (1000 deposited - 750 sent)
  Recipient:     0 mZAR     (750 received - 750 redeemed)
  Vault (fiat):  R250
  Circulating:   250 mZAR
  Pending:       0
  invariant_ok:  True
  verify_chain:  True


------------------------------------------------------------------------------
8. WHAT CHANGES BETWEEN RUNS
------------------------------------------------------------------------------
  New private keys are generated every run, and timestamps change, so these
  values WILL differ from the saved notebook output:
    - wallet addresses and public keys
    - transaction IDs, signatures, block hashes, receipt IDs
    - mining nonces, attempt counts and seconds
  The amounts, block numbers, statuses, HTTP codes and final balances are
  the same on every run.


------------------------------------------------------------------------------
9. TROUBLESHOOTING
------------------------------------------------------------------------------
  ModuleNotFoundError: flask / ecdsa
      Run the install cell, then restart the kernel.

  AssertionError in a demo cell
      Cells were probably run out of order or twice. Restart the kernel and
      run all cells from the top.

  "Mine pending transfers before changing reserves"
      /onramp was called while transfers were pending. Call /mine first.

  "Invalid nonce or repeated instruction"
      The instruction used the wrong nonce. Read the next nonce from
      platform.ledger["nonces"].

  OSError: address already in use (Section 13)
      Port 5000 is taken (another server, or an earlier run). Run the stop
      cell, restart the kernel, or change 5000 to another port.

  Explorer link does not open
      The server only runs while the kernel is alive and before the stop
      cell is run.


------------------------------------------------------------------------------
10. SCOPE, LIMITATIONS AND ATTRIBUTION
------------------------------------------------------------------------------
  Scope and limitations
    - All deposits, reserves and payouts are simulated. Real FX conversion
      and real cross-border payouts are out of scope.
    - Single process, in-memory state: no distributed consensus, no multi-node
      fork resolution, no real bank custody, no AML/KYC.
    - /onramp is open on localhost and stands in for a trusted issuer; in
      production it would sit behind KYC/AML checks and operator
      authentication.
    - verify_chain() proves the records are consistent, not that real bank
      reserves exist.
    - Difficulty 2 and 2 confirmations are classroom settings, not production
      security levels.
    - No mining reward is paid in mZAR, because unbacked issuance would break
      the 1:1 reserve rule.

  Design note
    Lecture 10 treats on-ramp and off-ramp as issuer-ledger events. This
    prototype also records mints and burns on-chain, so every reserve change
    has signed on-chain evidence, and redemption requires the owner's signed
    burn instruction.

  Reused components (group semester assignments)
    A1-A2  SHA-256 hashing, canonical JSON, blocks, proof of work, tampering
    A3     Transaction schema and mempool (adapted to signed dictionaries)
    A4     ECDSA wallets, addresses, signing and verification
    A5-A6  Blockchain, append_block, verify_chain, mining
    A7     Confirmation depth and probabilistic finality
    A8     Flask API, HTML explorer, localhost demonstration

  Stablecoin flow and reserve accounting (vault_fiat, circulating,
  invariant_ok) guided by Prof. Jules Mba's BLCH9X2 Lecture 10 slides and
  live-coding notebook (6 October 2026).

  AI declaration: OpenAI ChatGPT and Anthropic Claude assisted with code
  adaptation, integration, review, debugging and testing. Group members are
  responsible for reviewing, understanding and verifying the implementation.

==============================================================================
