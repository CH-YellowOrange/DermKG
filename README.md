# DermKG: A Concept-Centric Knowledge Graph for Interpretable Skin Disease Diagnosis

Official code for the paper **"DermKG: A Concept-Centric Knowledge Graph for Interpretable Skin Disease Diagnosis from Dermatological Morphology"** (IEEE EMBC 2027, under review).

<p align="center">
  <img src="figures_kg/kg07_whole_graph.png" width="520" alt="DermKG whole graph">
</p>

Dermatologists describe lesions with a standardized morphological lexicon (papule, plaque, scale, erythema, ...). DermKG turns this lexicon into a knowledge graph that links dermatologist-annotated concepts to diseases, a three-level diagnostic hierarchy and clinical morphology rules, and uses it to answer two questions:

1. **How much diagnostic information do morphological concepts carry?**
2. **How can concept-to-diagnosis knowledge be represented and used for interpretable diagnosis?**

## Highlights

- **DermKG**: 3,452 nodes of 8 types and 19,365 edges of 11 relation types, built from the SkinCon annotations of Fitzpatrick17k (3,230 images, 48 concepts, 113 diseases).
- **20 clinical rules**, all supported by the training data (15 strong, 5 moderate), plus 17 data-mined rules for long-tail diseases.
- **Information ceiling**: the 3,230 lesions collapse onto only 813 distinct concept vectors, bounding concept-only top-1 accuracy at **38.9%** for diseases and **77.6%** for the nine categories.
- **DermKG-Net**: a GNN that scores lesions against disease prototypes and aggregates them along the hierarchy, raising category-level balanced accuracy from **0.257 to 0.411** over the strongest of 11 baselines (paired bootstrap 95% CI [0.073, 0.231]).
- **Interpretability**: rule-based reasoning chains and concept-level attributions for every prediction.
- **Leakage control**: all label-dependent graph statistics are computed on the training split only, and validation/test diagnoses are removed from the inference graph.

## Repository structure

```
DermKG/
├── dermkg_build.ipynb            # 1. build the knowledge graph (no images needed)
├── paper1_hierarchical.ipynb     # 2. main experiments: ceiling, baselines, KG models, DermKG-Net
├── paper1_figures.ipynb          # 3. 31 publication figures (PDF + PNG, one per cell)
├── dermkg_visualization.ipynb    # 4. KG visualizations from one node to the whole graph
├── paper1_experiments.ipynb      # (optional) earlier 113-class disease-level experiments
├── image.csv                     # SkinCon concept annotations (download, see below)
├── fitzpatrick17k.csv            # Fitzpatrick17k labels (download, see below)
├── data/dermkg/                  # generated: graph, rules, split, triples
├── results/paper1_hier/          # generated: metrics, probabilities, bootstrap results
├── figures_paper1/               # generated: paper figures + captions.md
└── figures_kg/                   # generated: KG visualizations + interactive HTML
```

## Installation

```bash
conda create -n dermkg python=3.10 -y
conda activate dermkg
pip install pandas numpy scikit-learn networkx matplotlib pyvis ipykernel
pip install torch          # only needed for DermKG-Net
```

Users in mainland China can add `-i https://pypi.tuna.tsinghua.edu.cn/simple` to speed up installation.

## Data

DermKG uses only the concept annotations and labels, **no images are required**.

