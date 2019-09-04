# Safety Oracle
Safety oracles summaries, analysis and comparison.

The goal of this repository is written here: https://github.com/LayerXcom/safety-oracle/issues/1

## Definitions

`V`: The number of validators.

`M`: The number of messages(vertices) in MessageDAG.

`J`: The number of justifications(edges) in MessageDAG.

For simplicity, we assumed that `V <= M <= J <= MV`.

`CAN_ESTIMATE`: The candidate estimate.

`ADV_ESTIMATE`: The estimate that the adversary wants to finalize.

### Threshold



## Summaries

### Clique Oracle

#### Algorithm

1. Let V be a set of validtor that estimate `CAN_ESTIMATE` and G be an undirected graph with vertex set V.

2. Every pair (v1, v2) that satisfied the following conditions is connected by a edge. `O(V^2logV + VM) = O(V(VlogV + M))`
    - v1 ≠ v2
    - The justification of the v1 latest_message includes v2. `O(log V)`.
    - The justification of the v2 latest_message includes v1.
    - A v2 message in the v1 latest_message doesn't conflict with `CAN_ESTIMATE`
    - A v1 message in the v2 latest_message doesn't conflict with `CAN_ESTIMATE`
    - There is no message that conflicts with `CAN_ESTIMATE` among v2 messages that have not been seen yet by v1 but are in the view. `O(M)` for v1.
    - There is no message that conflicts with `CAN_ESTIMATE` among v1 messages that have not been seen yet by v2 but are in the view.

3. Find the maximum weighted clique C of G (exponential time).

4. fault_tolerance = 2 * C_weight - V_weight

See: https://en.wikipedia.org/wiki/Clique_problem#Finding_maximum_cliques_in_arbitrary_graphs


#### Metrics

|| Detect | Update |
-|-|-
| Time complexity | exponential (`O*(2^V)`, `O*(1.2599^V)`, etc) | |
| Space complexity | `O(V^2+J)` | |

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

1. Same as the method of clique oracle

2. Same as the method of clique oracle

3. Calculate the minimum size of the maximum weighted clique using above formula by `O(1)`.

4. Calculate the maximum weigted clique C.

5. fault_tolerance = 2 * C_weight - V_weight

#### Metrics

|| Detect | Update |
-|-|-
| Time complexity | `O(V^2logV + VM)`| |
| Space complexity | `O(V^2+J)` | |

### The Inspector

#### Algorithm

See: https://hackingresear.ch/cbc-inspector/

Recalculating levels happens worst `V` times and the time complexity of the recalculation is `O(J)`.

#### Metrics

|| Detect | Update |
-|-|-
| Time complexity | `O(VJ)` | |
| Space complexity | `O(J)` | |


### Adversary Oracle (necessary and sufficient conditions)

Simulate based on current MessageDAG.
If the result does not change no matter what happens in all future state, the property is finalized.
But, it is inefficient to simulate every possible future.

### Adversary Oracle (straightforward, in ethereum/cbc-casper)
Ref: https://github.com/ethereum/cbc-casper/blob/master/casper/safety_oracles/adversary_oracle.py

This oracle is the simplified simulation algorithm.

#### Algorithm

1. From `view`, we can see that there are validators from which the estimate of the latest messages is `CAN_ESTIMATE` or `ADV_ESTIMATE` or unknown.

2. If the estimate is unknown, we assume that it is `ADV_ESTIMATE`.


        C = set()
        for v in V:
            if v in view.latest_messages and view.latest_messages[v].estimate == CAN_ESTIMATE:
                C.add(v)


3. Build `viewables` with the following pseudo code.

        for v1 in V:
            for v2 in V:
                justification = view.latest_messages[v1].justification
                if v2 not in justification:
                    viewables[v1][v2] = ADV_ESTIMATE
                elif (there is a v2 message that estimate ADV_ESTIMATE and have seen from v1):
                    viewables[v1][v2] = ADV_ESTIMATE
                else:
                    viewables[v1][v2] = (message estimate that the hash is justification[v2])

4. Assume that all validator estimates that may change from `CAN_ESTIMATE` to `ADV_ESTIMATE` are `ADV_ESTIMATE`.

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


5. If (total weight of `CAN_ESTIMATE`) > (total weight of `ADV_ESTIMATE`) is finally statisfied, the property is finalized.

6. fault_tolerance = the minimum validator weight.

#### Metrics

|| Detect | Update |
-|-|-
| Time complexity | `O(V^3+J)` | |
| Space complexity | `O(V^2+J)`| |


### Adversary Oracle with Priority Queue

ethereum/cbc-capser の Adversary Oracle では fault tolerance が the minimum validator weight としているが、fault tolerance は min { can_weight - adv_weight } である。なぜなら can_weight - adv_weight が最も小さいバリデータが意見を変えなければ、Cは変わらず、(total weight of `CAN_ESTIMATE`) > (total weight of `ADV_ESTIMATE`) は依然成り立つためである。

また、意見を最も変えやすいノードを優先して調べればいいため、priority queue を使って調べるバリデータを管理すれば、より高速に最終的なCを調べることができる。


#### Algorithm


#### Metrics

|| Detect | Update |
-|-|-
| Time complexity | `O(V^2+J)` | |
| Space complexity | `O(V^2+J)` | |


## Comparison


### Detect finality
||Clique Oracle | Turán Oracle |  The Inspector | Adversary Oracle (straightforward) | Adversary Oracle with Priority Queue |
-|-|-|-|-|-
|Time Complexity |exponential|`O(V^2logV + VM)`| `O(VJ)` | `O(V^3+J)`| `O(V^2+J)` |
|Space Complexity |`O(V^2+J)`|`O(V^2+J)`| `O(J)` |`O(V^2+J)`|`O(V^2+J)`|


### Time to detection
||Clique Oracle | Turán Oracle | The Inspector |  Adversary Oracle (straightforward) | Adversary Oracle with Priority Queue |
-|-|-|-|-|-
|1 ||||
|2 ||||
|3 ||||

### Threshold

||Clique Oracle | Turán Oracle | The Inspector |  Adversary Oracle (straightforward) | Adversary Oracle with Priority Queue |
-|-|-|-|-|-
|1 ||||
|2 ||||
|3 ||||