In Part 2, we mastered the core "Engine vs. Driver's Seat" architecture.

Now, let's put that theory into practice. In this part, we're going to build the **Yield Engine**-the "funding source" for your ecosystem. We'll cover the three main "Lego blocks" for generating yield.

---

## **Part 3: The Yield Engine (Building Your Funding Source)**

This section is all about **Capital Accumulation**. We'll look at the "Portfolio Manager" that holds the funds and the two primary "factories" (YDS and YSS) that *generate* the sustainable yield.

###  portfolio The "Portfolio Manager": The Multi-Strategy Vault (MSV)

Think of the MSV as a "fund-of-funds" or the "Portfolio Manager" for your treasury.

* **Concept:** A "master vault" that holds your treasury's assets (e.g., `USDC`).
* **Problem It Solves:** You don't want to put all your eggs in one basket (one strategy), but you also don't want to manually hold 5 different vault tokens.
* **Solution:** You deposit 100% of your `USDC` into *one* MSV. The MSV, in turn, allocates that capital across multiple, separate strategies based on rules *you* set.
* **Benefit:** You hold *one* master vault share that represents your entire diversified portfolio.

#### Key Mechanics
As the "Portfolio Manager," you control four key things:

* **Debt Management (The "Allocation"):**
    * "Debt" is just the term for "capital allocated to a strategy."
    * You set a `maxDebt` for each strategy, capping your risk exposure.
* **Idle Funds (The "Liquidity"):**
    * This is the `USDC` held *inside* the MSV, not deployed to any strategy.
    * Its purpose is to provide immediate liquidity for user withdrawals.
* **The Strategy Queue (The "Withdrawal Order"):**
    * This is a critical, ordered list of your strategies.
    * When a user withdraws and `Idle Funds` are empty, the MSV pulls cash from strategies in this queue's *exact order*.
    * **Pro-Tip:** Put your most liquid strategies (like an Aave vault) first in the queue.
* **Rebalancing (The "Manager's Job"):**
    * You can call `updateDebt()` at any time to rebalance your portfolio-for example, by moving 2,000,000 `USDC` from a low-yield strategy to a new, high-yield one.

---

### 🛡️ The "Donation Vault": Yield Donating Strategy (YDS)

This is the first *buildable* strategy. It's the "Driver's Seat" you'll use for simple, profit-generating assets (like `USDC`, `DAI`, etc.).

* **Concept:** The **"capital-protected" vault.**
* **The Promise:** "Deposit your `USDC` here. You **forgo** all yield, but your principal is shielded. All yield is automatically donated."
* **Architecture:** It's a fork of Yearn V3's design. The "profit sink" is just "re-plumbed." Instead of profit increasing the user's Price-Per-Share (PPS), it's *diverted* to the donation address.

#### Key Mechanism: The Donation Buffer
This is the core innovation of YDS. It protects your users' principal by using the donated profits as a "shield."

* **On Profit:** A `report()` is called. The "Engine" sees a $100 profit. It **mints $100 worth of new vault shares** and sends them to the `donationAddress`. Your user's PPS stays flat.
* **On Loss (The Buffer):** A `report()` is called. The "Engine" sees a $40 loss. It **burns $40 worth of shares from the `donationAddress`'s buffer** *first*. Your user's PPS *still* stays flat.
* **On Catastrophe (Socialized Loss):** A `report()` is called. The "Engine" sees a $200 loss. The buffer only has $60 left. The "Engine" burns the entire $60 buffer, but there is still $140 of the loss remaining. This $140 loss is now **"socialized,"** and the PPS for *all* holders (including your users) finally drops to cover it.

#### Developer Guide: Building a YDS (Foundry Tutorial)
Let's build the `sDAI` strategy from the tutorial.

