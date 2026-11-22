# I rebuilt Netflix's artwork personalization system on MovieLens. Here's what actually matters.

*Netflix published MAPS in August 2026: how they pick which artwork to show you, and how CLIP embeddings fixed their cold-start problem. The paper is great but you can't touch it. So I rebuilt the mechanics on public data. Code on [GitHub](https://github.com/ZFayFay/netflix-maps-replication), notebook on Kaggle.*

---

## The problem Netflix was solving

Every title on Netflix has many artworks. Which one you see is a personalization problem. Their old models learned from clicks, but treated every artwork as an opaque ID. The model never saw the image.

That works fine for old titles. It fails completely for new ones. A new artwork has no click history, so an ID-based model knows nothing about it. Netflix had to fall back to popularity and wait for data. That's cold-start.

Their fix is simple to state: encode every artwork with CLIP, and concatenate that embedding with the learned ID embedding. Now a brand-new asset arrives with a vector the model already understands. If you click on artwork featuring a certain comedian, that preference lives in image space, not ID space — so it transfers to a title the model has never seen.

The second win: they collapsed five models (one per artwork canvas: billboard, vertical, landscape, and so on) into one. CLIP embeddings barely change under crop and resize, so the renderings of one image across canvases map to nearly the same vector. One model can pool signal across all of them. I checked this on real posters: the five canvas renders of one artwork have mean cosine 0.78 with each other, vs 0.56 for different crops of the same poster.

## What I could and couldn't replicate

Honesty first, because it shapes everything.

**Real:** MovieLens ratings, real movie posters, real CLIP embeddings, the paper's architecture, their offline evaluation method, their linear-probe trick.

**Simulated:** the impression logs. No public dataset has artwork impressions and clicks. So I simulated them from explicit, numbered assumptions. User taste = the average CLIP embedding of posters they actually rated highly. Click probability = a function of taste match plus a global "this artwork just works" term. Every assumption is written down in the notebook.

One more thing I won't hide: the simulation is built in CLIP space, and the winning model uses CLIP features. So the setup favors it by construction. This rebuild demonstrates the *mechanism*. It cannot prove real-world effect sizes. Netflix's proof was a month-long A/B test. Mine is a controlled sandbox.

## The setup

MovieLens gives ~890k real ratings with poster URLs. From each poster I cut four "artwork variants" (different crops showing different content — a face, title text, a scene) and render each variant on five canvases with different aspect ratios. 553 movies × 20 = 11,060 assets — my stand-in for Netflix's artwork catalog.

Then one CLIP pass over all of it. One vector per asset, saved to a numpy file. That file is a toy version of their Embedding Store: encode once, serve to every downstream model.

20% of movies are held out as cold-start. Their impressions never touch training.

## The bug that taught me the most

My first run failed in an instructive way. Every model — with CLIP, without CLIP — scored exactly like random picking. The reason: raw CLIP vectors are not spread around the sphere. They live in a narrow cone. The mean cosine between embeddings of completely unrelated movies was 0.28, not 0. So the cosine between a user's taste vector and the four crops of one poster was nearly identical for all four — no signal, nothing to personalize.

The fix is one line: subtract the mean embedding and renormalize before computing similarities. After centering, the best crop for a user gets a simulated click probability of ~0.20 vs ~0.06 for the worst. If you use CLIP cosines as a similarity signal anywhere, center first.

## The ablation that teaches the real lesson

Netflix tested three variants, and this is the part worth internalizing:

- **V1:** five per-canvas models, each with CLIP embeddings added
- **V2:** one unified model across all canvases, IDs only
- **V3:** one unified model, with CLIP

In their online A/B test, **V1 and V2 did nothing on their own. Only V3 moved the metrics.**

