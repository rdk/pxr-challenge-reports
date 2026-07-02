# PXR Challenge: Activity Model Report

## Task
Predict PXR (pregnane X receptor) activation potency, **pEC50**, for the challenge's 513-compound set, an expansion of known actives into close structural analogs. Predictions are scored against the measured potencies (mean absolute error / relative absolute error).

## Method
Each compound's pEC50 is predicted by a blend of two complementary model families: a **3D-structure-pretrained molecular encoder** that reads each molecule as a 3D shape, and an **ensemble of tabular models** built on molecular descriptors and fingerprints. The two are combined with a fixed weighting, with the 3D encoder as the load-bearing component.

The 3D encoder (UniMol-2) is fine-tuned on the assay data; the tabular ensemble combines gradient-boosted trees, tabular foundation models (TabICL, TabM), and chemical language-model embeddings into a single meta-prediction. Auxiliary assay data released with the challenge (a counter-screen and single-concentration measurements) are folded in as extra features, and the revealed Phase-1 labels are added to training.

In brief:
- **Inputs:** compound SMILES (models trained on ~4,100 dose-response compounds; predictions made for the 513-compound test set).
- **3D encoder:** UniMol-2 (84M-parameter variant), fine-tuned on the dose-response data and averaged over several random seeds (the load-bearing signal).
- **Tabular ensemble:** LightGBM gradient-boosted trees over Mordred descriptors, Morgan-count fingerprints, and structure-derived features, the tabular foundation models TabICL and TabM, and ChemBERTa / CheMeleon chemical-language-model embeddings, combined by a stacked meta-model.
- **Extra data:** the challenge's counter-screen (~2.6k compounds) and single-concentration (~10k compounds) measurements as surrogate features, plus the 253 revealed Phase-1 labels folded into training.
- **Blend:** 0.6 x tabular ensemble + 0.4 x 3D encoder.
- **Result:** one pEC50 per compound; leak-free (out-of-fold) held-out MAE ~0.43 (primary challenge metric: relative absolute error, RAE).

## Findings
The load-bearing signal is the **3D-pretrained encoder**. It generalizes from the training compounds to the novel analogs with almost no drop-off, whereas classical descriptor/fingerprint models fit the training set well but degrade on the analogs, which is the exact failure this challenge is designed to expose. Blending still helps a little, but the encoder is what carries generalization.

The residual error is concentrated in **activity cliffs** (pairs of near-identical molecules with very different potency). On smooth series the model is far more accurate, though its error still sits above the label-noise floor; the headline error is dominated by these cliffs, where a small structural change sharply lowers activity and the model, which saturates over a compressed output range, undershoots the size of the drop.

A single concern ran through every decision. This challenge is built around **novel analogs** (close cousins of known actives that models have not seen), which makes it easy to score well for the wrong reason: a model can look strong on *familiar* chemistry it has effectively memorized, and a pipeline tuned hard against the visible data can fit those examples without holding on new ones. Both edges vanish on genuinely novel chemistry, which is exactly the regime the blind set lives in. The pipeline was therefore kept **uncoupled and validation-gated**, aiming for a score that reflects honest generalization rather than a trick that won't survive the blind set.

## Acceptance gate
Ideas and new versions, borrowed or original, were adopted only if they beat the current best on **held-out validation** by a margin large and stable enough to trust, not on a single good run.

Honest note: this held for most decisions but not all. The validation signal was limited, so some comparisons were inconclusive; where the numbers couldn't settle a call, it came down to judgment, favoring the simpler, better-generalizing option.

## Idea lineage
- **3D-encoder QSAR** uses **UniMol / UniMol-2**, open pretrained molecular encoders (DP Technology); no encoder was trained from scratch.
- The **tabular ensemble** uses open gradient-boosting plus the tabular foundation models **TabICL** and **TabM**, and chemical language-model embeddings (**ChemBERTa**, **CheMeleon**).
- The **auxiliary assay features** (counter-screen, single-concentration) and the **revealed Phase-1 labels** come from the challenge's own released data; folding them in was a direction several participants converged on.
- The **analog-generalization discipline** matches the wider concern (echoed in the Boltz team's BoltzMol-1 report) that analog leakage inflates benchmark numbers.

## Tried but rejected
- **A newer tabular foundation model** (Google's TabFM): matched the existing tabular models in-distribution but was worst-in-class on the analog test; the descriptor-tabular family overfits regardless of the learner.
- **Descriptor/fingerprint models as the primary signal:** overfit training, generalize worse than the 3D encoder.
- **Conformer-ensemble re-fine-tuning of the encoder:** no gain over the existing version.
- **Calibrating predictions to the revealed labels:** overfit the small held-out set.
- **Crude high-throughput-chemistry labels as extra training data:** added noise and hurt the strong encoder.
- **Heavy blend-weight optimization:** overfit; a simple fixed weighting held up better.
- **Activity-cliff-contrastive learning:** the direction of a cliff is not learnable from the available features.
- **Additional docking poses and structure-derived energetics** (binding-pose energies, strain): no reliable signal for potency beyond the structure features already used in the tabular ensemble.

## Summary
A 3D-encoder-anchored blend, leak-free-validated (held-out MAE ~0.43). The main finding: 3D pretrained encoders generalize to novel analogs where descriptor models overfit, and the remaining error is concentrated in deliberate activity cliffs that are near-unlearnable from structure alone.
