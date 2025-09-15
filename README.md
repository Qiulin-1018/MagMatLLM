# MagMatLLM: LLM‑Guided Genetic Search for Magnetic Insulators

> Accelerated discovery of magnetic insulators via large language models + evolutionary search + first‑principles validation.

<p align="center">
  <img src="docs/figures/pipeline.png" alt="Pipeline diagram" width="720" />
</p>

[![Python 3.10+](https://img.shields.io/badge/python-3.10%2B-blue.svg)](#)
[![License](https://img.shields.io/badge/license-MIT-green.svg)](#license)
[![Reproducibility](https://img.shields.io/badge/reproducible-yes-brightgreen.svg)](#reproducing-our-results)

---

## ✨ Overview

**MagMatLLM** is a closed‑loop framework for discovering **magnetic insulators**. It integrates:

* **LLM** generation and variation of crystal structures (compose / permute / distort);
* **Genetic Algorithms (GA)** for multi‑objective selection across generations;
* **ML surrogate** (e.g., CHGNet) for fast screening of stability and properties;
* **DFT** (phonons, bands, DOS) for definitive validation.

The framework discovers previously unreported candidates under modest compute budgets and validates exemplars with finite band gaps, non‑zero total magnetic moments, and phonon stability.

> Paper (preprint): *Accelerated Discovery of Magnetic Insulators via LLM‑Guided Genetic Algorithm and First‑Principles Validation* (NeurIPS 2025 submission).

---

## 🧱 Key Features

* **LLM + GA**: Iterative generation & optimization for multiple objectives (stability, bulk modulus, total magnetic moment, etc.).
* **Two objective modes**: Weighted‑sum and lexicographic‑with‑thresholds.
* **Data augmentation**: Structure‑level augmentations (e.g., AugLiChem) to combat magnetic data scarcity.
* **Multi‑fidelity validation**: CHGNet relax & screen → DFT phonons/bands/DOS for final confirmation.
* **Extensible**: Portable to other property‑driven searches (MTIs, thermoelectrics, multiferroics, superconductors, photocatalysts…).

---

## 🗂️ Repository Structure

```
MagMatLLM/
├─ configs/                 # YAML configs (surrogate weights, thresholds, objectives)
├─ data/
│  ├─ raw/                  # Raw pulls (TMM, MP)
│  ├─ augmented/            # Augmented structures (AugLiChem outputs)
│  └─ candidates/           # LLM candidates (JSON/MCIF/POSCAR)
├─ docs/
│  └─ figures/              # Pipeline, bands, DOS, phonons, etc.
├─ scripts/
│  ├─ 00_prepare_dataset.py # Data crawl & cleaning (TMM+MP+AugLiChem)
│  ├─ 10_llm_generate.py    # LLM generation & mutations
│  ├─ 20_surrogate_eval.py  # CHGNet screening (E_form/E_hull/bulk/M_tot)
│  ├─ 30_selection.py       # Scoring, ranking, and parent update
│  ├─ 40_dft_preprocess.py  # DFT pre‑processing: supercells/spins/paths
│  └─ 50_analyze_dft.py     # Parse phonons/bands/DOS & export figures
├─ src/
│  ├─ magmatllm/            # Core: prompts, GA, metrics, I/O
│  └─ utils/                # Units, deduplication, geometry checks
├─ notebooks/               # Reproduction & visualization
├─ LICENSE
└─ README.md
```

---

## ⚙️ Installation

```bash
# 1) Create environment (Python 3.10+)
conda create -n magmatllm python=3.10 -y
conda activate magmatllm

# 2) Install dependencies
pip install -r requirements.txt
# Required: pymatgen, chgnet, ase, phonopy, numpy, pandas, torch, pydantic, ruamel.yaml, tqdm

# 3) Optional: GPU acceleration (CUDA/A100 recommended)
```

> **Tip:** On Google Colab, enable a GPU (A100 preferred) and use the demo notebooks in `notebooks/`.

---

## 🔑 API Keys

* **Materials Project** (for property enrichment):

  ```bash
  export MP_API_KEY=YOUR_KEY
  ```

---

## 🧪 Quickstart

### 1) Prepare datasets

```bash
python scripts/00_prepare_dataset.py \
  --tmm_url https://topologicalquantumchemistry.com/magnetic/index.html \
  --save_dir data/raw \
  --augment true --augment_ops rotate,swap,supercell,defect \
  --out data/augmented
```

### 2) LLM generation (composition/mutation)

```bash
python scripts/10_llm_generate.py \
  --parents data/augmented/parents.json \
  --template configs/prompt_template.yaml \
  --model llama-8b  # or llama-70b for more capacity \
  --n_children 200 --seed 42 \
  --out data/candidates/gen01
```

### 3) Surrogate screening & selection

```bash
python scripts/20_surrogate_eval.py \
  --in_dir data/candidates/gen01 \
  --chgnet relax --props e_form,e_hull,bulk,magmom \
  --out results/gen01_eval.csv

python scripts/30_selection.py \
  --scores results/gen01_eval.csv \
  --objective weighted_sum \
  --alpha_e 1.0 --alpha_b 0.5 --alpha_m 0.5 \
  --keep_top 128 --parent_k 16 --parents_per_group 8 \
  --next_parents data/candidates/parents_gen02.json
```

### 4) DFT pre/post‑processing

```bash
# Generate DFT inputs (e.g., QE/VASP)
python scripts/40_dft_preprocess.py --cands results/gen01_eval.csv --out dft/inputs

# After runs finish, parse phonons/bands/DOS
python scripts/50_analyze_dft.py --dft_out dft/outputs --fig_dir docs/figures
```

---

## 🎯 Objective Functions

* **Weighted‑Sum** (smooth trade‑off):
  $O_	ext{ws}(x) = lpha_E E^*_d(x) + lpha_b\, b(x) + lpha_m\, M_	ext{tot}(x)$
* **Lexicographic with thresholds**: satisfy stability thresholds first, then optimize bulk modulus and magnetization—useful when feasible solutions are sparse in early iterations.

> Metrics are normalized via winsorization and min–max. “Higher‑is‑better” metrics are inverted to unify the “lower‑is‑better” ranking semantics.

---

## 📊 Reproducing Our Results

* **Baseline**: MatLLMSearch (LLM+GA).
* **This work**: MagMatLLM (adds magnetic and band‑gap objectives).

Run `notebooks/reproduce_table1.ipynb` to regenerate summary tables and plots, including:

* Candidate counts, surrogate stability rate, DFT pass rate;
* Representative materials’ phonons/bands/DOS;
* Objective/ablation sensitivity.

> Note: DFT calculations require your own licenses/resources (VASP/QE/ABINIT, etc.).

---

## 📁 Data Sources

* **Topological Magnetic Materials (TMM) DB**: seeds for magnetic/topological compounds.
* **Materials Project (MP)**: PBE band gaps, bulk moduli, E\_hull, etc.
* **AugLiChem**: structure‑level augmentation to diversify sparse regimes.

Please follow each data source’s terms and citation requirements.

---

## 📌 Highlights

* **New candidates**: tens of previously unreported structures discovered under constrained compute.
* **Validation**: finite band gaps, non‑zero total magnetic moments, and no imaginary phonon frequencies for exemplars.
* **Efficiency**: 8B LLM + single A100 achieves competitive throughput and candidate quality versus much larger models.

See `docs/figures/` and the paper for details.

---

## 🛠️ Roadmap

* Incorporate **topological indicators** (symmetry indicators, Wilson‑loop proxies, inversion confidence) into objectives/filters.
* Fuse **retrieval‑augmented prompts** with **physics priors** (Wyckoff occupations, layered heavy‑element magnets, prototypes).
* Multi‑fidelity **active learning** to further reduce DFT budget.

---

## 🙋 FAQ

**Q1: Can I run without multi‑GPU A100s?**
Yes. An 8B model on a single A100 / high‑end workstation can complete multiple generate‑screen‑select cycles; offload the DFT stage to a queue or external cluster if needed.

**Q2: Do I have to use CHGNet?**
No. You can swap in M3GNet/others—adjust interfaces and normalization in `configs/`.

**Q3: What’s the working definition of “magnetic insulator” here?**
We require **M\_tot > 0** and **E\_g ≥ 0.025 eV**, plus phonon stability as final confirmation.

---

## 🤝 Contributing

PRs/issues are welcome, especially for:

* New prompt templates and generation operators (permutation/crossover/distortion);
* Robust deduplication and geometry validity;
* Surrogates and objective design;
* DFT workflows and visualization scripts.

Please see `CONTRIBUTING.md` (coming soon).

---

## 📚 Citation

If this repository helps your research, please cite:

```bibtex
@inproceedings{magmatllm2025,
  title     = {Accelerated Discovery of Magnetic Insulators via LLM-Guided Genetic Algorithm and First-Principles Validation},
  booktitle = {NeurIPS},
  year      = {2025},
  note      = {preprint}
}
```

---

## 📜 License

Released under the MIT License. See [LICENSE](LICENSE).

---

## 📝 Disclaimer

This codebase and predictions are for research/education only. Synthesis and applications require your own safety and compliance checks.

---

## 🔗 Acknowledgements

* Open‑source projects: CHGNet, pymatgen, ASE, phonopy, etc.
* Materials Project and TMM databases.
* AugLiChem for structure augmentation.
* The broader open‑source materials‑informatics community.
