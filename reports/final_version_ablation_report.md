# IMC2024 Final Version Ablation Report

Date: 2026-06-04

This report merges the experiment notes under `reports/` and the latest Kaggle scores reported during development.  The goal is to document what each version tested, which changes helped, which changes hurt, and why the final pipeline converged to the current best version.

## Executive Summary

The best validated submission is:

| Rank | Version | Score | Main idea |
| ---: | --- | ---: | --- |
| 1 | `ver141` | **0.292267** | `ver71` transparent ordering + RANSAC threshold `1.08` |
| 2 | `ver169` | `0.290150` | RANSAC threshold `1.075` |
| 3 | `ver144` | `0.289834` | `ver102` + low-weight DeDoDe dual-softmax transparent score, resized to `1600` |
| 4 | `ver136` | `0.289478` | `ver102` + transparent ALIKED resize `2400` + rotation search |
| 5 | `ver140` | `0.288992` | RANSAC threshold `1.03` |
| 6 | `ver154` | `0.288881` | RANSAC threshold `1.025` |
| 7 | `ver146` | `0.288362` | `ver102` + DeDoDe `3%` dual-softmax transparent score |
| 8 | `ver102` | `0.288293` | `ver71` + RANSAC threshold `1.00` |

The main conclusions are:

- The largest gain came from **transparent-scene ordering**, not from replacing the main SfM model.  `ver28 = 0.255489` became `ver71 = 0.282382` after introducing transparent ring ordering with a LoMa/ALIKED rank ensemble.
- The most reliable second gain came from **MAGSAC/RANSAC threshold tuning**.  The best value found was around `1.08`, giving `ver141 = 0.292267`.
- **LoMa-B remains the best primary non-transparent pipeline**.  Most attempts to replace it with GIM, DeDoDe, LoMa-L, MASt3R, GLOMAP, VGGSfM, or heavy PnP rescue reduced score.
- **DeDoDe did not work as a primary matcher or as a large-weight transparent signal**, but a very small transparent-ordering contribution sometimes produced competitive scores.  The best DeDoDe-related result was `ver144 = 0.289834`, still below `ver141`.
- More aggressive versions after `ver141` did not beat it.  The final recommendation is to keep `ver141` as the main submission and `ver169`, `ver144`, `ver136`, and `ver102` as backup references.

## Project Context

The competition target is the Kaggle **Image Matching Challenge 2024 - Hexathlon** notebook rerun setting.  The notebook receives hidden test scenes and must write:

```text
/kaggle/working/submission.csv
```

Each row contains one image pose:

```text
image_path,dataset,scene,rotation_matrix,translation_vector
```

The challenge score is sensitive to camera-pose quality, not only to the number of registered images.  This mattered throughout the project: several versions registered many images but still scored poorly because the pose graph was geometrically inconsistent or the inserted fallback poses were inaccurate.

The most important practical constraints were:

- Kaggle reruns the notebook with **internet disabled**.
- All Python wheels, model source trees, and weights must be provided as a Kaggle Dataset.
- Some GitHub/Hugging Face models try to download weights at runtime unless checkpoints are pre-copied into the expected torch cache paths.
- The hidden split can have different scene composition from the public notebook preview, so a small public-scene improvement was not trusted unless the Kaggle score confirmed it.
- Runtime matters.  Any method that processes all image pairs with a heavy dense model can time out even if it is theoretically strong.

## Evaluation Methodology

Each version changed a small number of controlled variables and was submitted to Kaggle for hidden-score validation.  The experiment style evolved over time:

1. **Direct model replacement:** try newer or stronger local matchers as primary models.
2. **SfM stability modules:** add shared intrinsics, scene fallback, and missing pose recovery.
3. **Routing and rescue modules:** try time-budget retry, PnP rescue, GLOMAP, MASt3R, PixSfM, SAM, and hybrid pair selection.
4. **Transparent-scene specialization:** stop treating transparent scenes as normal SfM scenes and infer a circular ordering instead.
5. **Fine-grained scalar tuning:** tune RANSAC threshold, transparent rank weights, ALIKED resolution, keypoint count, and DeDoDe auxiliary scores.

The main rule was: if a change did not improve the Kaggle score, it was not kept, even if it looked better locally or was more recent in the literature.

## Final Pipeline Overview

The final selected pipeline is `ver141`.  Conceptually it has two branches:

```text
Input scene
    |
    +-- if scene is transparent-like:
    |       compute LoMa pair evidence
    |       compute high-resolution ALIKED pair evidence
    |       combine rank scores with LoMa 20% + ALIKED 80%
    |       solve circular ordering / ring pose prior
    |       output ring poses
    |
    +-- otherwise:
            extract LoMa-B features
            match image pairs
            verify with MAGSAC/RANSAC threshold 1.08
            write COLMAP database
            share intrinsics for identical image sizes
            run PyCOLMAP incremental SfM
            if scene is weak, use scene-level ALIKED fallback
            recover remaining missing images with gated centroid/copy
            output poses
```

The final branch choices are intentionally conservative:

| Component | Final choice | Why |
| --- | --- | --- |
| Primary matcher | LoMa-B | Best non-transparent reconstruction quality among tested primary matchers. |
| Backend | PyCOLMAP incremental SfM | More stable than tested GLOMAP variants for this database. |
| Geometric verification | MAGSAC/RANSAC threshold `1.08` | Best hidden score. |
| Camera intrinsics | Shared by identical resolution | Reduces optimization degrees of freedom. |
| Scene fallback | Scene-level ALIKED + LightGlue | Useful as a safety net; edge-level mixing was unsafe. |
| Missing pose recovery | Centroid shift only when verified matches `>=80`, otherwise pose copy | Avoids low-confidence PnP/shift errors. |
| Transparent scenes | Ring-pose prior | Avoids impossible transparent/reflection SfM geometry. |
| Transparent ordering | LoMa 20% + ALIKED 80% rank ensemble | Largest score gain. |

