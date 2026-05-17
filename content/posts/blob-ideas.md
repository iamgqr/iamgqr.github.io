+++
date = '2026-05-17T22:11:15+08:00'
draft = false
title = 'Screeps Devlog #1: About my first iteration on "blob"'
+++

Last week I spent the whole week working on my "blob" code, and I finally used it to defeat a Level 4 Stronghold! It took about two 6-creep blob squads (T3 parts). A lot of people were wondering how I did it, so I decided to share my thoughts here.

## What is "blob"?

Literally, "blob" just means "a bunch of creeps." It doesn't have a fixed formation, which makes it way more flexible. In this post, when I say blob, I mean this specific approach: defining custom weights, letting behaviors emerge from those weights, and having each squad member coordinate with others to find the moves that maximize the total payoff.

## Why blob?

### Duo

Duo is the simplest example of a "fixed formation" - usually one attacker (with `ATTACK`, `RANGED_ATTACK` or `WORK`) and one healer. A lot of people who want to implement combat write this after finishing their basic blinky solo logic, because it's relatively simple. The downside is that there just aren't enough creeps. A lot of times, the healing simply can't keep up with 6 towers in an RCL8 room, especially if the enemy also has active defense creeps.

### Quad

Quad is a strategy popularized by Tigga, and it's pretty much the meta right now. A lot of PvP masters like Harabi use it. A Quad is basically four creeps moving in a 2x2 formation, usually 2 attackers and 2 healers. It solves the number problem, but introduces a new limit: "formation constraints". A fixed 2x2 group is hard to move flexibly through complex terrain, and you often have to write special logic just to get them through narrow gaps.

Also, there are some common issues with fixed formations that Quad doesn't really fix:

- It's really hard to make different squads coordinate with each other.
- There is the limitation of the decision model: its logic is entirely top-down "dead logic". All decisions are hardcoded if-else conditions - when to advance, when to retreat, when to take risks, it all relies on the programmer enumerating every edge case. And you can never enumerate *everything*.

### Blob

Because of this, I decided to focus on the blob idea. The core concept is simple: assign a value to the actions of the squad members, and find the actions that get the maximum overall value.

Compared to quad, blob solves a lot of things:

- No 4-player "roster limit": Blob supports any number of creeps. It's like a basket, you just throw in as many creeps as you want, and you can even add more dynamically.
- No 2x2 "formation limit": Blob has no fixed shape, it acts like a fluid, so it's much easier to coordinate in tricky terrain situations.
- No "dead logic" (if-else): Blob doesn't follow hardcoded if-else logic. Instead, it tries to find the optimal solution that maximizes the return, which avoids the problem of "edge cases" from the very root.

### Hungarian in blob

DroidFreak uses the Hungarian algorithm to implement blob. A weight is assigned to every tile a creep can choose, basically saying "if creep_x goes to pos_y, the score is Z". But because of the rule that "two creeps can't occupy the same tile", it's possible that the best move for both creep_1 and creep_2 is to go to pos_1, which is obviously illegal. So it becomes an optimal matching problem, assigning target tiles to all creeps to maximize the total score.

This Hungarian approach has one drawback: it can't express the value of "joint states". In real squad tactics, a lot of value is "joint" and can't be broken down into the sum of individual contributions:

- Healing overlap: You can calculate "value of healer A at pos_1" and "value of healer B at pos_2" separately, but the extra joint value of "A and B being at pos_1 and pos_2 simultaneously and being able to cover each other with heals" cannot be expressed easily by the Hungarian algorithm.
- Weakest link decides life and death: The safety of the squad is determined by its most fragile member. This is a global property, not a simple sum of each creep's individual score.
- Focus fire synergy: Multiple creeps attacking the same target yield the full kill reward. Evaluating "who can I attack" for each individual and then matching them completely misses this joint buff.

The root cause is actually this: Hungarian solves linear assignment problems (the objective function must be decomposable into the sum of independent single items). Real multi-player coordination is non-decomposable. So I thought about other ways to implement a blob.

## So, how blob?

### Introducing Simulated Annealing

