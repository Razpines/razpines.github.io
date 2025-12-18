---
layout: post
title: "I Built a Fully Automated Reels Factory"
description: "A practical, end-to-end experiment in automating short-form storytelling videos - and the real-world friction points you hit when you try to auto-publish."
date: 2025-06-21
tags: [automation, video, ffmpeg, llm, captions, instagram]
---

Short-form “storytime” reels have a very consistent shape: a first-person narrative read aloud over a loopable background clip, with large captions tuned for mobile viewing. The reason this format performs is not that any one video is magical - it’s that the production recipe is repeatable.

This post is a technical write-up of a small factory I built: a pipeline that starts from Reddit posts and produces ready-to-upload 9:16 MP4s (voiceover, styled captions, cropped background video, and mixed audio). It's intentionally slightly technical: enough detail for an engineer to understand the moving parts, but readable for non-engineers skimming for engineering judgment and tradeoffs.

---

## Example output

<video class="demo-video" controls playsinline preload="metadata">
  <source src="{{ '/assets/media/reels-factory/example.mp4' | relative_url }}" type="video/mp4" />
  Your browser doesn't support embedded videos. You can
  <a href="{{ '/assets/media/reels-factory/example.mp4' | relative_url }}">download the example video</a>.
</video>

---

## Goals and constraints

I wasn’t trying to generate a single impressive video by hand. The goal was an automated pipeline that:

- Takes messy human text as input
- Produces platform-ready vertical videos as output
- Can run in batch mode (overnight, or while I’m doing other work)
- Is easy to debug by inspecting intermediate artifacts
- Can optionally publish via the Instagram Graph API (but doesn’t depend on it)

At a high level, the stages are:

1. Find story candidates (Reddit)
2. Filter garbage and unsafe content
3. Rewrite into a spoken ~60-second script + hashtags (used when publishing via the Instagram Graph API)
4. Generate voice (TTS)
5. Generate timed captions
6. Render 9:16 video with burnt-in subs
7. Publish to Instagram (Graph API)

Runtime note: on my setup (RTX 3080 Ti, 12GB VRAM), I can generate about 20 reels in about 30 minutes end-to-end.

---

## Architecture

The pipeline is modular: each stage writes artifacts that the next stage consumes. That makes it easier to debug and cheaper to iterate - if something breaks, I can rerun only the failing stage instead of starting from scratch.

```
scrape -> rewrite -> tts+captions -> render -> publish
```

This separation matters because different stages fail for different reasons:

- Scraping is rate-limited and requires credentials.
- Rewriting is probabilistic and can produce unusable output.
- TTS and subtitle generation have edge cases around punctuation, profanity, and timing.
- Video rendering is deterministic but sensitive to codecs, filters, and media quirks.

---

## Content sourcing and caching (Reddit)

Reddit is a convenient source of story candidates because the posts already contain the core ingredients of a “storytime” format: first-person narration, clear conflict, and a built-in quality signal (votes and comments).

In practice, raw posts are not “ready”:

- Way too long
- Inconsistent formatting
- Often include content you wouldn’t want to display in captions verbatim

One design decision that paid off quickly was caching scraped data locally. Scraping is “low-variance work”: once you’ve fetched posts for a time window, re-fetching them usually doesn’t change your results much, but it does cost time and adds failure modes (rate limits, networking, auth).

So the scraper writes a local dataset (parquet) with the post text plus metadata and toxicity scores. That lets you iterate on rewriting, TTS, captions, and rendering without constantly re-scraping.

If you're curious about Reddit API setup, start by creating a Reddit app for credentials in [preferences/apps](https://www.reddit.com/prefs/apps), then use [PRAW](https://praw.readthedocs.io/en/stable/) as the client and keep the [Reddit API reference](https://www.reddit.com/dev/api/) nearby for edge cases.

---

## Filtering and content safety (Detoxify)

If you automate content generation at scale, filtering is not optional. Even if you never publish automatically, you want early gates that reduce the number of “obviously bad” candidates you spend compute on.

The pipeline checks:

- Length (will this fit in ~60s?)
- Unsafe/toxic content
- Profanity (especially for captions)

For toxicity scoring, I use Detoxify. Conceptually, it’s a pretrained text classifier (BERT-style under the hood) that predicts toxicity-related labels; it’s commonly trained/evaluated on large-scale “toxic comment” style datasets and returns a handful of scores you can threshold on. I don’t treat it as perfect moderation - it’s a pragmatic early filter.

