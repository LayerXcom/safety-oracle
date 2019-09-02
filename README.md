# Safety Oracle
Summaries and analysis of safety oracles.

The goal of this repository is written here: https://github.com/LayerXcom/safety-oracle/issues/1


## Definitions in big O notation

`V`: The number of validators

`M`: The number of messages(vertices) in MessageDAG.

`E`: The number of edges in MessageDAG. `E <= MV`.

## Summaries

### Clique Oracle

#### Algorithm

1. Let V be a set of validtor that estimate `CANDIDATE_ESTIMATE` and G be an undirected graph with vertex set V.

2. Every pair (v1, v2) that satisfied the following conditions is connected by a edge. `O(V^2log V + VM) = O(V(VlogV + M))`
    - v1 ≠ v2
    - The justification of the v1 latest_message includes v2. `O(log V)`.
    - The justification of the v2 latest_message includes v1.
    - A v2 message in the v1 latest_message doesn't conflict with `CANDIDATE_ESTIMATE`
    - A v1 message in the v2 latest_message doesn't conflict with `CANDIDATE_ESTIMATE`
    - There is no message that conflicts with `CANDIDATE_ESTIMATE` among v2 messages that have not been seen yet by v1 but are in the view. `O(M)` for v1.
    - There is no message that conflicts with `CANDIDATE_ESTIMATE` among v1 messages that have not been seen yet by v2 but are in the view.

3. Find the maximum weighted clique C of G (exponential time).

4. fault_tolerance = 2 * C_weight - V_weight

See: https://en.wikipedia.org/wiki/Clique_problem#Finding_maximum_cliques_in_arbitrary_graphs


#### Metrics

|| Detect | Update |
-|-|-
| Time complexity | exponential (`O*(2^V)`, `O*(1.2599^V)`, etc) | |
| Space complexity | `O(V^2+E)` | |
| Time to detection | | |
| Threshold | | |

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

5. fault_threshold = 2 * C_weight - V_weight

#### Metrics

|| Detect | Update |
-|-|-
| Time complexity | `O(V^2log V + VM)`| |
| Space complexity | `O(V^2+E)` | |
| Time to detection | | |
| Threshold | | |

### Adversary Oracle

#### Algorithm
Ref: https://github.com/ethereum/cbc-casper/blob/master/casper/safety_oracles/adversary_oracle.py

Simulate based on current MessageDAG.
If the result does not change no matter what happens in the future state, the property is finalized.

#### Metrics

|| Detect | Update |
-|-|-
| Time complexity | `O(V^3)` | |
| Space complexity | | |
| Time to detection | | |
| Threshold | | |

### The Inspector

#### Metrics

|| Detect | Update |
-|-|-
| Time complexity | | |
| Space complexity | | |
| Time to detection | | |
| Threshold | | |


#### Algorithm

See: https://hackingresear.ch/cbc-inspector/

### Ideal Oracle (Necessary and sufficient oracle)

#### Metrics

|| Detect | Update |
-|-|-
| Time complexity | | |
| Space complexity | | |
| Time to detection | | |
| Threshold | | |


## Comparison


### Complexity
||Clique Oracle | Turán Oracle | The Inspector | Ideal Oracle |
-|-|-|-|-
|Time Complexity |exponential|||
|Space Complexity ||||


### Threshold

||Clique Oracle | Turán Oracle | The Inspector | Ideal Oracle |
-|-|-|-|-
|1 ||||
|2 ||||
|3 ||||