With N creeps, each has 9 direction options, so the state space is 9^N. Even for N=7, that's millions of combinations, not to mention all the damage and heal decisions. Enumerating everything is definitely impossible. So I used Simulated Annealing (SA).

Simulated Annealing is a randomized optimization algorithm inspired by the annealing of metals. You start with an arbitrary initial solution, and repeatedly make random perturbations to the current plan to get a neighboring new plan. If the new plan is better, you accept it right away. If it's worse, you still accept it with a certain probability—this probability is `exp(Δ/T)`, where Δ is the score difference (a negative number) and T is the current "temperature". When the temperature is high, the probability of accepting worse solutions is large, so the algorithm is in a "broad exploration" state. As T gradually decreases ("cooling"), the acceptance probability drops, and the algorithm shifts into a "fine-tuning" state, trending towards convergence. To increase the chances of finding the global optimum, I run SA independently multiple times from different random initial states, and pick the best result out of all of them (multi-start).

The core advantage of using SA for a blob is that the evaluation function can be **any kind of joint function**, it doesn't need to be linearly separable. That means I can just throw the "overall squad state" into the function for scoring, naturally supporting all those joint values I mentioned earlier.

### How to apply SA in blob

What is the "state" in SA, in the context of blob? The most direct one is: **the movement direction of all creeps for this tick** (9 options: stay + 8 directions).

The process goes roughly like this:

- Initialization: all creeps don't move (stay).
- Each step, randomly perturb the next direction of one creep.
- Calculate the score of the new state.
- Decide whether to accept it based on the score difference and current temperature, following the Metropolis criterion.

**First challenge: How to make sure the movement is legal and two creeps don't crash into each other?**

Static constraints (hitting walls, going out of bounds, fatigue) are easy to handle - I just budget a "legal direction mask" for each creep in advance, and only sample from legal directions when perturbing.

Dynamic conflicts (two creeps wanting to go to the same tile) are more annoying. My solution is a **chained DFS perturbation**. If creep A wants to go to a tile that is already taken by creep B, I recursively force B to pick a new direction. If B's new direction is taken by C, I force C to change... and so on, until the end of the chain has no conflicts. The whole chain is either accepted or rejected together. This method also naturally supports "swaps" (A goes to B's current pos, B goes to A's current pos), because in the chained DFS this forms a valid closed loop.

**Second challenge: How to coordinate healing strategies with movement optimization?**

A healer's position decides if it can do close-range healing or ranged healing, and that also affects whether it can also do damage at the same time (`rangedHeal` conflicts with `rangedAttack`). If movement and healing are decided separately, this relationship is lost.

The fix is straightforward: **put healing targets into the SA state space too**. Each healer gets an extra decision dimension—"who to heal" (including healing itself). During perturbation, there's a certain probability to change a healer's target, letting SA jointly optimize "where to move" and "who to heal". This way, SA can naturally learn whether "moving closer to a hurt ally to use a more efficient close heal" is better than "staying far away to cast rangedHeal but wasting an attack opportunity".

### So how do you score a state?

My scoring function is made up of several independent sub-items weighted together, where each item corresponds to "what aspect of the squad's action I care about". The weights between sub-items can be adjusted independently, and when outputting the score, it also gives the breakdown of each item's contribution. This makes it easy to understand which weight is driving the squad's specific behavior when tuning parameters.

Before getting into specific items, there's a principle: **N-invariance**. A blob squad might have different numbers of creeps, and creeps might die during the fight. I want the same set of weight parameters to work out of the box regardless of the group size, without needing to retune. I implemented this by handling different aggregation types separately: for items that "sum everyone's score", I divide by N to get the average; for items that use softmax to "take the worst-case scenario", I apply an offset correction to the softmax that scales with N. This way, no matter how many creeps are in the team, the numerical scale of each item stays roughly stable.

#### Position Guidance (Attraction)