## Version Naming and Reproducibility Notes

Versions were generated as Kaggle notebooks.  The version number is not a chronological guarantee of quality; it records experiment order.  Later versions often test narrower hypotheses around an older strong baseline.  For example:

- `ver9` is the first strong LoMa-B baseline.
- `ver28` is the best pre-transparent-specialization baseline.
- `ver71` is the transparent-ordering breakthrough.
- `ver102` is `ver71` with RANSAC `1.00`.
- `ver141` is the final best RANSAC `1.08` version.

The final report uses Kaggle scores reported from submissions.  Some versions were diagnostic, timed out, or were not submitted after dependency issues.  Those are marked as `timeout`, `failed`, `diagnostic`, or `not scored`.

## Score Progression

The major improvements were not linear.  The best score moved through a small number of clear jumps:

| Stage | Version | Score | Gain from previous stage | Main reason |
| --- | --- | ---: | ---: | --- |
| Stable baseline | `ver1` | `0.201764` | - | ALIKED + LightGlue baseline. |
| Strong primary matcher | `ver9` | `0.241186` | `+0.039422` | LoMa-B primary reconstruction. |
| Clean SfM modules | `ver19` | `0.252244` | `+0.011058` | Shared intrinsics. |
| Safe recovery | `ver28` | `0.255489` | `+0.003245` | Gated centroid recovery. |
| Transparent specialization | `ver71` | `0.282382` | `+0.026893` | LoMa/ALIKED transparent ring ordering. |
| RANSAC tuning | `ver102` | `0.288293` | `+0.005911` | RANSAC threshold `1.00`. |
| Final threshold | `ver141` | `0.292267` | `+0.003974` | RANSAC threshold `1.08`. |

This progression explains the final strategy: the project did not win by stacking many new models, but by identifying two high-impact failure modes:

- normal-scene SfM needed stable geometry and intrinsics;
- transparent scenes needed a special non-SfM ordering prior.

## Version Score Timeline

### 1. Initial Matcher and Dependency Experiments

These versions established that a direct replacement of the baseline matcher was not enough.  They also exposed important offline Kaggle issues: missing wheels, torch checkpoint cache problems, forbidden zip filenames, and models trying to download weights during rerun.

| Version | Score / Status | Experiment | Conclusion |
| --- | ---: | --- | --- |
| `ver1` | `0.201764` | ALIKED + LightGlue + COLMAP stable baseline | Working but too weak. |
| `ver2` | `0.187977` | DISK + LightGlue | Worse than ALIKED. |
| `ver3` | timeout | ALIKED + RoMa fallback | Too slow for Kaggle runtime. |
| `ver4` | `0.200950` | ALIKED + EDM fallback | Similar to baseline, no useful gain. |
| `ver5` | failed / gauge issues | EDM primary | Reconstruction unstable. |
| `ver7` | failed / too slow | MASt3R primary attempt | Dense pairwise MASt3R was too expensive and produced weak COLMAP connectivity. |
| `ver8` | diagnostic only | Capped MASt3R fallback | Not competitive as a main path. |
| `ver9` | `0.241186` | LoMa-B primary baseline | First strong baseline; became the main model family. |
| `ver10` | `0.176683` | GIM-SuperPoint + GIM-LightGlue primary | High registration count did not translate to score; graph quality was worse. |
| `ver11` | timeout | Two-stage crop + GIM fallback + TSP/shared intrinsics | Too much complexity and runtime. |
| `ver12` | `0.126580` | LoMa-B + strict GIM fallback | GIM fallback poisoned or weakened scenes. |
| `ver13` | `0.215341` old, `0.116143` later rerun | LoMa-L / high-resolution LoMa line | Larger LoMa variant did not improve the Kaggle hidden score. |
| `ver14` | `0.116152` | LoMa-L + strict GIM fallback | Not useful. |

Key decision: keep **LoMa-B** as the primary matcher and stop trying heavy model replacement before the pipeline-level issues were fixed.

### 2. Core SfM Modules: Fallback, Shared Intrinsics, and Recovery

These versions isolated the three modules that later formed the strong `ver28` base:

- `B`: scene-level ALIKED fallback for disaster scenes.
- `C`: shared camera intrinsics by identical resolution.
- `D`: missing pose recovery with centroid shift.

| Version | Score | Modules | Conclusion |
| --- | ---: | --- | --- |
| `ver15` | `0.228614` | Transparent TSP only | Early transparent logic alone hurt the LoMa baseline. |
| `ver16` | `0.232378` | Transparent TSP + shared intrinsics | Better than `ver15`, still below `ver9`. |
| `ver17` | `0.248940` | Scene-level ALIKED fallback | Strong positive signal. |
| `ver18` | `0.230433` | Transparent TSP + fallback + shared intrinsics | Module interaction was bad in this form. |
| `ver19` | `0.252244` | Shared intrinsics only | Strongest clean improvement at this stage. |
| `ver20` | `0.252025` | Fallback + shared intrinsics | Slightly below `ver19`; fallback and shared intrinsics had mild conflict. |
| `ver21` | `0.252052` | Fallback + shared intrinsics retest | Confirmed the same level as `ver20`. |
| `ver22` | `0.221580` | DINOv2 hybrid pair selection | Pair pruning removed useful graph edges. |
| `ver23` | `0.252359` | Fallback + centroid recovery | Centroid recovery helped when paired with fallback. |
| `ver24` | `0.223009` | Fallback + shared intrinsics + DINOv2 pair selection | DINOv2 pair selection remained bad even with shared intrinsics. |
| `ver25` | `0.252761` | Fallback + shared intrinsics + centroid recovery | Best pre-gated recovery version. |
| `ver26` | `0.244680` | Shared intrinsics + centroid recovery, no fallback | Centroid recovery was harmful without the fallback context. |
| `ver27` | `0.248120` | `ver25` with centroid lambda `0.05` | Lower lambda hurt. |
| `ver28` | **`0.255489`** | `ver25` + centroid shift only when verified matches `>=80` | Best pre-transparent breakthrough baseline. |

