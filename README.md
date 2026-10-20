# TokenGauss-Former

### Group-aware Transformers for open-set IoT intrusion detection

TokenGauss-Former is a PyTorch implementation for classifying known network traffic and identifying attack families withheld from training. It combines feature and group tokens, a Transformer encoder, relative token-Gaussian scoring, and class-conditional conformal thresholds.

The accompanying [research notebook](TokenGaussFormer_NIDS_Final.ipynb) contains data preparation, model training, baseline comparisons, open-set evaluation, token-level explanations, ablation studies, and export of figures and tables.

## Main capabilities

- Feature tokenization with additional tokens for groups of related network measurements.
- Relative Mahalanobis scoring at the input embedding and Transformer layers.
- Fusion of token scores with the classification token representation.
- Validation-based thresholds for assigning a prediction to a known class or `UNKNOWN`.
- Comparisons with CNN-BiLSTM, FT-Transformer, RPL, CAC, GraphSAGE, and GAT.
- Supplementary mixture-Mahalanobis scoring with conformal thresholds across baseline backbones.
- Token contribution plots, attack-family fingerprints, and feature-replacement faithfulness analysis.
- Resumable training checkpoints, multiple random seeds, statistical comparisons, and computational measurements.

## Method

1. **Prepare network features.** Clean and deduplicate the data, apply class caps, create ordered train/validation/test partitions within each family, and transform the features. Standardization statistics are fitted using known-class training samples.
2. **Construct feature and group tokens.** Each feature receives a learned linear embedding. Related features also form learned group tokens. The grouping rules cover flags, protocol, size, rate/time, connection, DNS, TLS, HTTP, and other features when present.
3. **Train the Transformer classifier.** A classification token supplies the representation for known-class prediction. The main experiment uses class-weighted cross-entropy; the token-Gaussian regularizer is evaluated in separate ablations.
4. **Fit and combine anomaly scores.** Class-conditional and background Gaussian statistics are fitted to training representations. Relative Mahalanobis distances are computed for individual tokens and the classification token. The default score combines the largest 25% of normalized token scores with the classification-token score at equal weight.
5. **Calibrate the rejection rule.** Known validation samples determine thresholds grouped by predicted class. Classes with fewer than 30 validation predictions use a global threshold. A test sample whose score exceeds its assigned threshold receives the `UNKNOWN` label.

The saved token scores support examination of which input features and feature groups contribute to a flagged sample's anomaly score.

## Datasets and evaluation protocols

The notebook downloads CSV files from the following configured Hugging Face repositories.

