# Bridging Vectors and Graphs: Building a Hybrid Knowledge Graph Retrieval System

In recent years, knowledge graph (KG) systems and embedding (vector)-based representations have each proven powerful in their own right. But when used together in a hybrid setup, they unlock richer retrieval and reasoning capabilities. In this post I explain what vectorized knowledge graphs are, why “hybrid retrieval” (graph + vector) matters, the mathematical foundations, and a simple architecture you can build, with example formulas.

---

## What is a Vectorized Knowledge Graph?

A **knowledge graph** is a structured representation of entities (nodes) and relations (edges) that encodes “facts” or relationships in a domain: e.g. people, organizations, concepts, etc., and how they connect. On the other hand, **embeddings** or **vectorization** map items (text, nodes, relations, etc.) into a high-dimensional continuous vector space, where semantic similarity corresponds (approximately) to geometric closeness (direction/angle/distance).

A *vectorized knowledge graph* combines these:

* Each node (and/or relation or some textual description) has an embedding vector.
* The graph structure gives relational/topological information (neighborhoods, paths, graph distance).
* During retrieval or inference, both the vector similarity and graph structure signals are used.

Why combine them?

* Vector similarity captures semantic meaning (“these two texts talk about similar things”).
* Graph structure captures explicit relationships that embeddings might miss (e.g. siblings in KG, hierarchical relations, type constraints, etc.).
* Hybrid retrieval often gives better precision, recall, explainability, and robustness versus using only one of the methods.

---

## How Hybrid Retrieval Works – High-Level

When a query arrives (textual query), the system may do something like:

1. Embed the query → get vector $\mathbf{q}$.
2. Compute vector similarity between $\mathbf{q}$ and node embeddings $\mathbf{x}_i$ in KG.
3. Identify a small set of nodes with high vector similarity (top-K).
4. Use those as “anchors” or “seeds” in the graph; leverage graph structure (paths, distances, neighbors) to expand or refine results.
5. Combine the vector similarity scores and graph-based scores into a final ranking (hybrid score).

This allows balancing semantic match vs relational/inference structure.

---

## Key Mathematical Formulas

Here are the core formulas used in a hybrid retrieval / vectorized KG setup.

---

### 1. Cosine Similarity

Given a query embedding vector $\mathbf{q} \in \mathbb{R}^d$ and a node embedding $\mathbf{x}_i \in \mathbb{R}^d$:

$$
\mathrm{cosine}(\mathbf{q}, \mathbf{x}_i) = \frac{ \mathbf{q} \cdot \mathbf{x}_i }{ \| \mathbf{q} \| \cdot \| \mathbf{x}_i \| }
= \frac{ \sum_{j=1}^d q_j \cdot (x_i)_j }{ \sqrt{\sum_{j=1}^d q_j^2} \cdot \sqrt{\sum_{j=1}^d (x_i)_j^2} }
$$

* $\mathbf{q} \cdot \mathbf{x}_i = \sum_{j=1}^d q_j \cdot (x_i)_j$ is the dot product.
* $\| \mathbf{q} \|$ is the Euclidean norm (length) of $\mathbf{q}$, and similarly $\| \mathbf{x}_i \|$.
* The result is in the interval $[-1, 1]$, though in many embedding systems for text it is non-negative (if negative values in embeddings are uncommon or if vectors are non-negative).

This is a common way to measure semantic similarity.

---

### 2. Graph-Based Decay / Distance Signal

To incorporate graph structure, one often considers distances in the graph between nodes, for example via the shortest path length.

Let:

* $d(i, t)$ be the graph distance (number of edges in shortest path) between candidate node $i$ and some anchor node $t$.
* Define a decay function over graph distance, e.g.:

$$
g(i; t)=\exp(-\lambda \cdot d(i,t)) \quad (\lambda>0)
$$

If you have multiple anchor nodes $t \in A$ (from vector similarity or external seed), then you might take:

$$
g(i)=\max_{t \in \mathcal{N}_K} \exp(-\lambda \cdot d(i,t))
$$

or some other aggregate (average, sum, etc.). This gives a graph-based score that decreases as node is “farther” in the graph from the anchor(s).

---

### 3. Hybrid Score

To combine vector similarity and graph structure, use a weighted combination (linear blend) or another combining function. A simple linear hybrid score is:

$$
S(i)=\alpha \cdot \text{cos}(\mathbf{q},\mathbf{x}_i) + (1-\alpha)\cdot g(i), \quad \alpha\in[0,1].
$$

* $\alpha \in [0,1]$ controls the trade-off between semantic similarity vs graph proximity.
* Special cases:

  * If $\alpha = 1$, purely vector based.
  * If $\alpha = 0$, purely graph-based.

Optionally you can normalize both components into comparable scales before combining.

---

## Example Architecture / Implementation — Simple Case

Here is a sketch of how a minimal system might work.