Important interaction:

| Comparison | Difference | Interpretation |
| --- | ---: | --- |
| `ver9 -> ver17` | `+0.007754` | Scene fallback helped. |
| `ver9 -> ver19` | `+0.011058` | Shared intrinsics helped more. |
| `ver17 -> ver23` | `+0.003419` | Centroid recovery helped with fallback. |
| `ver19 -> ver26` | `-0.007564` | Centroid recovery hurt without fallback. |
| `ver25 -> ver28` | `+0.002728` | Gating centroid recovery was the correct safety mechanism. |

Key decision: preserve `ver28` as the clean strong baseline, especially the `verified_matches >= 80` gate.

### 3. Runtime Scheduling, Pair Selection, Rescue, and Refinement

This group tested whether `ver28` could be improved by smarter scheduling, alternative pair selection, PnP rescue, GLOMAP, MASt3R, PixSfM, or SAM.

| Version | Score / Status | Experiment | Conclusion |
| --- | ---: | --- | --- |
| `ver29` | `0.247571` | Time-budgeted retry | Retry was too aggressive or accepted worse reconstructions. |
| `ver30` | `0.200853` | Sliding-window pair selection | Strongly damaged graph connectivity. |
| `ver31` | `0.248542` | Hierarchical PnP / centroid / copy recovery | PnP rescue was weaker than gated centroid. |
| `ver32` | `0.206508` | Combined retry + sliding window + hierarchy | Combined weak ideas compounded. |
| `ver33` | `0.252289` | Strict disaster-only retry | Safer but still below `ver28`. |
| `ver34` | `0.253701` | Guarded ALIKED ensembling | Slight signal, not enough. |
| `ver35` | `0.252530` | Adaptive centroid threshold | Worse than fixed gate `80`. |
| `ver36` | `0.252882` | Strict retry + ALIKED ensemble + adaptive centroid | Still below `ver28`. |
| `ver37` | `0.214655` | GLOMAP backend replacement | GLOMAP integration hurt badly. |
| `ver38` | `0.239057` | GLOMAP + MASt3R scale-safe rescue | Better than `ver37`, still worse than LoMa/COLMAP. |
| `ver39` | about `0.248334` | MASt3R rescue on COLMAP | PnP-like rescue remained below `ver28`. |
| `ver40` | `0.217602` | GLOMAP skeleton + COLMAP triangulation | Did not recover GLOMAP quality. |
| `ver42` | `0.253093` | Destructive guided refinement | Slightly below `ver28`; not worth continuing. |
| `ver45` | `0.248854` | Multi-neighbor LoMa consensus PnP | PnP family still weak. |
| `ver46` | `0.252109` | Non-destructive guided refinement | Standard geometric refinement did not help. |
| `ver48` | `0.201753` | SAM transparent foreground masking | Catastrophic; masking removed useful transparent-scene evidence. |
| `ver49` | `0.252262` | PixSfM featuremetric refinement attempt | PixSfM extension unavailable in Kaggle; effectively fallback behavior. |

Key decision: stop PnP rescue, GLOMAP, SAM masking, DINOv2 pair pruning, and general BA/refinement tuning.  The remaining ceiling was not in generic backend replacement.

### 4. Transparent-Scene Breakthrough

Transparent scenes were the turning point.  Standard SfM is unreliable on transparent or reflective objects because local matchers can follow reflections and inconsistent background cues.  The successful approach was to bypass full SfM for transparent scenes and use a ring-pose prior, where the image order is inferred from pairwise matching evidence.

| Version | Score | Experiment | Conclusion |
| --- | ---: | --- | --- |
| `ver57` | `0.249944` | ALIKED-only transparent ordering, 8192 keypoints, resize `2048` | Better than old transparent TSP, but below `ver28`. |
| `ver58` | `0.259549` | LoMa 35% + ALIKED 65% rank ensemble | First clear transparent-ordering breakthrough. |
| `ver59` | `0.252881` | Remove fallback | Fallback still useful in the old base, but not the main gain. |
| `ver60` | `0.250030` | Disable shared intrinsics | Shared intrinsics should stay. |
| `ver64` | `0.256435` | LoMa-B 6144 keypoints | Small positive signal. |
| `ver66` | `0.261912` | ALIKED-only, resize `1600` | Strong pure-ALIKED ordering. |
| `ver67` | `0.261753` | ALIKED-only, resize `2400` | Similar to `ver66`. |
| `ver68` | `0.262593` | ALIKED-only + rotation search | Rotation helped ALIKED-only ordering. |
| `ver69` | `0.261313` | ALIKED-only + rotation + crop rematch | Crop rematch hurt. |
| `ver70` | `0.260061` | LoMa 50% + ALIKED 50% | Too much LoMa compared with the optimum. |
| `ver71` | **`0.282382`** | LoMa 20% + ALIKED 80% rank ensemble | Large breakthrough and new main baseline. |

Important conclusion: pure ALIKED was useful, but **not enough**.  The best transparent signal came from mostly ALIKED with a small LoMa contribution.  The LoMa 20% / ALIKED 80% balance was much better than 50/50 or 35/65.