| Dataset identifier | Configured source | Files used |
| --- | --- | --- |
| `CICIoT2023` | [bencorn/CIC-IoT-2023](https://huggingface.co/datasets/bencorn/CIC-IoT-2023) | `CSV/MERGED_CSV/Merged01.csv` through `Merged12.csv` |
| `ToN-IoT` | [codymlewis/TON_IoT_network](https://huggingface.co/datasets/codymlewis/TON_IoT_network) | `train_test_network.csv` |
| `CICIoT2023-B` | Same CIC-IoT2023 repository | `CSV/MERGED_CSV/Merged13.csv` through `Merged24.csv` |

CIC-IoT2023 labels are mapped to eight families: Benign, DDoS, DoS, Mirai, Recon, Spoofing, Web, and BruteForce. ToN-IoT families are read from the `type` column. Families that fall below the configured minimum size after preprocessing are removed.

| Dataset | Protocol | Attack families withheld from training and validation |
| --- | --- | --- |
| CIC-IoT2023 | S1 | Web, BruteForce |
| CIC-IoT2023 | S2 | Recon, Spoofing |
| CIC-IoT2023 | S3 | Mirai, Web |
| ToN-IoT | T1 | ransomware, backdoor |
| ToN-IoT | T2 | xss, password |
| ToN-IoT | T3 | mitm, scanning |

Only protocols whose configured unknown families remain after preprocessing are executed. The supplied notebook's saved output retains T2 and T3 for ToN-IoT; ransomware is absent from its retained families.

The default split is 60% training, 10% validation, and 30% testing within each family, ordered by the notebook's source-row `order` field. Held-out families are excluded from model training and threshold calibration. Test partitions contain known and held-out families.

Each dataset has its own fitted models and preprocessing statistics. The final stage repeats S1-S3 on the additional CIC-IoT2023 partitions, fitting models on the training split of `CICIoT2023-B` with the configured settings.

## Getting started

### Google Colab

1. Upload `TokenGaussFormer_NIDS_Final.ipynb` to the repository alongside this README.
2. Open the notebook in Google Colab and select a GPU runtime for the full experiments.
3. Execute cells in order, beginning with Stage 0. Mount Google Drive when prompted.
4. Check the printed configuration, selected device, retained families, and available protocols before training.
5. Continue through the training, analysis, export, and confirmation stages.

The default Colab output directory is:

```text
/content/drive/MyDrive/TempoShield_NIDS
```

`TempoShield_NIDS` is the directory name used in the supplied code. Change the `ROOT` assignment in the configuration cell to use another Colab destination.

### Local Jupyter environment

Install the imported packages in a Python 3 environment:

```bash
python -m pip install torch numpy pandas scipy scikit-learn matplotlib seaborn networkx pyarrow huggingface_hub jupyterlab
```

Launch JupyterLab from the directory containing the notebook:

```bash
jupyter lab TokenGaussFormer_NIDS_Final.ipynb
```

The notebook selects CUDA when available and otherwise selects the CPU. It uses `torch.amp` and the `weights_only` argument of `torch.load`; the installed PyTorch version must support these APIs. The first code cell also checks for `huggingface_hub`, `pyarrow`, `seaborn`, and `networkx` and attempts to install missing packages.

Outside Colab, the default output directory is `./TempoShield_NIDS`. Set `TS_ROOT` in a cell **before the configuration cell** to choose another location:

```python
import os

os.environ["TS_ROOT"] = os.path.abspath("TokenGaussFormer_runs")
```

### Reduced experiment mode

To reduce the per-class sample cap, number of epochs, and number of seeds, set the following environment variable before executing the configuration cell:

```python
import os

os.environ["TS_SMOKE"] = "1"
```

This mode uses a cap of 500 samples per family, up to 3 training epochs, seed 42, and a batch size of 128. It retains the configured source-file downloads. For full experiments, remove the variable or set it to `"0"` before configuration.

Use a separate output directory for reduced and full experiments so that their cached data and results remain distinct.

## Default configuration

The main settings are defined in `CFG` in Stage 0. Relative token-Gaussian scoring settings are defined in `TM` in Stage 9.

| Setting | Default |
| --- | --- |
| Maximum retained samples per family | 25,000 |
| Minimum retained family size | 300 |
| Train / validation / test fractions | 0.60 / 0.10 / 0.30 |
| Transformer embedding dimension | 96 |
| Transformer layers / attention heads | 2 / 4 |
| Feed-forward dimension | 192 |
| Dropout | 0.1 |
| Optimizer | AdamW |
| Learning rate / weight decay | `3e-4` / `1e-4` |
| Learning-rate schedule | OneCycleLR, 5% warm-up |
| Batch size / maximum epochs | 512 / 40 |
| Early stopping | Validation macro-F1; patience 8; minimum 12 epochs |
| Gradient norm clipping | 1.0 |
| Random seeds | 42, 7, 2026 |
| Graph-baseline neighbours | 10 |
| Gaussian covariance shrinkage | 0.1 |
| Gaussian fitting sample limit | 40,000 stratified training flows |
| Token aggregation | Largest 25% of token scores |
| Token / classification-token fusion weights | 0.5 / 0.5 |
| Conformal level, `alpha` | 0.05 |

The main TokenGauss-Former backbone is selected through `PROP_VAR = 'noTG'`, which disables the additional token regularizer during training. The regularized backbone, group-token removal, alternative scoring rules, and alternative thresholds are examined separately.

## Notebook workflow

| Stage | Contents |
| --- | --- |
| 0-3 | Environment, configuration, downloads, cleaning, deduplication, sampling, and exploratory analysis |
| 4-5 | Ordered data partitions, feature transformation, open-set protocols, and training-bank graph construction |
| 6-8 | Model definitions, resumable training, baseline scoring, and evaluation functions |
| 9-9b | TokenGauss-Former training, relative token-Gaussian scoring, and supplementary MixMaha-CP comparisons |
| 10-12b | Main comparisons, open-set behaviour, calibration analysis, token explanations, and feature-replacement analysis |
| 13-15 | Ablation studies, parameter sensitivity, and comparison of dataset-specific evaluations |
| 16-18 | Statistical comparisons, computational measurements, and CSV/LaTeX exports |
| 19 | Additional-partition experiments on CICIoT2023-B |

## Evaluation metrics

The notebook reports the following metrics as percentages.

| Metric | Evaluation target |
| --- | --- |
| `Closed-Acc` | Classification accuracy on known test samples |
| `Closed-MacroF1` | Macro-F1 on known test samples |
| `AUROC` | Separation of unknown and known test samples, with unknown samples positive |
| `AUPR-Out` | Average precision with unknown samples positive |
| `FPR95` | Fraction of known samples flagged at a score cutoff targeting 95% unknown detection |
| `Open-MacroF1` | Macro-F1 over known classes and the additional unknown class |
| `Open-Acc` | Accuracy over known classes and the unknown class |
| `Unknown-Recall` | Fraction of unknown test samples assigned to `UNKNOWN` |
| `Known-FRR` | Fraction of known test samples assigned to `UNKNOWN` |

Statistical analyses include one-sided Wilcoxon signed-rank comparisons with Holm adjustment, Cliff's delta, and Friedman-based ranking plots. The notebook also records parameter counts, storage estimates, inference timing, throughput, and GPU memory measurements when CUDA is available.

## Generated files

All generated files are written beneath `ROOT`.

| Location | Contents |
| --- | --- |
| `config.json` | Saved experiment configuration |
| `raw/` | Downloaded source CSV files |
| `interim/` | Cleaned data and feature metadata |
| `proc/` | Transformed feature arrays and partition metadata |
| `graph/` | Cached training-bank neighbour indices and distances |
| `ckpt/` | Resumable training states and best model weights |
| `runs/` | Per-run JSON summaries, compressed prediction arrays, scores, and analysis caches |
| `figs/` | Exported PNG figures |
| `tables/` | CSV summaries and LaTeX tables |

Representative table exports include `ALL_RUNS_long.csv`, `T_datasets.csv`, `T_hyperparameters.csv`, `T_conformal.csv`, `T_faithfulness.csv`, `T_wilcoxon_holm.csv`, `T_confirmation.csv`, and `T_confirmation_wilcoxon.csv`.

## Repeating experiments

- Run cells in their original order so that models, data bundles, and analysis variables are available to subsequent stages.
- Training checkpoints store model, optimizer, scheduler, epoch, and history information. Existing checkpoints and run summaries are reused on subsequent executions.
- When changing preprocessing, architecture, split definitions, or training settings, select a fresh `ROOT` directory to generate a separate set of caches and results. In Colab, update the `ROOT` assignment directly.
- Keep the configuration, dataset source identifiers, retained-family counts, software versions, and hardware information with any reported results.
- Obtain experiment values from the generated tables and run summaries associated with the chosen configuration.

## Dataset attribution and research use

The datasets are obtained from the sources listed above. Credit the original dataset creators and follow the terms associated with the datasets and their distribution repositories. The notebook downloads data during execution and stores local working copies beneath the configured output directory.

The implementation supports research on open-set network intrusion detection, anomaly scoring, calibration, and token-level explanation analysis.
