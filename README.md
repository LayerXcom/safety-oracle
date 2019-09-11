# Safety Oracle
Safety oracles summaries, analysis and comparison.

The goal of this repository is written here: https://github.com/LayerXcom/safety-oracle/issues/1

## Definitions

`CAN_ESTIMATE`: The candidate estimate.

`ADV_ESTIMATE`: The estimate that the adversary wants to finalize.

t: Byzantine fault tolerance threshold.

### In big-O notation

V: The number of validators.

M: The number of messages(vertices) in MessageDAG.

J: The number of arrows in MessageDAG.

For simplicity, we assumed that V <= M <= J <= MV.

### Lobbying Graph

The lobbying graph G(V,E) is constructed as follows.

1. Let V be a set of validtor that estimates `CAN_ESTIMATE` and G be a directed graph with vertex set V.

2. Every ordered pair (v1, v2) that satisfies the following conditions is connected by a arrow. 
    - v1 ≠ v2
    - The justification of the v1 latest_message includes v2.
    - The justification of the v2 latest_message includes v1.
    - A v2 message in the v1 latest_message doesn't conflict with `CAN_ESTIMATE`
    - A v1 message in the v2 latest_message doesn't conflict with `CAN_ESTIMATE`
    - There is no message that conflicts with `CAN_ESTIMATE` among v2 messages that have not been seen yet by v1 but are in the view.
    - There is no message that conflicts with `CAN_ESTIMATE` among v1 messages that have not been seen yet by v2 but are in the view.


#### Complexity

||Construct|Update|
-|-|-
|Time | O(V^2+VM) | O(V) |
|Space | O(V^2) | O(1) |

The above algorithm uses O(V^2) space because E is O(V^2) for any graph.

In 2, checking if the justification of the message includes a validator requires O(1) time on average case using a hash table. For each validator, getting latest messages of the other validators and checking if the latest message conflicts with `CAN_ESTIMATE` requires O(M), so the total running time is O(VM).

However, if you update the lobbying graph every time you get a message, this process can be improved. Updating the graph when a message comes requires O(V) time, because for a message the number of arrows that are newly connected in MessageDAG is at most V.

<!-- 
メッセージが来たとき、ADV_ESTIMATEだったら、そのバリデータへ入ってる辺を全て除く O(V)でできる
CAN_ESTIMATEだったら、色々更新する必要があるが、これも結局色々情報を持っておけばO(V)でできる
-->

## Summaries

### Clique Oracle

#### Algorithm

1. Construct the lobbying graph or get it.

2. Find the maximum weighted clique C of G in **exponential time**.

3. fault_tolerance = 2 * W(C) - W(V)

See: https://en.wikipedia.org/wiki/Clique_problem#Finding_maximum_cliques_in_arbitrary_graphs

#### Metrics

|| Detect | Detect (with updating)|
-|-|-
| Time complexity | exponential| exponential |
| Space complexity |  | |

Finding a clique requires O*(2^V) time. Even the fastest algorithm requires O*(1.1888^V). ( O*(f(k)) = O(f(k) poly(x)) )

### Turán Oracle

#### Turán theorem

This theorem gives a lower bound on the size of a clique in graphs.
If a graph has many edges, it contain a large clique.

Let G be any graph with n vertices, such that G is K_{r+1}-free graph that does not contain (r+1)-vertex clique.