### 5. RANSAC and Transparent Fine Tuning

Once `ver71` became the new main branch, the next reliable improvement was tuning the geometric verification threshold.

| Version | Score | Experiment | Conclusion |
| --- | ---: | --- | --- |
| `ver98` | `0.211439` | RANSAC threshold `0.50` | Far too strict; graph collapsed. |
| `ver99` | `0.263041` | RANSAC threshold `1.00` on older branch | Strong positive signal. |
| `ver100` | `0.254383` | RANSAC threshold `1.25` | Too loose. |
| `ver101` | `0.276049` | `ver71` + RANSAC `0.90` | Worse than `ver71`; still too strict. |
| `ver102` | **`0.288293`** | `ver71` + RANSAC `1.00` | Major gain over `ver71`. |
| `ver103` | `0.285598` | `ver71` + RANSAC `1.10` | Good, but below `1.00` in this batch. |
| `ver104` | `0.286062` | `ver71` + LoMa-B 6144 keypoints | Positive but below `ver102`. |
| `ver105` | `0.283976` | LoMa-B 6144 + RANSAC `1.00` | Combination did not add. |
| `ver106` | `0.282950` | Transparent ALIKED rotation search | Tiny improvement over `ver71`, much less than RANSAC. |
| `ver107` | `0.287379` | Rotation search + RANSAC `1.00` | Good but below `ver102`. |
| `ver108` | `0.284037` | Rotation + LoMa 6144 + RANSAC `1.00` | Too many small changes hurt. |

Key decision: RANSAC near `1.00` clearly helped.  Extra rotation/keypoint changes were not reliably additive.

### 6. Transparent Weight, Resolution, and DeDoDe Diagnostics

These versions explored whether the transparent ordering optimum could be pushed further with different rank weights, ALIKED resolution/keypoints, and DeDoDe ordering signals.

| Version | Score | Experiment | Conclusion |
| --- | ---: | --- | --- |
| `ver109` | `0.273276` | LoMa 10% / ALIKED 90% | Too much ALIKED; LoMa 20% was important. |
| `ver110` | `0.283894` | LoMa 15% / ALIKED 85% | Good, but below `ver102`. |
| `ver111` | `0.259946` | LoMa 25% / ALIKED 75% | Too much LoMa. |
| `ver112` | `0.250062` | LoMa 30% / ALIKED 70% | Much worse. |
| `ver113` | `0.283510` | `ver71` ensemble + ALIKED resize `1600` | Positive but below `ver102`. |
| `ver114` | `0.286659` | `ver71` ensemble + ALIKED resize `2400` | Strong but below `ver102`. |
| `ver115` | `0.280031` | ALIKED keypoints `6144` | Worse. |
| `ver116` | `0.287036` | ALIKED keypoints `12288` | Strong but below `ver102`. |
| `ver117` | `0.245100` | Transparent category lower-case diagnostic on old branch | Category handling alone was not the breakthrough. |
| `ver118` | `0.277662` | `ver71` without category lower-case change | Lower than `ver71`; robust category detection matters. |
| `ver119` | `0.285524` | `ver71` exact rerun | Confirms some score variance but same high-score family. |
| `ver120` | `0.285573` | DeDoDe 20% transparent ordering | DeDoDe as a large score source did not beat `ver102`. |
| `ver121` | `0.255498` | DeDoDe-only transparent ordering | DeDoDe alone was near `ver28`, far below ALIKED/LoMa ensemble. |
| `ver122` | `0.197291` | DeDoDe primary non-transparent + `ver71` transparent branch | DeDoDe primary reconstruction was not viable. |
| `ver123` | not scored | DINOv3-only transparent ordering | DINOv3 checkpoint was not available offline. |
| `ver124` | not scored | `ver71` + DINOv3 transparent score | Skipped without checkpoint. |
| `ver125` | not scored | LoMa + ALIKED + DeDoDe + DINOv3 | Skipped without checkpoint. |

Key decision: DeDoDe was not a good primary model and not strong enough as a main transparent ordering signal.  It could still be tested as a very low-weight signal, but the best path remained LoMa/ALIKED plus RANSAC tuning.

### 7. `ver102`-Based Expanded Sweep

This group tried to combine the new `ver102` baseline with transparent resolution/keypoints, RANSAC fine sweep, rotation search, and several DeDoDe scoring modes.

