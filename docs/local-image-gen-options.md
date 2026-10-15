# Local Image Generation: Options for Kids' Coloring Books

**Goal:** Run image generation locally, without per-image credits, using a workflow that feels like Leonardo or Krea. The setup has to hold one consistent art style across 4–8 children's coloring books: simple outlines, thick clean lines, large white areas and no shading.

_Last reviewed: 2026-10. This space changes monthly, so check model licences and versions before you commit._

---

## TL;DR recommendation

| Layer | Pick | Why |
|---|---|---|
| **Main app (Leonardo-like)** | **InvokeAI** (free Community Edition) | Canvas, inpainting/outpainting, boards/galleries, style reference and a model manager. Its UX is the closest local match to Leonardo. |
| **Batch / pipeline app (Krea-like)** | **ComfyUI** (or **SwarmUI**, a simpler front-end on top of ComfyUI) | Saved workflows that turn out 40 pages per book unattended. New models arrive here first. |
| **Base model** | **SDXL line-art fine-tune** to start; try **FLUX.1 [dev]** / **FLUX.1 Krea [dev]** if your GPU allows | SDXL is light, has a huge line-art LoRA ecosystem and allows commercial use. FLUX follows prompts better. |
| **Style consistency** | **Train your own "house style" LoRA** | This is the single biggest lever for a consistent series. Prompting alone drifts. |
| **Composition control** | ControlNet (lineart / scribble / canny) + IP-Adapter style reference | Sketch a layout and the model draws it in your style. |
| **Recurring characters** | Character LoRA, or FLUX.1 Kontext [dev] for edits | The same mascot can appear on every page. |
| **Post-processing** | Python script: threshold to pure B/W → despeckle → optional line thickening → 300 DPI print size | AI output always has some grey. Coloring pages need pure black on white. |

---

## 1. Front-end apps (the "Leonardo / Krea" layer)

| App | Feels like | Strengths | Weaknesses | Best for |
|---|---|---|---|---|
| **InvokeAI** | Leonardo (canvas + gallery) | Polished unified canvas, layers, regional prompts, inpainting, style/character reference, boards, workflow editor, one-click installer | Runs FLUX slower than ComfyUI; gets new models later | Day-to-day page design and touch-ups |
| **ComfyUI** | Krea's real-time / power tools | Fastest; supports every model; reusable node workflows; batch queue; API for scripting; real-time preview with turbo models | Node graphs take time to learn | Book-scale batch production and automation |
| **SwarmUI** | Simple UI over ComfyUI | Easy "generate" tab, with the full ComfyUI back end when you need it | Smaller community than the other two | A middle ground if ComfyUI feels intimidating |
| **Forge (SD WebUI Forge)** | Classic A1111 | Easy, very low-VRAM friendly, large extension library | Dated UX; slower development | Older or smaller GPUs (8 GB) |
| **Krita + AI Diffusion plugin** | Krea's live canvas | Draw-and-generate in a real paint app, with ComfyUI under the hood | Needs a ComfyUI back end | Hand-sketching layouts and cleaning lines |

Fooocus and A1111 are effectively in maintenance mode, so skip them.

**Suggested combo:** Use InvokeAI (or Krita + AI Diffusion) to explore and fix individual pages, and ComfyUI for the repeatable "make the book" workflow. All of these can share one models folder.

---

## 2. Base models

Coloring pages are a forgiving use case: no photorealism, no colour, simple shapes. You don't need the biggest model. You need one that follows "clean line art, no shading" reliably and that can be trained.

| Model | VRAM (practical) | Licence for selling output | Notes for coloring books |
|---|---|---|---|
| **SDXL 1.0 + line-art fine-tunes / LoRAs** | 8–12 GB | ✅ OpenRAIL++ (commercial OK) | Many existing "coloring book" and "line art" LoRAs on Civitai/Hugging Face. Cheapest to train your own LoRA. **Best starting point.** |
| **FLUX.1 [dev]** | 12–16 GB (fp8/GGUF), 24 GB full | ⚠️ Non-commercial *model* licence, but the licence says **outputs** may be used commercially. Read it yourself. | Excellent prompt adherence and clean lines. LoRA training needs about 16–24 GB. |
| **FLUX.1 Krea [dev]** | same as FLUX.1 dev | same as FLUX.1 dev | Krea's own open-weight model, and the closest thing to "Krea, locally". Aesthetic-tuned. |
| **FLUX.1 Kontext [dev]** | 12–24 GB | same as FLUX.1 dev | Instruction-based editing ("same bunny, now at the beach"), useful for keeping a character consistent. |
| **FLUX.2 [dev]** | 24 GB+ (heavily quantised below that) | Non-commercial model licence. Check the output terms. | Multi-reference consistency built in, but heavy hardware. |
| **Qwen-Image / Qwen-Image-Edit** | 16–24 GB (quantised) | ✅ Apache 2.0 | Strong prompt following and text rendering (for title pages and "Color the letter A!"). Permissive licence. |
| **Z-Image Turbo** | ~12–16 GB | ✅ Apache 2.0 | Fast few-step model with a permissive licence. Good for quick drafts. |

