# Causal Discovery and Rashomon Set Clustering Architecture

This document outlines the architecture for causal discovery, DAG enumeration, and plausibility analysis, incorporating the concepts of Rashomon sets, Wasserstein distances, and Bayesian updating.

## User Review Required

> [!WARNING]
> Please review the **Proposed Methodologies** section carefully. Since we are branching into advanced theoretical territory (clustering causal paths using Wasserstein distances and treating them as Rashomon sets), we need to ensure the mathematical assumptions align with your data. 

## Open Questions for Final Clarification

1. **Continuous vs. Discrete Data:** You mentioned modeling clusters as Dirichlet distributions, which implies discrete/categorical data. However, Wasserstein distance to a Normal distribution implies continuous data. Will the data in the CSV be purely continuous, strictly categorical, or mixed? (This determines whether we use Gaussian Mixtures/Wasserstein barycenters or Dirichlet distributions for the clusters).
2. **Interventions:** Since nodes can be labeled as interventional, will the CSV data contain the actual *interventional regimes* (e.g., $do(X=x)$) indicating which rows were intervened upon, or just a static label on the node for the whole dataset?

---

## Proposed Methodologies

### 1. Data and Traces
- **Format:** Tabular CSV files.
- **Scale:** Small graphs ($\approx 15$ nodes). This is computationally excellent, meaning we can afford exact DAG enumeration and dynamic programming for posterior calculations.
- **Annotations:** Nodes will be tagged via metadata as `observational`, `interventional`, or `outcome`. 

### 2. Edge & Path Plausibility
We will implement a modular `PlausibilityEngine` supporting multiple paradigms so you can experiment:

1. **Conditional Correlation (PC-Style):** Plausibility defined by the inverse of the p-value of partial correlation conditioned on a separating set (confounders).
2. **Distributional Distance:** Adjusted (deconfounded) conditional distribution $P(Y | do(X))$. Plausibility is the inverse of the Wasserstein distance between this distribution and a standard Normal (or a null hypothesis distribution).
3. **Bayesian Posterior (My Recommendation):** Because the graph is small ($\approx 15$ nodes), we can calculate the exact or highly-sampled posterior probability of an edge $P(X \rightarrow Y | Data)$ using Bayesian Dirichlet equivalent uniform (BDeu) scores or Bayesian Gaussian equivalent (BGe) scores depending on continuous vs. discrete data. 

### 3. Graph Similarity & Rashomon Set Clustering
Treating clusters of similar causal explanations as **Rashomon Sets** is a cutting-edge approach. We will structure the clustering module as follows:

1. **Markov Factorization of Paths:** For a given path in a DAG (e.g., $X \rightarrow Y \rightarrow Z$), we extract its joint probability distribution using the graph's Markov factorization: $P(X, Y, Z) = P(X)P(Y|X)P(Z|Y)$.
2. **Distance Metric:** We will compute the similarity between two paths by calculating the **Wasserstein Distance** or **KL Divergence** between their factored joint distributions. Recent research on *Causal Optimal Transport* (G-causal transport) supports using Wasserstein distance for causal models.
3. **Clustering (Rashomon Sets):** 
   - If the data is **discrete**, clusters can be modeled hierarchically using **Dirichlet distributions**.
   - If the data is **continuous**, we will use **Gaussian Mixture Models** or find **Wasserstein barycenters** to represent the "centroid" distribution of the cluster.
   - The resulting clusters represent sets of paths (causal explanations) that yield similar observational or interventional distributions (Rashomon sets).

---

## Architecture & Implementation Plan

### Environment Stack
- **Language:** Python 3.11
- **Environment:** Conda (`environment.yml` to be provided containing `causal-learn`, `networkx`, `scipy`, `pot` for Python Optimal Transport, `numpy`, `pandas`).

### Core Modules

#### 1. `DataModule`
- `TraceLoader`: Ingests CSV.
- `NodeAnnotator`: Manages labels (`observational`, `interventional`, `outcome`).

#### 2. `DiscoveryModule`
- `CausalDiscoveryEngine`: Wraps `causal-learn` (PC, GES) to handle observational data and structural constraints based on node labels.
- Outputs a CPDAG.

#### 3. `EnumerationModule`
- `MECEnumerator`: Because $N \approx 15$, we can use efficient backtracking or Dor & Tarsi algorithms to exhaustively orient the CPDAG into all valid DAGs within the Markov Equivalence Class.

#### 4. `PlausibilityModule`
- `EdgeEvaluator`: Implements BDeu/BGe scoring, conditional correlation, and Wasserstein-based plausibility for edges.
- `BayesianUpdater`: Updates edge priors as new CSV trace chunks arrive.

#### 5. `ClusteringModule` (The Rashomon Engine)
- `PathFactorizer`: Extracts the conditional probability tables/functions for a given sub-path.
- `DistributionDistanceCalculator`: Uses `pot` (Python Optimal Transport) to compute Wasserstein distances between paths.
- `RashomonClusterer`: Clusters paths. Evaluates path membership to a cluster centroid.

#### 6. `QueryEngine`
- `GraphMatcher`: Finds specific structural paths across the enumerated DAGs.
- `ResultRanker`: Sorts paths by their plausibility scores and cluster assignments.

## Verification Plan
1. **Sanity Check:** Generate a synthetic ground-truth DAG of 15 nodes. Sample a CSV. Run it through the pipeline and ensure the ground-truth DAG is within the highest-plausibility Rashomon set.
2. **Distance Verification:** Ensure that two paths with near-identical causal mechanisms yield a Wasserstein distance approaching $0$.