| Version | Score | Experiment | Conclusion |
| --- | ---: | --- | --- |
| `ver126` | `0.279013` | `ver102` + transparent resize `2400` | Hurt compared with `ver102`. |
| `ver127` | `0.279816` | `ver102` + transparent ALIKED keypoints `12288` | Hurt. |
| `ver128` | `0.283583` | Resize `2400` + ALIKED `12288` | Still below `ver102`. |
| `ver129` | `0.260424` | RANSAC `0.98` | Unexpectedly low; threshold interaction is noisy. |
| `ver130` | `0.288274` | RANSAC `1.02` | Essentially tied with `ver102`. |
| `ver131` | `0.280148` | RANSAC `1.05` | Worse in this run. |
| `ver132` | `0.286913` | DeDoDe 5% mutual transparent score | Close but below `ver102`. |
| `ver133` | `0.287927` | DeDoDe 10% mutual transparent score | Very close to `ver102`, not better. |
| `ver134` | `0.287921` | DeDoDe 5% mutual, resize `1600`, 4096 keypoints | Very close, not better. |
| `ver135` | `0.187817` | DeDoDe 5% mutual, resize `2048`, 4096 keypoints | Catastrophic. |
| `ver136` | `0.289478` | `ver102` + transparent resize `2400` + rotation | Strong, but below final `ver141`. |
| `ver137` | `0.289035` | `ver102` + ALIKED `12288` + rotation | Strong. |
| `ver138` | `0.287007` | Resize `2400` + `12288` + rotation | Combination did not add. |
| `ver139` | `0.268455` | RANSAC `0.95` | Too strict. |
| `ver140` | `0.288992` | RANSAC `1.03` | Strong. |
| `ver141` | **`0.292267`** | RANSAC `1.08` | Best overall. |
| `ver142` | `0.285991` | DeDoDe 5% dual-softmax transparent score | Below `ver102`. |
| `ver143` | `0.274656` | DeDoDe 10% dual-softmax transparent score | Too much DeDoDe hurt. |
| `ver144` | `0.289834` | DeDoDe 5% dual-softmax, resize `1600`, 4096 keypoints | Best DeDoDe-related version, still below `ver141`. |
| `ver145` | `0.192510` | DeDoDe 5% dual-softmax, resize `2048`, 4096 keypoints | Catastrophic. |
| `ver146` | `0.288362` | DeDoDe 3% dual-softmax transparent score | Competitive, not better. |
| `ver147` | `0.265685` | DeDoDe 15% dual-softmax transparent score | Too much DeDoDe. |
| `ver148` | `0.207482` | DeDoDe-only transparent dual-softmax | Very bad. |
| `ver149` | `0.186676` | DeDoDe primary non-transparent dual-softmax | Very bad. |
| `ver150` | `0.278612` | DeDoDe 5% dual-softmax MNN transparent score | Worse than simple low-weight versions. |

Key decision: `ver141` showed that RANSAC `1.08` was the best scalar improvement.  DeDoDe only made sense as a tiny auxiliary transparent score; high-weight or primary DeDoDe was consistently bad.

### 8. Final RANSAC, DeDoDe Tie-Break, and Transparent Search Sweep

These versions tested whether `ver141` could be beaten by finer RANSAC values, official DeDoDe APIs, multi-start TSP, or dynamic transparent settings.

| Version | Score | Experiment | Conclusion |
| --- | ---: | --- | --- |
| `ver151` | `0.279149` | RANSAC `0.99` | Worse. |
| `ver152` | `0.287748` | RANSAC `1.01` | Good, not best. |
| `ver153` | `0.283507` | RANSAC `1.015` | Worse. |
| `ver154` | `0.288881` | RANSAC `1.025` | Strong but below `ver141`. |
| `ver155` | `0.193192` | Official DeDoDe DualSoftMax transparent 5% | Official path failed badly. |
| `ver156` | `0.190960` | Official DeDoDe DualSoftMax transparent 10% | Failed badly. |
| `ver157` | `0.191512` | Official DeDoDe-only transparent DualSoftMax | Failed badly. |
| `ver158` | `0.287639` | Official DeDoDe tie-break 2% | Low-weight tie-break was safe but did not improve. |
| `ver159` | `0.287201` | Official DeDoDe tie-break 5% | Similar, below best. |
| `ver160` | `0.286954` | Official DeDoDe tie-break 10% | More DeDoDe weight hurt. |
| `ver161` | `0.286959` | Transparent multi-start 2-opt x8 | Multi-start did not improve. |
| `ver162` | `0.286521` | Transparent multi-start 2-opt x16 | More starts did not help. |
| `ver163` | `0.287413` | Multi-start + resize `2400` / keypoints `12288` | Still below best. |
| `ver164` | `0.287769` | Dynamic transparent boost for scenes <=60 images | Below best. |
| `ver165` | `0.271560` | Dynamic transparent boost for scenes <=45 images | Bad threshold. |
| `ver166` | `0.287822` | Dynamic transparent boost + multi-start | Below best. |
| `ver167` | `0.284601` | RANSAC `1.06` | Below `ver141`. |
| `ver168` | `0.286972` | RANSAC `1.07` | Below `ver141`. |
| `ver169` | `0.290150` | RANSAC `1.075` | Second-best overall, validates the `1.07`-`1.08` region. |
| `ver170` | `0.282453` | RANSAC `1.085` | Dropped sharply. |
| `ver171` | `0.274904` | RANSAC `1.09` | Too loose / bad interaction. |
| `ver172` | `0.283530` | RANSAC `1.08` + transparent rotation | Rotation hurt the best RANSAC version. |
| `ver173` | `0.286592` | RANSAC `1.08` + rotation + resize `2400` | Still below `ver141`. |
| `ver174` | `0.287140` | RANSAC `1.08` + rotation + ALIKED `12288` | Still below `ver141`. |

Key decision: stop adding extra transparent rotation/dynamic/multi-start layers on top of `ver141`.  The best hidden score was a simple RANSAC threshold change, not a more complicated transparent-ordering search.

## Method-Level Summary

The following table groups all tested ideas by practical outcome.

