# NeuroFuzzy: A Soft Computing Pipeline for Brain Tumor MRI Diagnosis

A five-stage soft computing pipeline for classifying brain MRI scans as healthy or tumor. It combines Fuzzy C-Means segmentation, GLCM texture features, Binary PSO feature selection, a neuro-fuzzy (ANFIS-style) classifier, a Mamdani fuzzy risk score, and a Kohonen self-organising map. The project report is `NeuroFuzzy_Project_Report.pdf`.

> **Research prototype.** This is an academic project. It is not validated for clinical use. See *Limitations* before relying on any result.

## Results

From the latest run of `Code.ipynb`, after the validation fix. The test set has 200 images (50 healthy, 150 tumor).

| Measure | Value |
|---|---|
| Test accuracy (neuro-fuzzy classifier, 8 selected features) | 88.50% |
| Healthy: precision / recall / F1 | 0.79 / 0.74 / 0.76 |
| Tumor: precision / recall / F1 | 0.92 / 0.93 / 0.92 |
| Accuracy under FGSM attack (epsilon 0.1) | 81.50% |
| Accuracy drop under attack | 7.00 points (notebook verdict: SECURE, using a 15-point cutoff) |
| Feature reduction (BPSO) | 19 to 8 features |
| Example Mamdani risk score | 67.88% (one true-positive test case) |

The earlier run, before the validation split, reported 88.00% and a 17.50-point drop (VULNERABLE). That run used the test set to choose features, so its numbers are not reliable.

## Pipeline

1. **Segmentation.** Each MRI is resized to 128x128 and clustered with Fuzzy C-Means (c=2, m=2). The brighter cluster is taken as the tumor region and thresholded at membership 0.5.
2. **Feature extraction.** GLCM texture (contrast, correlation, energy, homogeneity at 0, 45, 90 and 135 degrees), plus mean intensity, standard deviation and area of the segmented region. This gives 19 features per image.
3. **Feature selection.** Binary PSO (mealpy `OriginalPSO`, 30 epochs, 20 particles) picks a feature subset. Fitness is 3-NN accuracy minus a small penalty for the number of features, measured on a validation split taken from the training data. The test set is not used.
4. **Classification and risk.** A PyTorch neuro-fuzzy network with 10 Gaussian rules is trained with Adam and binary cross-entropy. Its output and the tumor area feed a Mamdani fuzzy system that gives a 0 to 100% risk score.
5. **Unsupervised check.** A 15x15 Kohonen SOM is trained on training features, and its map is overlaid with the test labels.

An adversarial test (FGSM, epsilon 0.1) is also run on the classifier.

## Dataset

The notebook expects this layout next to `Code.ipynb`:

```
dataset/
  Training/  glioma/  meningioma/  notumor/  pituitary/
  Testing/   glioma/  meningioma/  notumor/  pituitary/
```

The folder has 1,400 training and 400 test JPEGs per class (7,200 files, about 175 MB). The dataset is not in this repository. Download it from its original source, then place it in `dataset/`. The notebook uses 150 training and 50 test images per class.

## Setup

Python 3.9 or newer is recommended.

```bash
python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate
pip install -r requirements.txt
jupyter notebook Code.ipynb
```

Run the cells top to bottom.

## Files

| File | Purpose |
|---|---|
| `Code.ipynb` | Full pipeline: segmentation, features, BPSO, neuro-fuzzy classifier, Mamdani, SOM, FGSM |
| `requirements.txt` | Python packages |
| `NeuroFuzzy_Project_Report.pdf` | Project report |
| `23BCE8392_SoftComputing_Project_Report.pdf` / `.docx` | Original project report (before the fixes) |
| `23BCE8392_SoftComputing_Project_Ppt.pdf` / `23BCE8392_Project_Ppt.pptx` | Presentation |
| `(Original Paper)...Adaptive_Neuro-Fuzzy...pdf` | Base paper the project builds on |
| `.gitignore` | Keeps the dataset and checkpoints out of Git |

## Limitations

- **Single split and single run.** Results come from one train/test split and one run. There is no cross-validation or external test set.
- **Small subset.** Only 150 training and 50 test images per class are used.
- **Heuristic tumor segmentation.** The tumor is taken to be the brightest FCM cluster, for every image including healthy ones. The "area" feature is therefore not a true tumor measure for healthy scans.
- **Binary task only.** Glioma, meningioma and pituitary are merged into one "tumor" class.
- **Risk score is coarse.** In the example case, the Mamdani risk is 67.88% for both network probabilities 0.958 and 0.745, and 50% for both 0.2 and 0.5. The rule base does not respond to changes in the probability over these ranges.
- **Adversarial verdict depends on the cutoff.** The notebook calls the model SECURE because the drop is under 15 points. The accuracy still falls by 7 points under a small attack.
- **Not for clinical use.** No clinician review or regulatory validation has been done.
