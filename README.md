# Anatomy of a Suspected 'Flash-Loan Arbitrage Bot' Scam Pattern

[Deutsche Fassung](README.de.md)

<p align="center">
  <img src="scam-warning.png" alt="Thumbnail-style image: &quot;How is AI doing this? +$2,629/day&quot;, stamped with a large red &quot;SCAM&quot;" width="400">
</p>

## TL;DR

This repository documents the analysis of a "flash-loan arbitrage bot" promoted in a YouTube video. The analysed code is of unknown origin, and whether it matches the contract shown in the video is unresolved. The code file contains claimed evidence of profits that cannot be genuine. Its own tests only show profit because they create the price gap themselves. On an unmodified mainnet fork, our measurement found no organic profit.

**Do not deploy such code. Do not send ETH to any contract from such a video.**

## Pattern observed in the examined video

The following describes what we observed in the one video we examined. It is not a general statement about all such videos.

- The video promises passive income from an "arbitrage bot".
- Viewers are asked to copy a contract into a browser-based IDE, deploy it themselves, and deposit 1–2 ETH into it.
- Viewers are also asked to click a "verification" link.
- Short links sit in pinned comments, and the comment section shows uniformly positive, generic comments.

A flash loan is borrowed and repaid within a single transaction. It needs no capital of your own beyond gas. A request to deposit ETH into a bot that supposedly runs on flash loans is therefore the central red flag in this pattern.

## The code we analysed

The analysed code is a copy of unknown origin. Whether it matches the contract shown in the video is unresolved. The analysed code is not published here.

The original file asks for "1–2 ETH or more" in its header comment. The contract itself, however, rejects plain ETH transfers (receive/fallback). An off-chain keeper script in the original file refuses live mode unless its own wallet holds at least 2 ETH. With its default settings, the keeper only trades through Flashbots bundles, because sending to the public mempool is disabled (line 1754). A bundle only pays gas if it is included in a block, so 2 ETH is far more than the bot needs.

We found no trap in the contract. Our review was static only (see below).

## Red flags

- Guaranteed or "stable daily" income from a trading bot.
- You are asked to deploy pasted code yourself and fund it.
- "Verification" links, short links, or links in pinned comments.
- Many near-identical comments praising profits.
- A single huge file that bundles code, "audit" claims, "test results" and "DO NOT EDIT" banners.
- Claimed transactions and statistics that do not add up when you check them.
- Profit appears only in tests that manipulate prices first.

## Findings from the code review

The review was static: we read and searched the code without executing anything except the Foundry tests described below.

Line numbers in this section refer to the original file, which is not published. That file bundled the contract, the tests, an off-chain keeper and a simulation script, the latter two as commented-out JavaScript. The keeper and the simulation were never executed.

**Claimed evidence that cannot be genuine.** The "real mainnet tests" and statistics in the file header do not hold up:

- Two different trades share the same transaction hash (lines 395 and 498).
- Several trade calculations add up to a loss but are listed as profit. One trade turns 100,000 USDC into 100,412 USDC, lists 1,040 USDC in DEX fees, and still reports +255 USDC (lines 381–393). Another turns 50,000 DAI into 50,198 USDC, lists 300 USDC in fees, and reports +133.50 USDC (lines 339–351).
- That DAI trade starts with DAI and ends with USDC. The contract rejects any route that does not start and end with the same asset (`RouteMustStartAndEndOnAsset`), so this trade could not have run.
- Some swaps are impossible. Curve's 3pool is said to return WETH (line 320), but it holds no WETH. Routes via Curve and Balancer are listed, but the contract can only call Uniswap-V2- and V3-style routers.
- A "30-day" statistic (line 501) contains a "34-day" profitable streak (line 514).
- The "mainnet run" (line 169) is the simulation's mock mode. It uses hard-coded synthetic pool reserves (line 3769). Its numbers also contradict each other. A 1,000,000 USDC flash loan nets +2,572.64 USDC (line 183), but the execution guard reports an expected net of +12,572.64 USDC (line 192). A 250,000 USDC loan is said to net +3,184.27 USDC (line 185), which is more than the 1,000,000 USDC loan.
- The header advertises a "12-hop" spread (line 271), but the keeper searches at most 5 hops (line 1676). It advertises 1inch support (line 9), but the code does not implement 1inch. The closing lines promise "Guaranteed profit" and "Stable daily income" (lines 1582–1583).

**The file contradicts itself.** Its own simulation script states that on a healthy market, organic cross-DEX spreads beyond fees and premium are closed by competing bots within a block (lines 4337–4338).

**Misleading safety features in the contract.**