| Method family | Best related version | Best score | Outcome | Reason |
| --- | --- | ---: | --- | --- |
| LoMa-B primary SfM | `ver9` / later inherited | `0.241186` before extra modules | Good | Strong local matching and compatible with COLMAP. |
| Shared intrinsics | `ver19` | `0.252244` | Good | Reduces COLMAP degrees of freedom. |
| Scene-level fallback | `ver17` | `0.248940` | Good as safety net | Helps failed scenes without poisoning successful LoMa graphs. |
| Gated centroid recovery | `ver28` | `0.255489` | Good | Safe missing-pose strategy when enough match evidence exists. |
| Transparent ring ordering | `ver71` | `0.282382` | Excellent | Solves transparent scenes with a scene-specific prior. |
| RANSAC threshold tuning | `ver141` | `0.292267` | Excellent | Best graph connectivity/outlier tradeoff. |
| ALIKED-only transparent ordering | `ver68` | `0.262593` | Useful but incomplete | Good ordering evidence, but LoMa rank still needed. |
| Transparent rotation search | `ver136` | `0.289478` | Mixed | Helps some settings, hurts when stacked on `ver141`. |
| LoMa keypoint increase | `ver104` | `0.286062` | Mildly positive alone | More points can help but is not additive with best RANSAC. |
| DeDoDe low-weight transparent score | `ver144` | `0.289834` | Mixed / secondary | Can help as small auxiliary signal, not as main signal. |
| DeDoDe primary | `ver149` | `0.186676` | Bad | Graph quality and routing incompatible with the final SfM branch. |
| GIM primary / fallback | `ver10` / `ver12` | `0.176683` / `0.126580` | Bad | More matches did not mean better camera poses. |
| LoMa-L high-res | `ver13` / `ver14` | `0.215341` / `0.116152` | Bad | Heavier model/config did not generalize in hidden score. |
| DINOv2 pair selection | `ver22` / `ver24` | `0.221580` / `0.223009` | Bad | Pair pruning removed geometric bridge edges. |
| Sliding-window pairs | `ver30` | `0.200853` | Bad | Sequential assumption was unsafe for unordered scenes. |
| PnP / MASt3R rescue | `ver31` / `ver39` / `ver45` | around `0.248` | Bad | Recovery poses were less reliable than conservative copy/centroid. |
| GLOMAP backend | `ver37` / `ver40` | `0.214655` / `0.217602` | Bad | Integration/retriangulation produced worse model quality. |
| SAM transparent masking | `ver48` | `0.201753` | Bad | Masking removed useful context for transparent ordering. |
| PixSfM refinement | `ver49` | `0.252262` | Not useful here | Offline compiled extension unavailable; fallback only. |
| Multi-start transparent 2-opt | `ver161` / `ver162` | `0.286959` / `0.286521` | Not useful | More search did not beat simple rank ordering. |
| Dynamic transparent boost | `ver164` / `ver166` | `0.287769` / `0.287822` | Not useful | Extra conditional logic did not generalize. |

## What Worked

### LoMa-B as the Primary Non-Transparent Matcher

LoMa-B was the first major upgrade:

```text
0.039422 over ver1
ver1 = 0.201764
ver9 = 0.241186
```

It remained the best non-transparent base.  Larger or alternative primary matchers did not beat it:

- GIM primary: `ver10 = 0.176683`
- LoMa-L variants: about `0.116` to `0.215`
- DeDoDe primary: `ver122 = 0.197291`, `ver149 = 0.186676`
- MASt3R rescue/primary and GLOMAP variants were below the LoMa/COLMAP branch.

### Shared Intrinsics by Resolution

Shared camera intrinsics were one of the first clean improvements:

```text
0.011058 over ver9
ver9  = 0.241186
ver19 = 0.252244
```

This reduced the number of free camera parameters during COLMAP optimization.  Removing the idea later also hurt:

```text
ver60 = 0.250030
```

### Scene-Level Fallback, Not Edge-Level Mixing

Scene-level fallback helped early:

```text
ver17 = 0.248940
```

The useful lesson was not "mix all matchers into one graph"; it was "only retry whole failed scenes."  Edge-level or pair-level mixing often contaminated the graph.

### Gated Centroid Pose Recovery

Ungated pose recovery had unstable interactions.  The successful form was:

```text
verified matches >= 80: centroid-shift recovery
otherwise: conservative pose copy
```

This produced:

```text
ver25 = 0.252761
ver28 = 0.255489
```

The gate mattered because low-confidence 2D shifts are more likely to move cameras in the wrong direction than to rescue them.

### Transparent Ring Ordering

This was the largest breakthrough:

```text
ver28 = 0.255489
ver71 = 0.282382
gain  = +0.026893
```

The best transparent branch did not do normal SfM.  It used a ring-pose prior and estimated circular ordering with pairwise rank evidence:

```text
LoMa rank weight   = 0.20
ALIKED rank weight = 0.80
```

Why it worked:

- Transparent scenes produce reflection and refraction matches that violate 3D geometry.
- Full SfM can fail even when local matching has many confident-looking correspondences.
- The Kaggle scoring can still reward a plausible circular camera order for transparent objects.
- ALIKED gave strong ordering evidence, while a small LoMa component stabilized cases where ALIKED alone was ambiguous.

### RANSAC Threshold Around 1.08

RANSAC/MAGSAC threshold tuning was the best final-stage scalar improvement:

```text
ver71  = 0.282382
ver102 = 0.288293  # threshold 1.00
ver141 = 0.292267  # threshold 1.08
```

The threshold curve was not perfectly smooth, but the useful region was clear:

- Too strict: `0.50`, `0.90`, `0.95`, `0.98` were bad.
- Good: `1.00`, `1.02`, `1.03`, `1.075`, `1.08`.
- Too loose / unstable: `1.085`, `1.09`, `1.25`.

Best observed:

```text
ver141 = 1.08  -> 0.292267
ver169 = 1.075 -> 0.290150
```

## What Did Not Work

### DINOv2 Pair Selection and Sliding Windows

These hurt badly:

```text
ver22 = 0.221580
ver24 = 0.223009
ver30 = 0.200853
ver32 = 0.206508
```

Reason: global retrieval or temporal windows remove "bridge" edges that COLMAP needs.  Even if the selected pairs look semantically similar, SfM needs graph connectivity and geometric coverage, not only nearest-neighbor appearance.

### PnP / MASt3R / Multi-Neighbor Pose Rescue

