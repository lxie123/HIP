# HIP18: Oracle Arbitrage Fix

- Author: [@lthiery](https://github.com/lthiery)
- Start Date: 2020-10-22
- Category: Technical
- Original HIP PR: [#59](https://github.com/helium/HIP/pull/62)
- Tracking Issue: 

# Summary
[summary]: #summary
A new oracle price takes effect one hour in the future. The reason for this is to provide users with predictability when converting HNT to DC. 
 
 If one is trying to “sweep” all HNT from one wallet to another, one can leave just enough HNT to cover the transaction fee via implicit burn 
 If you are converting HNT into DC, you can know with relative certainty that your burn will yield a certain amount of DCs
 
 Unfortunately, this delay provides an on-chain arbitrage opportunity via state channels. Given the delay, an actor with HNT in-hand may see the forecasted drop in HNT price and thus convert HNT into DCs. After the drop takes effect, the actor may then “spend” the DCs in a state channel, naming a colluding gateway of theirs as the beneficiary. Effectively, the actor is burning and minting HNT on-chain with very little risk. 
 
 To demonstrate the “oracle arbitrage opportunity”, consider the following example:
 * At block 200, the current oracle price is $1.1 
 * Submitted oracle prices will set the new oracle price to $1; this will take effect in one hour or roughly at block 260
 * Between block 200 and block 260, the actor may burn 1000 HNT, yielding $1100 worth of DC
 * After block 260, the actor my close a state channel naming its colluding gateway as beneficiary of $1100 worth of DC, assuming:
    * the grace period on the state channel has expired
    * the oracle price has not changed again
    * the DC reward pool has not been maxed out
 	
 	⇒ the colluding gateway is awarded 1100 HNT
 

# Example
[example]: #example

The Dapper-Hickory-Meerkat, managed by [wallet 13ZiB74TJNdV5D5VFehEH3oMSuFNusjkho4gJYmnRmLRnXrZbE5](https://explorer.helium.com/accounts/13ZiB74TJNdV5D5VFehEH3oMSuFNusjkho4gJYmnRmLRnXrZbE5), has recently exploited this opportunity.

Here it is graphed from on-chain data showing HNT burn events, mining rewards from DC and what the oracle price was for all of these events:

![image MeerkatBurnAndMint](./0018-oracle-arb-fix/meerkat.png)


As you can see, “the big burn” of about 27k HNT was burned right before the oracle price dropped from about $1.2 to $1.1. Over time, the DC was “minted” back into HNT via the DC rewards.

It’s worth noting, despite the magnitude of burn and mint, at the time of this writing, only about 2k HNT was cleared by the owner of the Meerkat. Therefore, you could approximate:
* 27k HNT was burned
* 29k HNT was minted

While only 2k HNT was cleared, another side-effect of the arbitrage is that the Proof-of-Coverage (POC) earnings during the period of the mint back were severely impacted as when the DC reward pool grows, it grows at the expense of POC rewards during that same epoch. 

It’s also worth noting that the mint back took roughly 24 hours, so should the market price and then the oracle price have snapped back, the Meerkat may have lost out. The severe price drop may also be an artifact due to manual entry of oracle prices. This will not be the focus of this HIP but remains noteworthy and perhaps the subject matter for future HIPs.

# Winners and Losers as Is
[winners-and-losers-as-is]: #winners-and-losers-as-is

So who gains what from this opportunity? As previously stated, the owner of the Meerkat’s wallet has cleared about 2k HNT. The rest of the HNT in play was essentially burned and redeemed; in effect, HNT emissions were cut by about 30% during the 24 hour period of mint back. In a direct sense, this benefits:
* Current holders of HNT: effectively, inflation has been curbed
* non-POC earners: their emissions have not been affected

# Solution
[solution]: #solution
The opportunity for arbitrage is that the oracle price is forecasted to change before it actually changes. As previously mentioned, oracle reporting and oracle price adjustment may be compounding factors but are not the concern of the solution to be proposed herein.

**The proposed solution is to make HNT to DC conversion for transaction fees to be based on the minimum of (previous prices and dynamic price)  and to force burn transactions to accept a dynamic conversion rate, as long as it is below the transactions maximum acceptable conversion rate.**

Since transaction fees are relatively insignificant in the economics of the Helium blockchain, allowing for “opportunistic” transactions should not affect things the way that burning HNT for DC does. Thus allowing a minimum of previous and dynamic previous gives the same desired ease of use. 

For HNT to DC conversion, the dynamic price essentially means that any oracle price changes may change one’s conversion rate. In general, the DC burn transaction is only useful for OUI operators or facilitators of these parties. As such, we may expect a level of sophistication from these actors relative to regular wallet holders or miners who simply wish to transact funds.

However, to mitigate the impact of the dynamic price, we suggest the ability for dc_transactions_v2 to also stipulate a maximum oracle conversion rate. This will afford a level of assurance for these transactions with regards to oracle and market price volatility.

# Examples with Solution Implemented
[examples-with-solution-implemented]: #examples-with-solution-implemented

## Failed Burn

* A user submits a token burn right after block 200 is forged. Oracle price at that time is $1 and the *maximum conversion price set is $1*.
* At block 201, a new oracle price of $1.1. 
* The token burn transaction is now invalid and will be dropped. It could theoretically be rebroadcast at a later time or the user could submit a new transaction with the same nonce to invalidate the transaction.

## Successful Burn

* A user submits a token burn right after block 200 is forged. Oracle price at that time is $1 and the *maximum conversion price set is $1.1*.
* At block 201, enough oracle reports set an instantly new oracle price of $1.1. The token burn transaction remains valid.

## Payment Transaction

* Fees are set to allow the best price in the previous 100 blocks
* A user submits a payment of 1 HNT to another address. Going back 100 blocks, the lowest oracle price is 50 blocks ago at $1 and current oracle price is $1.1.
* If the transaction is processed within 50 blocks, the $1 oracle price is honored and 0.35 HNT is withdrawn from the account to cover the $0.35 fee

## Payment Transaction with Surprise Discount

* Fees are set to allow the best price in the previous 100 blocks
* A user submits a payment of 1 HNT to another address. Going back 100 blocks, the lowest oracle price is 50 blocks ago at $1
* After submission but before the transaction is processed, a new price of $0.5 is issued
* The $0.5 oracle price is honored and 0.175 HNT is withdrawn from the account to cover the $0.35 fee

# Impact
[impact]: #impact

So who gains what from this solution? While this solution is admittedly not the simplest from an implementation perspective, it attempts to resolve the logical existence of the arbitrage without affecting reward policy (no economic impact other than closing the arb) and without losing predictability in transaction fees. 

In effect, it simply tries to take the “forward looking window” that oracle prices provide for transaction fees and instead provide a “backwards looking window”, along with the potential to pay less if by chance the oracle price goes down after submission.

One minor drawback here is that someone wishing to “sweep” the contents out of a wallet might get a better deal than anticipated and would leave some dust (a few bones) behind. This could be addressed in the future with a specific sweep transaction which would indicate “empty the wallet to this wallet”, thus leaving no dust behind.

The biggest downside of this solution is an increase in complexity for the burn transaction as the user does not know ahead of time what the cost will be; the maximum conversion rate is added to the transaction, however, which provides a bounded conversion rate.

# Alternate Solutions
[alternate-solutions]: #alternate-solutions

## Delayed HNT Burn
Delayed burn is not logically that different from the proposal here, except instead of releasing the DC immediately, the DC would be held until the future oracle price takes effect. The downside of this approach is DCs cannot be created without a delay and thus a user must wait for DCs when they may need them quickly.
## 1.1:1 DC to HNT Conversion
Currently, DC rewards are determined by converting DC earned for a gateway into HNT at a 1:1 rate. Assuming the total HNT earned via DC rewards during the epoch is within the allocated percentage, the HNT rewarded is worth exactly the amount of DC earned by that gateway during the epoch. Recall that the 1:1 conversion was determined as a countermeasure to the DC spamming in August 2020, where traffic was low enough and so far below equilibrium, that every DC spent was effectively rewarding gateways 1:100 to 1:1000.

While this solution would effectively negate the arbitrage opportunity, it has a secondary economic impact of decreasing the earning potential of gateways that forward network data, which is precisely the reason for the Helium Network to exist. In fact, before DC spamming made the flaw in incentives apparent, we very much wanted to reward a large slice to gateways that were forwarding data.