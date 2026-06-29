# AUTH_IDS

**Multi-model intrusion detection for IoMT authentication attacks — Ensemble + G-KDE + LSTM fused via OR-logic.**

This is the Tier 2 detection layer from a two-tier authentication security architecture proposed for IoMT-based Remote Patient Monitoring (RPM) systems. Tier 1 (not in this repo) covers device-to-gateway authentication using physical-layer behavior and federated learning. Tier 2, implemented here, sits at the gateway-to-backend link and detects attacks in network traffic using machine learning.

## Table of Contents

- [Overview](#overview)
- [Project Structure](#project-structure)
- [Getting Started](#getting-started)
- [Methodology](#methodology)
- [Results](#results)
- [Limitations](#limitations)
- [Roadmap](#roadmap)
- [Citation](#citation)
- [Acknowledgments](#acknowledgments)

## Overview

Three models, each with a different strength, run in parallel and are combined with OR-logic so that a flag from any one of them is enough to escalate a session:

- **Ensemble (RF + AdaBoost + Bagging)** — hard-voting classifier for known multi-protocol attack types
- **G-KDE** — density-based anomaly detector, trained on benign traffic only, for zero-day coverage
- **LSTM** — sequence model trained on MQTT traffic to catch temporal attack patterns invisible at the single-flow level

The design priority is minimizing false negatives over false positives — in a clinical RPM context, a missed attack is a patient-safety issue, while a false positive only costs a re-authentication check.

> Tier 1 (federated device authentication) and the PQC/QKD key-exchange components from the paper are **not implemented** in this repo — they're architectural proposals evaluated separately, not in code.

## Project Structure

```
AUTH_IDS/
├── notebook.ipynb    # Data loading, preprocessing, model training, evaluation, interpretability
└── README.md
```

No `dataset/` folder is committed to this repo — see [Dataset Setup](#dataset-setup) below.

## Getting Started

### Prerequisites

```bash
pip install numpy pandas matplotlib seaborn scikit-learn tensorflow shap jupyter
```

No GPU required — all three models train and run on CPU.

### Dataset Setup

This project uses **[CICIoMT2024](https://www.unb.ca/cic/datasets/iomt-dataset-2024.html)** (Canadian Institute for Cybersecurity), a real multi-protocol IoMT capture covering Wi-Fi, BLE, and MQTT traffic.

The notebook works from a 9-file subset, relabeled into 5 classes:

| Source files | Label |
|---|---|
| Benign | `benign` |
| ARP_Spoofing | `spoofing` |
| MQTT-DDoS / MQTT-DoS Connect Flood | `mqtt_auth_attack` |
| Recon-Port_Scan, Recon-OS_Scan, Recon-VulScan | `recon` |
| TCP_IP-DDoS-SYN, TCP_IP-DoS-SYN | `syn_attack` |

1. Download the relevant `.pcap.csv` files from the CIC dataset page.
2. Create a `dataset/` folder locally (already in `.gitignore` — it won't get committed).
3. Drop the files in there before running the notebook.

### Usage

```bash
jupyter notebook notebook.ipynb
```

Run top to bottom: data loading/merging → preprocessing → ensemble → G-KDE → LSTM → fusion → evaluation/SHAP.

## Methodology

### 1. Data Preprocessing

- Merge individual attack/benign files into single train and test DataFrames
- Label each record by source file (`spoofing`, `mqtt_auth_attack`, `recon`, `syn_attack`, `benign`), plus a derived binary `benign`/`attack` target
- Clean numeric features (coerce types, zero out NaN/inf)
- Standardize features with `StandardScaler`
- Encode labels with `LabelEncoder`

### 2. Detection Models

**Ensemble Classifier**
- Random Forest + AdaBoost + Bagging (Decision Tree base estimators)
- Trained on the full multi-class feature set
- Combined via hard/majority voting
- Strongest on known, structurally distinctive attacks (floods, scans)

**G-KDE Anomaly Detector**
- Fit exclusively on benign traffic — no attack labels used
- Feature set narrowed to the top features by Random Forest importance
- Anomaly threshold set at the 5th percentile of benign validation scores
- Output: 0 = benign, 1 = anomaly
- Provides zero-day coverage the ensemble can't offer

**LSTM (Temporal Model)**
- Trained only on the MQTT benign/attack subset
- Input reshaped into sliding-window sequences
- Stacked LSTM layers + Dropout → Dense output with sigmoid activation
- Targets attacks (e.g. slow auth flooding) that look normal in any single record but not across a sequence

### 3. Fusion Strategy

OR-logic across all three binary outputs: if **any** model flags "attack," the fused result is "attack." This trades a higher false-positive rate for a lower false-negative rate — the explicit design choice for a clinical deployment context.

### 4. Evaluation

Each model, and the fused system, is scored with:
- Accuracy, Precision, Recall, F1
- False Positive Rate (FPR), False Negative Rate (FNR)
- AUC / ROC curves
- Confusion matrices

### 5. Interpretability

SHAP values are computed on the Random Forest component to identify which traffic features drive each attack-type prediction — useful for audit trails in a regulated healthcare setting.

## Results

| Component | Coverage | Binary FPR | Binary FNR |
|---|---|---|---|
| Ensemble (RF + AdaBoost + Bagging) | Known multi-protocol attacks | 0.0123 | 0.0027 |
| G-KDE | Any deviation from normal (zero-day) | 0.0469 | 0.0125 |
| LSTM (MQTT subset) | Temporal auth-flood patterns | 0.0000 | 0.0001 |
| **Combined OR-logic** | All of the above | **0.0557** | **0.0022** |

Ensemble binary AUC: **0.9998**. The combined FNR is lower than any individual component — the three models have largely non-overlapping failure modes, so fusing them closes gaps no single model covers alone.

**Notable weak point:** ARP spoofing (ensemble FNR ≈ 19.7% on that class). Spoofing injects a small number of forged packets rather than flooding traffic, so it doesn't trigger the volume/flag anomalies flow-level features are good at catching. G-KDE catches a portion of these misses through density deviation rather than class-boundary separation, and the LSTM exists specifically to cover temporal gaps like this on MQTT traffic.

## Limitations

- Evaluated on a 9-file CICIoMT2024 subset, not the full attack taxonomy
- Spoofing is the smallest, most imbalanced class in the subset
- LSTM coverage is MQTT v3.1.1 only — no BLE, Wi-Fi, recon, SYN, or MQTT v5.0 sequences
- All training/inference run on CPU infrastructure, not real IoMT gateway hardware
- No adversarial robustness testing — evaluated attacks make no attempt to evade detection

## Roadmap

- [ ] Add ARP-specific header features (sender/target MAC + IP) as a pre-filter to close the spoofing gap
- [ ] Extend LSTM temporal coverage to BLE and Wi-Fi sequences
- [ ] Benchmark inference on real gateway-class hardware (e.g. Raspberry Pi, ARM Cortex)
- [ ] Adversarial robustness testing against an informed attacker

## Citation

```
N. Ahmad, M. Ansari, A. AlYammahi, J. AlMekha, M. Fahmi,
"Two-Tier Authentication Security Architecture for IoMT-Based Remote Patient Monitoring (RPM)
Systems Using Machine Learning and Quantum Key Distribution,"
Rochester Institute of Technology, Dubai, 2026.
```

## Acknowledgments

- Hard-voting ensemble structure: Alharbi et al. (2025)
- G-KDE anomaly detection approach: Zachos et al. (2025)
- Temporal MQTT modeling: Lakshminarayana and Thilagam (2026)