For model details and code, see the [Detoxify project](https://github.com/unitaryai/detoxify).

---

## Rewriting with a local LLM (llama.cpp)

Early on, I assumed “summarize the post” would be enough. It wasn’t. Short-form narration has a different shape than a written post:

- It needs to be spoken, not read
- It needs to establish stakes quickly
- It needs to avoid Reddit artifacts (“EDIT:”, nested quotes, meta commentary)
- It needs a coherent arc that fits a strict length budget

In this project, rewriting is done locally using `llama.cpp` via `llama-cpp-python`. Running locally is a tradeoff: you give up some convenience compared to hosted APIs, but you gain reproducibility, lower marginal cost, and fewer external dependencies once it’s set up.

Resources: [llama.cpp](https://github.com/ggerganov/llama.cpp) + [llama-cpp-python](https://github.com/abetlen/llama-cpp-python). I'm using a quantized GGUF checkpoint so it runs comfortably on consumer hardware; in my local setup, the file I run is a `Q6_K` quant from [unsloth/Llama-3.1-8B-Instruct-GGUF](https://huggingface.co/unsloth/Llama-3.1-8B-Instruct-GGUF).

### Prompting strategy
I use the model for a few different “shapes” of work:

- **Rewrite**: turn the original post into a spoken script with a strict length budget.
- **Hook generation**: produce a short opening line (optional in my current implementation).
- **Hashtag generation**: produce a small set of relevant, platform-friendly hashtags.

One practical detail: I structure prompts as a stable instruction prefix (“system prompt”) followed by the variable content (the post text) and a clear delimiter. This helps in two ways:
- It’s easier to debug failures (“which instruction caused that behavior?”).
- It enables prefix reuse when many prompts share the same beginning.

I also keep the constraints explicit (word count, first-person voice, no rambling). “Technically correct” output is easy; “watchable” output is what takes iteration.

### Why KV caching matters
When you run an LLM, a lot of the cost is in processing the prompt (the “prefill” stage). If you run many generations with a shared prefix (for example: the same instruction scaffold plus different story text), it’s much cheaper to reuse the same model instance so the runtime can take advantage of KV cache behavior rather than repeatedly cold-starting.

In my current codebase, the rewrite stage already reuses a single `llama_cpp.Llama` instance across multiple posts in a batch, which is the simplest form of “implicit caching”: you avoid model reload overhead and keep inference hot.

Under the hood, `llama-cpp-python` can also reuse work when consecutive prompts share a token prefix (a “prefix match”), which is exactly what you get when the instruction prefix stays constant and only the story text changes.

I’m also working toward making the “per-reel” classification step benefit from the same idea (reusing a single instance across many reels), because the system instructions are essentially identical from one item to the next.

### Constrained outputs with grammar
One feature I like in `llama-cpp-python` is grammar-constrained generation (`LlamaGrammar`). For small structured tasks (like “return one of {male,female}”), grammar constraints can be more reliable than hoping the model behaves. It’s a nice example of using an LLM like a component in a system, not a mystical oracle.

---

## Voice generation (Kokoro TTS)

For voice, I use Kokoro, an open-weight text-to-speech model. The goal here is not cinematic acting; it’s consistent, intelligible narration that’s fast enough to generate in batch.

Kokoro resources: [Kokoro-82M model page](https://huggingface.co/hexgrad/Kokoro-82M).

In the pipeline, the TTS step turns the rewritten script into a waveform (audio). The code also tweaks speaking rate (e.g., slightly faster delivery to fit a 60-second target).

---

## Captions: why I use Whisper “in reverse”

Captions are part of the product, not a nice-to-have. For this format, you want large captions that appear in sync with the narration.

Here’s the catch: even though I already have the narration text (because I wrote it), Kokoro does not output word-level timestamps for the audio it generates. Without timestamps, you can’t reliably place each word on screen at the right moment.

So I do something slightly funny but very effective: I run speech-to-text on my own generated audio using Whisper, purely to recover timestamps for captioning.

Resources: [Whisper repository](https://github.com/openai/whisper), [Whisper paper](https://arxiv.org/abs/2212.04356), and the [whisper-medium model page](https://huggingface.co/openai/whisper-medium).

### Word-level timestamps (for highlight effects)
Whisper can produce word-level timestamps, not just sentence-level segments. That enables “highlight the current word” captions (often used in reels) while keeping everything fully automated. In my implementation, I enable word timestamps and write VTT captions with per-word timing.

---

## Subtitle formats: VTT -> ASS (old-school, reliable)

Whisper gives me VTT subtitles. VTT is easy to generate and inspect, but it’s limited for styling. For the “big, readable, mobile-first captions” look, I convert VTT into an ASS subtitle file.

ASS (Advanced SubStation Alpha) is very old-school, but it has two properties that matter a lot for automation:
- It’s expressive enough to style captions consistently (font, outline, positioning).
- It’s reliably supported by FFmpeg via `libass`, so I can burn subtitles into the final MP4 without manual editing steps.

This VTT->ASS step is one of those pragmatic engineering choices: not the trendiest format, but dependable when your goal is “100% automated rendering.”

---

## Rendering

Rendering is where all the pieces become a single artifact: an upload-ready vertical MP4.

This stage uses FFmpeg (via the `ffmpeg-python` wrapper) for a few reasons:
- It’s the most battle-tested way to manipulate media programmatically.
- It has stable primitives for crop, overlay, subtitle burn-in, resampling, and mixing.
- It supports hardware acceleration paths when available.

What happens during rendering:
- Choose a random segment from a background clip long enough to cover narration
- Crop the background video to 9:16
- Burn in the ASS subtitles (so the result looks the same everywhere)
- Mix narration audio with any background audio (at a low level)
- Encode as H.264 + AAC with faststart (friendlier for uploading/streaming)

### GPU note (CUDA / NVENC)
On my machine (RTX 3080 Ti, 12GB VRAM), most of the heavy lifting is GPU-accelerated, but in different ways:
- The LLM and Whisper can use CUDA through PyTorch / GPU support.
- FFmpeg can use the GPU for decoding and encoding depending on how it’s installed and which codecs are enabled. In my render code I request CUDA hardware acceleration for input and use NVIDIA’s H.264 encoder (`h264_nvenc`) for output. If your FFmpeg build doesn’t include NVENC support, it will fall back to CPU encoding.

FFmpeg docs and build info: [ffmpeg.org](https://ffmpeg.org/).

---

## Publishing (the hard part)

Once you can generate videos, it's tempting to aim for "one-click generate + publish." Instagram does offer an API for publishing reels, but the real-world friction is non-trivial.

In my experience, the hard parts are almost never “the HTTP request.” They’re account and permission requirements:

- You need the right account type
- Often need a connected Facebook Page
- App setup, roles, permissions, tokens
- Opaque errors like **"Insufficient Developer Role"**
- Token refresh, review processes, webhook hell

So I treat publishing as an optional extension:

- Core factory generates assets
- Publishing is documented but not required

*Important: This is a technical experiment. Review outputs for rights/attribution and platform compliance before publishing.*

This keeps the core project shippable. Platform integration work is slow and bureaucratic; it’s better as a separate track than a blocker for the media pipeline itself.

---

## What I learned

**Caching makes iteration practical**  
Separating stages and caching intermediate artifacts makes it feasible to iterate on prompts, caption styling, and rendering without paying the full “scrape everything again” cost.

**“Local” is a product decision, not just a technical decision**  
Local inference is more setup work, but it makes the pipeline cheaper per run and easier to run repeatedly without thinking about API budgets.

**Captions are a core engineering problem**  
Generating readable, synchronized captions is where a lot of the polish lives. Whisper is “overkill” if you only care about text, but it’s extremely convenient when what you actually need is timestamps.

**Publishing is a separate system**  
API permissions, tokens, account constraints, and operational reliability deserve their own design, observability, and retry logic. It's not just "the last step."

## What works now

The repo can:

- Scrape and cache stories
- Rewrite them into Reel-ready narration (+ hashtag suggestions for publishing)
- Generate TTS
- Generate timed captions
- Render vertical videos with burnt-in subs
- Output ready-to-post MP4s
- Document the publishing extension (and its headaches)

---

## Next steps

- **Content quality scoring** (LLM-as-a-judge to identify "good" reels and support ranking)
- **More caption styles** (themes, per-word highlights, dynamic pacing)
- **Performance tracking** (close the loop using the Instagram API)
- **Multi-platform publishing** (TikTok and Facebook in parallel with Instagram)
- **Robust publishing service** (token refresh, scheduling, retries)

---

## Try it yourself

If you want to explore the repo, start by generating one reel end-to-end, then iterate on one stage at a time (rewrite prompts, TTS settings, caption styling, render settings). The “factory” framing is the point: the value is in fast iteration.

**Repo:** [reels_factory](https://github.com/Razpines/reels_factory)

---

If you’re building something similar, I’d be curious where the time went for you: sourcing, filtering, rewriting, captions, rendering, or publishing. My experience was that the “glue” (formats, timing, failure modes, and reproducibility) mattered at least as much as the models.
