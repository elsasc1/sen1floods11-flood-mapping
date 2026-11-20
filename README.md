# Flood mapping with Sentinel-1 / Sentinel-2 fusion (Sen1Floods11)

Course project at IMT Atlantique (done in a team of five): does fusing radar (Sentinel-1) and optical
(Sentinel-2) data, plus terrain and rainfall, improve pixel-level flood mapping compared
with a single sensor or a simple threshold?

## Data

[Sen1Floods11](https://github.com/cloudtostreet/Sen1Floods11) (Bonafilia et al., CVPR Workshops 2020,
CC BY-SA 4.0), in its 8-channel Kaggle version: **446 scenes** of 512×512 pixels,
split into 312 train / 67 validation / 67 test scenes. Labels: 1 flood, 0 no flood, −1 invalid.
About 11 % of valid pixels are flooded, so accuracy is uninformative; we report **IoU, F1,
precision and recall on the flood class** (recall matters most: a missed flood is worse
than a false alarm).

**Channel check.** The channel order in the Kaggle documentation did not match the data. By
comparing class histograms and a vegetation index we established the real order:
channels 0–2 visible bands, 3 near-infrared, 4 VV, 5 VH (dB), 6 DEM, 7 7-day rainfall.
There is no SWIR band in this version.

## Method

1. Sample pixels per scene (1,500 for training, 4,000 for validation and test), keep valid labels only.
2. Features: the 8 channels plus VV − VH, a water index `(ch0 − NIR) / (ch0 + NIR)` and a visible-band ratio.
3. Baselines: a single threshold on VV, and on the water index, tuned on the validation set.
4. Random Forest on four feature sets (radar only, optical only, both, everything), `class_weight="balanced"`.
5. RBF-SVM on a 15,000-pixel subsample (standardised features).
6. Unsupervised, radar only (the realistic cloudy-flood case): k-means and Gaussian mixtures, including a
   per-scene GMM with and without a 5×5 median filter against speckle.

## Results (test, sampled pixels)

| Model | Features | IoU | F1 | Precision | Recall |
|---|---|---|---|---|---|
| Threshold VV < −16.5 dB | 1 channel | 0.388 | 0.559 | 0.704 | 0.464 |
| Threshold water index > −0.02 | 1 channel | 0.574 | 0.730 | 0.819 | 0.658 |
| Random Forest | radar (S1) | 0.459 | 0.629 | 0.544 | 0.746 |
| Random Forest | optical (S2) | 0.693 | 0.818 | 0.771 | 0.872 |
| Random Forest | S1 + S2 | 0.694 | 0.820 | 0.772 | 0.874 |
| **Random Forest** | **S1 + S2 + DEM + rain** | **0.737** | **0.848** | **0.810** | **0.890** |
| SVM (RBF) | S1 + S2 | 0.660 | 0.795 | 0.700 | 0.920 |
| SVM (RBF) | S1 + S2 + DEM + rain | 0.681 | 0.810 | 0.719 | 0.929 |

Take-aways: the best model uses every source; adding radar to optical alone brings almost
nothing here (0.693 → 0.694), whereas terrain and rainfall add about 0.04 IoU. Radar alone
is the weakest supervised option, yet it is the only one available under cloud cover.

## Limitations

- Metrics are computed on sampled pixels, not on full scenes.
- The dataset has no cloud mask, so "optical fails under clouds" is a plausible explanation, not a proof.
- Single run with a fixed seed; no confidence intervals.

## Reproduce

Open the notebook in Google Colab, run the setup cell (it downloads the dataset with `kagglehub`;
provide your own Kaggle credentials when prompted, never commit them), then run all cells.
