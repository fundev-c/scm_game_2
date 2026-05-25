SYSTEM DESIGN DOCUMENT

Multiplayer Classroom Supply Chain Simulation

Version 1.0

April 2026

Table of Contents

1. Game Overview

This document defines the complete system design for a multiplayer classroom simulation game. Teams of 4 players compete against each other in a supply chain management scenario, making strategic investment decisions each round to maximise production and score.

1.1 Core Concept

Teams of 4 players compete against other teams

Each player assumes one role: Supplier, Manufacturing, Transport, or Sales

The game runs in daily rounds with changing demand

Teams must coordinate investments to balance their supply chain

The slowest role (bottleneck) limits total team output

Score rewards matching production to demand; penalties for over/under production

1.2 Design Goals

Teach supply chain bottleneck theory through gameplay

Force team communication and strategic coordination

Create meaningful trade-offs between short-term and long-term investment

Ensure deterministic, reproducible results for classroom analysis

2. Game Loop

Each round (day) follows a strict 7-phase sequence. All teams execute the same phases simultaneously but cannot see each other’s decisions.

2.1 Phase Sequence

Start of Day — Increment currentDay. Check if currentDay > totalDays; if so, transition to GAME_OVER. Reset all per-round flags (submission status, round investment totals).

Demand Reveal — The server reads today’s demand from the pre-generated demandSequence and broadcasts it to all teams simultaneously. No team sees it before another.

Each player receives capitalPerPlayer units at the start of each round.

Unspent capital carries over to the next round, up to a maximum carryover limit.Discussion and Investment (Timed) — A countdown timer runs (e.g., 90 seconds). Players see a team chat and an investment interface. Each player chooses how much of their capital to invest in upgrades for their role. Unspent capital carries over to the next round up to maxCarryover. Any capital beyond maxCarryover is lost. Investments apply immediately to that player’s role stats.

Production Calculation — The system computes each role’s effective capacity, identifies the bottleneck (minimum capacity), and determines units produced. See Section 4 for exact formulas.

Scoring — Apply the scoring formula to determine the round score. Update cumulative team score and leaderboard. See Section 5.

End of Day — Display results to each team (their own stats only, not other teams’). Persist state. Loop back to Phase 1. 

Teams are shown:

- Bottleneck role

- Capacity of each role

- Missed demand or excess cause

2.2 Timing Rules

Phase 4 (Discussion + Investment) has a configurable timer, default 90 seconds

All other phases are instantaneous from the players’ perspective

The server waits for all teams to acknowledge results before advancing to the next round

If a player does not submit before the timer expires, their investment defaults to 0

3. State Model

All variables required to fully describe the game at any point in time. The state model is divided into four entities: Global Config, Team, Player, and Role Stats.

3.1 Global Config

3.2 Team State

3.3 Player State

3.4 Role Stats (per Player)

3.5 RoundResult (per Team per Day)

4. Production Calculation

Production follows a strict sequential pipeline. Each role represents one stage in the supply chain. The output of the entire chain is limited by the weakest link (bottleneck).

4.1 Pipeline Model

The four roles form a serial pipeline: Supplier → Manufacturing → Transport. Each role has an effective capacity. The team can only produce as many units as the lowest-capacity role allows.

4.2 Step-by-Step Calculation

bottleneckSeverity = maxCapacity / minCapacity

Step A: Compute each role’s effective capacity

effectiveCapacity(role) = predefined capacity based on level

Step B: Find the bottleneck

minCapacity = minimum of all role capacities

maxCapacity = maximum of all role capacities

unitsProduced = minCapacity

Step C: Determine units sold vs excess

unitsSold = min(unitsProduced, demand × salesEfficiency)

excessUnits = max(0, unitsProduced − demand)

unmetDemand = max(0, demand − unitsProduced)

If system is highly imbalance:
If (maxCapacity > 1.5 × minCapacity):

    unitsProduced = unitsProduced × 0.9

Step D:
unitsProduced = floor(unitsProduced)

4.3 Example