All PnP-style missing pose recovery variants stayed around `0.248`:

```text
ver31 ~= 0.248542
ver39 ~= 0.248334
ver45 ~= 0.248854
```

Reason: missing-image rescue is high risk.  If a camera pose is inserted with the wrong scale, wrong neighbor, or slightly wrong translation, the mAA score can drop.  The simple gated centroid/copy rule was safer.

### GLOMAP Backend Replacement

GLOMAP looked promising in theory, but the actual Kaggle pipeline got worse:

```text
ver37 = 0.214655
ver38 = 0.239057
ver40 = 0.217602
```

Possible reasons:

- GLOMAP retriangulation was difficult to integrate with the shared-intrinsics COLMAP database.
- Skipping or modifying retriangulation reduced point/track quality.
- Global SfM was not automatically better for this hidden split and this LoMa-generated graph.

### SAM Transparent Masking

SAM foreground masking failed:

```text
ver48 = 0.201753
```

Likely reasons:

- Transparent objects are not always centered.
- Background seen through glass can still be useful for image ordering.
- Masking or darkening the background destroys stable context.
- The branch changed high-value transparent scenes, so failures were heavily penalized.

### PixSfM / Standard Refinement

Standard COLMAP refinement variants did not help:

```text
ver42 = 0.253093
ver46 = 0.252109
```

PixSfM also did not become a useful branch because the compiled extension was unavailable in the Kaggle offline environment:

```text
PixSfM unavailable: ModuleNotFoundError: No module named 'pixsfm._pixsfm._base'
ver49 = 0.252262
```

### DeDoDe as Primary or High-Weight Signal

DeDoDe was extensively tested.  It did not become competitive:

```text
ver121 = 0.255498  # DeDoDe-only transparent ordering
ver122 = 0.197291  # DeDoDe primary non-transparent
ver148 = 0.207482  # DeDoDe-only transparent dual-softmax
ver149 = 0.186676  # DeDoDe primary non-transparent dual-softmax
```

High-weight DeDoDe also hurt:

```text
ver143 = 0.274656  # 10% dual-softmax
ver147 = 0.265685  # 15% dual-softmax
```

The only somewhat useful DeDoDe form was a small auxiliary transparent score:

```text
ver144 = 0.289834
ver146 = 0.288362
ver158 = 0.287639
```

This suggests DeDoDe may contain some complementary ordering evidence, but it is weaker than the ALIKED/LoMa rank signal and easy to overweight.

Additional observations from the DeDoDe sweep:

- DeDoDe-only transparent ordering did not carry enough stable circular-order information.
- DeDoDe primary non-transparent reconstruction was much worse than LoMa-B, suggesting that even a strong detector/descriptor can be a poor drop-in replacement if the downstream pair filtering and COLMAP graph are not tuned for it.
- Small DeDoDe weights sometimes produced competitive scores because they acted as a weak tie-breaker, not because they replaced the LoMa/ALIKED signal.
- Official dual-softmax variants were surprisingly poor in this pipeline.  A likely reason is that the score distribution was not calibrated to the existing rank ensemble; adding the raw matching evidence changed the order of many transparent edges rather than only resolving ties.
- DeDoDe resize/keypoint settings were fragile.  Versions using resize `2048` with 4096 keypoints collapsed, which suggests this branch was not robust enough for hidden-scene routing.

### More Complex Transparent Search

After `ver141`, extra transparent complexity did not improve:

```text
ver161 = 0.286959  # multi-start x8
ver162 = 0.286521  # multi-start x16
ver164 = 0.287769  # dynamic small-scene boost
ver166 = 0.287822  # dynamic + multistart
ver172 = 0.283530  # best RANSAC + rotation
ver173 = 0.286592  # best RANSAC + rotation + resize
ver174 = 0.287140  # best RANSAC + rotation + keypoints
```

Reason: once the LoMa/ALIKED ring ordering was strong, more search or resolution changes likely overfit public patterns or changed hidden-scene ordering in harmful ways.

## Error Analysis and Lessons

### Registered Count Was Not Enough

Several logs showed high registration counts or many verified matches, but the final Kaggle score still dropped.  This means the score rewarded accurate relative camera geometry more than simple coverage.  This was visible in:

- `ver10`, where GIM produced a connected graph but scored poorly;
- PnP/MASt3R rescue versions, where extra recovered poses lowered score;
- DeDoDe primary versions, where the graph existed but the pose quality was poor.

The final design therefore accepts fewer aggressive pose changes and prefers conservative recovery.

### Model Recency Was Not a Reliable Predictor

Many newer or more ambitious methods were tested:

- MASt3R / DUSt3R-style rescue;
- GLOMAP global SfM;
- DeDoDe;
- SAM;
- PixSfM;
- DINO-based pair selection;
- LoMa-L.

Most did not improve the final score.  The likely reason is that IMC submission quality depends on the whole system:

```text
feature extraction
pair selection
geometric verification
database writing
camera model assumptions
SfM backend
fallback routing
pose recovery
scene category handling
offline execution
```

A stronger isolated model can still hurt if its outputs are not calibrated for the rest of the pipeline.

### Transparent Scenes Needed a Different Objective

Transparent scenes are not just "hard normal scenes."  They violate assumptions behind standard local-feature SfM:

- reflections can look like repeatable features but correspond to virtual objects;
- background through transparent material can change relative to the object;
- the real 3D surface can be textureless or ambiguous;
- Fundamental matrix verification can accept visually plausible but physically inconsistent matches.

The successful transparent branch changed the objective from "estimate full SfM" to "estimate a plausible circular order."  This was the largest conceptual change in the project.

### RANSAC Threshold Had Hidden-Split Sensitivity

