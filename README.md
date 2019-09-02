# Safety Oracle
Analysis of safety oracles.

The goal of this repository is written here: https://github.com/LayerXcom/safety-oracle/issues/1

## Summaries


### Definitions

`V`: Number of validators.

`M`: Number of messages(vertices) in MessageDAG.

`E`: Number of edges in MessageDAG. `E <= MV`

### Clique Oracle

Time complexity: exponential (`O*(2^n)`, `O*(1.2599^n)`, etc)

Space complexity: `O(V^2+M)`

Time to detection: 

Threshold: 

#### Algorithm

See: https://en.wikipedia.org/wiki/Clique_problem#Finding_maximum_cliques_in_arbitrary_graphs


### Turán Oracle


Time complexity: `O(1)`

Space complexity: `O(V+M)`

Time to detection: 

Threshold: 

#### Turán theorem

This theorem gives a lower bound on the size of a clique in graphs.
If a graph has many edges, it contain a large clique.

Let G be any graph with n vertices, such that G is K_{r+1}-free graph that does not contain (r+1)-vertex clique.

The upper bound of the number of edge is ![](https://i.gyazo.com/0dca1e7495205a9ddd8277a5bd13e6fa.png).

Let E be the set of edges in G.

![](https://i.gyazo.com/db867537543776cfc9a2ad872d5d7322.png)

r is a lower bound on the size of clique in graphs with n vertices and |E| edges.

#### Algorithm

Getting the number of edges is `O(1)`.

Getting minimum size of maximum clique is `O(1)`.

Getting sorted validators is `O(1)`.

### Adversary Oracle
Time complexity: `O(V^3)`

Space complexity: 

Time to detection: 

Threshold: 

#### Algorithm
Ref: https://github.com/ethereum/cbc-casper/blob/master/casper/safety_oracles/adversary_oracle.py

Simulate based on current MessageDAG.
If the result does not change no matter what happens in the future state, the property is finalized.

### The Inspector

Time complexity: 

Space complexity: 

Time to detection: 

Threshold: 

#### Algorithm

See: https://hackingresear.ch/cbc-inspector/

### Ideal Oracle (Necessary and sufficient oracle)

Time complexity: 

Space complexity: 

Time to detection: 

Threshold: 

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