Suppose on Day 3, demand is 85. The team’s effective capacities are:

Supplier: 50 × 1.20 = 60

Manufacturing: 65 × 1.10 = 71.5 → floor to 71

Transport: 50 × 1.00 = 50

Sales: 65 × 1.10 = 71.5 → floor to 71

Bottleneck = Transport at 50. Units produced = 50. Units sold = min(50, 85) = 50. Unmet demand = 35. Excess = 0.

5. Scoring System

5.1 Round Score Formula

roundScore = (+1 × unitsSold) + (−2 × unmetDemand) + (−1 × excessUnits)

5.2 Score Components

5.3 Scoring Example

Continuing the example from Section 4.3 (demand = 85, produced = 50, sold = 50, unmet = 35, excess = 0):

roundScore = (50 × 1) + (35 × −2) + (0 × −1)  = 50 − 70 − 0 = −20

This demonstrates that heavy investment without addressing the bottleneck (Transport) results in a strongly negative score.

5.4 Cumulative Score

cumulativeScore = sum of all roundScores across all completed days. The cumulative score can go negative. The team with the highest cumulative score at the end of the game wins.

5.5 Tiebreakers

Fewer total rounds with negative scores

Lower total investment spent across all rounds

If still tied: co-winners

6. Role Definitions

All four roles start with identical stats (base capacity 50, efficiency multiplier 1.0). The strategic depth comes from teams deciding which roles to upgrade relative to current and anticipated demand.

6.1 Supplier

Theme: Raw material sourcing. Represents the speed and volume at which raw materials can be acquired.

Investment effect: Increases material throughput. Each level adds +15 to base capacity and +0.10 to efficiency multiplier.

Strategic implication: A weak supplier starves the entire chain. Even world-class manufacturing cannot produce without materials.

6.2 Manufacturing

Theme: Production line conversion. Transforms raw materials into finished goods.

Investment effect: Increases production line speed. Each level adds +15 to base capacity and +0.10 to efficiency multiplier (reduces defect rate).

Strategic implication: Often the most capital-hungry role. High demand requires significant manufacturing investment.

6.3 Transport

Theme: Logistics and distribution. Moves finished goods from factory to market.

Investment effect: Increases shipping volume. Each level adds +15 to base capacity and +0.10 to efficiency multiplier (reduces transit loss).

Strategic implication: A bottleneck here means goods pile up unsold at the factory despite being produced successfully.

6.4 Sales

Theme: Market conversion and customer reach.

Sales no longer contributes to production capacity.

Instead, it determines how much of produced units can actually be sold.

Formula:

unitsSold = min(unitsProduced, demand × salesEfficiency)

Where:

salesEfficiency starts at 1.0 and increases with upgrades.

Investment effect:

Each level increases salesEfficiency by +0.1.

Strategic implication:

Even if production is high, poor sales limits actual revenue.

Overproduction is penalised if sales cannot match output.

7. Investment System

7.1 Upgrade Tiers

Each role follows the same upgrade progression. Costs escalate at each level, creating diminishing returns on heavy single-role investment.

7.2 Effective Capacity at Each Level

7.3 Investment Mechanics

Each round, a player can invest between 0 and capitalPerPlayer (e.g., 100)

totalInvested for that player accumulates across rounds

Level upgrades happen instantly when totalInvested crosses the cumulative cost threshold

Players cannot downgrade or reallocate past investments

Max level is 5. Once a role reaches max level, further investment is blocked.The UI must prevent additional investment.

If a player invests enough to skip levels (e.g., 200 on day 1), they jump from level 1 to level 3, with the remainder counting toward level 4

8. Demand Generation

8.1 Formula

demand(day) = baseDemand + amplitude × sin(2π × day / period) + noise(seed, day)

8.2 Default Parameters

8.3 Demand Range

For the first 3 rounds, demand is restricted to a narrower range (e.g., 50–70) to allow teams to stabilise their system before full variability is introduced.

With default parameters, demand oscillates between approximately 30 and 110 units. This range is carefully chosen: at level 1, all roles have effective capacity of 50, meaning teams start significantly below peak demand. At max level 5, effective capacity is 154, which comfortably exceeds even the highest demand.