---

### Data / Indexing Phase

* Collect data: nodes with textual descriptions, relations among nodes.
* Compute embeddings for node texts (e.g. using an LLM embedding model).
* Build:

  1. A vector store: mapping node IDs → embeddings.
  2. A graph structure: adjacency lists or a graph database storing edges among nodes.

---

### Query Phase

Given a query $Q$:

1. Embed $Q$ → $\mathbf{q}$.
2. Compute vector similarities: $\mathrm{cosine}(\mathbf{q}, \mathbf{x}_i)$ for many $i$. Retrieve top-K nodes by vector similarity.
3. These top-K nodes are used as **anchors** $A = \{ t_1, t_2, \dots, t_K\}$.
4. For every candidate node $i$ in a larger pool (or all nodes), compute graph distance $d(i, t)$ to anchors. Then compute $g(i)$ by e.g. $\max_t \exp(-\lambda d(i, t))$.
5. Compute hybrid score:

$$
S(i) = \alpha \cdot \mathrm{cosine}(\mathbf{q}, \mathbf{x}_i) + (1 - \alpha)\cdot g(i)
$$

6. Rank nodes by $S(i)$, return top results.

---

## Variants and Practical Considerations

* **Scale & Efficiency**: Computing distances from many anchors to many nodes can be expensive; often cut off at some‐maximum graph distance (e.g. only nodes within hop ≤ H).
* **Multiple anchors**: Can use more than one anchor; aggregate graph signal via max, average, or something like softmax over distances.
* **Normalization**: Cosine similarity is often between -1 and +1; graph decay $g(i)$ is often between 0 and 1. If cosine is always non-negative (or adjusted to \[0,1]), blending is easier.
* **Tuning hyperparameters**:

  * $\alpha$ (how much weight vector vs graph)
  * $\lambda$ (decay rate over distance)
  * $K$ (how many anchors)
* **Edge types / weighted graph**: Edges might have types or weights; distance could be weighted shortest path, not just unweighted hop count.
* **Embedding quality**: The better / more meaningful the embeddings, the more reliable the semantic similarity component.

---

## Example: Hybrid Retrieval in Practice

Suppose you have nodes:

* Node A: “Quantum computing fundamentals”
* Node B: “Classical algorithms overview”
* Node C: “Quantum algorithm for factoring”
* Node D: “Applications of quantum computing in cryptography”

Edges: e.g., A connected to C, C connected to D, etc.

Query: “What quantum algorithms help factor large numbers?”

* Embedding of query $\mathbf{q}$ is close (in cosine measure) to embedding of Node C and D.
* Anchors = say top-2 vector matches → C, D.
* For another node B, which is semantically somewhat related via embeddings, but graph distance to anchors may be larger. Graph decay signal reduces score.
* Final ranking might prefer C (highest both), then D, then B, etc.

---

## Template / Pseudocode

```text
Inputs:
  Query Q
  Node set N = {n_i}
  Embeddings X = { x_i for n_i }
  Graph G with adjacency or distance function d(.,.)
  Hyperparameters: K (anchors count), α, λ

Procedure:
  1. q = Embed(Q)
  2. For all i in N, compute s_vec[i] = cosine(q, x_i)
  3. Find top K indices A = { t_1, …, t_K } by highest s_vec[·]
  4. For each i in N:
        g_i = max_{t in A} exp( - λ * d(i, t) )
  5. For each i in N:
        S[i] = α * s_vec[i]  + (1 − α) * g_i
  6. Return nodes i with highest S[i]
```

---

## Mathematical Summary

Putting all together, the core formulas are:

$$
\mathrm{cosine}(\mathbf{q}, \mathbf{x}_i) = \frac{ \mathbf{q} \cdot \mathbf{x}_i }{ \|\mathbf{q}\| \; \|\mathbf{x}_i\| }
$$

$$
g(i) = \max_{t \in A} \; \exp( - \lambda \cdot d(i, t) )
$$

$$
S(i) = \alpha \cdot \mathrm{cosine}(\mathbf{q}, \mathbf{x}_i) \;+\; (1 - \alpha) \cdot g(i)
$$

---

## Code