The RANSAC threshold sweep was not perfectly monotonic:

```text
0.98  -> 0.260424
1.00  -> 0.288293
1.02  -> 0.288274
1.03  -> 0.288992
1.075 -> 0.290150
1.08  -> 0.292267
1.085 -> 0.282453
1.09  -> 0.274904
```

This suggests a sharp tradeoff:

- too strict: removes useful matches and weakens COLMAP connectivity;
- too loose: accepts outliers and corrupts the pose graph;
- best region: enough extra edges for connectivity while still keeping outliers manageable.

The best hidden score happened at `1.08`, but `1.075` and `1.03` remain reasonable backup settings because small hidden-set variation can change the optimum.

## Implementation Reliability Notes

The following engineering details were necessary for Kaggle rerun reliability:

- All model weights must be present in the offline dependency dataset.
- LoMa's DaD detector weight must be pre-cached as `dad.pth`; otherwise LoMa tries to download it.
- ALIKED and ALIKED-LightGlue checkpoints must be copied into torch hub cache paths.
- Dependencies that contain forbidden Kaggle dataset filenames must be cleaned before zipping.
- Native binaries copied from Kaggle input may not be executable, so executable runtimes need special handling if used.
- Some compiled libraries such as PixSfM are fragile across Python/CUDA/Ceres/PyCOLMAP versions.
- DINOv3 checkpoints require gated Hugging Face access, so DINOv3 variants were left disabled unless the checkpoint is explicitly included.

These reliability constraints explain why the final solution favors models already verified in the offline notebook environment.

## Recommended Submission Strategy

Use `ver141` as the final main submission.  It is the highest score and does not rely on fragile optional dependencies.

If multiple submissions are available, use backup versions that are methodologically different enough to cover hidden-score variance:

| Priority | Version | Reason |
| ---: | --- | --- |
| 1 | `ver141` | Best score, simplest final high-performing setup. |
| 2 | `ver169` | Very close RANSAC threshold; guards against `1.08` being slightly overfit. |
| 3 | `ver144` | Best low-weight DeDoDe auxiliary branch; covers a different transparent-ordering signal. |
| 4 | `ver136` | Strong transparent rotation/resize variant. |
| 5 | `ver102` | Simpler strong baseline; useful as a reliable fallback. |

Do not submit:

- DeDoDe primary versions;
- GLOMAP versions;
- SAM transparent mask versions;
- DINOv2 pair-selection versions;
- sliding-window pair versions;
- PnP/MASt3R rescue versions;
- official DeDoDe-only or high-weight versions.

## Final Recommended Pipeline

The recommended final version is `ver141`.

It contains:

- LoMa-B primary matching for normal scenes.
- MAGSAC/RANSAC threshold `1.08`.
- PyCOLMAP incremental SfM.
- Shared camera intrinsics by identical resolution.
- Scene-level ALIKED + LightGlue fallback retained from the strong baseline.
- Transparent-scene ring-pose prior.
- Transparent ordering from LoMa 20% + ALIKED 80% rank ensemble.
- Gated centroid recovery for missing poses.

Final configuration checklist:

| Setting | Final value / behavior |
| --- | --- |
| Main version | `ver141` |
| Base lineage | `ver71` transparent ordering branch |
| Normal-scene matcher | LoMa-B |
| Normal-scene SfM | PyCOLMAP incremental mapping |
| RANSAC / MAGSAC threshold | `1.08` |
| Shared intrinsics | enabled, grouped by exact `(width, height)` |
| Scene fallback | ALIKED + LightGlue scene-level fallback for weak reconstructions |
| Fallback style | scene-level only, not pair-level mixing |
| Missing pose recovery | centroid shift only for verified matches `>=80`; otherwise pose copy |
| Transparent-scene detection | category-based transparent routing with robust lower-case handling |
| Transparent pose model | circular / ring-pose prior |
| Transparent ordering evidence | LoMa rank + ALIKED rank |
| Transparent rank weights | LoMa `0.20`, ALIKED `0.80` |
| DeDoDe in final main version | disabled |
| GLOMAP in final main version | disabled |
| SAM masking in final main version | disabled |
| DINOv2/DINOv3 pair selection | disabled |
| Sliding-window pair pruning | disabled |
| PnP / MASt3R rescue | disabled |
| Extra transparent multi-start | disabled |
| Extra transparent rotation on final RANSAC | disabled |

Recommended backup versions:

| Backup | Score | Why keep it |
| --- | ---: | --- |
| `ver169` | `0.290150` | Very close RANSAC setting `1.075`; confirms final region. |
| `ver144` | `0.289834` | Best DeDoDe-assisted version; useful if hidden split favors its low-weight transparent score. |
| `ver136` | `0.289478` | Best transparent resize/rotation variant. |
| `ver102` | `0.288293` | Simpler, strong baseline with RANSAC `1.00`. |
| `ver140` | `0.288992` | RANSAC `1.03`, another stable scalar-only backup. |

## Final Takeaways

The project started as a model-replacement search, but the final gains came from understanding the failure modes of the dataset:

1. Normal scenes needed a clean LoMa-B + COLMAP graph, not more matcher mixing.
2. Transparent scenes needed a special geometric prior, not full SfM.
3. Missing-pose recovery needed conservative gating, not aggressive PnP.
4. RANSAC threshold tuning had large hidden-score impact because it controls the tradeoff between graph connectivity and outlier contamination.
5. Newer models were not automatically better.  DeDoDe, MASt3R, GLOMAP, GIM, and SAM were all plausible, but the hidden score showed that integration quality and routing mattered more than paper recency.

The final pipeline therefore favors a small number of validated, stable mechanisms over a large ensemble of untrusted model outputs.
