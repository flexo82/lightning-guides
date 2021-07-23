## Intro
I am running [McDouchebag](https://1ml.com/node/03b75897555da10fc84c93fd1543f4e166a025582057dd58a97c029baba2deb1ab) and this is a guide of how i have chosen to manage fees on my node. I will apologize for my English as that is not my first language, so please bare with me.

When i set up my node for the first time in 2018 i only setup 2 channels which did not see a single transaction for more than a year. There was no good guides on how to run routing nodes out there and do not even talk about managing fees. To this day i have not been able to find a single guide about fee structure which made me want to write one. 
I struggled as a newcommer to the lightning network and i did not know how to make my node succesfull or even profitable. 
Success for me was not to be profitable but rather having transaction flowing through my node while trying to break even, knowing that i was helping the network instead of maximizing profit.

## Routing
In order to become a routing node you will need to have a few channels open. The better connected your node is, the higher the likelyhood of transactions being routed through your node.
This is not a guide on how to get liquidity for you node. There are plenty of resources out there that will help. Here is a few of the better ones.

- [How to get Inbound Liquidity on LN](https://gist.github.com/bretton/53bc511b6fdafef31951199dd25bbf88)
- [Lightning Liquidity](https://coincharge.io/en/lightning-liquidity/)
- [Bootstrapping Liquidity](https://wiki.ion.radar.tech/tutorials/bootstrapping-liquidity)

And last a shoutout to [LightingNetwork+](https://lightningnetwork.plus/) for creating an awesome service for routing node operaters.

## Fees
Fees on the lightning network is still a very mysterious thing. There is no right way of doing it. Fee management seems to be shrouded in secrecy, however, i think the reason there are no guides is that noone has found a the secret sauce to run a profitable node without taking care of your channels as they were your kids.
Channels needs constant care and grooming if you want to run a profitable routing node. 

As i mentioned in the intro, my aim is not to be profitable, however it is a nice side effect of running a routing node. I have earned back the sats i have spend on opening channels and rebalancing, but it is not enough for me to retire in any way. I may be lucky if i can buy a cup of coffee once in a while.
When i started my routing node, my aim was to help the network. I decided i wanted to run a low fee node in the hopes people would automatically discover it on [1ML](https://1ml.com/node/03b75897555da10fc84c93fd1543f4e166a025582057dd58a97c029baba2deb1ab), but of course that did not happen.

I started out like most new people having my fees at the default you get out of the box. I run [RaspiBliz](https://github.com/rootzoll/raspiblitz) node on a RP4 which uses [LND](https://github.com/lightningnetwork/lnd).
The default fees sits the following.
```
Base Fee: 1.000 mSat
Fee Rate: 1 ppm
```

The Base Fee is denominated in milli-satoshis, `1 satotoshi = 1.000 milli-satoshis` and is flat rate that will always be payed when routing a transaction
The Fee Rate is denominated micro-statoshis, `1 satoshi = 1.000.000 micro-satoshis` and is proportional to the size of the transaction.
This means if you route a transaction of 100.000 satoshis, you will earn 1,100 satoshis.

This fee structure seems good, right?! Its cheap and you earn a few sats when routing transactions. Well, my mission was to help the network and lightning can route anything from large multi-million sats transaction to transaction of only a few sats. In order to route micro-transaction of a few sats, it does not make sense to have a base fee of 1 satoshi. On a 5 satoshi transaction, that would be a 20% fee.

Making a long story short, i ended up at the conclusion at keeping a low base fee would allow micro as well as macro transaction and then trying to earn a little satoshis on the fee rate. I searched far and wide for something to help me balance the Fee Rate automatically so i did not have to manually do that all the time.
There are multiple tools out there that can help and i eventually settled on using [charge-lnd](https://github.com/accumulator/charge-lnd) with cron. Its an easy to use command-line tool that can balance your channels based on a configuration. The cron job is set to run every 2 minutes on my node which updates the fees.

My config for charge-lnd is [here](https://github.com/flexo82/lightning-guides/blob/main/config/charge-lnd.config). I will keep updating the config and feel free to use or get inspired from it.

Lets break it up.

First the defaults that applies to all channels and the `[default]` rule is a catch all
```
[default]
# 'default' is special, it is used if no other policy matches a channel
strategy = static
base_fee_msat = 1
fee_ppm = 50

[mydefaults]
# no strategy, so this only sets some defaults
base_fee_msat = 1
min_fee_ppm_delta = 50
time_lock_delta = 40
min_htlc_msat = 1
```


The `[known-one-way-drains]` is to ensure some of the nodes the are mainly one way will pay for sucking a channel dry. I am currently connected to 3 of suchs nodes which funnily enough is also some of the larger routing nodes or exchanges.
The `[expensive]` rule is to penalize channels that are trying to earn a lot on my node for having relatively low fees. I may drop this rules as this is really hard to automate your way out of.
```
[known-one-way-drains]
# Big nodes that tends to drain funds
node.id = 033d8656219478701227199cbd6f670335c8d408a92ae88b962c49d4dc0e83e025,
          03cde60a6323f7122d5178255766e38114b4722ede08f7c9e0c5df9b912cc201d6,
          0390b5d4492dc2f5318e5233ab2cebf6d48914881a33ef6a9c6bcdbb433ad986d0

strategy = static
fee_ppm = 300

[expensive]
# match channels where the peer node has set a high ppm fee rate
# and set the same fee rate on our side (strategy=match_peer)
chan.min_fee_ppm = 1_000
chan.max_ratio = 0.3

strategy = match_peer
```


I am having low constant fee for non-routing nodes. They are typically mobile wallets.
```
[leafnode]
# Non-routing channels have a constant fee
chan.private = true

strategy = static
fee_ppm = 25
base_fee_msat = 1
```


This is the rule that applies to most channels. It sets the fees based on the current balance of funds. As funds are pushed to me, the fees lower to encourage more outbound transactions through the channel and when funds are pushed to the peer, fees get higher to encourage incomming transactions. This works great to "autobalance" channels.
```
[proportional]
# Set fees proportionally to how the channel is funds are balanced

strategy = proportional
base_fee_msat = 1
min_fee_ppm = 10
max_fee_ppm = 150
```


## Conclusion
It has been a journey for me to find some fees that works. I have tried to make it such that i am running a routing node with relatively cheap fees in a way that makes me break more or less even when considering cost of hardware, electricity, channel creation fees and rebelancing.

I have tried to ensure the big nodes that drain funds are paying more for doing so in order to keep my channels balanced.

If you think this guide has been helpful and want to support me and/or the network, feel free to set up a channel to my node, [McDouchebag](https://1ml.com/node/03b75897555da10fc84c93fd1543f4e166a025582057dd58a97c029baba2deb1ab)
