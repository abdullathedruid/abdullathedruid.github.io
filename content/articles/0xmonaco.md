---
title: 0xMonaco
date: 2022-08-22 20:27
category: Smart Contracts
tag: ethereum
---

## Introduction

Last weekend saw the release of the highly anticipated Paradigm CTF. Amongst the many challenges of varying difficulty, there was a PvP game which involves racing cars against each other. The only catch, however, is that the cars would be controlled by smart contracts! Over the course of 24 hours, developers would deploy new contracts to try and beat each other and to end the competition with the highest ELO. 

Within the last minutes, it was neck-and-neck between three teams (OpenSea, JustLurking, and myself) with OpenSea just snatching the win in the very last races. Although I'm gutted I didn't win, I'll take the 2nd place finish. I will include the source code for my car at the bottom of the post, but I'm not sure how much it makes sense without understanding the thought process that went into writing it! Feel free to scroll down to the bottom to have a look.

![Alt Text]({static}/images/monaco-scoreboard.png)

## The Beginnings

When the competition started on Saturday night, I was very excited and deployed my first car. My first car (MaxBidCar) was very simple. Just buy 11 acceleration, every round. I didn't win many races, but I certainly managed to get off the start line quickly. I then started to realise that the strategy you should take depends completely on the strategy taken by other players. In this example, I realised that if I had started quickly (because I bought up all the acceleration) there was a chance that the guy behind me hadn't coded any shell mechanism. Indeed, the only races I managed to win are the ones where I didn't get shelled.

I realised I needed to be more clever than this, so I decided to change the amount of acceleration being bought. There was a trade-off between buying as much acceleration as possible vs being economic. I started to think that the optimal strategy would be an economic strategy. After re-reading the rules, I tried to understand the exponential pricing algorithm and come up with a solution. I started with the following:

```solidity
// This is cheap speed
while (monaco.getAccelerateCost(1) < 20) {
    ourCar.balance -= uint24(monaco.buyAcceleration(1));
}

// shell someone if shells are cheap, and we are not in lead
if (ourCarIndex != 0 && monaco.getShellCost(1) < 200) {
    ourCar.balance -= uint24(monaco.buyShell(1));
}
```

The documentation had suggested that the "default" price for acceleration was 20, and for shells was 200. Therefore we buy these whenever items whenever we can get them at a bargain - regardless of what our current speed is! I noticed that this strategy was very effective in certain scenarios. When the other two racers were very aggressive, they would spend all their coins early on in the game. Eventually they will both be priced out of taking additional turns. At this point, I can use all the cheap moves to slowly sail across the finish line.

## Economics and Meta-Game Theory

Any competitive gamer will understand the concept of the 'meta' or 'metagame'. Over time, it became apparent that multiple different strategies were evolving - some would frontload the start of their race, others would save their economy for a push at the end. Some people had different thresholds for which they would prioritise their own speed vs shelling others to reduce their speed. This metagame became apparent to me after watching some of my own races. It almost appeared like the game evolved into rock paper scissors, where the three different behaviours were "acceleratooor", "shellooor", "economicooor".

