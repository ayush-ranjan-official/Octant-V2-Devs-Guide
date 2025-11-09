# Octant V2 Under the Hood: A Developer's Guide to the Core Architecture

![Logo](Assets/ocatant_logo.svg)

Hello and Welcome! This repository contains the complete 5-part technical guide to Octant V2. This tutorial is designed to take you from the core vision and architecture to writing, testing, and deploying your own custom Yield Strategies (YDS) and Governance Mechanisms (TAM).

This tutorial is a submission for the **"Best tutorial for Octant v2"** track in the Octant DeFi Hackathon.

## 📚 Content List (Table of Contents)

Here are the links to each part of the tutorial.

* **[Part 1: The Vision & Core Concepts](./1-core-concepts.md)**
* **[Part 2: The Core Architecture (The "How It Works")](./2-core-architecture.md)**
* **[Part 3: The Yield Engine (Building Your Funding Source)](./3-yield-engine.md)**
* **[Part 4: The Governance Engine (Distributing Your Funds)](./4-capital-allocation.md)**
* **[Part 5: The Economic Flywheel (Tying It All Together)](./5-economics.md)**

## 👩‍💻 Who is this Guide For?

This guide is for developers, DAO operators, and protocol engineers who want a deep, hands-on understanding of Octant V2. You should be familiar with smart contracts and concepts like ERC-20, but no prior experience with Octant V2 or Yearn V3 architecture is required.

## 🗺️ A Guide to the Content

This tutorial is broken into 5 parts, designed to be read in order. Each part builds on the last, taking you from the "why" to the "how."

* **Part 1: The Vision & Core Concepts**
    * We'll start with the "why." You'll learn the high-level vision of Octant V2 as an "Ecosystem Growth Engine," the problems it solves, and the fundamental "Lego blocks" (like ERC-4626) you'll be using.

* **Part 2: The Core Architecture (The "How It Works")**
    * This is the most important part. We'll master the **"Engine vs. Driver's Seat"** pattern, the core `delegatecall` architecture that powers *all* of Octant's components. Understand this, and you'll understand the whole system.

* **Part 3: The Yield Engine (Building Your Funding Source)**
    * Our first hands-on section. We'll build the "funding" half of our machine. You'll learn the difference between **YDS (Yield Donating)** and **YSS (Yield Skimming)** and walk through a step-by-step tutorial to build your first YDS.

* **Part 4: The Governance Engine (Distributing Your Funds)**
    * With a funding source built, we'll build the "distribution" half. You'll learn how the `Regen Staker` and `Tokenized Allocation Mechanism (TAM)` work. We'll walk through a tutorial to build a **1-Person-1-Vote (1p1v)** governance mechanism from scratch.

* **Part 5: The Economic Flywheel (Tying It All Together)**
    * In the final part, we zoom out. We'll see how the tools we built are used to power high-level economic models like the **Sustainability Pool** and **Impact Bonds**, creating a true economic flywheel.

---

Made with ❤️ by Ayush Ranjan
