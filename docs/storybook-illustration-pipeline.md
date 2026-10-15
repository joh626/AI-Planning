# Storybook Illustration Pipeline: Tenniel ("Alice in Wonderland") Line Style

**Goal:** Produce sets of illustrations for a series of children's novels. The pages should be black-and-white pen-and-ink, in the style of John Tenniel's original *Alice* engravings. The same characters and places must look the same across every chapter and every book. Everything runs locally on the RTX 5060 Ti 8 GB PC (see `local-image-gen-options.md`).

This is a different job from the coloring books:

| | Coloring books | Novel illustrations |
|---|---|---|
| Line style | Thick, uniform, no shading | Fine pen and ink, **crosshatching**, Victorian engraving |
| Consistency problem | Style only | Style **plus recurring characters, plus recurring places** |
| Volume | 30–50 single-subject pages per book | Maybe 10–30 scene illustrations per book, many with 2–3 characters |
| Main risk | Grey fills | A character's face or costume drifting between chapters |

---

## 1. Three layers of consistency, three tools

Think of every illustration as **style + who + where**. Lock each one separately:

| Layer | Tool | Trained on / built from |
|---|---|---|
| **Style** (Tenniel ink) | **Style LoRA** (one for the whole series) | Tenniel's public-domain illustrations |
| **Who** (recurring characters) | **One character LoRA per main character**, with a unique trigger word (e.g. `pipwick_char`) | A character sheet: 15–30 images of that character |
| **Where** (recurring locations or "sets") | Saved **layout references**: a ControlNet depth or line map of each location | A rough sketch or simple 3D block-out of each room or landscape |

### Style: training on Tenniel is legally clean
- John Tenniel died in 1914. His *Alice's Adventures in Wonderland* (1865) and *Through the Looking-Glass* (1871) illustrations are **public domain worldwide**. That's about 92 images, plus hundreds of *Punch* cartoons for more training data.
- Use scans of the **original engravings**, from sources such as the British Library, Internet Archive or Wikimedia Commons. Avoid modern colourised or "restored" editions, which may carry new copyright.
- **Avoid Disney's *Alice* designs entirely.** They are under copyright and are a different look anyway.
- Be careful with other "golden age" illustrators. Arthur Rackham's pre-1929 work is generally fine. E.H. Shepard's *Pooh* is public domain in the US but **still under copyright in the UK until 2047**.
- Caption the training images by **content** ("a girl talking to a caterpillar on a mushroom") so the LoRA learns the ink style, not Alice herself. Otherwise every girl comes out as Alice.

### Who: building a character from nothing to a LoRA
1. Design the character with the style LoRA alone, and generate until one image is "the one".
2. Make a **character sheet** from it: front, 3/4, side and back views, plus a few expressions and poses. Use ControlNet OpenPose for the poses and inpainting to fix details.
3. Hand-pick 15–30 consistent images and fix inconsistencies (buttons, hat, hair) by hand in Krita.
4. Train a **character LoRA** with a unique trigger word.
5. Write a **character card** listing the fixed attributes: costume, props, proportions and "never" rules (e.g. "never without the pocket watch").

### Where: keeping recurring locations consistent
- For each recurring location (the cottage kitchen, the clock tower), sketch it once or block it out in **Blender** or another free 3D tool.
- Render a **depth map and a line map** from the 2–4 camera angles you'll reuse, and save them as the location's set.
- Every scene in that location uses the same depth/line ControlNet, so walls, windows and furniture stay put.

---

## 2. Scenes with several characters

This is the hardest part with any model, and SDXL tends to blend two character LoRAs together. Approaches, from simplest:

1. **Generate one character, then inpaint the second** into a masked area with only that character's LoRA active. This is the most reliable method on 8 GB.
2. **Regional prompting with masked LoRAs** in ComfyUI: character A's LoRA applies only to the left mask, character B's only to the right.
3. **Pose first:** use an OpenPose skeleton from a rough sketch so each figure's position is fixed before style and identity are applied.
4. (Cloud only) **FLUX.1 Kontext** or **Qwen-Image-Edit** with reference images: "put this character into this scene". These are good for hard cases but too heavy for this PC.

