# WIDS: Wi-Fi Deauthentication and Disassociation Flood Detection

A prototype Wireless Intrusion Detection System (WIDS) that uses simple machine learning to detect deauthentication and disassociation flood attacks in 802.11 (Wi-Fi) traffic, and raises an alert when one occurs.

## What it does

1. Reads 802.11 frames from the AWID3 dataset, one capture part at a time (batching), so large files fit in memory.
2. Turns frames into 1-second windows with simple features: total frames, management frames, deauthentication frames, disassociation frames and their share of management traffic.
3. Trains scikit-learn classifiers (Decision Tree, Random Forest, Logistic Regression and XGBoost) to label each window as normal or attack.
4. Replays the test windows in 10-second batches, as a live detector would receive them. For each window flagged as an attack, it prints an alert and writes it to a log file with the target BSSID and the claimed sender addresses.

This is a **WIDS**. It alerts and logs but does not block anything (see "Why it does not block" below).

## Dataset

AWID3, from the University of the Aegean. The notebook reads a Kaggle copy of the dataset (`suumia/awid3-dataset`). For citations and licence terms, use the original AWID source: [add the AWID3 reference and link].

The dataset is not included in this repo. Only these folders are used:

| Folder | Attack | Parts | Parts with attack frames |
|---|---|---|---|
| `1.Deauth` | Deauthentication | 33 | 21 to 32 |
| `2.Disas` | Disassociation | 41 | 28 to 40 |

Attack frames are a small share of the data (about 2.4% for Deauth and 3.7% for Disas), so results are reported with precision, recall and F1 as well as accuracy.

## Method

- **Features per second:** `n_frames`, `n_mgmt`, `n_deauth`, `n_disas`, and the fraction of management frames that are deauth or disassociation.
- **Labels:** a window is an attack if it contains any frame labelled with the attack.
- **Split:** Normal and attack windows are split separately, with the earlier 70% of each class for training and the later 30% for testing. There is no random shuffling, because neighbouring seconds are almost identical and would leak.
- **MAC addresses are never features.** They are used only in the alert text.
- **Checks:** a volume-only leakage check, and a held-out parts test (train on some capture parts, test on others), both done for Deauth.

## Results

Test sets hold about 55 attack windows each, so one window changes recall by about 2 points.

| | Deauthentication | Disassociation |
|---|---|---|
| Test windows (normal / attack) | 128 / 55 | 127 / 55 |
| Decision Tree accuracy | 1.000 | 0.973 |
| Decision Tree recall (attack) | 1.000 | 0.909 (5 missed) |
| False alarms | 0 | 0 |
| Alerts in replay | 55 of 55 | 50 of 55 |

- **Deauth:** Decision Tree and Random Forest both score 1.00. On the held-out parts test, 46 of 47 attack windows were detected and none of 12 normal windows was flagged. The one miss had only 5 deauth frames.
- **Disas:** Logistic Regression and XGBoost score 1.00. Decision Tree and Random Forest each missed 5 of 55 attack windows.

**How to read these scores:** no normal window contained a deauth frame, and every attack window contained at least 45, so these floods are very easy to see in a per-second count. High scores show the attack is loud, not that the model is sophisticated. The leakage check also found that traffic volume alone partly separates the classes, because normal and attack windows come from different parts of the capture.

## Why it does not block

The sender addresses in the attack frames appear to be forged. The alerts show the access point's own address and the broadcast address (`ff:ff:ff:ff:ff:ff`), which cannot be a real sender. Blocking a claimed sender could disconnect legitimate devices, so each alert logs the target BSSID and the claimed senders only.

## How to run

The notebook was developed on Kaggle with the AWID3 dataset attached, and reads from `/kaggle/input/datasets/suumia/awid3-dataset/CSV`.

1. Create a Kaggle notebook and add the AWID3 dataset.
2. Upload or copy in `[notebook-name].ipynb`.
3. Run all cells from the top. Building the windows reads many files and takes a few minutes.

To run elsewhere, download only the `1.Deauth` and `2.Disas` folders and change the `root` path in the first code cell.

**Requirements:** Python 3, `pandas`, `numpy`, `scikit-learn`, `matplotlib`, and `xgboost` (optional: the notebook falls back to scikit-learn's `HistGradientBoostingClassifier` if it is missing).

## Limitations

- AWID3 is testbed data, not live traffic. Results do not show how the system would perform on another network.
- All parts come from one capture session, so the held-out test shows consistency within the testbed, not generalisation.
- The floods are loud, so near-perfect scores are expected and say little about quieter attacks.
- The replay uses the same test windows as the evaluation, so it demonstrates alerting and logging, not new accuracy. Log-file timestamps show when the replay ran, and the `t=` value in each alert is the capture's own time.
- (Re)association, evil twin and rogue AP attacks were not analysed.
- The leakage check and held-out test were run for Deauth only.

## Possible improvements

- Detect (re)association, evil twin and rogue AP attacks, which need different features (request counts, beacons per BSSID, SSID/BSSID mismatches).
- Build windows per BSSID and use a rolling count over several seconds, so weak or partial seconds are caught.
- Run the leakage check and held-out test on Disassociation.
- Test on a different capture or an authorised lab network.
- Try a sequence model (such as an LSTM) over consecutive windows, and tune the decision threshold.

## Ethics and scope

All analysis uses a public, pre-captured dataset. No traffic was captured and no attacks were generated as part of this project. Any live capture or attack generation must stay on your own network or an approved testbed.

## AI assistance statement

I used Claude (Anthropic's AI assistant) while building this project: to plan the pipeline, write and debug code, interpret outputs and draft the markdown text in the notebook and this README. I ran the code, checked the results against the data, and can explain how each part works. 

## Acknowledgements

- AWID3 dataset, University of the Aegean.
- Kaggle copy of the dataset uploaded by `suumia`.