The upper bound of the number of edge is ![](https://i.gyazo.com/0dca1e7495205a9ddd8277a5bd13e6fa.png).

Let E be the set of edges in G.

![](https://i.gyazo.com/db867537543776cfc9a2ad872d5d7322.png)

r is a lower bound on the size of clique in graphs with n vertices and |E| edges.

#### Algorithm

1. Construct the lobbying graph or get it.

2. Calculate the minimum size of the maximum weighted clique using above formula in O(1) time.

3. Calculate the maximum weigted clique C.

4. fault_tolerance = 2 * W(C) - W(V)

#### Metrics

|| Detect | Detect (with updating)|
-|-|-
| Time complexity | O(V^2 + VM)| O(1) |
| Space complexity | O(V^2) | O(1) |


### Simple Finality Detector

Simple finality detector is a generalization of Clique oracle.

#### Algorithm

1. Construct the lobbying graph G or get it.
2. Let q = n/2 + t.
3. Compute outdegree of vertices in G.
4. C = V.
4. Look for the vertice with outdegree of q or less in C, remove it from C, and update the outdegree counters.
5. Repeat 4.
6. If W(C) > q, the property is finalized.

#### Metrics

|| Detect | Detect (with updating)|
-|-|-
| Time complexity | O(V^2 + VM)| O(V^2) |
| Space complexity | O(V^2 + J) |  |

### The Inspector (Improved Finality Detector)

#### Algorithm

See: https://hackingresear.ch/cbc-inspector/

Recalculating levels happens worst V times and the recalculation runs in O(J) time, so the total time complexity is O(VJ).

#### Metrics

|| Detect | Detect (with updating) |
-|-|-
| Time complexity | O(VJ) |  |
| Space complexity | O(J) | |


### Adversary Oracle (necessary and sufficient conditions)

Simulate based on current MessageDAG.
If the result does not change no matter what happens in all future state, the property is finalized.
But, it is inefficient to simulate every possible future.

### Adversary Oracle (straightforward, like ethereum/cbc-casper's one)
Ref: https://github.com/ethereum/cbc-casper/blob/master/casper/safety_oracles/adversary_oracle.py

This oracle is the simplified simulation algorithm.

#### Algorithm

1. Construct the lobbyin graph or get it.

2. From `view`, we can see that there are validators from which the estimate of the latest messages is `CAN_ESTIMATE` or `ADV_ESTIMATE` or unknown.

3. If the estimate is unknown, we assume that it is `ADV_ESTIMATE` in O(V^2) time.

        C = set()
        for v in V:
            if v in view.latest_messages and view.latest_messages[v].estimate == CAN_ESTIMATE:
                C.add(v)


4. Build `viewables` with the following pseudo code in O(V^2) time.

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
            for v in C:
                can_weight = sum(v2.weight for v2 in viewables[v] if viewables[v2] == CAN_ESTIMATE and v2 in C))
                adv_weight = sum(v2.weight for v2 in viewables[v] if viewables[v2] == ADV_ESTIMATE or v2 not in C))

                if can_weight <= adv_weight:
                    to_remove.add(v)
                    progress_mode = True
                
            C.difference_update(to_remove)
        
        if (total weight of CAN_ESTIMATE) > (total weight of ADV_ESTIMATE):
            the property is finalized


6. If (total weight of `CAN_ESTIMATE`) > (total weight of `ADV_ESTIMATE`) is finally statisfied, the property is finalized.

7. fault_tolerance = the minimum validator weight in C.

#### Metrics

|| Detect | Detect (with updating) |
-|-|-
| Time complexity | O(V^3 + VM) | O(V^3) |
| Space complexity | O(V^2) | |

### Adversary Oracle with Priority Queue (WIP)

#### Fault tolerance

#### Using a priority queue

<!-- 
ethereum/cbc-capser の Adversary Oracle では fault tolerance を the minimum validator weight としているが、fault tolerance は min { can_weight - adv_weight } でもいいと考えられる。なぜなら can_weight - adv_weight が最も小さいバリデータが意見を変えなければ、Cは変わらない。そして、(total weight of `CAN_ESTIMATE`) > (total weight of `ADV_ESTIMATE`) は依然成り立つためである。

また、意見を最も変えやすいノードを優先して調べればいいため、priority queue を使って調べるバリデータを管理すれば、より高速に最終的なCを調べることができる。
-->

#### 


#### Algorithm (without priority queue)

        C = set()
        for v in V:
            if v in view.latest_messages and view.latest_messages[v].estimate == CAN_ESTIMATE:
                C.add(v)

        for v1 in V:
            for v2 in V:
                justification = view.latest_messages[v1].justification
                if v2 not in justification:
                    viewables[v1][v2] = ADV_ESTIMATE
                elif (there is a v2 message that estimate ADV_ESTIMATE and have seen from v1):
                    viewables[v1][v2] = ADV_ESTIMATE
                else:
                    viewables[v1][v2] = (message estimate that the hash is justification[v2])

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
                    fault_tolerance = min(fault_tolerance, can_weight - adv_weight)
                
            C.difference_update(to_remove)
        
        if (total weight of CAN_ESTIMATE) > (total weight of ADV_ESTIMATE):
            the property is finalized


#### Detect all finality that Clique oracle can do




#### Metrics

|| Detect | Detect (with updating) |
-|-|-
| Time complexity | O(V^2 + VM) | O(V^2) |
| Space complexity | O(V^2) | |


## Comparison


### Complexity

#### Detect
||Clique Oracle | Turán Oracle |  The Inspector | Adversary Oracle (straightforward) | Adversary Oracle with Priority Queue |
-|-|-|-|-|-
|Time |exponential| O(V^2 + VM) |  O(VJ)  |  O(V^3 + VM) |  O(V^2 + VM)  |
|Space |  | O(V^2) |  O(J)  | O(V^2) |  O(V^2) |

#### Detect with updating when a message comes
||Clique Oracle | Turán Oracle |  The Inspector | Adversary Oracle (straightforward) | Adversary Oracle with Priority Queue |
-|-|-|-|-|-
|Time |exponential| O(1) |    |  O(V^3) |  O(V^2)  |
|Space| | O(1) |    |   |    |


### Fault tolerance threshold and quorum
![](https://i.gyazo.com/02131195fbf9df360f36f36ae5e135a4.png)


||Clique Oracle | Turán Oracle | The Inspector |  Adversary Oracle (straightforward) | Adversary Oracle with Priority Queue |
-|-|-|-|-|-
|1 ||||
|2 ||||
|3 ||||
