How do we *distribute* the yield to our community and fund public goods?

That's where the **Governance Engine** comes in. In this part, we'll build the "Capital Allocation" half of our system. We'll learn how to create a staking pool to engage our community and a secure grants program to distribute our funds.

---

## **Part 4: The Governance Engine (Distributing Your Funds)**

This section covers the two key components for building a community-driven allocation system.

### 🗳️ The "Staking Toolkit": Regen Staker

The `Regen Staker` is not a single, one-size-fits-all contract. Instead, it's a **"Staking-as-a-Service" toolkit.** Think of it as a *factory* that builds and deploys a secure, custom-configured staking pool just for your community.

* **Concept:** A "factory" (`RegenStakerFactory`) that you use to build and deploy your own staking pool.
* **Purpose:** To let your community members (the "Stakers") lock up your native token. In exchange, they get:
    1.  **Rewards:** A share of the yield you're generating (from your Part 3 vaults).
    2.  **Power:** The ability to *participate* in the allocation process (i.e., vote in your new grants program).

#### Key Architecture
When you use the "factory," you get to choose your "Lego blocks":

* **The "Contract-Maker" (`RegenStakerFactory`):** This is the main "factory" contract, deployed by Octant. You call it, feed it your config, and it builds and deploys your unique staking contract.
* **The 2 Variants (Your "Chassis"):** You must choose one:
    1.  `WITH_DELEGATION`: The "governance" flavor. Use this if your stake token is a governance token (like `COMP` or `UNI`). It allows your users to delegate their voting power *while* their tokens remain safely staked. It's more complex and costs more gas.
    2.  `WITHOUT_DELEGATION`: The "simple" flavor. Use this for standard ERC20 tokens where on-chain voting delegation isn't needed. It's simpler and much cheaper for your users.