My rebuild reproduces the same pattern on cold-start movies, measured as IPS lift over a random policy: V2 (ID-only) and V1 (per-canvas + CLIP) bounce between −5% and +7% — noise. V3 is positive on all five canvases: +5% on billboard, +9% vertical, +20% landscape, +19% short-panel, +3% panel. A new asset is just an unseen ID for V2; a sparse canvas gives V1 too few examples to use the image content.

The two ingredients need each other. CLIP tells a model what an image contains, but a sparse canvas has too few examples to learn how to use that. Consolidation supplies the data, but ID-only data transfers nothing to a new asset. Together, the rich canvases teach the shared model how image content maps to taste, and that mapping transfers straight to the sparse canvases.

The lesson I'd take into any ML project: when a feature "doesn't help," look for a second blocking factor before throwing the feature away.

## IPS: the most transferable idea in the paper

How do you evaluate a new ranking policy offline? The naive way — replay it on logs from the current system — is biased. The old policy over-showed its favorites, so anything that disagrees with it looks bad, because the impressions it would have picked barely exist in the logs.

Netflix's answer is inverse propensity scoring on a randomized slice. A small fraction of traffic picks artwork at random from a known distribution, and the probability of each impression is logged at serving time. Reweight each observation by 1/propensity and you get an unbiased estimate of what a candidate policy would have earned. Netflix says having propensities that are known by construction — not modeled after the fact — is the single biggest reason their offline numbers track online results.

In my rebuild the whole simulated log is randomized (propensity = 1/4 everywhere), which makes the estimator easy to see working. Whatever system you work on, this is the pattern: reserve a small randomized slice, log the propensities, and you've bought yourself a trustworthy offline metric.

## The linear probe: a $0 gate for new embeddings

New embedding models show up constantly. A full trial — data pipeline, retraining, weeks of A/B traffic — is expensive. Netflix gates candidates with one cheap question: from the content embedding alone, can a *linear* model predict which asset wins under an unpersonalized policy?

Keeping it linear is the point. If the embedding encodes the semantic drivers of appeal, a linear classifier finds them. If it doesn't, you're at chance, and you just saved yourself a failed experiment.

In my rebuild the probe hits 35.5% winner accuracy on cold-start movies vs the 25% random baseline, from the CLIP vector alone — modest, because winners are estimated from few impressions, but clearly above chance. Two cells of code. I'd use this pattern anywhere embeddings feed a bigger system.

## Query-aware ranking, almost for free

CLIP puts text and images in one space. So search-aware artwork is one extra term: blend the personalization score with the cosine between the CLIP embedding of the query and the CLIP embedding of the asset. Search "an explosion" and the ranker surfaces artwork that contains one. No new model, no new infrastructure — the same stored vectors, one similarity at scoring time.

This is the quiet argument for an embedding store. Once content vectors are computed and served from one place, each new use case is a configuration change, not a migration.

## What I'd tell you to remember

1. **Cold-start is an information problem, not a data-volume problem.** The information was in the image the whole time; ID-based models just couldn't see it.
2. **Ablate combinations, not just components.** Netflix's content features and model consolidation were each worthless alone and significant together. My rebuild shows the same: V1 and V2 ≈ 0, V3 +3% to +20%.
3. **Randomize a slice and log propensities.** It's the cheapest path to offline metrics you can trust.
4. **Gate expensive experiments with cheap probes.**
5. **Center CLIP embeddings before using cosines.** Raw CLIP lives in a cone; unrelated items still have cosine ~0.3.

The obvious extension is video: Netflix's MediaFM adds audio and captions to the visual signal, and it beat the visual-only embedding decisively. There's no public equivalent of their preview data, so my rebuild stops at images.

---

*Full notebook and code: [github.com/ZFayFay/netflix-maps-replication](https://github.com/ZFayFay/netflix-maps-replication). All ideas belong to the Netflix authors — this is an independent educational rebuild of their [MAPS post](https://netflixtechblog.com/maps-netflixs-multimodal-asset-personalization-at-scale-32f96320785e).*