8.4 Pre-generation Rule

The entire demand sequence is generated before the game starts using the randomSeed. This ensures all teams see identical demand values and the instructor can reproduce the exact game for analysis.

9. Edge Cases and Rules

9.1 Investment Edge Cases

9.2 Production Edge Cases

9.3 Scoring Edge Cases

9.4 Communication Rules

Cross-team communication is strictly forbidden. Teams cannot see other teams’ investments, capacities, or strategies.

If a player disconnects mid-round, their investment defaults to 0. Role stats remain unchanged. Team is notified.

If a player does not submit before the timer expires, same rule as disconnect: investment = 0.

9.5 Game Boundary Rules

Exactly 4 players per team. No partial teams. Lobby waits until all teams are full.

Minimum 2 teams (otherwise no competition). Maximum limited by server capacity.

Recommended game length: 10–20 rounds. Shorter games favour aggressive early investment; longer games reward balanced strategies.

10. System Architecture

The system follows a three-tier architecture: Client, Server, and Data layers. All game logic runs on the server to prevent tampering.

10.1 Client Layer

Player UI: Shows current demand, personal capital, role stats, investment slider, and round results. One instance per player.

Team Chat: Real-time text communication within a team. Only visible to teammates. Active during Phase 4.

Instructor Dashboard: Overview of all teams’ scores, current game phase, ability to pause/advance rounds, and post-game analytics.

10.2 Server Layer

Game Engine: Owns the phase state machine. Advances through phases, manages timers, coordinates other modules. Only module that transitions gamePhase.

Investment Processor: Validates each player’s investment (within capital bounds), applies to totalInvested, checks level thresholds, updates role stats.

Production Calculator: Pure function. Takes 4 effective capacities + demand, returns unitsProduced, unitsSold, excessUnits, unmetDemand, bottleneckRole.

Scorer: Pure function. Takes production results + total investment cost, returns roundScore. Appends to roundHistory and updates cumulativeScore.

Demand Generator: Called once at game creation to produce the full demandSequence array. Never called during gameplay.

10.3 Data Layer

Game State Store: Persists all state entities. On disconnect/reconnect, the full game state is rehydrated from here.

Leaderboard: Derived view: sorted teams by cumulativeScore, updated after each round.

10.4 Communication Protocol

Client-server communication uses WebSockets for real-time bidirectional events. Key events include:

Server → Client: PHASE_CHANGE, DEMAND_REVEAL, CAPITAL_GRANTED, ROUND_RESULTS, TIMER_TICK

Client → Server: SUBMIT_INVESTMENT, CHAT_MESSAGE, PLAYER_READY

All game logic runs server-side. The client is a thin view layer that displays state and collects input.

11. Formula Reference

All formulas in one place for quick reference during implementation.


---TABLES---

## TABLE 0:
| Variable | Type | Description |
| totalDays | int | Total number of rounds in the game |
| currentDay | int | Current round number (starts at 1) |
| capitalPerPlayer | int | Fixed capital given to each player per round (e.g., 100) |
| demandSequence | int[] | Pre-generated array of demand values, one per day |
| phaseTimerSeconds | int | Duration of the investment phase timer (e.g., 90) |
| gamePhase | enum | LOBBY, DEMAND_REVEAL, INVESTMENT, PROCESSING, RESULTS, GAME_OVER |
| randomSeed | int | Seed for demand generation, ensures reproducibility |

## TABLE 1:
| Variable | Type | Description |
| teamId | string | Unique team identifier |
| cumulativeScore | int | Running total score across all rounds (can be negative) |
| roundHistory | RoundResult[] | Array of per-round results for post-game analysis |

## TABLE 2:
| Variable | Type | Description |
| playerId | string | Unique player identifier |
| teamId | string | Team this player belongs to |
| role | enum | SUPPLIER, MANUFACTURING, TRANSPORT, or SALES |
| capitalThisRound | int | Capital available this round (reset each round) |