As my previous strategies had been quite poor, my elo dropped down to the 900s. This became quite frustrating as it became clear that the metagame at elo ~900 was different to ~1100 and ~1300 and ~1500 were all different (Although I didn't have that much time to learn the top meta!). A user that wastes all his coins buying acceleration should be punished by a swift shell from the player behind them, however in some elos there were many cars which never bought any shells! So a "Greedy" algorithm which just max bids acceleration, and hopes that the player behind them doesn't shell them becomes very effective.

Going back to my algorithm, I wanted to expand on the code I highlighted above. I liked this algorithm because it would be cautious when the other two racers were aggressive, and was aggressive when the other players were passive. I decided to converge around the current price of acceleration/shells as an indicator for whether these actions were overbought or not. If acceleration was expensive, I would just sit and wait for it to slow down whilst also shelling everyone in front of me. Unfortunately, I found that most of the time, my car was being too stingy and avoided paying for most actions and so just sat at the start line for most of the race.

After finding out that my car was ending the races with most of its coins, I realised I could be more spend more buying actions than the default cost. Deciding to buy acceleration whenever its price was less than `20` meant that I didn't buy acceleration a lot of the time, especially at the end of the races when you need acceleration the most! I then created a function `getMultipliedInput()` which I would use to scale this value. At first I tried 150% scaling (so I would buy acceleration at 30 instead of 20), and then 200%. I then experimented with using other parameters as part of this scaling. I tried looking at the difference between my balance and the average of the others players balance - this wasn't an effective strategy!. I also looked at the number of turns, and distance travelled as methods of estimating "how close are we to the end of the game".

In the end, I chose to use an algorithm that was very similar to the compound interest rate formula utilising a kink. I looked at distance travelled, and number of turns taken in order to calculate how much my car was willing to bid on acceleration/shells. I was able to use the test file supplied in the project folder to test different cars by running `forge test -vv` to test my car against other example cars. I was able to use this to set some initial variables for the kink distance and sprint factor. This was my first time using foundry, and I was impressed by the speed I was able to iterate over making changes to these variables.

## Summary

You will have noticed throughout the article I have made many assessments to the functioning of my car. Indeed, it is pointless to make changes to your cars code without having a hypothesis to validate/invalidate these changes. There are many different frameworks for continuous improvement, but broadly they involve 1. coming up with a theory 2. making a change 3. assess response of change 4. repeat. I don't think it's any coincidence that the three teams that came in the top three, all had approximately submissions. I feel like any less than this implied that users were less willing to experiment/make changes with their code. People with more submissions would probably not have had sufficient time to assess the impact of their changes!

Naturally, I would also assume that other people may have spent their time working on the rest of the CTF challenges. So, thanks again to Paradigm for creating an entertaining set of challenges. This was definitely the most amount of fun I've had on-chain, and would love to see some on-going development for 0xMonaco. Now, here's the code you all wanted to see:


## The Code

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.13;

import "./Car.sol";
import "forge-std/console.sol";

// elo: 965.90 when we start

contract ZoomCar is Car {
    constructor(Monaco _monaco) Car(_monaco) {}

    uint256 internal constant KINK_DISTANCE = 700;
    uint256 internal constant KINK_FACTOR = 3_000;
    uint256 internal constant SPRINT_FACTOR = 11_500;
    uint256 internal constant FINISH_DISTANCE = 1000;

    function getMultipliedInput(
        uint256 input,
        uint32 distance,
        uint16 turn
    ) internal view returns (uint256 output) {
        if (distance < KINK_DISTANCE) {
            output =
                input +
                ((turn * 3) / 4) +
                ((input * ((distance * KINK_FACTOR) / KINK_DISTANCE)) / 1000);
        } else {
            output =
                input +
                ((turn * 3) / 4) +
                ((input *
                    (KINK_FACTOR +
                        (((distance - KINK_DISTANCE) * SPRINT_FACTOR) /
                            (FINISH_DISTANCE - KINK_DISTANCE)))) / 1000);
        }
        console.log(output);
    }

    function takeYourTurn(
        Monaco.CarData[] calldata allCars,
        uint256 ourCarIndex
    ) external override {
        Monaco.CarData memory ourCar = allCars[ourCarIndex];

        uint16 turn = monaco.turns();

        // In the final sprint, if we are 2nd:
        // shell the guy ahead if he finishes before us
        if (ourCarIndex == 1 && allCars[ourCarIndex].y >= KINK_DISTANCE) {
            uint256 our_ttg = 1 +
                (FINISH_DISTANCE - allCars[ourCarIndex].y) /
                allCars[ourCarIndex].speed;
            uint256 their_ttg = 1 +
                (FINISH_DISTANCE - allCars[ourCarIndex - 1].y) /
                allCars[ourCarIndex - 1].speed;
            if (
                their_ttg <= our_ttg && ourCar.balance > monaco.getShellCost(1)
            ) {
                ourCar.balance -= uint24(monaco.buyShell(1));
            }
        }

        // This is cheap speed
        for (uint256 i; i < 10; i++) {
            if (
                monaco.getAccelerateCost(1) <=
                getMultipliedInput(20, allCars[ourCarIndex].y, turn)
            ) {
                ourCar.balance -= uint24(monaco.buyAcceleration(1));
            }
        }

        // if we are in lead, price gouge shells
        if (
            ourCarIndex == 0 &&
            monaco.getShellCost(1) <=
            getMultipliedInput(200, allCars[ourCarIndex].y, turn)
        ) {
            ourCar.balance -= uint24(monaco.buyShell(1));
        }

        // If we aren't leading:
        if (ourCarIndex > 0) {
            uint32 ahead_speed = allCars[ourCarIndex - 1].speed;
            uint256 shell_cost = monaco.getShellCost(1);

            // If we're in last, and the guy in first is faster than second
            // We shouldn't shell, because we want the second guy to shell the first
            bool shouldShell = true;
            if (ourCarIndex == 2) {
                if (ahead_speed < allCars[ourCarIndex - 2].speed) {
                    shouldShell = false;
                }
            }

            // shell person ahead if their speed is greater than cost of shell
            if (
                shouldShell &&
                (shell_cost <
                    getMultipliedInput(
                        ahead_speed * 16,
                        allCars[ourCarIndex].y,
                        turn
                    ) &&
                    ahead_speed > 1)
            ) {
                ourCar.balance -= uint24(monaco.buyShell(1));
            }
        }
    }
}

```