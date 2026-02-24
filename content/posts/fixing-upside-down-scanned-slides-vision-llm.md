---
title: "Fixing 10,000 Upside-Down Scanned Slides with a Local Vision LLM"
date: 2026-02-23
draft: false
tags: ["python", "ollama", "llm", "vision", "photography", "opencv"]
---

My grandfather had about 10,000 35mm slides. I rented a [SlideSnap X1](https://www.slidesnap.com/) and spent a weekend feeding them through — 33 boxes worth, organized into folders by box. The scanner itself was great, but when you're pushing through thousands of slides in a weekend, some inevitably go in upside down or backwards. No metadata, no EXIF orientation flags — just thousands of JPEGs, some right-side up, some not, sitting on a NAS.

Manually reviewing 10,000 images isn't realistic. But a vision LLM running on local hardware can look at each one and answer a simple question: is this upside down?

## The problem

Scanned slides can be wrong in four orientations:

| Orientation | What happened |
|-------------|---------------|
| Correct | Slide was loaded properly |
| Upside down (180°) | Slide was inserted flipped vertically |
| Mirrored | Slide was scanned from the emulsion side |
| Mirrored + upside down | Both problems at once |

Upside-down images are by far the most common issue. Mirror flips are harder to detect without readable text in the image, so I focused on rotation first.

## Why not digiKam or OpenCV?

I tried [digiKam](https://www.digikam.org/) first. It has an auto-rotate feature that uses DNN-based orientation detection, and it kind of works — but it felt painfully laggy when processing thousands of images, and the accuracy wasn't great. It would confidently leave obviously upside-down photos untouched. For a few hundred photos it's probably fine, but at this scale I needed something I could script, run unattended, and verify afterwards.

Pure classical computer vision doesn't help much either. You could try detecting faces and checking if they're inverted, but many of these photos are landscapes, buildings, or candid shots without clear faces. Edge detection and gradient analysis can suggest a dominant "up" direction, but they're unreliable on photos with ambiguous composition — a snow-covered mountain reflected in a lake, for example.

The task requires the kind of semantic understanding that a human uses: sky goes up, people stand on their feet, buildings point upward, text reads left-to-right. A vision language model does exactly this.

## The setup

I already had [Ollama](https://ollama.ai) running on a Mac Studio (M1 Ultra, 64 GB RAM) for [Frigate's](https://frigate.video) camera event descriptions. The Mac Studio sits on the same network as the NAS, so the pipeline is:

1. Read image from NAS (SMB mount)
2. Send to Ollama's vision model via HTTP API
3. Parse the response
4. Record the result

For the model, I used `llama3.2-vision` (11B parameters). It fits comfortably in 64 GB of RAM and processes each image in about 15 seconds. The smaller `minicpm-v` was faster but significantly less accurate.

## What didn't work: the grid approach

My first attempt was clever but wrong. I created a 2x2 grid showing all four possible orientations of each image (original, mirrored, rotated 180°, mirrored+rotated), labeled A through D, and asked the model to pick which one looked correct.

```text
┌─────────┬─────────┐
│ A: orig │ B: mirr │
├─────────┼─────────┤
│ C: 180° │ D: both │
└─────────┴─────────┘
```

The model said "A" for every single image. The grid made each sub-image too small to reason about, and the model defaulted to the first option. Five for five wrong.

## What worked: one question at a time

Splitting the problem into a simple binary question on the full-resolution image worked much better. The prompt:

```text
Look at this scanned photograph. Is it UPSIDE DOWN?

Check these things:
- Are people's heads at the BOTTOM of the image? That means upside down.
- Is the sky or ceiling at the BOTTOM? That means upside down.
- Are buildings, trees, or poles pointing DOWNWARD? That means upside down.
- Is the ground or floor at the TOP of the image? That means upside down.

Respond in EXACTLY this format (two lines, nothing else):
UPSIDE_DOWN: YES or NO
REASON: Brief explanation
```

On a test set of 5 images (3 upside-down, 2 correct), this got all 5 right. The key insight: give the model the full image at a reasonable resolution, ask one simple question, and tell it exactly what format to respond in.

## Blur detection for free

While I was processing each image, I also wanted to flag blurry scans that might need to be re-done. This part doesn't need an LLM at all — the [Laplacian variance](https://docs.opencv.org/4.x/d5/db5/tutorial_laplacian.html) method works well:

```python
def calculate_blur_score(image_path):
    img = cv2.imread(str(image_path), cv2.IMREAD_GRAYSCALE)
    # Normalize for different scan resolutions
    h, w = img.shape
    if max(h, w) > 1024:
        scale = 1024 / max(h, w)
        img = cv2.resize(img, (int(w * scale), int(h * scale)))
    return float(cv2.Laplacian(img, cv2.CV_64F).var())
```

The Laplacian operator approximates the second derivative of the image — sharp edges produce high values, blurry regions produce low values. The variance of the Laplacian across the whole image gives a single sharpness score. Higher is sharper.

A threshold of 100 worked well for these scans. Anything below that was visibly soft.

## Sampling before committing

Running 10,000 images through a vision model at 15 seconds each would take about 42 hours. Before committing to that, I sampled 3 random images from each of the 33 folders to estimate which ones had problems:

```text
Folder                     Total  Sampled  Upside↓      %
------------------------- ------ -------- -------- ------
Batch 01                      81        3        2    67% <<<
Batch 02                      70        3        2    67% <<<
Batch 03                      79        3        2    67% <<<
Batch 04                      81        3        2    67% <<<
Batch 05                      78        3        1    33%
Batch 06                     541        3        1    33%
Batch 07                     441        3        1    33%
Batch 08                     420        3        0     0%
Batch 09                    1305        3        0     0%
Batch 10                    1085        3        0     0%
Batch 11                    1373        3        0     0%
...
```

99 sample images, 13 minutes, and I had a priority map. Four folders had 67% upside-down rates — those ~311 images should be processed first. The large folders (4,000+ images) looked clean. The sampling pass likely saved 30+ hours of unnecessary processing.

## The script

The full script is about 400 lines of Python. The core loop is straightforward:

```python
for img_path in image_files:
    # Blur score (instant, no LLM needed)
    blur_score = calculate_blur_score(img_path)

    # Orientation check (sends image to Ollama)
    img_b64 = image_to_b64(img_path, max_size=1024)
    text = ask_vision(ollama_url, model, UPSIDE_DOWN_PROMPT, img_b64)
    is_upside_down = parse_response(text)

    results.append(ImageResult(
        path=str(img_path),
        blur_score=blur_score,
        orientation="C" if is_upside_down else "A",
        correction="rotate_180" if is_upside_down else "none",
    ))
```

It generates an HTML report with side-by-side thumbnails (as-scanned vs. corrected) so you can visually verify before applying fixes. The fixes themselves are non-destructive — each corrected image gets a `.original` backup:

```bash
# Review first
python slide_analyzer.py "/Volumes/slides/batch-01"

# Apply corrections from the report
python slide_analyzer.py --fix-from slide_report.json
```

## Performance

| Metric | Value |
|--------|-------|
| Model | `llama3.2-vision` (11B) |
| Hardware | Mac Studio M1 Ultra, 64 GB |
| Time per image | ~15 seconds |
| Accuracy (test set) | 5/5 |
| Sample pass (99 images) | 13 minutes |
| Estimated full run (10,000 images) | ~42 hours |
| Estimated full run (priority folders only) | ~1.3 hours |

## What I'd do differently

**Mirror detection is hard.** I initially tried to detect horizontal flips too, but the model was unreliable — it flagged correctly-oriented images as mirrored. Without readable text in the photo, there often aren't enough visual cues. Mirror detection probably needs a more capable model or a different approach (OCR on the image, then check if the text is backwards).

**A larger model might help.** The 11B `llama3.2-vision` worked well for upside-down detection, which is a relatively easy spatial reasoning task. Mirror detection is subtler and might benefit from a 34B+ parameter model. With 64 GB of RAM, `llava:34b` would fit, but I haven't tested it yet.

**Batch the Ollama calls.** The current script processes images sequentially because Ollama handles one inference at a time on a single GPU. If you had multiple GPUs or were using a cloud API, you could parallelize this significantly.

## Conclusion

The combination of "cheap classical CV for the easy stuff" (blur detection) and "local vision LLM for the semantic stuff" (orientation detection) turned a multi-day manual review into something that runs overnight. The sampling strategy — check a few images per folder before processing everything — is the kind of optimization that seems obvious in retrospect but saves enormous amounts of time.

The script, the Ollama model, and the NAS are all local. No images leave the network, no API costs, no rate limits. That matters when the photos are your grandfather's personal history.