## TABLE 3:
| Variable | Type | Description |
| level | int | Current upgrade level (starts at 1, max 5) |
| baseCapacity | int | Base units per day at current level |
| efficiencyMultiplier | float | Multiplier applied to base capacity (starts at 1.0) |
| currentCapacity | float | Computed: baseCapacity x efficiencyMultiplier |
| totalInvested | int | Cumulative investment across all rounds |
| upgradeCost | int | Cost threshold for next level |

## TABLE 4:
| Variable | Type | Description |
| day | int | Round number |
| demand | int | Demand value for this round |
| unitsProduced | int | Total units produced (limited by bottleneck) |
| unitsSold | int | min(unitsProduced, demand) |
| excessUnits | int | max(0, unitsProduced - demand) |
| unmetDemand | int | max(0, demand - unitsProduced) |
| bottleneckRole | enum | Which role was the limiting factor |
| totalInvestmentCost | int | Sum of all 4 players’ investments this round |
| roundScore | int | Computed score for this round |

## TABLE 5:
| Component | Points | Rationale |
| Units sold | +1 each | Revenue from meeting demand |
| Unmet demand | −2 each | Lost customers and reputation damage (harsh penalty) |
| Excess production | −1 each | Wasted resources from overproduction |

## TABLE 6:
| Level | Cumulative Cost | Cost This Level | Base Capacity | Efficiency Mult. |
| 1 (start) | 0 | 0 | 50 | 1.00 |
| 2 | 50 | 50 | 65 | 1.10 |
| 3 | 130 | 80 | 80 | 1.20 |
| 4 | 250 | 120 | 95 | 1.30 |
| 5 (max) | 420 | 170 | 110 | 1.40 |

## TABLE 7:
| Level | Base Cap. | Eff. Mult. | Effective Cap. |
| 1 | 50 | 1.00 | 50 |
| 2 | 65 | 1.10 | 71 |
| 3 | 80 | 1.20 | 96 |
| 4 | 95 | 1.30 | 123 |
| 5 | 110 | 1.40 | 154 |

## TABLE 8:
| Parameter | Default | Description |
| baseDemand | 70 | Centre point of demand oscillation |
| amplitude | 30 | How far demand swings above/below base |
| period | 8 days | Length of one full demand cycle |
| noise range | −10 to +10 | Random integer added each day (seeded) |

## TABLE 9:
| Scenario | Resolution |
| Player invests 0 | Allowed. No upgrade, no cost deducted. Capital is lost. |
| Investment exceeds capital | Rejected. Investment is clamped to available capital. |
| Player at max level invests | Investment accepted, cost deducted, no stat improvement. UI should warn. |
| Investment crosses two levels | Allowed. Player jumps levels. Remainder applies toward next. |

## TABLE 10:
| Scenario | Resolution |
| All roles have equal capacity | No single bottleneck highlighted. Production equals shared capacity. |
| Demand is 0 | All production is excess. Score = −1 × unitsProduced − investmentCosts. |
| Fractional capacity | All capacities and outputs are floored to integers before calculations. |

## TABLE 11:
| Scenario | Resolution |
| Negative round score | Allowed. Cumulative score can go negative. |
| Perfect round | Score = demand − investmentCosts. Best possible outcome. |
| Tied cumulative scores | Tiebreaker: fewer negative rounds, then lower total investment. |

## TABLE 12:
| Formula | Expression |
| Effective capacity | baseCapacity(level) × efficiencyMultiplier(level) |
| Units produced | min(effCap_supplier, effCap_manufacturing, effCap_transport) |
| Units sold | min(unitsProduced, demand) |
| Excess units | max(0, unitsProduced − demand) |
| Unmet demand | max(0, demand − unitsProduced) |
| Round score | (+1 × unitsSold) + (−2 × unmetDemand) + (−1 × excessUnits) |
| Demand | baseDemand + amplitude × sin(2π × day / period) + noise(seed, day) |
| Level threshold | Player levels up when totalInvested ≥ cumulativeCost(nextLevel) |