* **The "Plugins" (Your "Custom Features"):**
    * `EarningPowerCalculator`: This is your custom "Rewards Rulebook." You can write logic here to, for example, give a 2x reward multiplier to users on an "OG Member" whitelist.
    * `Whitelists`: These are three optional "Bouncer Lists" for access control:
        * `Staker Whitelist`: Who is *allowed* to stake?
        * `Contribution Whitelist`: Who is *allowed* to add rewards to the pool? (e.g., only your DAO's multisig).
        * `Allocation Mechanism Whitelist`: A key security feature. Where are stakers *allowed* to donate their rewards? (Prevents users from sending funds to a malicious contract).

#### The User Flow
Once deployed, your community members can:
* `stake()`: Lock up their tokens.
* `claimReward()`: Collect their share of the yield.
* `contribute()`: This is the "Regen" part. Instead of claiming rewards for themselves, a user can **donate** their rewards directly to a public goods project (one that's on your `Allocation Mechanism Whitelist`).

> **Critical Security Warning:** The Regen Staker system is built for standard ERC20 tokens. It **DOES NOT** support fee-on-transfer, rebasing, or deflationary tokens. Using a "weird token" *will* break the accounting math and lead to lost funds.

---

### 🏛️ The "Grants Program": Tokenized Allocation Mechanism (TAM)

Now we have a way for our community to get rewards and voting power. But how do they *use* that power?

This is where the **Tokenized Allocation Mechanism (TAM)** comes in. It's the secure, on-chain framework for running your entire grants program.

* **Concept:** This is the **"Engine vs. Driver's Seat"** pattern from Part 2, but applied to **governance**.
* **The "Engine" (`TokenizedAllocationMechanism`):** The shared, audited "guts" that handle the *entire, complex lifecycle* of a grant round (timing, state, timelocks, accounting).
* **The "Driver's Seat" (`BaseAllocationMechanism`):** The *simple proxy* you build that only contains your *policy* (your custom voting rules).

#### Key Mechanism: Quadratic Funding (QF)
TAM is built to handle *any* voting mechanism, but its "Quadratic Funding (QF)" flavor is a key feature.

* **What is QF?** It's a democratic voting system. It values **broad community support** (many small votes) far more than **deep pockets** (one big vote).
* **How?** The cost of votes is *quadratic*. Your 1st vote-credit might cost $1, your 2nd $2, your 3rd $3, and so on. This means 100 people buying 1 vote-credit each is *far* more powerful than 1 person buying 100.
* **The `α` (Alpha) "Slider":** This is a "knob" in the QF math: $F_j = \alpha(\sum_i \sqrt{c_{i,j}})^2 + (1-\alpha)\sum_i c_{i,j}$
    * `α = 1`: **Pure Quadratic.** Only broad support (the `∑√c` part) matters.
    * `α = 0`: **Pure Linear.** It's just a 1-dollar-1-vote poll (the `∑c` part).
    * `α = 0.5`: A **blend** of both.

#### The 7-Step Secure Lifecycle
When you use TAM, the "Engine" *gives you this entire secure lifecycle for free*. This is the *process* it manages for you:

1.  **Register:** Voters sign up and get their voting power.
2.  **Propose:** Authorized users submit their grant proposals.
3.  **Vote:** The community votes during the open "voting window."
4.  **Finalize (Owner-gated):** An admin (your DAO) calls this to end the voting and "lock in" the results.
5.  **Queue (Permissionless):** *Anyone* can now call `queueProposal()` for a *winning* proposal. This "puts it in line" for funding.
6.  **Timelock (The "Safety Net"):** This is a mandatory waiting period after a proposal is queued. It gives the DAO's "emergency admins" time to spot a malicious proposal and veto it.
7.  **Redeem / Sweep (The Payout):** After the timelock, the "redemption window" opens. The winning project can now call `redeem()` to claim their funds. After the window closes, the admin can `sweep()` any *unclaimed* funds.

#### Developer Guide: Building a TAM (Foundry Tutorial)
The best way to understand this is to build one. Let's walk through the "1-Person-1-Vote" (1p1v) tutorial.

* **The Goal:** Build a "Driver's Seat" (a mechanism) with a "1-Person-1-Vote" rulebook.
* **The "Filing Cabinet" (`OPOV_SLOT`):**
    * **Problem:** Your 1p1v rules need to *store* data (like who has voted). How do you do this in your proxy ("Driver's Seat") without "colliding" with the "Engine's" own storage?
    * **Solution:** Use **EIP-1967 Unstructured Storage**. This is like creating a "filing cabinet" (`OPOV` struct) and giving it a unique, secret key (`OPOV_SLOT`). You tell the blockchain, "Put my cabinet at this *exact* secret location." This guarantees it will *never* interfere with the "Engine's" standard data.
* **Implementing the Hooks (Your "Rulebook"):**
    Your job is just to write the "hooks" that define your 1p1v rules. The "Engine" will call them at the right time.

    * **`_beforeSignupHook` & `_getVotingPowerHook` (The "Bouncer"):**
        * This is the 1-power-once logic.
        * When a user signs up, you check the "Engine's" records: `_tokenizedAllocation().votingPower(user) == 0 ? 1 : 0;`
        * This line means: "If this user has *zero* power, give them 1. If they already have power (even from a re-registration), give them 0."
    * **`_processVoteHook` (The "Ballot Box"):**
        * This is where you enforce your voting rules.
        * `require(!s.voted[pid][voter], "already voted");` (One vote per person, per proposal).
        * `require(weight == 1, "1p1v: weight must be 1");` (Each vote has a "weight" of exactly 1).
        * `return oldPower - 1;` (It "costs" 1 power unit to vote).
    * **`_hasQuorumHook` (The "Win-Checker"):**
        * The "Engine" asks: "Did proposal `pid` win?"
        * You define the "win condition": `s.forVotes[pid] >= minVotes && s.forVotes[pid] > s.againstVotes[pid];`
    * **`_beforeFinalize...` & `_convertVotesToShares` (The "Prize Calculator"):**
        * This is a two-step process for calculating the *pro-rata* (proportional) payout.
        * **`_beforeFinalize...`:** Runs *once* when the admin finalizes the round. You loop through all proposals, check which ones *won* (using your `_hasQuorumHook`), and sum their `forVotes` into a "grand total" (`totalForVotesSucceeded`).
        * **`_convertVotesToShares`:** Runs for *each* winning proposal when it's queued. You just do the simple proportional math:
            * `Prize = (Total Budget) * (Proposal's For Votes) / (Grand Total of Winning Votes)`
    * **`_availableWithdrawLimit` (The "Bank Teller's Window"):**
        * This is a critical security hook. The "Engine" asks: "Is the 'bank window' open for withdrawals?"
        * Your hook *must* enforce the "Engine's" global schedule. You return `type(uint256).max` (unlimited) *only* during the `[start, start + grace]` window.
        * At *all other times*, you return `0`. This is what enforces the timelock and allows the `sweep()` function to work.

---

And that's it! You've now built *both* halves of the Octant V2 system.

* In **Part 3**, you built the **Yield Engine** to generate a sustainable stream of funds.
* In **Part 4**, you built the **Governance Engine** to engage your community and let them securely and democratically *distribute* those funds.
