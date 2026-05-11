# 7.0 Expanded Dataset Model Analysis: What Worked, What Didn't, and Why¶
Insight on:
> *What worked and what didn't? Why did one model outperform another? What are the failure modes? What would you do with more time?

## Loading Saved Metrics¶
Results come from three saved JSON files:

  * `bert-baseline_v1.json` — original BERT, 4 entity types
  * `bert-expanded_v1.json` — BERT with 8 entity types (run 2.1-bert-expanded.ipynb first)
  * `layoutlmv3-lr5e-5_bs2_v1.json` — LayoutLMv3 with 8 entity types

    Loaded files:
      bert-baseline             F1=0.368
      bert-expanded             F1=0.5012
      layoutlmv3-bestmodel      F1=0.782
      layoutlmv3-expanded       F1=0.7094
    

## Overall Model Comparison¶    
    Category                   BERT     LMv3     Diff  Winner
    ----------------------------------------------------------
    HEADER                   0.2821   0.5315  +0.2494  LMv3
    QUESTION                 0.6382   0.8193  +0.1811  LMv3
    ANSWER                   0.5070   0.7422  +0.2352  LMv3
    OTHER-STAMP              0.2759   0.3095  +0.0336  Tie
    OTHER-BATES              0.7885   0.9053  +0.1168  LMv3
    OTHER-LOGO               0.1045   0.3371  +0.2326  LMv3
    OTHER-INSTRUCTION        0.0323   0.2491  +0.2168  LMv3
    

## Interpretation¶

The most obvious takeaway is that LayoutLMv3 beats BERT on almost everything. That is not surprising — LMv3 uses text, layout coordinates, and image patches together, while BERT only sees the words. But the interesting part is how much the gap varies depending on the category.

For the core classes — HEADER, QUESTION, ANSWER — LMv3 wins by a large margin. HEADER is the best example. BERT gets 0.28, LMv3 gets 0.53. Nearly double. This makes sense because headers are often visually distinct — larger font, different positioning — and BERT has no way to see that. It only reads the words, and header words are not always unique enough on their own.

Now the subcategories tell a more nuanced story. OTHER-BATES is the standout. BERT gets 0.788 and LMv3 gets 0.905. The gap is smaller here than anywhere else. The reason is straightforward — Bates numbers are just digit sequences like "000123." You do not need to see the layout or the image to recognize them. The text alone is enough. This is actually a useful finding because it shows that not all subcategories need a multimodal model. Some can be handled with simpler tools.

OTHER-STAMP is the opposite story. Both models struggle — BERT gets 0.276 and LMv3 gets 0.309. Essentially a tie, and both are bad. Stamps are purely visual. They are rubber-stamped marks with no consistent text. Even LMv3's image patches are not enough to reliably detect them. This suggests that document stamps might need a dedicated object detection approach, not a sequence labeling model.

OTHER-LOGO and OTHER-INSTRUCTION follow the same pattern. BERT nearly fails on both (0.104 and 0.032). LMv3 does better but still not well — 0.337 and 0.249. These are genuinely hard categories. Logos are visual and varied by definition. Instructions are long text blocks that look similar to answer fields. There is not enough signal in either text or layout patches alone to distinguish them consistently.

Overall, expanding the OTHER category was the right decision for analysis purposes. It shows clearly that "OTHER" was never one coherent thing — it was a mix of very different elements, some text-detectable, some layout-dependent, and some that current models cannot handle well at all.


## Interpretation¶

The overall F1 drop is expected — and that is okay Going from original to expanded, BERT drops from 0.573 → 0.501, and LMv3 drops from 0.782 → 0.709. Both drop by almost exactly 0.07. This is not a failure. The expanded model is solving a harder problem with more classes and fewer examples per class. The drop is the cost of getting more information.

OTHER collapses to 0.00 in both expanded models This is the most striking number in the plot. In the expanded schema, the residual OTHER category — items that do not fit any subcategory — gets F1=0.00 for both BERT and LMv3. This actually means the subcategory split was well-designed. Most of what used to be "OTHER" got absorbed into the four subcategories, leaving almost no examples for the residual class. The model cannot learn a class with near-zero examples. This is expected, not a bug.