| File | Source | Notes |
|---|---|---|
| `image.csv` | [SkinCon](https://skincon-dataset.github.io) | Download "SKINCON Fitzpatrick17k annotations" and rename to `image.csv` |
| `fitzpatrick17k.csv` | [Fitzpatrick17k](https://github.com/mattgroh/fitzpatrick17k) | `fitzpatrick17k.csv` from the repository root |

Place both files in the repository root. SkinCon `ImageID` (without `.jpg`) matches Fitzpatrick17k `md5hash`. Images flagged "Do not consider this image" by the annotating dermatologists are removed, leaving 3,230 images.

## Quick start

Run the notebooks in order:

```text
1. dermkg_build.ipynb          ->  data/dermkg/
2. paper1_hierarchical.ipynb   ->  results/paper1_hier/
3. paper1_figures.ipynb        ->  figures_paper1/
4. dermkg_visualization.ipynb  ->  figures_kg/
```

Steps 1 and 2 take a few minutes on a laptop CPU; DermKG-Net (4 variants x 3 seeds) takes about 2 minutes on CPU. Set `RUN_GNN = False` in `paper1_hierarchical.ipynb` to skip it.

## DermKG schema

| Node type | Count | Description |
|---|---|---|
| SkinImage | 3,230 | annotated clinical image |
| Concept | 48 | dermatologist-annotated morphological descriptor |
| ConceptGroup | 5 | primary lesion, secondary change, color, shape/surface, vascular pattern |
| Disease | 113 | skin condition |
| DiseaseCategory | 9 | Fitzpatrick17k nine-partition label |
| SuperCategory | 3 | benign, malignant, non-neoplastic |
| SkinType | 7 | Fitzpatrick I to VI, plus unknown |
| ClinicalRule | 37 | 20 expert + 17 mined rules |

**Structural relations** (read from annotations): `HAS_CONCEPT`, `HAS_SKIN_TYPE`, `DIAGNOSED_AS`, `BELONGS_TO`, `IS_A`, `SUBCLASS_OF`

**Knowledge / inference relations** (training split only): `CHARACTERIZED_BY` (smoothed P(concept | disease)), `CO_OCCURS` (PMI), `GOVERNED_BY` (rule premise), `INDICATES` (rule conclusion), `ACTIVATES` (image to rule)

Clinical rules are conjunctions of disjunctive concept clauses, for example

```text
R01  Telangiectasia AND (Papule OR Nodule)  ->  basal cell carcinoma
R03  Burrow                                 ->  scabies
R13  Friable AND (Papule OR Nodule)          ->  pyogenic granuloma
```

Each rule is validated on the training split by support, precision and lift, and assigned a strength: strong (lift >= 5, support >= 10), moderate (lift >= 2) or weak (lift >= 1).

## Main results

Category-level test results (9 classes). DermKG-Net: mean over 3 seeds.

| Model | Acc | Macro-F1 | Balanced Acc | Minority recall |
|---|---|---|---|---|
| Majority class | 0.649 | 0.087 | 0.111 | 0.000 |
| LogReg (bottom-up), best baseline | 0.674 | 0.299 | 0.257 | 0.173 |
| KG-Mixture | 0.688 | 0.271 | 0.237 | 0.149 |
| KG-Fusion | 0.688 | 0.271 | 0.237 | 0.149 |
| DermKG-Net (noKG) | 0.518 | 0.308 | 0.392 | 0.366 |
| DermKG-Net (GCN) | 0.522 | 0.312 | 0.393 | 0.367 |
| DermKG-Net (GCN, no rules) | 0.519 | **0.326** | **0.411** | **0.389** |

Concept-only ceilings (in-sample oracle):

| Level | Accuracy ceiling | Balanced-accuracy ceiling | Majority class |
|---|---|---|---|
| Disease (113) | 0.389 | 0.414 | 0.039 |
| Category (9) | 0.776 | 0.623 | 0.651 |
| Super-category (3) | 0.822 | 0.722 | 0.718 |

## Outputs

| File | Content |
|---|---|
| `data/dermkg/triples_infer.csv` | inference graph as (head, relation, tail, weight) triples, val/test labels removed |
| `data/dermkg/triples_full.csv` | full graph including all `DIAGNOSED_AS` edges |
| `data/dermkg/clinical_rules.csv` | expert and mined rules with support, precision, recall, lift, strength |
| `data/dermkg/dermkg.graphml` | graph for Gephi / Cytoscape |
| `data/dermkg/dermkg_hetero.npz` | per-relation edge indices for PyTorch Geometric `HeteroData` |
| `results/paper1_hier/summary.json` | all test metrics, ceilings and significance tests |
| `figures_kg/*.html` | interactive graph views (open offline in any browser) |

## Limitations

Concepts are provided by dermatologists, so results are an upper bound for automated pipelines. The expert rules were curated from textbook morphology and validated statistically but have not yet been reviewed by an independent dermatologist panel. Fitzpatrick17k labels come from online atlases rather than biopsy-confirmed diagnoses.

## Citation

Code for the paper **"DermKG: A Concept-Centric Knowledge Graph for Interpretable Skin Disease Diagnosis from Dermatological Morphology"** (under review).

Please also cite the datasets:

```bibtex
@inproceedings{groh2021fitz,
  author    = {Groh, Matthew and Harris, Caleb and Soenksen, Luis and others},
  title     = {Evaluating Deep Neural Networks Trained on Clinical Images in Dermatology with the {Fitzpatrick} 17k Dataset},
  booktitle = {Proc. IEEE/CVF CVPR Workshops},
  pages     = {1820--1828}, year = {2021}
}
@inproceedings{daneshjou2022skincon,
  author    = {Daneshjou, Roxana and Yuksekgonul, Mert and Cai, Zhuo Ran and Novoa, Roberto and Zou, James},
  title     = {{SkinCon}: A Skin Disease Dataset Densely Annotated by Domain Experts for Fine-Grained Debugging and Analysis},
  booktitle = {NeurIPS Datasets and Benchmarks Track}, year = {2022}
}
```

## License

Code is released under the MIT License. The SkinCon annotations and Fitzpatrick17k labels are subject to their original licenses (Fitzpatrick17k: CC BY-NC-SA 3.0) and may be used for non-commercial research only. This repository does not redistribute any images.

## Contact

Cheng Huang, Department of Genetics, Washington University in St. Louis (chehua@wustl.edu)
