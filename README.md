# Robust Keystroke Classification under Adversarial Conditions

**A multi-model evaluation framework for acoustic side-channel keystroke inference, with adversarial robustness testing and a blockchain-backed audit trail.**

[![Python](https://img.shields.io/badge/Python-3.10%2B-blue.svg)](https://www.python.org/)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![Status](https://img.shields.io/badge/status-research%20prototype-orange.svg)]()
[![Ethereum](https://img.shields.io/badge/audit-Sepolia%20testnet-627EEA.svg)](https://sepolia.etherscan.io/)

> MSc Cyber Security group project — *Research Methods and Group Project in Security and Resilience (CSC8208)*, Newcastle University.

---

## Overview

Acoustic side-channel attacks recover typed text from the unintended sounds a keyboard makes. While prior work shows that machine-learning models can classify keystroke audio in controlled settings, most studies optimise for **clean accuracy** and largely ignore how these models behave under **adversarial perturbation**.

This project closes that gap. It builds an end-to-end pipeline that converts keystroke input to audio, extracts spectral features, applies controlled adversarial perturbations, runs a panel of classifiers in parallel, and then **selects the most robust model** — recording the decision immutably on a blockchain ledger for a verifiable audit trail.

The headline finding: the model with the *highest clean accuracy is not the most secure one*. A sequential **LSTM** trades a little clean accuracy for dramatically better adversarial robustness, making it the strongest practical choice.

## Key contributions

- **Integrated evaluation framework** — data generation, feature extraction, adversarial attack, inference, and audit in a single reproducible pipeline.
- **Fair multi-architecture comparison** — classical ML (SVM, Random Forest), recurrent (LSTM), convolutional (LogMel CNN), and hybrid convolution-attention (CoAtNet / CoAtNet + Self-Attention) models, all trained and tested on identical data.
- **Structure-aware adversarial testing** — gradient-based attacks (FGSM, BIM, PGD) for differentiable models; frequency-domain (DFT) perturbations for non-differentiable models.
- **Robustness over raw accuracy** — a combined robustness score, not clean accuracy alone, drives model selection.
- **Tamper-evident auditing** — the winning model and its metrics are hashed and committed to an Ethereum (Sepolia) ledger for verifiability.

## System architecture

```
 Sensor Node  ─►  Text-to-Audio  ─►  Adversarial  ─►   Parallel ML        ─►  Audit & Comparison
 (keystrokes)     Converter (TTA)     Proxy             Inference Engine        Layer (blockchain)
                                                        ├─ SVM / RandomForest
                                                        ├─ LSTM
                                                        ├─ LogMel CNN
                                                        └─ CoAtNet (+ Self-Attn)
```

A full diagram is in [`docs/images/architecture.png`](docs/images/architecture.png).

## Results

Models ranked by **robustness score** (combining clean accuracy, mean adversarial accuracy, and mean attack success rate):

| Rank | Model                      | Clean Acc | Mean Adv. Acc | Mean ASR | Robustness |
|:----:|----------------------------|:---------:|:-------------:|:--------:|:----------:|
| 🥇 1 | **LSTM**                   | 91.81%    | **91.38%**    | **0.48%**| **91.33%** |
| 2    | CoAtNet                    | 97.81%    | 89.54%        | 8.46%    | 89.35%     |
| 3    | Skypetype                  | 95.75%    | 81.20%        | 15.19%   | 80.56%     |
| 4    | CoAtNet + Self-Attention   | 96.00%    | 71.17%        | 25.86%   | 70.14%     |
| 5    | SVM                        | 96.06%    | 58.50%        | 39.10%   | 56.96%     |
| 6    | Random Forest              | 90.62%    | 57.30%        | 36.78%   | 53.84%     |
| 7    | LogMel CNN                 | 67.38%    | 48.17%        | 28.51%   | 38.87%     |

**Takeaways**

- **PGD** was consistently the most effective attack; FGSM the weakest; BIM in between — iterative optimisation matters.
- **Recurrent (LSTM)** models degrade gracefully under perturbation; **convolutional** models that lean on spatial frequency patterns degrade sharply.
- **Classical models** appear "robust" to gradient attacks only because they are non-differentiable — an incompatibility, not genuine resilience. They fall over under DFT-based attacks.

## Threat model & attacks

The adversary can perturb the input representation to flip a prediction, ranging from black-box (input-only) to white-box (gradient access).

| Attack | Type | Applies to |
|--------|------|-----------|
| FGSM   | Single-step gradient | Differentiable models |
| BIM    | Iterative gradient   | Differentiable models |
| PGD    | Iterative, projected gradient | Differentiable models |
| DFT    | Frequency-domain     | All models (incl. SVM, Random Forest) |

## Repository structure

```
acoustic-keystroke-adversarial/
├── README.md
├── LICENSE
├── requirements.txt
├── .gitignore
├── .env.example                 # template for RPC URL / keys (never commit real keys)
├── report/
│   └── Report.pdf
├── docs/
│   └── images/                  # architecture diagram + result figures
├── src/
│   ├── tta/
│   │   └── tta.py               # Text-to-Audio converter
│   ├── inference/
│   │   ├── models/              # model definitions / training scripts
│   │   └── features/            # MFCC + log-Mel extraction
│   └── audit/
│       └── best_model_selection_and_blockchain.ipynb
├── models/                      # gitignored — trained weights (large)
└── results/                     # metrics, plots, blockchain receipts
```

## Installation

```bash
git clone https://github.com/Base-Karthik-S/Robust-Keystroke-Classification-under-Adversarial-Conditions-with-Secure-Audit-Framework/Adversarial-Acoustic-Keystroke.git
cd acoustic-keystroke-adversarial

python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

## Usage

```bash
# 1. Generate audio from keystroke input
python src/tta/tta.py

# 2. Extract features, train and evaluate the model panel
#    (run the inference scripts / notebooks in src/inference)

# 3. Select the most robust model and write the audit record
#    open src/audit/best_model_selection_and_blockchain.ipynb
```

For the blockchain step, copy `.env.example` to `.env` and add your Sepolia RPC endpoint and a **testnet-only** wallet key. Never commit `.env`.

## Tech stack

Python · NumPy · librosa · scikit-learn · PyTorch / TensorFlow · Matplotlib · web3.py · Ethereum (Sepolia testnet)

## Limitations & future work

- Models are evaluated on **single keystrokes**, not full typing sequences — this enables clean per-key analysis but does not capture inter-keystroke dynamics.
- Audio is **synthetically generated** for consistency rather than recorded in the wild.
- Future directions: adversarial training and certified robustness, multimodal fusion, and real-time deployment feasibility.

## Report

The full write-up — methodology, threat model, evaluation metrics, and discussion — is in [`report/Report.pdf`](report/Report.pdf).

## Authors

Abdul Hamdhan Ahamed · Karthik S · Karuppasamy Karuppasamy · Manav Sharma · Miran Mansuri · Nadeem Mohamed Abdul. (Newcastle University - MSc Cyber Security)

## License

Released under the [MIT License](LICENSE).

## Acknowledgements

Builds on foundational acoustic side-channel work (Asonov & Agrawal; Zhuang et al.) and adversarial ML (Goodfellow et al.; Madry et al.). Full citations are in the report's references.