OTHER-BATES is the only subcategory that both models handle well BERT gets 0.79, LMv3 gets 0.91. This is the biggest gap between OTHER-BATES and every other subcategory. The reason is purely textual — Bates numbers are digit sequences like "000234." BERT, which has no layout or image information at all, still gets 0.79. That tells you the text signal alone is enough. LMv3 adds a small bump to 0.91 but the heavy lifting was already done by the words. This is actually a useful finding — for Bates number detection specifically, you do not need a complex multimodal model.

HEADER is the hardest core class, consistently across all four panels BERT original: 0.26. LMv3 original: 0.55. BERT expanded: 0.28. LMv3 expanded: 0.53. It is the lowest core class in every single panel. Headers are tricky because the words inside a header are not always unique — they can look like questions or answers in isolation. What makes a header a header is mostly its visual position and font size. BERT has no access to that. LMv3 does better with layout coordinates, but still struggles compared to QUESTION and ANSWER.

QUESTION is the easiest class for both models BERT original: 0.71. LMv3 original: 0.85. Questions have consistent surface patterns — short phrases, colon endings, specific positional layout. Both models pick this up well. LMv3's layout features push it to 0.85, which is the highest single score in either original panel.

The visual subcategories are hard for everyone OTHER-STAMP: BERT=0.28, LMv3=0.31. OTHER-LOGO: BERT=0.10, LMv3=0.34. OTHER-INSTRUCTION: BERT=0.03, LMv3=0.25. None of these are good numbers. Stamps and logos are purely visual — they have no consistent text, and even LMv3's image patches (14×14 grid over the whole page) are too coarse to reliably detect a small rubber stamp or a logo in the corner. Instructions are long text blocks that look almost identical to answer fields in terms of layout. The model has no reliable signal to separate them.

The multimodal gap is real and consistent Across every category, LMv3 beats BERT. But the size of the gap varies. For HEADER it is 0.29 (original) — the biggest gap. For OTHER-BATES it is only 0.12 — the smallest gap. This pattern makes sense. Categories that rely on visual or positional cues benefit the most from LMv3's extra modalities. Categories with strong text signals barely benefit at all.

What this plot actually tells you Expanding OTHER was the right call for understanding the data. The original OTHER=0.28 (BERT) and 0.52 (LMv3) were hiding a very mixed bag — one subcategory that is nearly trivial to detect (Bates), one that is hard even for the best model (Stamp), and two that are somewhere in between. A single F1 score for "OTHER" was meaningless. The expanded schema makes the difficulty structure visible.

## Subcategory F1 — BERT Expanded vs LayoutLMv3 Expanded¶

The central question: **does layout + vision help for each subcategory differently?**

Expected story:

  * **OTHER-BATES** : sequential digits → strong text signal → BERT should be competitive
  * **OTHER-LOGO** : visual graphic → BERT blind → LMv3 should dominate
  * **OTHER-STAMP** : positional + visual → LMv3 advantage
  * **OTHER-INSTRUCTION** : looks like QUESTION text → both struggle


    Subcategory                BERT     LMv3     Diff Winner
    -------------------------------------------------------
    OTHER-STAMP              0.2759   0.3095  +0.0336  Tie
    OTHER-BATES              0.7885   0.9053  +0.1168  LMv3
    OTHER-LOGO               0.1045   0.3371  +0.2326  LMv3
    OTHER-INSTRUCTION        0.0323   0.2491  +0.2168  LMv3
    

## Interpretation¶

OTHER-BATES: the one that works BERT gets 0.788 and LMv3 gets 0.905. Both are above 0.5. Both are good. The gap between them is only 0.117 — the smallest gap across all four subcategories. This is the key insight. Bates numbers are digit-only strings. You do not need to see the page layout or the image to find them. The text alone is a strong enough signal. BERT, which sees nothing but words, already gets 0.788. LMv3 adds layout and image and gets 0.905, but BERT was already doing most of the work. If your only goal was to detect Bates numbers in legal documents, you could use a much simpler model and still get reasonable results.

