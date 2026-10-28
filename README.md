# Variance Detection: is cancer a cooperative defector that gives itself away by variance?

An open hypothesis, open code and three open preprints: **cancer cells, as defectors from the cellular cooperative,
show higher temporal variance in gene expression than healthy cells, enough to detect them from stability alone.**

> **Status, 28 October 2026: the hypothesis is untested on real data.** An adversarial internal review on 16 July 2026
> found the simulation results in version 1 of the papers circular: the synthetic generator *defines* defecting cells as
> the higher-variance population, and the classification threshold was fitted on the same ground-truth labels with no
> held-out data. The F1 = 1.000, AUC = 1.000 and ~1,800x separation below are therefore **not evidence** for the
> hypothesis; they show only that the code runs. Synchronized cooperative systems (coupled oscillators, bursting
> neurons) are counterexamples to a *universal* variance law. Version 2 of each paper states this correction. The
> useful, known part: trust = 1/variance is inverse-variance weighting (the Kalman gain, Beta reputation), a known
> estimator applied to cell behaviour, not a new law.

**What would actually test it** (pull requests welcome):
1. a held-out threshold (train/test split, or cross-validation across donors);
2. defectors defined independently of variance (tumour annotation, copy-number calls, clinical labels);
3. at least one real single-cell RNA-seq dataset with matched normal tissue (TCGA, Human Cell Atlas, GEO, Tabula Sapiens);
4. the claim scoped to the tissues and conditions tested.

## Papers (Zenodo, CC BY 4.0)

| Paper | DOI (concept: always the latest version) | For |
|---|---|---|
| Variance as a Universal Defection Signal: A Framework for Early Detection of Cooperative Failure in Biological Systems | [10.5281/zenodo.19005449](https://doi.org/10.5281/zenodo.19005449) (v2 with correction: [10.5281/zenodo.23731290](https://doi.org/10.5281/zenodo.23731290)) | researchers |
| Your Body Already Knows: The Easy Way to See Cancer | [10.5281/zenodo.19004940](https://doi.org/10.5281/zenodo.19004940) | everyone |
| Light Is All You Need: Six Shadows That Keep Eight Billion People in the Dark | [10.5281/zenodo.19005308](https://doi.org/10.5281/zenodo.19005308) | everyone, deeper |

Author: Sohail Mohammad Siddique Batliwalla Merchant ("Blaze"), AXXIS / DOT Protocol, sohail@blockend.com.

## Run it

Simulation (stages 1 and 2, no data needed):

```bash
pip install numpy scipy matplotlib scikit-learn
python variance_detection_proof.py
```

It reports a 2.4x separation on a cooperative network and ~1,800x on simulated gene expression, with F1 = 1.000. Read
the status note above: these numbers come from data built to separate.

Real data (stage 3, the test that matters):

```bash
pip install scanpy anndata pandas numpy scipy scikit-learn
python real_data_connector.py
```

The connector loads AnnData from TCGA, Human Cell Atlas, GEO or Tabula Sapiens. Report results, positive or negative,
with the issue template **Validation result** (`.github/ISSUE_TEMPLATE/validation_result.md`).

## Files

| File | What |
|---|---|
| `variance_detection_proof.py` | the simulations and figures (circular as described above) |
| `real_data_connector.py` | adapter for real scRNA-seq data |
| `arxiv/main.tex`, `arxiv/main.pdf`, `arxiv/figure1.png` | paper source, compiled PDF, figure |
| `USER_MANUAL.md` | install, run, outputs |
| `CITATION.cff` | how to cite this repository |

## Not medical advice

Nothing here diagnoses or treats anything. Checkpoint-inhibitor immunotherapy (for example pembrolizumab, nivolumab)
is a real, approved class of treatment; whether it suits anyone is a question for their oncologist.

## Licence

Code: Apache License 2.0 (`LICENSE`). Papers and figures: Creative Commons Attribution 4.0.

## Cite

```bibtex
@misc{batliwallamerchant2026variance,
  title     = {Variance as a Universal Defection Signal: A Framework for Early Detection of Cooperative Failure in Biological Systems},
  author    = {Batliwalla Merchant, Sohail Mohammad Siddique},
  year      = {2026},
  publisher = {Zenodo},
  version   = {v2},
  doi       = {10.5281/zenodo.23731290},
  note      = {Version 2 states that the version 1 simulation results are circular}
}
```

For Mumtaz. For Carl. For the researcher at 3 AM.
