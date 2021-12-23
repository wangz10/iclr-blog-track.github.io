---
layout: post
title: Euclidean geometry meets graph, a geometric deep learning perspective
tags: []
authors: Wang, Zichen, Amazon Web Services; Shi, Yunzhi, Amazon Web Services; Chen, Xin, Amazon Web Services
---

Graph neural networks (GNN) have been an active area of machine learning research to tackle various problems in graph data. Graph is a powerful way of representing relationships among entities as nodes connected by edges. Sometimes nodes and edges can have spatial features, such as 3D coordinates of nodes and directions along edges. How do we reason over the topology of graphs while considering those geometric features? In this post, we discuss a paper published in ICLR 2021: 

* Bowen Jing, Stephan Eismann, Patricia Suriana, Raphael J Townshend, Ron Dror (2021): [Learning from Protein Structure with Geometric Vector Perceptrons](https://iclr.cc/virtual/2021/spotlight/3449). 


## Graphs and geometric graph
Graph is a common data structure using nodes connected by edges to represent complex relationships among entities. Nodes and edges usually have features on them for GNN to learn from. Sometimes nodes and edges can have spatial features, such as 3D coordinates of nodes and directions along edges, adding a layer of geometric relationships (including distance, direction, angle, etc.) among the entities. In this post, we term this type of graph with vector features on nodes and/or edges as **geometric graph**. 

Small molecules, macro-molecules, and geospatial data are all examples where one can construct geometric graphs: small molecules can be represented by a graph of atoms connected by chemical bonds, where atoms can be described by their coordinates in 3D Euclidean space and chemical bonds can be described by vectors along their directions. The angles between adjacent chemical bonds are known to be indicative of the chemical properties of the molecule such as stability. Similarly, macro-molecules such as proteins can be represented by a graph of amino acid residues connected based on their spatial proximity. Each amino acid residue contains $k$ key atoms, making the node vector feature to be $\mathbf{V}_n \in \mathbb{R}^{k \times 3}$. Spatial networks or transportation networks can be constructed by treating intersections as nodes and roads as edges from urban geospatial data. Furthermore, the geographic coordinates of the intersections and the directions of the roads can be encoded as vector features on nodes and edges, respectively. 

## A Geometric Deep Learning perspective
The term “geometry” in a narrow sense is equivalent to the classic Euclidean geometry. However, other schools of geometries exist such as Riemannian geometry. To unify divergent schools of geometries, the [Klein’s Erlangen Program](https://en.wikipedia.org/wiki/Erlangen_program), views geometry as a study of **invariance** with respect to various transformations.

In the broader sense of geometry, a graph is also an important geometric object because its nodes are invariant to permutations: the ordering of its nodes does not change the property of the entire graph. At a local level, the ordering of immediate neighbors does not affect a node’s attributes such as degree and centrality. With this perspective, when we impose Euclidean geometric features onto a graph to form a *geometric graph*, it is further invariant to Euclidean group transformations $E(n)$ including rotations, translations and reflections.

Following the geometric deep learning blueprint ([Bronstein et al. 2021](https://arxiv.org/abs/2104.13478)), in order to develop an effective geometric deep learning algorithm on *geometric graphs*, one needs to invent equivariant and invariant layers with respect to permutation and $E(n)$ symmetry groups. This can be achieved by extending the GNN to operate in $E(3)$ symmetries (assuming we are dealing with 3D Euclidean space).


## Introducing the GVP paper
Most GNN architectures are unable to reason over the spatial relationships among the nodes and edges in a geometric graph. The study by Jing et al. focuses on tasks operating on the tertiary structures of proteins and frames the method as “learning from structures”. In fact, the problem of learning from geometric graph can be naively solved by ignoring either aspect of the data: 1) relational reasoning on graphs using GNN while ignoring the Euclidean geometry, and 2) geometric reasoning on the protein structures using 3D CNN with voxelized representations. Motivated by combining the best from both worlds of GNN and CNN, the authors developed GVP-GNN, which is a novel GNN architecture with Geometric Vector Perceptrons (GVP) as building blocks. 

### GVP, the Euclid’s neuron 
As its name suggests, GVP is a type of perceptron, just like a basic perceptron used in dense layers. What’s novel about GVP is its ability to jointly process data points with both scalar and vector features. 

So how does GVP do that? Let’s first revisit how perceptrons work. We denote $\mathbf{s} \in \mathbb{R}^h$ as a feature vector where each value in $\mathbf{s}$ has nothing to do with spatial information. A perceptron is simply

$$
\begin{equation}
    \mathbf{s}' = Perceptron(\mathbf{s}) = \sigma(\mathbf{w}^\intercal\mathbf{s} + b)
    \tag{1}
\end{equation}
$$

A GVP deals with a data point that possess vector features $\mathbf{V} \in \mathbb{R}^{\nu \times 3}$ and jointly process $\mathbf{s}$ and $\mathbf{V}$ with the following computations:
$$
\begin{equation}
    \mathbf{s}', \mathbf{V}' = GVP(\mathbf{s}, \mathbf{V})
    \tag{2}
\end{equation}
$$

$$
\begin{equation}
    \mathbf{s}' = \sigma(\mathbf{W}_m
    \begin{bmatrix}
        \| \mathbf{W}_h \mathbf{V} \|_2\\
        \mathbf{s}
    \end{bmatrix}  + \mathbf{b})
    \tag{3}
\end{equation}
$$

$$
\begin{equation}
    \mathbf{V}' = \sigma^+(\| \mathbf{W}_{\mu} \mathbf{W}_{h} \mathbf{V} \|_2) \odot \mathbf{W}_{\mu} \mathbf{W}_{h} \mathbf{V} 
    \tag{4}
\end{equation}
$$

The output $\mathbf{s}'$, $\mathbf{V}'$ from GVP is provable $E(3)$ invariant and equivariant, respectively, with respect to rotations and reflections. As we can easily see in equation (3) that the scalar output $\mathbf{s}'$ only depends on the l2-norm of the vector, which is constant after any rotations/reflections. Similarly, in equation (4), the directions in the vector output $\mathbf{V}'$ is a function of $$\mathbf{W}_{\mu} \mathbf{W}_{h} \mathbf{V}$$, which means it only changes with transformation of $\mathbf{V}$. 

The authors further extended the universal approximation theorem from perceptron to GVP: they proved that GVP can approximate any rotation-invariant functions. Now that we have GVP to take care of the Euclidean transformations, we can use it to build a GNN to tackle geometric graphs. 

### Refresher on GNN basics

GNN typically reasons among neighboring nodes by exchanging messages between them to update node representations with node and edge features, thus integrating the topological structures of the graph when making inferences. The message passing paradigm ([Gilmer et al. 2017](https://arxiv.org/abs/1704.01212)) unifies many GNN architectures as collecting and aggregating messages from neighbors to update the node representation. 

The message function describes how a pair of nodes and the edge connecting them generate the message: 
$$
\begin{equation}
    \mathbf{m}_{ij} = M(\mathbf{h}_i, \mathbf{h}_j, \mathbf{e}_{ij})    
    \tag{5}
\end{equation}
$$

Afterwards, messages collected from all neighbors of a node are aggregated and used for updating the node’s representation:
$$
\begin{equation}
    \mathbf{h}_{i} \leftarrow U(\mathbf{h}_i, AGG_{j\in \mathcal{N}(i)}\mathbf{m}_{ij} )
    \tag{6}
\end{equation}
$$

### GVP-GNN: Message from vector features

GVP-GNN adopts the general message passing scheme of GNNs, allowing nodes and edges to have scalar and vector features. It generates messages using both node and edge features:
$$
\begin{align}
\mathbf{m}_{ij} &= M_{GVP}(\mathbf{s}_j, \mathbf{V}_j, \mathbf{s}_{ij}, \mathbf{V}_{ij})\\
 &= GVP(concat(\mathbf{s}_j, \mathbf{s}_{ij}), concat(\mathbf{V}_j, \mathbf{V}_{ij})) \\
 \tag{7}
\end{align}
$$

We use $\mathbf{h}=(\mathbf{s}, \mathbf{V})$ to simplify notations, GVP-GNN’s update function aggregates scalar and vector messages:
$$
\begin{equation}
\mathbf{h}_{i} \leftarrow LayerNorm(\mathbf{h}_i + \frac{1}{|\mathcal{N}(i)|} Dropout( \sum_{j\in \mathcal{N}(i)}\mathbf{m}_{ij}))
\tag{8}
\end{equation}
$$

It’s worth mention that GVP-GNN is not the first neural architecture to reason over geometric graphs. Other prominent algorithms developed prior to GVP-GNN include SchNet ([Schütt et al., 2017](https://arxiv.org/abs/1706.08566)), Tensor Field Network (TFN) ([Thomas et al., 2018](https://arxiv.org/abs/1802.08219)), and SE(3)-Transformer ([Fuchs et al., 2020](https://arxiv.org/abs/2006.10503)). 

- SchNet is a form of continuous convolution on GNN:
$$
\begin{align}
\mathbf{m}_{ij} 
% &= M_{SchNet}(\mathbf{s}_j, \mathbf{V}_j, \mathbf{V}_{i}) \\
&=  \phi_{CF}(\Vert\mathbf{V}_j - \mathbf{V}_i\Vert) \phi_{S}(\mathbf{s}_j)
\tag{9}
\end{align}
$$

- TFN uses [spherical harmonics](https://en.wikipedia.org/wiki/Spherical_harmonics) to compute its learnable weight kernel $\mathbf{W}^{lk}$ to preserves $SE(3)$ equivariance: 
$$
\begin{align}
\mathbf{m}_{ij} 
% &= M_{TFN}(\mathbf{s}_j, \mathbf{V}_j, \mathbf{V}_{i}) \\
&= \sum_k\mathbf{W}^{lk}(\mathbf{V}_j - \mathbf{V}_i) \mathbf{s}_j^{lk}
\tag{10}
\end{align}
$$

- $SE(3)$-transformer enhanced TFN with the attention mechanims:
$$
\begin{align}
\mathbf{m}_{ij} 
% &= M_{SE3}(\mathbf{s}_i, \mathbf{V}_i, \mathbf{V}_{j}) \\
&= \sum_k\alpha_{ij} \mathbf{W}^{lk}(\mathbf{V}_j - \mathbf{V}_i) \mathbf{s}_j^{lk}
\tag{11}
\end{align}
$$

Observing the message functions (equations 9, 10, 11) from the three preceding algorithms, we note that they all depend on the direction between the vector features: $$\mathbf{V}_j - \mathbf{V}_i$$. In contrast, GVP-GNN achieves $SE(3)$ equivariance of the message via concatenation of vector features of node **and** edge: $$concat(\mathbf{V}_j, \mathbf{V}_{ij})$$. The concatenation allows GVP-GNN to generate an ensemble of vector messages from both nodes and edges, making it more flexible and versatile than the competing methods. 

More concretely, the competing methods are unable to reason with edge vector features that are independent of node vector features. Having indepedent vector features from nodes and edges allows one to construct more expressive geometric graphs. For instance, in the protein graph, the authors encoded unit vector in the direction of neighboring amnio acids as edge vector feature $C\alpha_j - C\alpha_i$, whereas node vector features can represent amino acid's interal direction along $C\beta_i - C\alpha_i$. 

![protein geomtric graph]({{ site.url }}/public/images/2021-12-20-euclidean_geometric_graph/geometric_protein_graph.png)
*Illustration of vector features on a protein geometric graph.* The left panel depicts amino acid residues from a local neighborhood on a protein in 3D Euclidean space. Amino acid residues are colored by their types, with key atoms ($C$, $O$, $C\alpha$, $C\beta$) labeled for two selected residues. Within residues, two vectors ($C\alpha - C\beta$ and $C\alpha - C$) plotted in dashed arrows form the node feature $$\mathbf{V}_i \in \mathbb{R}^{2 \times 3}$$. Between residue i and j, the vector $C\alpha_i - C\alpha_j$ can be used as the edge vector features $\mathbf{V}_{ij} \in \mathbb{R}^{1 \times 3}$. The right panel abstracts the protein geometric graphs and their vector features.

### Inherently interpretable by visualizing the learned vector field

Because GVP-GNN learns and updates vector features on nodes and edges, it’s possible to visualize the learned vector features at the intermediate GVP-GNN layers. The set of learned vector features on a geometric graph actually resembles a **vector field**. We can try to interpret the the learned vector field with some intuitions from physics. For instance, we can imagine the learned vector field represents force or velocity, although such interpretation needs to be rigorously investigated. 

The authors visualized the learned node vector features on protein geometric graphs. We can clearly see the distinctive patterns on different protein graphs: contracting (A and D), rotating (B) expanding (C). It would be really interesting to study if those learned vector fields actually are correlated with molecular forces among the amino acid residues in the protein graphs.


![Figure 2 in the GVP paper]({{ site.url }}/public/images/2021-12-20-euclidean_geometric_graph/GVP_figure2.png)

In addition to looking at the resultant vector fields (analogous to feature maps in CNN), one should also analyze GVP’s kernels ($$\mathbf{W}_{\mu}$$ and $$\mathbf{W}_{h}$$) because they are the counterpart to CNN’s kernels (aka filters), where lower level kernels learns to detect edge and texture and higher level learns more complex images components such as animal eyes (think DeepDream/[Inceptionism](_https://ai.googleblog.com/2015/06/inceptionism-going-deeper-into-neural.html_)).

## Applications of GVP-GNN within and beyond macro-molecules

Jing et al. demonstrated the application of their GVP-GNN to various supervised learning tasks on proteins represented as geometric graphs of amino acid residues. A natural next step is to model the protein geometric graphs at atomic level. With its expressiveness in geometric graphs, it would be interesting to see if high-resolution atomic graph representation of proteins improve various benchmarks in protein biology including protein design, and model quality assessment. Looking ahead, GVP-GNN has promising potential in improving protein structure prediction. For instance, in the AlphaFold2 framework ([Jumper et al. 2021](http://doi.org/10.1038/s41586-021-03819-2)), invariant point attention (IPA) module is currently used to refine the high-resolution residue protein structure. Can GVP-GNN perform better than the IPA module on this task? Geometric graphs constructed from other macro-molecules like RNA and small molecules are immediate testbed for GVP-GNN, we also foresee broader applications in other domains.

### Geospatial data as geometric graphs

In recent development of intelligent transportation systems (ITS), forecasting traffic speed, volume or the density of roads in traffic networks is fundamentally important. Considering the traffic network as a spatial-temporal graph, GNNs are ideally suited to traffic forecasting problems because of their ability to capture spatial dependency. A widely used application is the estimated times of arrival (ETAs) prediction in Google Map by DeepMind ([Lange and Perez, 2020](https://deepmind.com/blog/article/traffic-prediction-with-advanced-graph-neural-networks)). However, current GNN solutions for these use cases only represent the data using regular graphs without geometric features. Traffic accident and anomaly prediction is a particularly challenging example: it involves accident records, geometric road structures, and meteorological conditions in addition to road networks. For example, the direct sunlight behind a steep hill or a sharp turn with no lane separation significantly increase the accident risk, and could be modeled in the GVP-GNN by considering geometric relations between road nodes and edges, i.e., encoding nodes with 3D vector features representing the geographic coordinates (latitude, longitude, and altitude). [Jiang and Luo (2021)](https://arxiv.org/abs/2101.11174) mentioned other use cases such as parking availability and lane occupancy forecasting that are likely to benefit from introducing geometric relations.

Another geospatial use case is the problem of collision-free navigation in multi-robot systems where the robots are restricted in observation and communication range. Positional relations between the robot fleet is crucial to the task. In [Li et al. (2019)](https://arxiv.org/abs/1912.06095), they jointly trained two networks: a CNN that extracts adequate features from local observations, and a GNN that learns to explicitly communicate these features among robots. GVP-GNN has a potential to determine what information is necessary and effectively communicate to facilitate efficnet path planning.

### Application in graph drawing 

Graph drawing algorithms (aka network layout algorithms) are used to compute the 2D or 3D coordinates of nodes from a graph to preserve their local and global topological structures. They are commonly used for visualizing graphs and are closely related to some non-linear dimensionality reduction algorithms such as Laplacian Eigenmaps, which applies principal component analysis on the Lapacian matrix of the affinity graph constructed from the input data. Common graph drawing algorithms simulate physical forces among nodes and edges. For instance, the Fruchterman-Reingold algorithm is an iterative method that defines attractive and repulsive forces among nodes based on whether they are connected, with the objective of minimizing the energy of the system until equilibrium. Force-directed graph drawing algorithms are computationally expensive: cubic time to the number of nodes $\mathcal{O}(n^3)$. This is because the number of iterations is estimated to be $\mathcal{O}(n)$, and in every iteration, all pairs of nodes are visited and their mutual repulsive forces computed ($\mathcal{O}(n^2)$ complexity).

When we revisit the graph drawing problem, we noticed it can be formulated as a node-level tasks on a geometric graph: given a graph, we compute the coordinates of nodes ($\mathbf{V}_i \in \mathbb{R}^{1 \times d}$, where $d$ equals 2 or 3, for visualizing in 2D or 3D) that minimize the energy function. GVP-GNN can be plugged in as a function approximator to solve this problem. The node vector features can be initialized randomly, just like in force-directed graph drawing algorithms where the positions of the nodes are initialized randomly. Graph drawing with GVP-GNN can reduce the time complexity of force-directed algorithms because each message passing epoch of GVP-GNN only costs $\mathcal{O}(m)$, where $m$ is the number of edges, which is guaranteed to be $m \leq n^2$.

## References
* Justin Gilmer, Samuel S. Schoenholz, Patrick F. Riley, Oriol Vinyals and George E. Dahl (2017): “Neural Message Passing for Quantum Chemistry” [arXiv:1704.01212](https://arxiv.org/abs/1704.01212)
* Kristof T. Schütt, Pieter-Jan Kindermans, Huziel E. Sauceda, Stefan Chmiela, Alexandre Tkatchenko and Klaus-Robert Müller (2017): “SchNet: A continuous-filter convolutional neural network for modeling quantum interactions” [arXiv:1706.08566](https://arxiv.org/abs/1706.08566)
* Nathaniel Thomas, Tess Smidt, Steven Kearnes, Lusann Yang, Li Li, Kai Kohlhoff and Patrick Riley (2018): “Tensor field networks: Rotation- and translation-equivariant neural networks for 3D point clouds” [arXiv:1802.08219](https://arxiv.org/abs/1802.08219)
* Fabian B. Fuchs, Daniel E. Worrall, Volker Fischer and Max Welling (2020): “SE(3)-Transformers: 3D Roto-Translation Equivariant Attention Networks” [arXiv:2006.10503](https://arxiv.org/abs/2006.10503)
* Michael M. Bronstein, Joan Bruna, Taco Cohen and Petar Veličković (2021): “Geometric Deep Learning: Grids, Groups, Graphs, Geodesics, and Gauges” [arXiv:2104.13478](https://arxiv.org/abs/2104.13478)
* John Jumper et al. (2021) “Highly accurate protein structure prediction with AlphaFold” Nature [doi.org/10.1038/s41586-021-03819-2](http://doi.org/10.1038/s41586-021-03819-2)
* Andrew Y. Ng, Michael I. Jordan, Yair Weiss (2001): [“On Spectral Clustering: Analysis and an algorithm”](http://citeseerx.ist.psu.edu/viewdoc/summary?doi=10.1.1.19.8100)
* Fruchterman, Thomas M. J.; Reingold, Edward M. (1991), "Graph Drawing by Force-Directed Placement", Software – Practice & Experience, Wiley, 21 (11): 1129–1164, [doi:10.1002/spe.4380211102](https://doi.org/10.1002/spe.4380211102)
* Oliver Lange, Luis Perez (2020): [“Traffic prediction with advanced Graph Neural Networks”](https://deepmind.com/blog/article/traffic-prediction-with-advanced-graph-neural-networks)
* Weiwei Jiang, Jiayun Luo (2021): “Graph Neural Network for Traffic Forecasting: A Survey” [arXiv:2101.11174](https://arxiv.org/abs/2101.11174)
* Qingbiao Li, Fernando Gama, Alejandro Ribeiro, Amanda Prorok (2019): “Graph Neural Networks for Decentralized Multi-Robot Path Planning” [arXiv:1912.06095](https://arxiv.org/abs/1912.06095)