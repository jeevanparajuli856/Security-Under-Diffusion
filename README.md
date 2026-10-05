# Security Under Diffusion

**A benchmark showing how diffusion-deepfake detectors hold up against image generators they were never trained on, measured at the false-positive rates that identity verification (KYC) and security operations (SOC) teams actually run at.**

Most deepfake detectors are evaluated on the same generator they were trained on, at whatever threshold makes the numbers look good. Real fraud pipelines don't work that way. Attackers use the newest model available, and a fraud team can only afford to flag a tiny fraction of real customers. This project measures the gap between those two worlds.

📄 [Paper (PDF)](Security%20Under%20Diffusion_Paper.pdf) · ✍️ [Write-up on Medium](https://medium.com/@jeevanparajuli856/what-my-deepfake-security-benchmark-taught-me-3f60411a0cac)

---

## Key findings

| | |
|---|---|
| **Frequency beats reconstruction** | A small CNN on the Fourier spectrum (HFreq) reached AUROC 0.95–1.00 on the seen generator in every scenario. A pretrained DIRE checkpoint was near random on headshots (0.58) and inverted on scenes (0.08). |
| **New generators break detectors** | On headshots, HFreq caught **49%** of Stable Diffusion fakes at a 1% false-positive rate, but only **4–8%** of Gemini fakes it had never seen. |
| **Ranking ≠ deployability** | HFreq's ranking stayed strong on unseen generators (AUROC 0.79–1.00), but strong ranking didn't translate into catches at strict, pre-set thresholds. AUROC alone overstates how ready a detector is for production. |
| **JPEG doesn't matter (at q=90)** | Re-encoding every image as JPEG-90 changed AUROC by less than 0.01 for both detectors. |

### AUROC by scenario (PNG)

| Detector | Scenario | Seen: SD v2 | Unseen: Gemini 2.5 Flash Image | Unseen: Gemini 3 Pro Image |
|---|---|---|---|---|
| HFreq | Document | 0.999 | 0.785 | 0.797 |
| HFreq | Headshot | 0.946 | 0.896 | 0.899 |
| HFreq | Scene | 1.000 | 1.000 | 1.000 |
| DIRE | Document | 0.821 | 0.728 | 0.381 |
| DIRE | Headshot | 0.581 | 0.446 | 0.491 |
| DIRE | Scene | 0.078 | 0.290 | 0.163 |

Full tables, including TPR at 1%, 0.5% and 0.1% FPR and the JPEG-90 results, are in the [paper](Security%20Under%20Diffusion_Paper.pdf).

---

## What's in the benchmark

**Threat model.** An attacker submits an AI-generated ID photo, selfie, or scene image to an automated check. They have access to current commercial generators. The defender must keep false positives low, because every false alarm is a blocked real customer or a wasted analyst hour.

**Dataset: 4,499 images, 128×128, three scenarios**

| Source | Images | Used for |
|---|---|---|
| Real (bona fide) | 1,500 | train / val / seen test |
| Stable Diffusion v2 | 1,500 | train / val / seen test |
| Gemini 2.5 Flash Image ("Nano Banana") | 1,001 | unseen test only |
| Gemini 3 Pro Image ("Nano Banana Pro") | 499 | unseen test only |

- Scenarios: `doc` (privacy-masked ID documents), `headshot` (selfies), `scene` (natural scenes, no people)
- Every image is stored as a lossless PNG master plus a JPEG-90 copy
- Detectors are trained and calibrated **only** on real + SD v2. The Gemini images are held out completely.

**Detectors**

- **DIRE**: diffusion reconstruction error with the pretrained ImageNet-ADM checkpoint (ResNet-50), run zero-shot.
- **HFreq**: a compact ResNet-style CNN on the Hamming-windowed log-magnitude FFT spectrum, trained per scenario (Adam, early stopping on validation AUROC; under 15 minutes per scenario on one RTX 4090).

**Evaluation protocol**

1. Calibrate a decision threshold per detector and scenario on the **validation** split, targeting fixed FPRs (1%, 0.1%).
2. Freeze those thresholds and apply them to the seen and unseen test sets, the way a deployed system would.
3. Report AUROC plus TPR at each FPR target, separately for each generator.

---

## Repository layout

```
configs/         global.yaml (seed, FPR targets, device), dire.yaml, hfreq.yml
detectors/       DIRE and HFreq model code + checkpoints
preprocessing/   per-detector input pipelines (normalization, FFT)
runners/         run_all.py orchestrates every detector × scenario × variant
evaluation/      calibrate.py, scorer.py, metrics.py, plot_results.py
utils/           logging
```

Everything is driven by a CSV manifest, so new generators or scenarios can be added without code changes:

```
image_id,path,label,scenario,source,generator_family,split
```

- `label`: 0 = real, 1 = AI-generated
- `scenario`: `doc` | `headshot` | `scene`
- `generator_family`: `real` | `sd` | `nano25` | `nanopro`
- `split`: `train` | `val` | `test_seen` | `test_unseen`

---

## Quick start

```bash
python -m venv venv
source venv/bin/activate            # Windows: venv\Scripts\activate
pip install -r requirements.txt
```

Put the data here (relative paths are stored in the manifest):

```
data/
  images/{doc,headshot,scene}/          # PNG masters
  images_jpeg/{doc,headshot,scene}/     # JPEG-90 copies
  manifests/master_manifest.csv
```

Run the full benchmark (PNG):

```bash
python -m runners.run_all --dire_ckpt detectors/checkpoints/imagenet_adm.pth --plot
```

Re-run on JPEG-90 using the trained models and the existing calibration:

```bash
python -m runners.run_all --jpeg --dire_ckpt detectors/checkpoints/imagenet_adm.pth --plot --plot_variant jpeg
```

Results, metrics JSON, and figures are written to `outputs/`. Seeds are fixed in `configs/global.yaml`.

---

## Limitations

- 128×128 resolution simulates low-resource capture but removes high-frequency detail that full-resolution detectors rely on.
- DIRE was evaluated zero-shot. Fine-tuning it on in-domain data would likely close part of the gap.
- The unseen set covers two commercial generators. Results may differ for open-weight models such as FLUX or SDXL.
- The ID documents are privacy-masked or synthetic, not real government IDs.

## Recommendations for practitioners

- Don't ship on AUROC. Measure TPR at your actual false-positive budget, on generators you didn't train on.
- Prefer frequency-domain features over a pretrained reconstruction score when you can't fine-tune.
- Treat the detector as one signal: add provenance (C2PA, SynthID watermarks), metadata checks, and human review for borderline scores.

---

## Citation

```
Jeevan Parajuli. "Security Under Diffusion: Evaluating Pretrained DIRE and
Frequency-Based Deepfake Detectors for KYC and SOC Operations."
University of Louisiana Monroe, 2026.
https://github.com/jeevanparajuli856/Security-Under-Diffusion
```

**Contact:** Jeevan Parajuli · [LinkedIn](https://www.linkedin.com/in/jeevanparajuli856) · parajulij@warhawks.ulm.edu
