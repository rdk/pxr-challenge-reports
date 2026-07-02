# PXR Challenge: Structure Model Report

## Task
Predict the 3D structure of each protein-ligand complex (the PXR ligand-binding domain with one of the challenge's 184 small molecules bound in it), one predicted complex per ligand. Scoring is by **LDDT-PLI**, which measures how closely the predicted protein-ligand contacts match the crystal structure.

## Method
The structure pipeline is built on **co-folding**: rather than docking a ligand into a fixed receptor, an open-source model (Boltz-2) predicts the full protein-ligand complex directly from the protein sequence and the ligand's SMILES. For each ligand it generates a large pool of candidate poses and submits the single one the model is most confident about, after confirming that pose can be scored by the official scorer.

Two ingredients shape generation: a deep multiple-sequence alignment gives the model evolutionary context for the protein, and a small set of experimental PXR structures is supplied as templates to anchor the binding pocket (one with its mobile activation helix removed so the model can sample it freely). One template does most of the work, and adding more beyond a small core did not help. Because the challenge references are blind, pose selection relies only on the model's own interface confidence (max-iPTM), with no post-hoc adjustment that could not be validated on held-out crystals.

In brief:
- **Inputs:** PXR ligand-binding-domain sequence and each ligand's SMILES.
- **Engine:** Boltz-2 co-folds the full complex in one shot (open-source; no model trained from scratch).
- **Context:** a deep MSA for the protein, plus a small set of experimental PXR templates to anchor the pocket.
- **Sampling:** ~150 candidate poses per ligand.
- **Selection:** the highest interface-pTM (max-iPTM) pose per ligand.
- **Quality check:** every submitted pose is verified scorable by the official scorer; any that is not is regenerated.
- **Result:** one predicted complex per ligand; **LDDT-PLI 0.5016** on the blind set.

## Findings
The limiting factor is pose **selection**, not **generation**. For almost every ligand a good pose is already present in the pool of 150: the best-achievable score across the pool is ~0.77, well above the ~0.50 that can be selected blind. The open problem is identifying that pose without the answer. For this target the model's confidence is close to uninformative for ranking poses, and no alternative selector beat taking the most confident pose. This appears to be a small-data limit: with ~30 reference PXR crystals for validation, the difference between a good and a bad pose-ranker falls below the detection threshold. Good poses exist in the pool; selecting them is the unsolved part.

A single concern ran through every decision. This challenge is built around **novel analogs** (close cousins of known actives that models have not seen), which makes it easy to score well for the wrong reason: a bigger model can look strong on *familiar* chemistry it has effectively memorized, and a pipeline tuned hard against the visible data can fit those examples without holding on new ones. Both edges vanish on genuinely novel chemistry, which is exactly the regime the blind set lives in. The pipeline was therefore kept **uncoupled and validation-gated**, aiming for a score that reflects honest generalization rather than a trick that won't survive the blind set.

## Acceptance gate
Ideas and new versions, borrowed or original, were adopted only if they beat the current best on **held-out validation** by a margin large and stable enough to trust, not on a single good run.

Honest note: this held for most decisions but not all. The validation signal was limited, so some comparisons were inconclusive; where the numbers couldn't settle a call, it came down to judgment, favoring the simpler, better-generalizing option.

## Idea lineage
- **The templated co-folding recipe** is adopted from **discoverybytes'** publicly shared method, reproduced on Boltz-2 and re-validated.
- **Boltz-2** is the open AlphaFold3-class model from the Boltz team; no structure model was trained from scratch.
- **Confidence-based pose selection** draws on the **AF2Rank / actifpTM / ipSAE** line of work on reusing a model's confidence to rank near-native poses.
- The **analog-generalization discipline** is consistent with the benchmark-leakage analysis in the Boltz team's **BoltzMol-1** report.
- The **scorability check** was added after an early pose passed the official format validator but was then rejected by the scorer.

## Tried but rejected
- **Smarter pose selectors:** physics scores (GNINA, Vina), geometric pocket-contact filters, internal confidence signals (ipSAE, actifpTM), per-atom confidence, cross-model agreement, trained rerankers. None beat max-confidence.
- **A second engine as a wholesale replacement:** Chai-1, OpenFold3, and ESMFold2 scored below Boltz-2.
- **A stronger/newer co-folder** (Protenix): finds better poses in-distribution, but the gain is memorization on familiar chemistry that does not transfer, and the poses are not selectable regardless.
- **Physics-based pose-validity filtering (PoseBusters):** misaligned with the contact-recovery metric (LDDT-PLI does not penalize the clashes it flags), so it discards good poses; it hurt the production model.
- **Conditioning the fold on the pocket:** the predicted pose was insensitive to pocket residues.
- **Pinning the receptor to a single crystal:** over-constrained the flexible pocket and lowered scores.
- **Dense template ensembles:** no gain beyond the small core.
- **More diffusion samples / sampling-parameter tuning:** the selected pose plateaus even as more (unselectable) good poses appear.
- **Transplanting a similar crystal ligand's coordinates onto the shared scaffold:** helped a few very-close analogs, too few to matter overall.

## Summary
An uncoupled, validation-gated Boltz-2 co-folding submission (**LDDT-PLI 0.5016**), building on discoverybytes' public templating recipe. The main finding is that **selection, not generation, is the binding constraint** for this target: a limited-reference-data problem. Good poses are already in the pool; selecting them is the open challenge.
