# Safety Oracle
This is a comprehensive summary of safety oracle, finality detection mechanisms of CBC Casper.  
(See [this slide](https://docs.google.com/presentation/d/1j8u8RabU_p7gc6kP3SEoMn_-8Ezmh-7mMij_Iy4DsOE) for a general introduction of CBC Casper and [this post](https://hackingresear.ch/cbc-finality/index.html) about finality detection in CBC Casper.)  
We introduce various existing proposals on safety oracle, describe the algorithms and compare them.
The goal of this project is [here](https://github.com/LayerXcom/safety-oracle/issues/1)


## Table of Contents
* [Definitions](#definitions)
   * [MessageDAG](#messagedag)
      * [Message types](#message-types)
      * [Example](#example)
   * [In big-O notation](#in-big-o-notation)
   * [Lobbying Graph](#lobbying-graph)
      * [Example](#example-1)
      * [Complexity](#complexity)
* [Summaries](#summaries)
   * [Clique Oracle](#clique-oracle)
      * [Algorithm](#algorithm)
      * [Metrics](#metrics)
      * [Why q &gt; n/2   t?](#why-q--n2--t)
   * [Turán Oracle](#turán-oracle)
      * [Turán theorem](#turán-theorem)
      * [Algorithm](#algorithm-1)
      * [Metrics](#metrics-1)
   * [Simple Inspector](#simple-inspector)
      * [Algorithm](#algorithm-2)
      * [Metrics](#metrics-2)
   * [Inspector](#inspector)
      * [Algorithm](#algorithm-3)
      * [Metrics](#metrics-3)
   * [Adversary Oracle (necessary and sufficient conditions for finality)](#adversary-oracle-necessary-and-sufficient-conditions-for-finality)
   * [Adversary Oracle (straightforward, like ethereum/cbc-casper's one)](#adversary-oracle-straightforward-like-ethereumcbc-caspers-one)
      * [Algorithm](#algorithm-4)
      * [Metrics](#metrics-4)
   * [Adversary Oracle with Priority Queue (WIP)](#adversary-oracle-with-priority-queue-wip)
      * [Using a priority queue](#using-a-priority-queue)
      * [Detect all finality that Clique Oracle can do](#detect-all-finality-that-clique-oracle-can-do)
      * [Metrics](#metrics-5)
* [Comparison](#comparison)
   * [Complexity](#complexity-1)
      * [Detect finality](#detect-finality)
      * [Detect finality when a new message comes](#detect-finality-when-a-new-message-comes)
   * [Fault tolerance threshold and quorum](#fault-tolerance-threshold-and-quorum)
   * [Sample 1](#sample-1)
   * [Sample 2](#sample-2)
   * [Sample 3](#sample-3)
   * [Sample 4](#sample-4)

## Definitions

`t`: Byzantine (or equivocation) fault tolerance.

`CAN_ESTIMATE`: The candidate estimate.

`ADV_ESTIMATE`: The estimate that the adversary wants to finalize.


### MessageDAG

*MessageDAG* is a DAG (directed acyclic graph) of messages where edges are directed from a message to the messages in its justification.

#### Message types

![](https://i.gyazo.com/6640a8550a7ddebdec50e9d804e98fde.png)

#### Example

![](https://i.gyazo.com/500e7f5223a2c6552e62cbb7d78ac517.png)


### Notations

`V`: The number of validators.

`M`: The number of messages(vertices) in the MessageDAG.

`J`: The number of arrows in the MessageDAG.

In practice, we can assume that V <= M <= J <= MV.

### Lobbying Graph

We construct a *lobbying graph* `G(V,E)` as follows.

1. Let V be a set of validators that estimates `CAN_ESTIMATE` and `G` be a directed graph with a set of vertices `V`.

2. An arrow connects every ordered pair (v1, v2) that satisfies the following conditions. 
    - v1 ≠ v2
    - The justification of the v1 latest_message includes v2.
    - The justification of the v2 latest_message includes v1.
    - A v2 message in the v1 latest_message doesn't conflict with `CAN_ESTIMATE`
    - A v1 message in the v2 latest_message doesn't conflict with `CAN_ESTIMATE`
    - No message conflicts with `CAN_ESTIMATE` among v2 messages that have not been seen yet by v1 but are in the view.
    - No message conflicts with `CAN_ESTIMATE` among v1 messages that have not been seen yet by v2 but are in the view.

#### Example

![](https://i.gyazo.com/c67c8f09f148fdd775d5c639fdbca3c4.png)

The lobbying graph constructed from the above MessageDAG is:

![](https://i.gyazo.com/ee4c3b1d709eec9fb7c6f6e30a7cf05a.png)

#### Complexity

||Construct|Update when a new message comes|
-|-|-
|Time | O(V^2 + VM) | O(V) |
|Space | O(V^2) | - |

In 2, checking if the justification of the message includes a validator requires `O(1)` time on average case using a hash table. For each validator, getting the latest messages of the other validators and checking if the latest message conflicts with `CAN_ESTIMATE` requires `O(M)`.
Therefore, the total running time is `O(VM)`.

However, if you update the lobbying graph every time you get a message, this process can be improved. Updating the graph when a message comes requires O(V) time because for a message the number of newly connected arrows in the MessageDAG is at most `V`.

The space complexity is `O(V^2)` because `E` is `O(V^2)` for any graph. Of course, the space complexity of the MessageDAG is `O(J)`. However, a validator must always have it, so we don't consider it's space in safety oracles.


## Summaries

### Clique Oracle

#### Algorithm

1. Construct the lobbying graph or get it.

2. Convert the lobbying graph to an undirected graph by connecting edges (valdiators) connected bidirectionally.

3. Find the maximum weighted clique C of G in **exponential time**.

4. Byzantine fault tolerance threshold `t = ceil(W(C) - W(V)/2) - 1`. (`q > n/2 + t`)

See: https://en.wikipedia.org/wiki/Clique_problem#Finding_maximum_cliques_in_arbitrary_graphs

#### Metrics

|| Detect | Detect when a new message comes|
-|-|-
| Time complexity | exponential| exponential |
| Space complexity | -  | - |

Finding a clique requires O*(2^V) time. Even [the fastest algorithm](https://www.labri.fr/perso/robson/mis/techrep.html) requires O*(1.1888^V). ( O*(f(k)) = O(f(k) poly(x)) )

#### Why q > n/2 + t?

`q > n/2 + t`, so `t < q - n/2`.

![](https://i.gyazo.com/578fab8c55b2c1b83d48013366837dcf.png)

In the above case, `n = 8, q = 7, t < 7 - 8/2 = 3`.

This result means that up to t-1=2 equivocation failures can be tolerated.

2 equivocating validators:

![](https://i.gyazo.com/6ebde608f9d3fef9054d13d9607248b4.png)

3 equivocating validators:

![](https://i.gyazo.com/927ff421ec5fef56121922d05a188990.png)

Therefore, Clique Oracle fault tolerant threshold formula isn't `q > n/2 + t/2`.

When satisfying the formula `q > n/2 + t/2`, `t < 2q - n = 14 - 8 = 6`.

### Turán Oracle

#### Turán theorem

This theorem gives a lower bound on the size of a clique in graphs.
If a graph has many edges, it contains a large clique.

Let G be any graph with n vertices, such that G is K_{r+1}-free graph that does not contain (r+1)-vertex clique.

The upper bound of the number of edge is 

![](https://i.gyazo.com/0dca1e7495205a9ddd8277a5bd13e6fa.png).

Let E be the set of edges in G.

![](https://i.gyazo.com/db867537543776cfc9a2ad872d5d7322.png)

r is a lower bound on the size of clique in graphs with n vertices and |E| edges.

#### Algorithm

1. Construct the lobbying graph or get it.

2. Convert the lobbying graph to an undirected graph by connecting edges (valdiators) connected bidirectionally.

3. Calculate the minimum size of the maximum weighted clique using the above formula in O(1) time.

4. Calculate the maximum weighted clique C.

5. `t = ceil(W(C) - W(V)/2) - 1`. (`q > n/2 + t`)

#### Metrics

|| Detect | Detect when a new message comes|
-|-|-
| Time complexity | O(V^2 + VM)| O(1) |
| Space complexity | O(V^2) | - |


### Simple Inspector

Simple Inspector is a generalization of Clique Oracle.

#### Algorithm

1. Construct the lobbying graph G or get it.
2. Let `q > n/2 + t`.
3. Compute outdegree of vertices in G.
4. C = V.
4. Look for the vertice with outdegree of q or less in C, remove it from C, and update the outdegree counters.
5. Repeat 4.
6. If `W(C) >= q`, the property is finalized.

#### Metrics

|| Detect | Detect when a new message comes|
-|-|-
| Time complexity | O(V^2 + VM)| O(V^2) |
| Space complexity | O(V^2) | - |

### Inspector

#### Algorithm

See: https://hackingresear.ch/cbc-inspector/

For some q (<= V), computing levels happen worst V times, and the computation runs in O(J) time.
Therefore the total time complexity is O(V * V * J) = O(V^2J).

![](https://i.gyazo.com/76474dcf86e942204f51fdfb83206d22.png)

#### Metrics

|| Detect | Detect when a new message comes |
-|-|-
| Time complexity | O(V^2J) | O(V^2J) |
| Space complexity | O(J) | - |


### Adversary Oracle (necessary and sufficient conditions for finality)

Simulate based on current MessageDAG.
If the result does not change no matter what happens in all future states, the property is finalized.
But, it is inefficient to simulate every possible future.

### Adversary Oracle (straightforward, like ethereum/cbc-casper's one)
Ref: https://github.com/ethereum/cbc-casper/blob/master/casper/safety_oracles/adversary_oracle.py

This oracle is the simplified simulation algorithm.

#### Algorithm

1. Construct the lobbying graph or get it.

2. From `view`, we can see that there are validators from which the estimate of the latest messages is `CAN_ESTIMATE` or `ADV_ESTIMATE` or unknown.

3. If the estimate is unknown, we assume that it is `ADV_ESTIMATE` in O(V^2) time.

        C = set()
        for v in V:
            if v in view.latest_messages and view.latest_messages[v].estimate == CAN_ESTIMATE:
                C.add(v)


4. Build `viewables` with the following pseudocode in O(V^2) time.

        for v1 in V:
            for v2 in V:
                justification = view.latest_messages[v1].justification
                if v2 not in justification:
                    viewables[v1][v2] = ADV_ESTIMATE
                elif (there is a v2 message that estimate ADV_ESTIMATE and have seen from v1):
                    viewables[v1][v2] = ADV_ESTIMATE
                else:
                    viewables[v1][v2] = (message estimate that the hash is justification[v2])

5. Assume that all validator estimates that may change from `CAN_ESTIMATE` to `ADV_ESTIMATE` are `ADV_ESTIMATE`.

        progress_mode = True
        while progress_mode:
            progress_mode = False

            to_remove = set()
            fault_tolerance = -1
            for v in C:
                can_weight = sum(v2.weight for v2 in viewables[v] if viewables[v2] == CAN_ESTIMATE and v2 in C))
                adv_weight = sum(v2.weight for v2 in viewables[v] if viewables[v2] == ADV_ESTIMATE or v2 not in C))

                if can_weight <= adv_weight:
                    to_remove.add(v)
                    progress_mode = True
                else:
                    if fault_tolerance == -1:
                        fault_tolerance = can_weight - adv_weight
                    fault_tolerance = ceil(min(fault_tolerance, can_weight - adv_weight) / 2) - 1
                
            C.difference_update(to_remove)
        
        if (total weight of CAN_ESTIMATE) > (total weight of ADV_ESTIMATE):
            the property is finalized

6. If (total weight of `CAN_ESTIMATE`) > (total weight of `ADV_ESTIMATE`) is finally satisfied, the property is finalized.

7. `t = ceil(min_{v in C}{(can_weight of v) - (adv_weight of v)} / 2) - 1`.

**N.B. The original ethereum/casper's fault tolerance threshold t is the minimum validator weight in C, but we think that it is min{can_weight - adv_weight}/2**

#### Metrics

|| Detect | Detect when a new message comes |
-|-|-
| Time complexity | O(V^3 + VM) | O(V^3) |
| Space complexity | O(V^2) | - |

### Adversary Oracle with Priority Queue (WIP)

We think that Adversary Oracle is faster using a priority queue.

#### Using a priority queue

<!-- 
ethereum/cbc-capser の Adversary Oracle では fault tolerance を the minimum validator weight としているが、fault tolerance は min { can_weight - adv_weight } でもいいと考えられる。なぜなら can_weight - adv_weight が最も小さいバリデータが意見を変えなければ、Cは変わらない。そして、(total weight of `CAN_ESTIMATE`) > (total weight of `ADV_ESTIMATE`) は依然成り立つためである。

また、意見を最も変えやすいノードを優先して調べればいいため、priority queue を使って調べるバリデータを管理すれば、より高速に最終的なCを調べることができる。
-->

#### Detect all finality that Clique Oracle can do

// introducing the idea of observable equivocations

#### Metrics

|| Detect | Detect when a new message comes |
-|-|-
| Time complexity | O(V^2 + VM) | O(V^2) |
| Space complexity | O(V^2) | - |


## Comparison


### Complexity

#### Detect finality
||Clique Oracle | Turán Oracle | Simple Inspector | Inspector | Adversary Oracle | Adversary Oracle with Priority Queue |
-|-|-|-|-|-|-
|Time |exponential| O(V^2 + VM) | O(V^2 + VM) |  O(V^2J)  |  O(V^3 + VM) |  O(V^2 + VM)  |
|Space | - | O(V^2) |  O(V^2) | O(J)  | O(V^2) |  O(V^2) |

#### Detect finality when a new message comes
||Clique Oracle | Turán Oracle | Simple Inspector | Inspector | Adversary Oracle | Adversary Oracle with Priority Queue |
-|-|-|-|-|-|-
|Time |exponential| O(1) | O(V^2) |  O(V^2J)  |  O(V^3) |  O(V^2)  |

### Fault tolerance threshold and quorum
![](https://i.gyazo.com/02131195fbf9df360f36f36ae5e135a4.png)

This image is the relationship between Byzantine (or equivocation) fault tolerance threshold and quorum.

The line `q = n - t` represents the maximum number of honest validators.

Safety oracles that satisfies `q > n/2 + t` can achieve that Byzantine fault tolerance threshold is `1/4`.
On the other hand, safety oracles that satisfies `q > n/2 + t/2` can achieve that it is `1/3`.

### Sample 1

![](https://i.gyazo.com/f10641dcf34ae64353666fb32b41a63f.png)

||Clique Oracle | Turán Oracle | Simple Inspector | Inspector |  Adversary Oracle |
-|-|-|-|-|-
t|1|1|1|1|1

### Sample 2

![](https://i.gyazo.com/9d69aa78d3ea09a9b2b8dc4bd051388c.png)

||Clique Oracle | Turán Oracle | Simple Inspector | Inspector |  Adversary Oracle |
-|-|-|-|-|-
t|1|1|1|2|1

In this sample, Inspector fault tolerance threshold is: 

`t = ceil((1-2^(-2))(2q - n)) - 1 = ceil((3/4)*(8-4)) - 1 = 2`.

### Sample 3

![](https://i.gyazo.com/57bb43bd029919552f6f11ebbd074261.png)

||Clique Oracle | Turán Oracle | Simple Inspector | Inspector |  Adversary Oracle |
-|-|-|-|-|-
t|-1|-1|1|1|1

In this case, Clique Oracle and Turán Oracle has not yet detected finality.

{A,B,C,D,E} is the quorum set and `q = 4`, so fault tolerance threshold of Simple Inspector and Inspector `t = ceil(q - n/2) - 1 = ceil(4 - 5/2) - 1 = 1`.

The `t` of Adversary Oracle is also `ceil((4-1)/2) - 1 = 1`.

### Sample 4

![](https://i.gyazo.com/578fab8c55b2c1b83d48013366837dcf.png)

||Clique Oracle | Turán Oracle | Simple Inspector | Inspector |  Adversary Oracle |
-|-|-|-|-|-
t|2|2|2|2|2

