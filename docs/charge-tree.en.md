# Charge Trees: Definition and Foundations

## 1. Scope

This document records the general idea of **Charge Trees** independently of any particular geometry, metric, or task. The **Dimensional Charge Tree (DCT/KDCT)** is one instantiation of this model and is described separately.

The purpose is not to claim originality or present experimental results. The proposal is to formalize a family of trees whose structure adapts locally to a data stream through an explicit notion of **charge**.

## 2. Central idea

A Charge Tree separates two decisions that many data structures handle as one:

1. **When should a local region change?** — decided by the charge function.
2. **How should that region change?** — decided by the split, merge, or reorganization policy.

\[
\underbrace{\phi(S_v)}_{\text{when to reorganize}}
\qquad\text{and}\qquad
\underbrace{\Pi(S_v)}_{\text{how to split}}.
\]

This separation is the defining idea of the family. A KD-tree, a metric tree, or a tree of local classifiers may lead to different Charge Tree variants when local maintenance is explicitly controlled by charge.

## 3. General definition

A **Charge Tree** is a rooted, dynamically maintained tree in which:

- each leaf \(v\) represents a logical region and stores a set or multiset of elements \(S_v\);
- each internal node represents a decomposition of its region among child nodes;
- a charge function \(\phi\) maps the local state of a node to a charge value \(\chi_v\);
- an upper capacity \(C\), or an equivalent condition, determines when a leaf is overloaded;
- a split policy \(\Pi\) replaces an overloaded leaf with a local decomposition;
- optionally, a lower threshold \(Y<C\) and a merge policy recompose regions whose charge has decreased;
- insertions and removals normally affect only the relevant path and its local structural neighborhood.

For a leaf \(v\):

\[
\chi_v=\phi(S_v,A_v),
\]

where \(A_v\) is optional auxiliary state, such as statistics, summaries, histograms, local models, or temporal information. The simplest form is:

\[
\phi(S_v)=|S_v|.
\]

Charge therefore does not have to be merely an element count.

## 4. Fundamental components

### 4.1 Element

An element \(x\) is the unit stored by the tree. Depending on the instantiation, it may be a feature vector, an object in a metric space, a labeled sample, a stream event, or another representation accepted by the routing and query functions.

The identity of an element and its routing representation do not have to be the same. An identifier may support removal, while attributes, distances, or learned rules determine the path through the tree.

### 4.2 Region

Each node represents a logical region \(R_v\) of the domain. A region may be:

- an interval or hyperrectangle in a dimensional tree;
- a range defined by a center and radius in a metric tree;
- a set of conditions in a classification tree;
- any partition with consistent routing rules.

An internal node does not need to store elements directly. Its minimum responsibility is to hold the decision that routes an element to one of its children.

### 4.3 Charge

Charge measures the structural or semantic pressure placed on a region. Possible charge functions include:

- number of elements;
- number of distinct representations;
- accumulated weight or volume;
- class impurity;
- error of a local model;
- temporal instability or evidence of concept drift;
- a combination of mass, impurity, error, and time.

For example, a supervised variant might use:

\[
\phi(v)=\alpha\,\operatorname{mass}(v)
       +\beta\,\operatorname{impurity}(v).
\]

Two leaves containing the same number of elements may consequently have different charges.

### 4.4 Capacity and hysteresis

The capacity \(C\) is the upper charge threshold. The basic split condition is:

\[
\chi_v\ge C.
\]

An implementation must state whether a split happens at \(\chi_v=C\) or only when \(\chi_v>C\).

When merging is supported, a lower threshold \(Y\) is normally used:

\[
Y<C.
\]

The gap between \(Y\) and \(C\) creates **hysteresis**, preventing small charge variations from continuously alternating between split and merge.

### 4.5 Routing policy

The routing function of an internal node selects the child that should receive an element:

\[
\rho_v(x)\in\operatorname{children}(v).
\]

The rule used for future insertions must remain compatible with the partition created by the split.

In supervised applications, labels may influence charge or split selection during training. Routing an unknown sample, however, must depend only on information available at prediction time.

### 4.6 Split policy

The split policy receives the state of a leaf and produces a routing decision and two or more children:

\[
\Pi(v,S_v)=(\theta_v,v_1,\ldots,v_m),
\]

where \(\theta_v\) describes the rule installed in the new internal node.

A valid split should:

1. produce at least two nonempty groups, unless a variant explicitly defines another behavior;
2. assign every element to a child;
3. create a rule that can route future insertions;
4. preserve the invariants of the instantiation;
5. terminate even when the data cannot be separated.

If all elements are indistinguishable to the split policy, the tree must not generate empty splits indefinitely. Possible responses include temporarily accepting an over-capacity leaf, storing multiplicities, or choosing another valid split rule.

### 4.7 Merge policy

Merge is optional in the broad definition, but it is especially useful for streams in which elements expire.

A merge policy \(M\) may replace an internal node and its descendants with a leaf when their combined charge becomes sufficiently small:

\[
\phi\!\left(\bigcup_i S_{v_i}\right)\le Y.
\]

The merge must preserve all active elements, rebuild or combine their Internal Structures, and update local summaries.

### 4.8 Internal Structure (EI)

The **Internal Structure (EI)** is the local mechanism a leaf uses to store, organize, and access its elements. The acronym **EI** is retained from the Portuguese term *Estrutura Interna*. It is an abstraction rather than a fixed structure or index.

This separation allows:

- the Charge Tree to organize relationships among regions;
- the EI to implement operations within a leaf;
- different workloads to use different local representations without changing the definition of the tree.

A possible conceptual EI interface is:

~~~text
insert(x)               insert an element
remove(id or x)         remove an occurrence or a specific identity
elements()              iterate over active elements
size()                  return the stored amount
chargeState()           expose state needed by the charge function
partition(theta)        distribute elements according to a split decision
merge(EI_1, ..., EI_m)  combine Internal Structures during a merge
~~~

Optional operations include:

~~~text
contains(x)
count(x)
query(q)
summary()
compact()
~~~

An EI may be implemented as a vector, list, hash table, multiset, ordered buffer, small local index, or specialized model. This choice changes local costs but not the identity of the Charge Tree.

## 5. Basic invariants

### 5.1 Unique membership

Each active element logically belongs to a single leaf, unless replication is explicitly part of the variant.

### 5.2 Consistent routing

For every element \(x\) stored below an internal node \(v\), \(\rho_v(x)\) must select the descendant containing \(x\).

### 5.3 Conservation across split and merge

Split and merge must neither create nor lose elements:

\[
S_v=\biguplus_i S_{v_i},
\]

including multiplicities when \(S_v\) is a multiset.

### 5.4 Consistent charge

After an insertion, removal, split, or merge, the stored or computed charge must match the current leaf state.

### 5.5 Feasible split

Exceeding capacity does not imply that a valid split always exists. An implementation must distinguish an **overloaded leaf** from a **separable leaf**.

### 5.6 Safe summaries

If a node summary is used for exact pruning, it must provide a proven safe bound. Heuristic summaries may guide traversal order but must not eliminate potentially valid results from a query declared exact.

## 6. Fundamental operations

### 6.1 Leaf location

~~~text
locateLeaf(x):
    v <- root
    while v is internal:
        v <- rho_v(x)
    return v
~~~

If \(h\) is the height and \(R\) the cost of one routing decision:

\[
T_{\mathrm{route}}=O(hR).
\]

### 6.2 Insertion

~~~text
insert(x):
    v <- locateLeaf(x)
    EI_v.insert(x)
    update charge and summaries along the affected path
    if v is overloaded and separable:
        split(v)
~~~

An ordinary insertion costs:

\[
O(hR+I_{EI}+U),
\]

where \(I_{EI}\) is the EI insertion cost and \(U\) is the cost of updating charge and summaries. A split adds the partition cost \(P(|S_v|)\).

### 6.3 Split

~~~text
split(v):
    decision, groups <- Pi(v, EI_v.elements())
    if the partition is invalid:
        keep v as a leaf that this policy cannot currently separate
        return
    create the children and their EIs
    distribute all elements
    replace v with the routing decision
    recompute local charge and summaries
~~~

A split is **local**: by definition, it does not require rebuilding the entire tree.

### 6.4 Removal

If removal receives only an element, the tree may need to locate it again. A FIFO stream or sliding window may optionally keep a map from an element identifier to its leaf.

~~~text
remove(id or x):
    v <- locate the element
    EI_v.remove(id or x)
    update charge and summaries
    evaluate merges starting from v
~~~

With an **id-to-leaf** map, location may be constant on average. The reference map is an operational optimization rather than a mandatory part of the definition.

### 6.5 Merge

~~~text
tryMerge(v):
    p <- parent of v
    while p can be replaced by a leaf:
        if the combined charge of eligible descendants <= Y:
            p <- merge(descendants)
        else:
            stop