> **Licence note:** For a commercial KDP/Etsy product, the Apache 2.0 and SDXL OpenRAIL++ models carry the least licence risk. The FLUX [dev] family generally allows commercial use of *outputs*, but restricts the model weights. Confirm the current terms on each model card before you publish.

---

## 3. Getting a consistent style across 4–8 books

This is where local generation beats Leonardo or Krea on cost: you can train and iterate as much as you like for free.

1. **Lock a "style bible" first.**
   Pick 20–40 example pages that look exactly right. Generate them, hand-fix them, or draw them. The style rules are:
   - Thick, uniform black outlines (for example, the look of a 2–3 pt line at print size)
   - No shading, no hatching, no grey, no filled black areas
   - Large closed shapes (age 3–6) or moderate detail (age 6–10)
   - A plain or very simple background, with plenty of white space
   - One subject per page, centred, with a margin from the edges
2. **Train a style LoRA** on that set.
   - Tools: **OneTrainer**, **kohya_ss**, or **ai-toolkit** (FLUX).
   - Typical run: SDXL takes 30–60 min on a 12–16 GB card. FLUX needs 16–24 GB and longer.
   - Caption the *content* ("a bunny holding a carrot"), not the style, so the LoRA absorbs the style.
   - Consider one **house-style LoRA** shared by all books, plus an optional **character LoRA** per book series.
3. **Freeze a prompt template**, and change only the subject:
   ```
   <lora:myhouse_style:0.8> coloring book page, clean black line art, thick outlines,
   simple shapes, white background, no shading, {SUBJECT}, centered, kids illustration
   negative: shading, gradient, grey, hatching, texture, color, photo, text, watermark, cropped
   ```
4. **Control composition** with ControlNet (lineart, scribble or canny) on rough sketches, or on simple 3D/clip-art layouts. Use an **IP-Adapter / style reference** that points at 1–2 "golden" pages.
5. **Fix seeds and settings per book.** Save the ComfyUI workflow JSON in this repo for each book, so any page can be regenerated exactly.
6. **Keep a reject checklist:** grey fills, broken (non-closed) shapes, tiny details, extra fingers, anything scary or ambiguous for kids.

---

## 4. Post-processing pipeline (print-ready pages)

AI line art is never pure black and white. A small script should:

1. Convert to greyscale, then **threshold** to pure B/W (Otsu or a fixed ~200 level).
2. **Despeckle** by removing tiny specks or islands.
3. Optionally **thicken lines** (morphological dilate) for younger ages.
4. Optionally **vectorise** with `potrace` for perfectly crisp, scalable lines.
5. Place the image on a **print canvas**, for example 8.5×11 in @ 300 DPI (2550×3300 px), with safe margins and bleed if needed.
6. Optionally add a blank back page (single-sided pages stop markers bleeding through).
7. Export the pages to a single interior PDF.

This is easy to automate with Python (Pillow, OpenCV, potrace) and can be added to this repo as `tools/`.

---

## 5. Hardware tiers

| Budget | GPU | What runs well |
|---|---|---|
| Entry | 8 GB (RTX 3060 Ti/4060) | SDXL in Forge/ComfyUI, SDXL LoRA training (slow), FLUX only at heavy quantisation |
| **Sweet spot** | **12–16 GB (RTX 3060 12GB, 4060 Ti 16GB, 4070 Ti Super, 5070 Ti)** | SDXL comfortably, FLUX / Z-Image quantised, SDXL LoRA training |
| Comfortable | 24–32 GB (RTX 3090 / 4090 / 5090) | Everything, including FLUX/Qwen LoRA training and FLUX.2 |
| Mac | Apple Silicon, 32 GB+ unified memory | Works (Draw Things, ComfyUI, InvokeAI), but slower. Draw Things is the most Mac-native. |