```javascript
import OpenAI from "openai";

// --- Konfigurasi client OpenAI-compatible ---
const client = new OpenAI({
  apiKey: process.env.OPENAI_API_KEY,
  baseURL: process.env.OPENAI_BASE_URL || undefined,
});
const EMB_MODEL = process.env.EMBEDDING_MODEL || "text-embedding-3-small";

/**
 * Node bertipe:
 * - person, org, paper, concept, dsb.
 * Setiap node punya 'text' untuk di-embed
 */
const nodes = [
  { id: "n1", type: "person", text: "Ada Lovelace, pioneer of computing and algorithms." },
  { id: "n2", type: "concept", text: "Analytical Engine: early mechanical general-purpose computer." },
  { id: "n3", type: "paper", text: "A note on Bernoulli numbers and algorithmic computation." },
  { id: "n4", type: "concept", text: "Knowledge Graph: entities and relations capturing real-world facts." },
  { id: "n5", type: "org", text: "Royal Society: scientific academy supporting research and publications." },
];

const edges = [
  ["n1", "n2"], // Ada -> Analytical Engine
  ["n1", "n3"], // Ada -> paper
  ["n3", "n5"], // paper -> Royal Society
  ["n4", "n2"], // KG -> Analytical Engine (contoh relasi konseptual)
];

// --- Utility graph ---
const adj = new Map();
for (const n of nodes) adj.set(n.id, new Set());
for (const [a, b] of edges) { adj.get(a).add(b); adj.get(b).add(a); }

function shortestPathLen(start, goal) {
  if (start === goal) return 0;
  const q = [start];
  const dist = new Map([[start, 0]]);
  while (q.length) {
    const u = q.shift();
    for (const v of adj.get(u) || []) {
      if (!dist.has(v)) {
        dist.set(v, dist.get(u) + 1);
        if (v === goal) return dist.get(v);
        q.push(v);
      }
    }
  }
  return Infinity;
}

// --- Embeddings ---
async function embedTexts(texts) {
  const res = await client.embeddings.create({
    model: EMB_MODEL,
    input: texts,
  });
  return res.data.map((d) => d.embedding);
}

// Cosine similarity
function cosine(a, b) {
  let dot = 0, na = 0, nb = 0;
  for (let i = 0; i < a.length; i++) {
    dot += a[i] * b[i];
    na += a[i] * a[i];
    nb += b[i] * b[i];
  }
  return dot / (Math.sqrt(na) * Math.sqrt(nb));
}

// Hybrid score: alpha*cos + (1-alpha)*graphDecay
function hybridScore(cosScore, graphDecay, alpha = 0.7) {
  return alpha * cosScore + (1 - alpha) * graphDecay;
}

// Graph decay g(i) = max_{t in topK} exp(-lambda * d(i,t))
function graphDecayForNode(nodeId, anchorIds, lambda = 0.7) {
  let best = 0;
  for (const t of anchorIds) {
    const d = shortestPathLen(nodeId, t);
    if (d === Infinity) continue;
    const val = Math.exp(-lambda * d);
    if (val > best) best = val;
  }
  return best; // 0..1
}

async function main() {
  // 1) Embed semua node
  const nodeEmbeddings = await embedTexts(nodes.map(n => n.text));

  // 2) Query
  const query = "early computer pioneers and their work on mechanical computation";
  const [qEmb] = await embedTexts([query]);

  // 3) Skor cosine awal
  const cosScores = nodeEmbeddings.map(e => cosine(qEmb, e));

  // 4) Ambil anchor topK (berdasar cosine) untuk graph signal
  const K = 2;
  const topKIdx = cosScores
    .map((s, i) => [s, i])
    .sort((a, b) => b[0] - a[0])
    .slice(0, K)
    .map(([_, i]) => i);
  const anchorIds = topKIdx.map(i => nodes[i].id);

  // 5) Hitung skor hybrid
  const alpha = 0.7;       // lebih berat ke semantic
  const lambda = 0.7;
  const results = nodes.map((n, i) => {
    const g = graphDecayForNode(n.id, anchorIds, lambda);
    const score = hybridScore(cosScores[i], g, alpha);
    return { node: n, cos: cosScores[i], g, score };
  }).sort((a, b) => b.score - a.score);

  // 6) result
  console.table(results.map(r => ({
    id: r.node.id,
    type: r.node.type,
    text: r.node.text.slice(0, 60) + (r.node.text.length > 60 ? "..." : ""),
    cos: +r.cos.toFixed(4),
    g: +r.g.toFixed(4),
    hybrid: +r.score.toFixed(4),
  })));
}

main().catch(err => {
  console.error(err);
  process.exit(1);
});
```

---

## Conclusion

Vectorized knowledge graphs with hybrid retrieval strategies provide more robust, semantically rich, and structurally aware information retrieval than methods relying solely on embeddings *or* graph structure. By balancing the two, you can capture both meaning and explicit relational structure, improving relevance, explainability, and flexibility.

In practice, the choice of embeddings, graph modeling (edge weights, node types), decay formulas, and blending weights will all influence performance. But the high-level blueprint above gives a solid foundation to build systems that combine what embeddings and graphs individually do well.

[1]: https://en.wikipedia.org/wiki/Cosine_similarity "Cosine similarity"
[2]: https://neo4j.com/blog/developer/hybrid-retrieval-graphrag-python-package "Hybrid Retrieval Using the Neo4j GraphRAG Package for ..."
[3]: https://arxiv.org/html/2408.04948v1 "HybridRAG: Integrating Knowledge Graphs and Vector ..."