OTHER-STAMP: both models are stuck BERT gets 0.276, LMv3 gets 0.309. The gap is only 0.033 — the smallest in the entire chart. But unlike OTHER-BATES where a small gap means both are good, here a small gap means both are equally bad. Stamps are rubber-stamped images — they often contain text like "RECEIVED" or "CONFIDENTIAL," but the text is inconsistent across documents and the visual appearance varies heavily by ink, rotation, and degradation. LMv3's image patches cover a 14×14 grid over the whole page, which is too coarse to reliably detect a stamp in one corner. Sequence labeling models in general are not designed for this. Object detection would be a more appropriate tool.

OTHER-LOGO: BERT almost completely fails BERT gets 0.104. LMv3 gets 0.337. The gap here is 0.233 — one of the largest in the chart. BERT's near-failure makes sense. Logos rarely have useful text inside them, and even when they do, the words are company names or taglines that appear in various contexts. Without seeing where on the page something is or what it looks like, BERT has almost no signal to work with. LMv3 does better because layout coordinates and image patches give some positional and visual cues. But 0.337 is still not a usable number. Logo detection in the general case likely needs a dedicated visual model fine-tuned on logo images.

OTHER-INSTRUCTION: BERT's biggest failure BERT gets 0.032. That is essentially zero. LMv3 gets 0.249, which is better, but still not good. Instruction blocks are long paragraphs of text that explain how to fill out a form. The problem is that their text content overlaps heavily with answer fields — both are long, both can use similar vocabulary. What makes an instruction an instruction is its position on the form, its formatting, and its context relative to other elements. BERT cannot access any of that. LMv3 gets some of the positional signal from bounding boxes and reaches 0.249, but it is still below any practical threshold. This subcategory might need document-level context, not just token-level features.

The bigger picture The four subcategories are ordered by how much the multimodal advantage matters. For OTHER-BATES, text is enough — the gap is small. For OTHER-STAMP, neither modality helps enough — the gap is also small but for the opposite reason. For OTHER-LOGO and OTHER-INSTRUCTION, layout and image features matter a lot — the gaps are large. This pattern shows that "OTHER" was never one thing. It was four different detection problems bundled into one label, each requiring different information to solve.

## Summary Table¶
    
    
    ================================================================================
      FORMVISION — Full Model Comparison
    ================================================================================
    Model                             Overall   HEADER   QUESTION   ANSWER    OTHER
    --------------------------------------------------------------------------------
      Simple rule-based (1.0)          0.0370   0.0490     0.0470   0.0000   0.0370
      BERT baseline (2.0)              0.3680   0.0000     0.4592   0.3776   0.0250
      BERT expanded (2.1)              0.5012   0.2821     0.6382   0.5070   0.0000
      LayoutLMv3 original (4.0)        0.7820   0.5477     0.8542   0.8069   0.5204
      LayoutLMv3 expanded (4.1)        0.7094   0.5315     0.8193   0.7422   0.0000
    ================================================================================
    
    OTHER subcategory breakdown (expanded models only):
    Subcategory              BERT exp   LMv3 exp Advantage
    -------------------------------------------------------
      OTHER-BATES              0.7885     0.9053  LMv3
      OTHER-STAMP              0.2759     0.3095  Tie
      OTHER-LOGO               0.1045     0.3371  LMv3
      OTHER-INSTRUCTION        0.0323     0.2491  LMv3
    

## Conclusion¶

  * Simple rule-based gets 0.037 overall. That number grounds everything. Without it, the reader does not know where you started. With it, the jump to LMv3's 0.782 becomes meaningful.

  * BERT baseline gets 0.368 with HEADER=0.000. This is a critical data point. The baseline BERT completely fails on headers. The expanded BERT (2.1) gets HEADER=0.282. That improvement came just from restructuring the labels, not changing the model at all.

  * LMv3 expanded drops to 0.709 from 0.782. Without this table, a reader might wonder if the expanded model is just worse. This table makes it clear that LMv3 original was the peak overall F1, and the exp



