# Rebuilding Netflix's MAPS on MovieLens

An independent, educational rebuild of the ideas in Netflix's [MAPS: Multimodal Asset Personalization at Scale](https://netflixtechblog.com/maps-netflixs-multimodal-asset-personalization-at-scale-32f96320785e) (Aug 2026), using public data.

Netflix's problem: which artwork to show each member for each title — and what to do right after launch, when an asset has no interaction history (cold-start). Their answer: feed CLIP image embeddings into the ranker so a new asset arrives with a representation the model already understands, and consolidate five per-canvas models into one.

## What's in the notebook

`maps_replication.ipynb` — self-contained: downloads its own data (MovieLens ratings parquet from HuggingFace + poster images) inside the notebook. Runs on Kaggle/Colab with internet on; GPU recommended, CPU works (~15 min at the 600-movie cap).

1. **Data** — ~890k MovieLens ratings joined with poster URLs, most-rated 600 movies kept.
2. **Assets** — 4 artwork variants (crops of each poster) × 5 canvases (aspect-ratio renders), mirroring Netflix's canvas setup and their crop-invariance argument.
3. **CLIP embeddings** — ViT-B/32, one vector per asset, cached to a numpy file (a tiny "Embedding Store").
4. **Simulated impression logs** — Netflix's logs are private, so clicks are simulated from numbered, stated assumptions. A uniform logging policy gives known propensities, like Netflix's exploration slice.
5. **V1 / V2 / V3 ablation** — per-canvas + CLIP, unified ID-only, unified + CLIP. Same architecture as the paper: `h_a = MLP([e_id(a); e_clip(a)])` scored against a user embedding, with reward-weighted training.
6. **IPS offline evaluation** — inverse propensity scoring on an exploration slice of cold-start movies.
7. **Linear probe** — the cheap gate: predict each title's winning artwork from the embedding alone.
8. **Query-aware ranking** — blend the personalization score with cosine(CLIP text query, asset image).

## Result (verified run: 553 movies, 11,060 assets, seed 42)

IPS lift vs the random policy on held-out cold-start movies:

| canvas | V2 ID-only | V1 per-canvas+CLIP | V3 unified+CLIP |
|---|---|---|---|
| billboard | −4.4% | −3.7% | **+5.2%** |
| vertical | +0.5% | −4.5% | **+8.8%** |
| landscape | −1.4% | −3.3% | **+20.3%** |
| short_panel | +5.4% | −3.7% | **+19.0%** |
| panel | +0.7% | +6.6% | **+3.1%** |

Same qualitative story as the paper: V1 and V2 are noise around zero, only V3 wins — neither content embeddings nor model consolidation works alone. Linear probe: 0.355 winner accuracy vs 0.25 random.

One finding worth knowing: raw CLIP vectors share a common direction (mean cosine between *unrelated* movies ≈ 0.28, the "cone effect"). Our first simulation skipped centering and every model scored ≈ random. Center the embeddings before using cosines as a similarity signal.

## What's real vs simulated

**Real:** ratings, posters, CLIP embeddings, architecture, IPS estimator, probe.
**Simulated:** impressions and clicks (Assumptions 1–6 in the notebook). The ground truth is built from CLIP embeddings, which favors the CLIP model by construction — so this shows the *mechanics*, not real-world effect sizes. Netflix's evidence is their A/B tests.

## Run it

Open the notebook on Kaggle or Colab, enable internet (and ideally a GPU), run all. No datasets to attach.

## Credit

All ideas belong to the Netflix authors: Emma Yanyang Kong, Aditya Deshpande, Asad Abbasi, Bowei Yan, David Fagnan, Ashish Rastogi, Dhaval Patel, Ray Zhang. This repo is an unaffiliated educational replication.
