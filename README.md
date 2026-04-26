# FUSED — Federated Unlearning via Selective Sparse Adapters

FUSED is a **reversible federated machine-unlearning** framework. After standard FedAvg training, it surgically erases the influence of targeted clients, classes, or samples from a global model by attaching lightweight LoRA adapters only to the layers identified as most sensitive to the forget-target data — using the **Fisher Information Matrix (FIM)** to guide placement. The original frozen model is preserved and can be fully recovered.

![Architecture overview](img_1.png)

---

## Architecture

```
novelty_2/
├── main.py                  # Entry point — parses args, orchestrates all phases
├── algs/
│   ├── fl_base.py           # Base federated learning class (FedAvg, local train, test)
│   └── fused_unlearning.py  # FUSED class: train_normal, forget_client/class/sample,
│                            #   relearn, verify_restored_model
├── models/
│   ├── Model_base.py        # Lora / DynamicLora wrappers (PEFT), MyModel base
│   ├── LeNet_FashionMNIST.py
│   ├── CNN_Cifar10.py       # ResNet-18 wrapper
│   ├── CNN_Cifar100.py      # ResNet-18 wrapper (100-class head)
│   └── ViT_Cifar100.py
├── dataset/
│   ├── data_utils.py        # Dataset loading, Dirichlet / pathological partitioning,
│   │                        #   train/test/proxy splitting per client
│   └── generate_data.py     # data_init() / cross_data_init() called from main
├── fim_utils.py             # Diagonal FIM computation + layer sensitivity scoring
├── distillation_utils.py    # Feature Distribution Matching (MMD) dataset distillation
├── utils.py                 # Model init, attack helpers, MIA evaluation
├── alg_utils/
│   └── ada_hessian.py       # AdaHessian optimiser (optional)
├── results/                 # CSV outputs per paradigm (auto-created)
└── save_model/              # Saved .pth checkpoints (auto-created)
```

### Training pipeline (3 phases)

| Phase | What happens |
|---|---|
| **Phase 1** — FedAvg | Standard federated training across all clients (`train_normal`). Global model + per-epoch param-change CSV saved. |
| **Phase 1.5** — Distillation *(optional)* | Dataset distillation via Feature Distribution Matching (MMD). Each client's data compressed to `ipc` synthetic images per class and saved to `distilled_data/`. Enabled with `--distill_data`. |
| **Phase 2** — Unlearning | FIM computed on clean vs. forget-target data. Layers with sensitivity score above the `fim_percentile`-th percentile receive a LoRA adapter (`DynamicLora`). Only adapter weights are trained — base model stays frozen. |
| **Post-unlearning** | Membership Inference Attack (MIA) evaluation; optional relearning resistance test; restored Phase 1 model verification. |

### Unlearning paradigms

| `--forget_paradigm` | Target | Description |
|---|---|---|
| `client` | Whole clients | Erases all knowledge contributed by specified clients (default mode). |
| `class` | Class labels | Relabels the forget class to random other classes before unlearning. |
| `sample` | Backdoor samples | Removes a backdoor trigger implanted in early batches of forget-clients. |

---

## Environment Setup

Python 3.10+ and a CUDA-capable GPU are recommended.

```bash
# 1. Create a virtual environment (optional but recommended)
python -m venv .venv
source .venv/bin/activate        # Linux / macOS
.venv\Scripts\activate           # Windows

# 2. Install dependencies
pip install -r requirements.txt
```

> **Google Colab**: paste `!pip install -r requirements.txt` in a cell. The file uses `>=` bounds for packages Colab pre-installs, so nothing is downgraded.

> **Windows**: `bitsandbytes` may require the `bitsandbytes-windows` wheel. If the install fails, run:
> ```bash
> pip install bitsandbytes --prefer-binary
> ```

---

## Datasets

All datasets are downloaded automatically via `torchvision` on first run. No manual download is needed.

| `--data_name` | Dataset | Default model |
|---|---|---|
| `fashionmnist` | FashionMNIST (28×28, 1ch) | `LeNet_FashionMNIST` |
| `mnist` | MNIST (28×28, 1ch) | `LeNet_FashionMNIST` |
| `cifar10` | CIFAR-10 (32×32, 3ch) | `CNN_Cifar10` (ResNet-18) |
| `cifar100` | CIFAR-100 (32×32, 3ch) | `CNN_Cifar100` (ResNet-18) |

Downloaded files are cached under `./dataset/<data_name>/`.

Data is partitioned across clients using a **Dirichlet distribution** (`--partition dir`, controlled by `--alpha`). Lower alpha = more heterogeneous (non-IID). Each client's data is then split into train / test / proxy sets; the proxy set is used for the shadow-model MIA evaluation.

