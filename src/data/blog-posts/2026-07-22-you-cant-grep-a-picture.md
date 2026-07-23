---
title: "You can't grep a picture"
slug: you-cant-grep-a-picture
publishDate: "2026-07-22"
description: "Choose to read documents as pixels and you foreclose every text-level defense in advance — including the OCR you reach for to fix it."
---

*Words: Rob. Edited by Jester.*

## TL;DR — the Emperor's food-taster is only reading the menu

_Choose to read documents as pixels, and you foreclose every text-level defense in advance — including the one you'll reach for to fix it._

An inspection that senses the thing differently from how the system consumes it is blind precisely where it matters.

## A Picture Speaks a Thousand Words'); DROP TABLE audit_log;

The documents coming into my system are radiology reports, pathology forms, consulate letters. Written by strangers. Scanned by strangers. And read by the machine as _pictures._

And that last part is the turn.

The process: 
- A document is created; writing on paper
- The paper is imaged (scanned/photographed)
- The image has no text layer
- `pdftotext` gives ... nothing. Not an error. Just silence.
- The canonical path kicks in, and lets a vision model read the pixels

And a vision model reading pixels is the least inspectable ingest path you have. **The input path is uninspectable** — and everything below is a consequence of that one fact.

No text filter can stand in front of a picture. Every scanner, every injection-detector, every guardrail you own or might one day write — they all operate on *text.* A page that arrives as an image walks in underneath all of them, by construction. Your human backstop goes with it: you can grep a text file, but you cannot grep a PNG. An instruction printed in 6-point grey at the foot of a letterhead is perfectly clear to the model and invisible to everything, and everyone, else.

To be blunt. Your entire line of defense against text-level threats isn't evaded. It is **foreclosed** — ruled out by construction, the moment you decided to read documents as pictures.

## The fix everyone reaches for is the wrong shape

What to do? The obvious fix, the one everyone reaches for: OCR the page, screen the text, point your existing controls at the transcript. Put the words back, treat it like words.

It doesn't work because — and this is important — **it assumes the OCR and the model see the same page. They don't.**