- The header advertises an "emergency profit withdrawal with 24h timelock" (line 630). The lock only applies during the first 24 hours after deployment (line 920) and never again after that. `sweepToken` (line 999) can withdraw any token while the contract is paused, without any lock.
- The contract stores a `slippageBps` setting (line 870) but never reads it. Slippage protection depends entirely on the minimum amounts the keeper passes in.

**Risky defaults in the keeper** (from reading only, the keeper was never executed).

- `DRY_RUN` defaults to false and `ENABLE_LIVE` to true (lines 1739, 1761). After 24 hours of uptime and enough passed simulations, the bot trades live without further confirmation. `LIVE_FORCE` skips this gate (line 1976). The gate only evaluates simulations, not realised results.
- `MAX_FLASH_AMOUNT` defaults to 0, which means no limit (line 1735). There is no daily gas or loss budget.
- The private key is read in plain text from the environment variable `KEEPER_PK` (line 1716). The same key signs trades and Flashbots requests.
- Unless a start block is configured, pool discovery only scans the last 5,000 blocks, about 17 hours (lines 2114, 2134). Established, liquid pools are missed. What it finds are mostly newly created pools and tokens, a group in which scam tokens are common.

**No drainer found, with limits.** In the contract and the off-chain scripts we found no hard-coded recipient addresses, no `selfdestruct` or `delegatecall`, and no code that sends a private key anywhere. The flash-loan callback checks its caller and initiator. We also found no hidden Unicode control characters ("Trojan Source"), no `eval`, no obfuscated or base64-encoded code and no install scripts.

These results come from reading only. The 16 contract addresses in the original file were not checked against Etherscan.

**Not confirmed.** The review raised three suspected functional bugs that we have not tested:

- a mismatch between the contract's Uniswap V3 swap call and the default V3 router;
- a wrong V3 quoter interface in the keeper;
- an incorrect Flashbots request signature.

Treat these as hypotheses.

## Our experiment

Setup: Foundry tests on a mainnet fork at block 25733035 (details under "How we measured").

**The profit tests create their own opportunity.** The tests PROOF1 to PROOF3 first let a simulated "whale" (funded via Foundry's `vm.deal`) sell WETH on Uniswap V2. The bot then trades against the gap this creates. The profits they log are the contract's USDC balance change, measured after the Aave premium has been repaid and before gas:

- 3,530.15 USDC: 20,000 USDC flash loan, whale sells 1,000 WETH.
- 41.02 USDC: 2,000 USDC flash loan, whale sells 100 WETH.
- 26,848.25 USDC: 50,000 USDC flash loan, whale sells 5,000 WETH.

**The "honest organic baseline" tests nothing.** Test T4 quotes a round trip but contains no assertion, and one of its variables is never used.

**Our own measurement (T4b).** Our test `test_T4b_organicRoundTrip` makes no whale move and executes nothing. It only asks the routers for quotes, so the Aave fee and gas are not included. A round trip USDC → WETH → USDC loses money at every size we tested (rounded to two decimal places):

| Stake (USDC) | Uniswap → Sushi | Sushi → Uniswap |
|---:|---:|---:|
| 100 | −0.49 % | −0.88 % |
| 1,000 | −1.23 % | −1.61 % |
| 5,000 | −4.38 % | −4.75 % |
| 20,000 | −14.60 % | −14.94 % |

**Limits of the measurement.** It covers one block and one pair (USDC/WETH on Uniswap V2 and SushiSwap). It uses quotes only, with no gas and no competing bots. It is not a statement about all markets. It does show that, in this setup, the advertised profit did not exist without artificial price moves.

## How we measured

- Tool: Foundry 1.8.5.
- Environment: a local mainnet fork pinned to block 25733035. No transactions were sent to the real network.
- Method for T4b: quotes from the routers' `getAmountsOut` for the round trip USDC → WETH → USDC, in both router orders (Uniswap V2 and SushiSwap), for stakes of 100, 1,000, 5,000 and 20,000 USDC. The result is the difference between USDC out and USDC in, relative to USDC in.
- Not included: the Aave flash-loan fee and gas.

The analysed code is not published here.

## If you already deployed or deposited

- Check and revoke token approvals for your wallet, for example with revoke.cash. Type the address into your browser yourself. Never use a link from a video or its comments.
- Move your remaining funds to a new wallet. If you ever entered a seed phrase or private key on a site linked from such a video, treat the old wallet as compromised.
- Report the video to YouTube.

## About this repository

This repository contains only this documentation, in English and German.

## Disclaimer

This repository is for education and research only. It is provided as is, without warranty of any kind. It is not financial advice. It describes one code sample and a suspected scam pattern observed in one video. It makes no statement about any individual person.