Also plan for 32 GB system RAM and 200–500 GB free SSD space for models.

**No suitable GPU?** Rent one by the hour (RunPod, Vast.ai) and run the same ComfyUI workflows. It costs a few dollars per book and is still far cheaper than credits.

---

### Our machine: HP OMEN 16L (RTX 5060 Ti 8 GB, i5-14400F, 16 GB DDR5, 1 TB SSD)

The 8 GB of GPU memory (VRAM) is the deciding limit. Plan around **SDXL**.

| Task | Verdict on this PC |
|---|---|
| SDXL + line-art LoRAs + ControlNet at 1024×1024 | ✅ **Primary path.** About 5–15 s per image; batches of 4 are fine. |
| SD 1.5 line-art models | ✅ Very fast. Useful for quick drafts. |
| Z-Image Turbo (quantised GGUF/fp8) | ⚠️ Possible in ComfyUI. Worth testing as an upgrade. |
| FLUX.1 dev / Krea dev / Kontext (GGUF Q4–Q5) | ⚠️ Runs in ComfyUI, but slowly (around 1 min or more per image). With 16 GB of system RAM it will swap heavily. |
| Qwen-Image, FLUX.2 | ❌ Too large for this PC. Rent a cloud GPU instead. |
| Train an SDXL style LoRA | ⚠️ Possible in OneTrainer/kohya with low-VRAM settings (batch 1, gradient checkpointing, Adafactor/8-bit, fp8 base). Takes about 1–3 h. **Alternative:** rent a 24 GB cloud GPU for about an hour (a few dollars per LoRA). |
| Train a FLUX LoRA | ❌ Use a cloud GPU. |

**Recommended set-up for this PC**
1. **Upgrade system RAM to 32 GB** (2×16 GB DDR5). It's the cheapest upgrade with the biggest effect: it stops model swapping and makes FLUX/Z-Image tests bearable. Check which RAM the HP motherboard supports first.
2. **Front-end:** use **ComfyUI** (desktop or portable) as the engine. It handles 8 GB best. Add **Krita + AI Diffusion** on top for a Krea-like "sketch and generate" canvas. **InvokeAI** also runs SDXL on 8 GB, so use it if you prefer Leonardo's feel. **Forge** is the simplest fallback.
3. **RTX 50-series note:** Blackwell cards need **recent builds** (PyTorch with CUDA 12.8+). Use the current installers. Older guides and old one-click packages often fail on 50-series cards with "no kernel image" errors.
4. **Resolution isn't a problem.** Generate at SDXL's native ~1024 px, then upscale 2–3× or vectorise with potrace. Line art vectorises cleanly to any print size, so 8 GB doesn't limit print quality.
5. **Storage:** 1 TB is enough. SDXL checkpoints are about 7 GB each and LoRAs about 50–200 MB. Keep it to 2–3 base models.

---

## 6. Cost comparison (rough)

- **Leonardo / Krea:** subscription plus credits. Iterating on 4–8 books × 30–50 pages × many retries adds up, and LoRA training costs extra.
- **Local:** a one-off GPU (or one you already own), plus electricity (about $0.02–0.05 per hour of generation). Unlimited retries and training.
- **Hybrid:** prototype in Leonardo or Krea if you already pay for them, then produce locally.

---

## 7. Publishing gotchas

- **Amazon KDP requires disclosure** of AI-generated content when you upload.
- **Copyright:** in the US, purely AI-generated images generally cannot be copyrighted. Hand edits, your selection and arrangement, and original text improve your position. Keep your sketches and edit files.
- Avoid prompts that name artists or copyrighted characters.

---

## Suggested next steps

1. ~~Confirm the hardware.~~ Done: RTX 5060 Ti 8 GB with 16 GB RAM (see section 5). Go SDXL-first and consider upgrading to 32 GB RAM.
2. Install **InvokeAI** and **ComfyUI** with a shared models folder, and download an SDXL base model, 1–2 line-art LoRAs, and ControlNet lineart.
3. Generate about 50 test pages, then pick 20–40 "golden" pages to form the style bible.
4. Train the house-style LoRA, then compare it against the off-the-shelf LoRAs.
5. Build the batch ComfyUI workflow and the B/W post-processing script, and save both in this repo.
6. Produce book 1 end-to-end, then reuse the same workflow for books 2–8.