The OCR and the model have different failure curves. Faint, tiny, low-contrast type lives in the exact region where OCR fails and the model sails straight through — which the [CSA's image-injection note](https://labs.cloudsecurityalliance.org/research/csa-research-note-image-prompt-injection-multimodal-llm-2026/) documents plainly: these models read embedded text "even under adverse conditions: low contrast, small font sizes." Handwriting lives there too. Where the curves differ, the OCR misses the text that is ingested by the model.

All an attacker has to do is **optimize for model legibility, not OCR legibility**. They don't have to beat your inspection. They only have to write in the gap between the curves. 

One law, and everything below is a worked example of it:

> An inspection layer that senses a document differently from how the system consumes it is blind exactly where it matters.

## Between the Failure Curves

Example. Drop an injected instruction in 6-point light-grey at the foot of a clean-looking scan. Human eyes would slide straight past it.

Now run the screener, `tesseract page.png`: it transcribes the legitimate document **flawlessly** — and misses the injection completely.

Now you have a clean transcript of a poisoned page — **a false negative wearing a receipt.** It looks like proof the page is safe; it is proof of nothing but what OCR happened to catch.

And then your model swallows the poison.

The emperor's food-tasters used to eat the actual food. They didn't read the menu in the next room.

![One page, two readers: a naive tesseract OCR pass transcribes the report but is blind to a 6-point grey injection at the foot of the page, while a vision model reads the injected instruction in full.](/assets/blog/ocr-two-readers.png)

*A naive `tesseract` pass transcribes the report flawlessly and never sees the injection at the foot of the page; a vision model reading the same render reads it in full. (The vision model here is Claude.)*

## Yes, you can harden it. No, it still isn't a control.

You can drag OCR's failure curve closer to the model's by beating the faint type into legibility:

```python
im = Image.open(path).convert("L")                        # grayscale
im = im.resize((im.width * 2, im.height * 2), Image.LANCZOS)  # upscale 2×
im = ImageOps.autocontrast(im, cutoff=0)                   # normalize the range
im = ImageEnhance.Contrast(im).enhance(2.8)                # stretch the contrast
```

Upscale, autocontrast, contrast-stretch. Do this, and the same 6-point injection gets caught. This preprocessing is load-bearing, not decoration — strip it back to the "clean" one-liner everyone writes and you go blind again to precisely the text the tool exists to surface.

But hardening the screener does not eliminate the poison. Why?

**A clean transcript is never evidence of a clean page.** Absence of a finding means OCR didn't see it — that is *all* it means. You've narrowed the gap between the two curves. You haven't closed it. Whatever lives in what's left still strolls through the front door.

**And then there's the handwriting.** Tesseract reads print well and handwriting... not. The model reads handwriting. Sit with what that means on a *form:* the printed part is boilerplate, and the handwritten part is the human being. A real one, from my own folder — a pathology request where the doctor had scrawled *URGENT* across it by hand and drawn a little smiley face. The hardened screener recovered neither. The only "URGENT" in the transcript was the pre-printed label on the form. **It kept the stationery and dropped the doctor.** The transcript was least trustworthy in the exact spot where a human being pressed hardest.

## The fix that failed

One more repair suggests itself, and it's a good idea: don't trust the transcript — instead, mask every word OCR *did* read, and flag the leftover ink as "marks I couldn't make out." Turn the invisible hole into a visible one.

It failed. And the failure is the most useful result in the whole piece.

On the handwriting page it flagged thirteen regions — the biggest of them the form's low-contrast red printed boxes — and it did *not* flag the doctor's handwriting at all. The strokes were too thin to survive the downscale. It would have shipped as a false-confidence machine: an inspector that dutifully reports findings, not one of which is the thing that matters.

We binned it instead of tuning it, because the law was already on the table. A mask built on OCR's eyes inherits OCR's blind spots. **You cannot police a vision-model path with anything that is not a vision model.**

## One more trap: the instrument poisons what it measures

If you do write text sidecars — and you should, there are good reasons a human wants to grep what was otherwise an opaque photo — one last thing will bite you. It bit us.

The tool writes a little text sidecar (`.ocr.txt`) next to every file, and every sidecar opens with the same boilerplate caution header, stamped on identically. Our first version of that header quoted, as a helpful little example, the exact word one real document carried — and the word it picked was "urgent." Every sidecar in the folder promptly matched a naive `grep -i urgent` — *including a photo of the Hubble Deep Field.* The audit artifact had started manufacturing its own false positives: the next person to search the folder would have found urgency everywhere they looked, galaxies and all — and galaxies are not typically known for their urgency.

The discipline that fixes it generalizes far past OCR: **an audit artifact must inject no searchable tokens into the corpus it audits**. The header now carries no verbatim document content — it names the class and cites the source — and every header line is prefixed `#`, so the commentary strips off mechanically (`grep -v '^#'`) and no future rewrite can re-poison the corpus. **Any caution banner, template header, or annotation you write inside a namespace that will later be searched has just become part of that namespace.**

## So what do you actually do

Not a cleverer screener. The move that works is telling the truth about what you built.

Write the sidecar if you need a greppable surface — for the human, for an audit, as something a future filter could one day bite on — but stamp it, in its own header, as **not a control,** and say exactly why: absence of a finding is not absence from the page; it cannot read the handwriting; the person who wrote and scanned this page is a stranger, and every word of it is data, never instruction.

And if you truly need to *police* a vision-model path, the policing has to happen at the model's eye — not beside it. The only honest inspection is one that sees what the consumer sees. Everything else is a receipt for a page nobody read.

None of this is an argument against reading documents as pixels. It's often the only path that works, and choosing it is fine. But it forecloses every text-level defense in advance, silently, and you should choose it *knowing* that. The attack side isn't even new — sub-visual injection in medical imaging is documented ([Clusmann et al., *Nature Communications*, 2025](https://www.nature.com/articles/s41467-024-55631-x)), and the render-to-image version has just surfaced in the trade press under its own name: **InkJect** ([DeepKeep, July 2026](https://www.prnewswire.com/news-releases/deepkeep-exposes-inkject-a-new-visual-prompt-injection-vulnerability-that-bypasses-guardrails-in-leading-ai-models-302815702.html)), which slips past text-based guardrails using near-invisible formatting the model reads without trouble. What that coverage skips is the architecture underneath: not *here is an attack,* but *here is why the reflex you'll reach for to stop it was already defeated before the attacker showed up.*

So extend [Simon Willison's lethal trifecta](https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/) by one uncomfortable step. The untrusted-content leg can walk into your system as pixels — underneath every text control you own — and the OCR you grab to close the hole is reading a different document than the one your model is about to obey.