---

## 3. Getting real crosshatching

Tenniel's look depends on fine, consistent hatching, and models tend to turn it into grey mush at low resolution.

- Generate the composition at SDXL's native ~1024 px.
- Do a **second "detail" pass** with a 1.5–2× upscale, tiled diffusion or ControlNet Tile, and the style LoRA at low denoise (0.3–0.45). This is where the hatching lines resolve.
- Post-process to **pure black ink on white**: threshold, despeckle, and optionally potrace to vector. This is the same script as for the coloring books, with thinner-line settings.
- Optionally add a slight paper tone and line jitter to sell the "engraved" look. Keep it subtle and consistent across books.

---

## 4. Managing illustration sets across books

The production side is what keeps 4–8 books consistent: everything is written down and repeatable.

### Folder structure
```
series/
  bible/
    style/            # style LoRA, golden reference images, prompt template
    characters/
      pipwick/        # character sheet, LoRA, character card (YAML)
    locations/
      clock-tower/    # depth/line maps per camera angle, reference render
  book-01/
    manifest.csv      # one row per illustration (see below)
    sketches/         # rough composition/pose sketches
    output/raw/
    output/final/     # cleaned, print-ready
  book-02/ ...
```

### The illustration manifest (one row per picture)
| id | chapter | caption / moment in text | characters | location + angle | pose sketch | prompt extras | seed | status |
|---|---|---|---|---|---|---|---|---|
| b01-c03-01 | 3 | Pip climbs the clock face | pipwick | clock-tower / A | sk_0301.png | moonlight, wind | 81231 | approved |

A small script can read this manifest and send each row to **ComfyUI's API**. For each row it loads the style LoRA, the listed character LoRAs, the location's ControlNet map and the pose sketch, then batch-generates 4–8 options. You only pick winners and do touch-ups.

### Version pinning
- Record **which LoRA versions** each book used (e.g. `style_v3`, `pipwick_v2`). Re-training a LoRA mid-series is the most common cause of drift.
- Keep the ComfyUI workflow JSON for each book in the repo.

---

## 5. What this PC can and can't do

| Task | RTX 5060 Ti 8 GB / 16 GB RAM |
|---|---|
| SDXL + style LoRA + 1–2 character LoRAs + ControlNet | ✅ Fine in ComfyUI. |
| Hatching detail pass (tiled upscale) | ✅ Slower, but fine; tiling keeps memory low. |
| Inpainting the second character | ✅ |
| Training **one** SDXL LoRA | ⚠️ Possible overnight with low-VRAM settings. |
| Training the style LoRA plus 4–10 character LoRAs | 💡 **Rent a cloud GPU.** It costs a few dollars per LoRA and takes an hour each, rather than many nights at home. |
| FLUX Kontext / Qwen-Image-Edit for multi-character edits | ❌ Rent a cloud GPU for the occasional hard scene. |

32 GB of system RAM helps even more here, because several LoRAs and ControlNets are loaded at once.

---

## 6. Publishing notes
- Tenniel's *style* and his public-domain images are fine to learn from. Your **characters are your own designs**, so keep the character sheets and design history to show authorship.
- In the US, purely AI-generated images generally can't be registered for copyright. Human selection, arrangement and hand edits strengthen your position, and the **text of the novels is fully yours**.
- Amazon KDP requires disclosure of AI-generated images.

---

## Suggested next steps
1. Collect the Tenniel training set (about 92 *Alice* illustrations plus a selection of *Punch* cartoons) and caption it.
2. Train the **style LoRA** (cloud GPU recommended), then test it on generic scenes.
3. Design the main cast. Build a character sheet, then a LoRA, then a card for each character.
4. Block out the 3–5 most-used locations.
5. Build the **manifest-to-ComfyUI batch script**, then pilot it on one chapter.
6. Lock all versions, then produce book 1.