This expresses "how far the squad is from the siege target". I use a pre-calculated **cost map** instead of a simple straight-line distance. Using Dijkstra starting from the outermost defense structures and expanding outwards, plains have 1 cost and swamps have 5 cost. SA will automatically "pathfind" during optimization, guiding them to the enemy's outermost defenses. I check the cost map at each creep's **post-tick position**, and minimize the average value for the whole squad. More on "post-tick position" later.

#### Attack Output

This expresses "how much effective damage the squad can deal to enemy targets after moving". For each creep, I enumerate all enemy targets within attack range at its post-tick position, and pick the optimal attack plan (single target rangedAttack vs AoE rangedMassAttack - whichever scores higher). Different target types are weighted slightly differently based on importance.

#### Safety

This is the most crucial and complicated item in the scoring function.

The core idea: **Enumerate the assumption of "the enemy focuses fire on my creep_i", calculate the health state for each assumption, and use the worst case among the iterations as the safety score.**

Specifically, for every "enemy focuses fire on creep_i" scenario, I budget how each enemy unit (towers, enemy creeps) would distribute their damage under that assumption - if they can hit i, they *will* focus on i. This allows me to calculate the expected health at the end of the tick for every single one of my creeps (subtract expected damage, add healing). The heal_received is determined by the current healTargets state: who heals who, and how much, directly affects each creep's end-of-tick health, which in turn affects the safety score. This is the core value of putting healing into the SA state space - SA can jointly optimize "who heals who" to find the overall plan that yields the highest safety score.

For aggregation, I use **softmax** instead of a hard min or an average: it behaves closely to "taking the safety of the weakest creep", but it preserves gradients, giving SA a signal to optimize, rather than being like a hard min where it says "nobody else matters, only look at the weakest one". Also, **health needs non-linear processing** - the last few HP before death are worth way more than HP in the front. I use an exponential decay function to model this: when at full health, the penalty is close to zero, but as it gets closer to death, the penalty increases drastically, and when health is negative ("over-killed"), the penalty spikes sharply. This way, "one creep having 1% HP left" will be penalized much more than "three creeps each having 60% HP left".

#### Next Tick Safety

Safety evaluates the "health state at the end of this tick", but the movement direction SA optimizes decides **where the creep will be next tick** - if it stands on a dangerous tile, it will keep taking hits next tick. NextSafety exists to make the blob proactively dodge these dangerous spots. It considers the damage the creep would take if it continues to be focused by enemies at its next-tick position, and how much adjacent healing will be available there next tick. Combining these gives the lowest possible health it might be hit down to. This is the key mechanism that makes the blob proactively "dodge" - when SA sees a tile has a high threat value, it will lean towards picking a tile with a lower threat.

### About Cross-room

Cross-room movement is a pretty headache-inducing problem, so I just integrated it directly into the blob implementation instead of writing a separate cross-room logic layer outside.

My approach is: include the target room plus the four adjacent rooms into the calculation scope. Actually, only rooms where members are actively present, or rooms a member can step into next tick, are included, keeping the overhead bounded. In Screeps, cross-room teleportation happens at the end of the tick—a creep standing on an exit tile will automatically swap with the corresponding exit tile in the other room. In SA, I uniformly use the **post-tick position** (the position after applying the room swap) as the "next tick's position".

There is something very elegant about this design: a creep just needs to choose "stay on the exit tile" (dir=stay), and the SA evaluation function will score it based on its post-tick position (already in the other room). For example, after cross-room, tower damage becomes 0. SA will naturally discover that "standing on the exit tile waiting to cross" is a high-scoring choice, without needing any extra cross-room logic to "tell" it that it's time to cross over.

## What's the future of my blob?

- Add damage targets into the state space too
- Mechanically we can only look-ahead 1 tick, so might add "potential future damage taken" and "potential future damage dealt"
- Top-level decision design: when to retreat
- Implement support for field battles based on the above points
- Handling design for swamp tiles, and implementing pull
- Incremental evaluation, instead of re-calcing everything after perturbation
- WASM

## Bonus video

We don't have a video yet, but watch this replay against Harabi's room (before like 5/20/2026): https://screeps.com/a/#!/history/shardX/W39S1?t=1002126

Until next time, my friends!