* **The "Lego" Insight:** The `sDAI` vault is *already* an ERC-4626 vault. This means your YDS (also an ERC-4626) is just a thin "wrapper." You're just plugging one "USB-C" port into another. It's that simple.
* **Your "Driver's Seat" Job:** The "Engine" (`YieldDonatingTokenizedStrategy`) handles all the complex mint/burn/buffer logic. Your contract only needs to implement **three "hooks"**:

    1.  `_deployFunds(uint256 amount)`
        * **The "Engine" asks:** "I just got `amount` `DAI`. What do I do with it?"
        * **Your "Hook":** `sDAI.deposit(amount, address(this));`
    2.  `_freeFunds(uint256 amount)`
        * **The "Engine" asks:** "A user wants `amount` `DAI` back, but I'm empty. Please get it."
        * **Your "Hook":** `sDAI.withdraw(amount, address(this), address(this));`
    3.  `_harvestAndReport() returns (uint256 totalAssets)`
        * **The "Engine" asks:** "It's accounting time. What's the *total value* of all assets you manage?"
        * **Your "Hook":** `return sDAI.convertToAssets(sDAI.balanceOf(address(this))) + DAI.balanceOf(address(this));`
        * **Note:** You just *report* the total. The "Engine" does all the complex profit/loss math based on the number you return.

* **Security (Do This!):**
    * **`BaseHealthCheck` (The "Circuit Breaker"):** Inherit this. It's a safety module. You set a `profitLimitRatio` (e.g., 10%) and `lossLimitRatio` (e.g., 5%). If an oracle bug makes your `_harvestAndReport()` return a 40% profit, this health check **makes the transaction revert**, saving your vault.
    * **Mirror Upstream Limits:** Also implement `availableDepositLimit()`. Your hook should just check `sDAI.maxDeposit()`. This stops user deposits from failing (and wasting gas) if the `sDAI` vault is full.

---

### 📈 The "Skimming Vault": Yield Skimming Strategy (YSS)

This is the second "Driver's Seat" you can build. It's designed for a completely different class of asset.

* **Use Case:** For **appreciating** assets (like `stETH`, `rETH`, or other Liquid Staking Tokens).
* **Concept:** The **"price-stabilizing" vault.**
* **The Promise:** "Deposit your volatile `stETH`. You **forgo** all appreciation. In return, your **1 vault share will always be worth 1 `ETH`**."

#### Key Mechanism: The 1-to-1 Value Promise
The YSS works by "skimming" the appreciation and using it as a "buffer" against depreciation.

* **On Appreciation (The "Skim"):** `stETH` is now worth 1.05 `ETH`. The "Engine" sees this 0.05 `ETH` of "un-promised" value and **"skims"** it by **minting new shares** worth 0.05 `ETH` to the `dragonAddress`. Your user's 1 share is still worth 1 `ETH`.
* **On Depreciation (The "Buffer"):** `stETH` is now worth 0.98 `ETH`. The "Engine" **burns shares from the "Dragon Buffer"** to cover the 0.02 `ETH` loss. Your user's 1 share is *still* worth 1 `ETH`.
* **On Catastrophe (The "Insolvency"):** `stETH` de-pegs to 0.90 `ETH`. The loss is *bigger* than the entire buffer. The 1-to-1 promise is **broken.** The vault enters **"Insolvency Mode,"** and the remaining loss is socialized.

#### Developer Guide: Building a YSS
This is even simpler than a YDS.

* **Architecture:** A YDS is an *active* strategy (it has to `_deployFunds` to Aave). A YSS is a *passive* strategy-its only job is to *hold* the `stETH` and *track* its value.
* **The Developer's *One* Job:** The "Engine" (`YieldSkimmingTokenizedStrategy`) handles all the complex logic. Your "Driver's Seat" only needs to implement **one critical hook**:

    1.  `_getCurrentExchangeRate() returns (uint256)`
        * **The "Engine" asks:** "It's accounting time. What is the real-time exchange rate? How much `ETH` is 1 `stETH` worth right now?"
        * **Your "Hook":** (Your code to get the rate from an oracle).
        * **Note:** The *entire* skimming, buffering, and solvency system runs off the single number *you* provide in this function.

---

And there you have it! You now know how to build the *entire* "funding source" of your ecosystem.

* You can use the **MSV** as your "Portfolio Manager" to diversify your treasury.
* You can build a **YDS** (in 3 simple functions) to donate the *profit* from your stablecoins.
* You can build a **YSS** (in 1 simple function) to donate the *appreciation* from your LSTs.

You now have a powerful, automated engine for generating a sustainable yield stream. But what's the point of all this yield if you can't *distribute* it?
