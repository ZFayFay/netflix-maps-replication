# Netflix fixed cold-start with two ideas. Either one alone does nothing.

## I rebuilt Netflix's artwork personalization on MovieLens. The lesson is about combinations, not features.

*Part 1 of **Netflix Tech Blog, rebuilt**. [Part 2: the billboard bandit](https://medium.com/@faycal.zouine.usa/i-rebuilt-netflixs-billboard-bandit-the-metric-picked-the-winner-686296a3b72f) · [Part 3: what recommendations are worth](https://medium.com/@faycal.zouine.usa/your-dashboard-says-recommendations-multiply-viewing-by-7-5-the-truth-is-3-1-a7b68e8334c3)*

*6-minute read. In a hurry? Read the bold lines.*

Every new title on Netflix arrives with the same handicap. Its artwork has never been clicked. So a model that learns from clicks knows nothing about it.

That's cold-start. And Netflix's older artwork models had it bad. They treated each image as an ID number. The model never actually *saw* the picture.

In August 2026, Netflix published [MAPS](https://netflixtechblog.com/maps-netflixs-multimodal-asset-personalization-at-scale-32f96320785e), their fix. I rebuilt it on public data to see what really drives it.

The short answer: two ideas. Neither works alone.

> When a feature "doesn't help," look for the second thing it needs.

## The two ideas

**Idea 1: let the model see the image.** Run every artwork through CLIP (a model that turns an image into a list of numbers that capture what's in it). Glue those numbers to the old ID. Now a brand-new artwork shows up with a description the model already understands.

**Idea 2: one model instead of five.** Netflix shows artwork in five shapes (billboard, vertical, landscape and so on). They used to train one model per shape. MAPS trains one shared model.

Why can one model handle five shapes? Because CLIP barely cares about cropping. I checked on real posters: the five shapes of one image score 0.78 similarity with each other, vs 0.56 for different crops of the same poster.

**Same picture, different crop, almost the same numbers. So the shapes can share what they learn.**

## What's real, what's simulated

Honesty first, because it shapes everything.

Real: 890k MovieLens ratings, real movie posters, real CLIP, Netflix's architecture and their evaluation method.

Simulated: the clicks. No public dataset logs artwork impressions. So I simulated them, with every assumption written down in the notebook.

One more thing. My simulated users have tastes built from CLIP, and the winning model uses CLIP. That stacks the deck. **So this shows the mechanism, not the size of the effect.** Netflix's proof is their A/B test. Mine is a sandbox.

## The bug that taught me the most

My first run failed. Every model, with or without CLIP, scored like random guessing.

The cause: CLIP vectors all point roughly the same way. Two completely unrelated movies still scored 0.28 similarity, not 0. So all four crops of a poster looked equally good to every user. Nothing to personalize.

The fix is one line. Subtract the average vector first (centering). After that, a user's best crop got a click chance of about 0.20, vs about 0.06 for the worst.

**If you use CLIP similarities anywhere, center them first.**

## The result: only the combination wins

Netflix tested three versions. I rebuilt all three and scored them on movies the model had never seen:

- **V1:** five separate models, with CLIP
- **V2:** one shared model, no CLIP
- **V3:** one shared model, with CLIP

V1 and V2 bounced between −5% and +7% vs random. That's noise. V3 won on all five shapes: +5% billboard, +9% vertical, +20% landscape, +19% short panel, +3% panel.

![IPS lift by canvas](https://raw.githubusercontent.com/ZFayFay/netflix-maps-replication/main/figures/fig2_ips.png)

Netflix saw the same thing in their live test. V1 and V2 did nothing. Only V3 moved the metrics.

Here's why. CLIP tells the model *what's in* the image. But a rarely used shape has too few clicks to learn what to *do* with that. The shared model fixes that: the busy shapes teach it how image content maps to taste, and the quiet shapes borrow it.

**Two ingredients that each look useless can be the whole answer together. Test combinations, not just parts.**

## How to grade a model without an A/B test

You can't judge a new model on logs from the old one. The old model mostly showed its favorites. Anything that disagrees with it looks bad, because its picks barely appear in the data.

Netflix's fix: show artwork at random to a small slice of traffic, and write down the odds of each pick at the moment it's shown (the propensity). Then weight each result by 1 ÷ those odds (inverse propensity scoring). That gives a fair estimate of how any new model would have done.

It's the same logic as the geo lift tests I ran on the agency side. You buy a clean comparison with a small, deliberate dose of randomness.

**Keep a small random slice and log the odds. It's the cheapest trustworthy metric you'll ever buy.**

## A free check before an expensive test

New image models come out all the time. Testing one properly takes weeks. Netflix asks one cheap question first: using only the image numbers, can a simple straight-line model (a linear probe) guess which artwork wins?

If it can't, skip the expensive test. In my rebuild it picked the winner 35.5% of the time vs 25% for a coin flip. Modest, but clearly above chance.

**Gate expensive experiments with cheap checks.**

## Search, almost for free

CLIP puts text and images in the same space. So "show artwork that matches what the user searched" is one extra term. Search "an explosion" and explosions come up.

![Query-aware ranking](https://raw.githubusercontent.com/ZFayFay/netflix-maps-replication/main/figures/fig4_query.png)

No new model. Same stored numbers, one more comparison.

## What to remember

1. **Cold-start is an information problem.** The information was in the image all along. The old model just couldn't see it.
2. **Test combinations, not just parts.** Each idea alone: noise. Together: +3% to +20%.
3. **Log your odds.** A small random slice buys you honest offline numbers.
4. **Check cheap before you test expensive.**
5. **Center your embeddings.**

The obvious next step is video. Netflix's newer model adds audio and captions to the image signal. There's no public data like it, so my rebuild stops at posters.

---

*Notebook and code: [github.com/ZFayFay/netflix-maps-replication](https://github.com/ZFayFay/netflix-maps-replication). All ideas belong to the Netflix authors. This is an independent rebuild of their [MAPS post](https://netflixtechblog.com/maps-netflixs-multimodal-asset-personalization-at-scale-32f96320785e).*