~~~

A variant should specify:

- which descendants may be merged;
- whether all of them must be leaves;
- how their EIs are combined;
- how charge and summaries are rebuilt;
- whether merging may continue toward the root.

### 6.6 Query

The general Charge Tree definition does not impose a particular query. A query may follow one route to a leaf, visit several regions through branch-and-bound, query a local model, or combine answers from multiple leaves. The query algorithm and cost therefore belong to the concrete instantiation.

## 7. Data-stream maintenance

Charge Trees were conceived with data streams in mind:

~~~text
for each new element x_t:
    insert(x_t)
    if the window exceeds its limit:
        remove(oldest_element)
~~~

Structural effects remain local:

- increasing charge may trigger a split;
- decreasing charge may trigger a merge;
- distribution changes may trigger reorganization when the chosen charge function detects them;
- the structure does not rely on mandatory global rebuilding.

This does not guarantee global balance or eliminate optional rebuilds. The more modest conceptual guarantee is that the normal unit of adaptation is local.

## 8. Parametric costs

Because a Charge Tree is a framework, its costs depend on concrete choices. Let:

- \(h\): tree height;
- \(R\): cost of one routing decision;
- \(I_{EI}\): EI insertion cost;
- \(D_{EI}\): EI removal cost;
- \(Q_{EI}\): local query cost;
- \(U\): charge and summary update cost;
- \(P(n)\): cost of splitting a leaf containing \(n\) elements;
- \(M(n)\): cost of merging regions containing \(n\) elements in total.

| Operation | General cost |
|---|---:|
| Route to a leaf | \(O(hR)\) |
| Insert without split | \(O(hR+I_{EI}+U)\) |
| Insert with split | \(O(hR+I_{EI}+U+P(n))\) |
| Remove with direct reference | \(O(D_{EI}+U)\), plus an optional merge |
| Remove without direct reference | location cost \(+O(D_{EI}+U)\) |
| Merge | \(O(M(n))\) |
| Query | variant traversal cost \(+Q_{EI}\) in visited leaves |

No general operation should be claimed to cost \(O(\log N)\) unless the concrete variant establishes suitable height and balance properties.

## 9. Properties and limitations

### 9.1 Local adaptation

The basic unit of change is a region. This suits streams in which only part of the domain changes at a given time.

### 9.2 Structural reversibility

When both split and merge exist, a region may specialize and later return to a simpler representation. Reversal does not need to restore the same memory representation; it must preserve the logical element set and all invariants.

### 9.3 Independence between charge and partition

The same partition policy may be paired with different charge functions. Conversely, one charge function may control trees using different partition rules.

### 9.4 No automatic balance guarantee

Limiting leaf charge does not by itself limit tree height. A variant that requires balance guarantees must impose suitable properties on its split policy.

### 9.5 Exactness versus heuristics

A heuristic may choose the most promising child or order traversal. Pruning a region from an exact query requires a safe condition proving that the region cannot contain a better answer.

## 10. Possible instantiations

### 10.1 Dimensional Charge Tree

Uses vectors, axis-aligned cuts, and spatial regions. This is the DCT/KDCT family.

### 10.2 Metric Charge Tree

Uses distances, representatives, and radii to partition a metric space. A VP-tree made dynamic through charge, local splits, and merges is one possible instantiation. “VP Charge Tree” is not presented as an established name in the literature.

### 10.3 Classification Charge Tree

Each leaf may maintain a histogram or local expert. Charge may incorporate mass, impurity, error, or drift. Labels can guide training-time maintenance but cannot be required to route a sample whose class is still unknown.

## 11. Core definition and extensions

The core includes:

- leaves containing elements or local representations;
- an explicit charge function;
- an overload condition;
- a local reorganization policy;
- consistent routing after reorganization;
- preservation of elements and invariants.

Optional mechanisms and variant choices include:

- merge and the threshold \(Y\);
- the concrete EI type;
- an identifier-to-leaf map;
- region summaries;
- branch-and-bound;
- a distance metric;
- boxes, pivots, histograms, or experts;
- compaction and rebuilding policies.

## 12. Short definition

> A Charge Tree is an adaptive tree in which an explicit charge function decides when a local representation should be reorganized, while an independent policy determines how to split, merge, or replace it. Elements are kept in leaves through configurable Internal Structures, and normal maintenance occurs locally in response to insertions, removals, or changes in the observed state.