---

## Running the Code

All runs go through `main.py`. The `--paradigm fused` flag selects FUSED (the default).

### Basic client unlearning (FashionMNIST)

```bash
python main.py \
  --data_name fashionmnist \
  --forget_paradigm client \
  --paradigm fused \
  --global_epoch 100 \
  --local_epoch 5 \
  --alpha 1.0
```

### Client unlearning — CIFAR-10 (9 forget clients)

```bash
python main.py \
  --data_name cifar10 \
  --model CNN_Cifar10 \
  --forget_paradigm client \
  --paradigm fused \
  --global_epoch 100 \
  --local_epoch 5 \
  --alpha 1.0 \
  --forget_client_idx 3 9 13 19 23 29 33 39 43
```

### Client unlearning — CIFAR-100 (9 forget clients)

```bash
python main.py \
  --data_name cifar100 \
  --model CNN_Cifar100 \
  --forget_paradigm client \
  --paradigm fused \
  --global_epoch 100 \
  --local_epoch 5 \
  --alpha 1.0 \
  --forget_client_idx 3 9 13 19 23 29 33 39 43
```

### Surgical unlearning with FIM percentile tuning

```bash
python main.py \
  --data_name fashionmnist \
  --forget_paradigm client \
  --paradigm fused \
  --global_epoch 50 \
  --local_epoch 5 \
  --alpha 1.0 \
  --forget_client_idx 4 9 13 17 23 28 34 39 47 \
  --fim_percentile 70
```

Higher `--fim_percentile` means fewer (more selectively chosen) layers receive LoRA adapters.

### With dataset distillation (Phase 1.5)

Distillation compresses each client's data into `ipc` synthetic images before Phase 2, reducing unlearning time significantly.

```bash
python main.py \
  --data_name fashionmnist \
  --forget_paradigm client \
  --paradigm fused \
  --global_epoch 50 \
  --local_epoch 5 \
  --alpha 1.0 \
  --forget_client_idx 4 9 13 17 23 28 34 39 47 \
  --fim_percentile 70 \
  --distill_data \
  --ipc 5 \
  --dm_iterations 500
```

---

## Key Arguments Reference

| Argument | Default | Description |
|---|---|---|
| `--data_name` | `fashionmnist` | Dataset name |
| `--model` | `LeNet_FashionMNIST` | Model architecture |
| `--forget_paradigm` | `class` | Unlearning target: `client`, `class`, or `sample` |
| `--paradigm` | `fused` | Algorithm: `fused`, `federaser`, `retrain`, `exactfun`, `eraseclient`, `fl` |
| `--forget_client_idx` | `[0]` | Space-separated list of client indices to forget |
| `--forget_class_idx` | `[0]` | Space-separated list of class indices to forget |
| `--global_epoch` | `2` | Number of global FL rounds |
| `--local_epoch` | `1` | Local SGD epochs per client per round |
| `--num_user` | `50` | Total number of clients |
| `--alpha` | `1.0` | Dirichlet concentration (data heterogeneity) |
| `--fim_percentile` | `70` | Layer selection threshold for LoRA placement |
| `--fim_max_batches` | `10` | Batches used for FIM estimation |
| `--distill_data` | `False` | Enable Phase 1.5 dataset distillation |
| `--ipc` | `5` | Synthetic images per class (distillation) |
| `--dm_iterations` | `500` | Pixel optimisation steps (distillation) |
| `--MIT` | `True` | Run Membership Inference Attack evaluation |
| `--relearn` | `True` | Run relearning resistance test after unlearning |
| `--proxy_frac` | `0.2` | Fraction of each client's data reserved as proxy for MIA |

---

## Output Files

All CSVs are written to `./results/<forget_paradigm>/`.

| File pattern | Contents |
|---|---|
| `Acc_loss_fl_<paradigm>_data_<dataset>_distri_<alpha>.csv` | Per-epoch accuracy during Phase 1 training |
| `Acc_loss_fused_<paradigm>_data_<dataset>_...csv` | Per-epoch accuracy during Phase 2 unlearning |
| `relearn_data_<dataset>_...csv` | Accuracy trajectory during relearning test |
| `MIA_<paradigm>_...csv` | Membership inference attack accuracy |
| `restored_model_verification_...csv` | Accuracy of the restored Phase 1 model |

Model checkpoints are saved to `./save_model/`:
- `global_model_<dataset>.pth` — end of Phase 1
- `global_fusedmodel_<dataset>.pth` — end of Phase 2 (with adapters)
- `restored_phase1_model_<dataset>.pth` — Phase 1 base extracted via LoRA unload
