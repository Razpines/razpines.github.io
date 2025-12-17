---
layout: post
title: "I Built a Fully Automated Reels Factory (Reddit → Rewrite → Voice → Captions → 9:16 Video → Publish)"
description: "A practical, end-to-end experiment in automating short-form storytelling videos — and the real-world friction points you hit when you try to auto-publish."
tags: [automation, video, ffmpeg, llm, captions, instagram]
---

You've seen them: first-person story over Minecraft gameplay, calm voiceover, perfect captions. They get millions of views. The format works because it's *repeatable*.

So I automated it.

This is about building a pipeline that eats Reddit posts and spits out ready-to-post vertical videos. Also: why the "last mile" of actually publishing them automatically is harder than generating the videos themselves.

**Repo:** https://github.com/Razpines/reels_factory

---

## The goal

I didn't want to generate one cool video. I wanted a factory:

- Feed it messy human text
- Get back platform-ready Reels
- Run it overnight, wake up to a batch
- Iterate fast because everything's reproducible
- Publish automatically (or at least try)

The full scope:

1. Find story candidates (Reddit)
2. Filter garbage and unsafe content
3. Rewrite into ~60s narration + hook
4. Generate voice (TTS)
5. Generate timed captions
6. Render 9:16 video with burnt-in subs
7. Publish to Instagram (Graph API)

---

## Architecture

Each stage produces artifacts the next stage consumes. When something breaks—and it will—you can inspect the output and rerun just that piece.

```
scrape → rewrite → tts+captions → render → publish
```

Why this matters: scraping hits rate limits. Rewriting produces garbage. TTS chokes on edge cases. FFmpeg will ruin your day over a missing codec.

Modular means you don't start from scratch every time.

---

## Content sourcing

Reddit posts are already optimized for engagement: first-person narratives, clear conflict, upvotes as a quality signal.

But raw posts are a mess:

- Way too long
- Inconsistent formatting
- Full of content that'll get you banned instantly

The scraper collects candidates into a local dataset with metadata. Filter and iterate without constantly re-scraping.

---

## Filtering

If you're automating at scale, filtering isn't optional.

I check for:

- Length (will this fit in ~60s?)
- Unsafe/toxic content
- Profanity (especially for captions)

This is the unsexy part that eats most of your time. Your model can be perfect; bad input distribution will still wreck you.

---

## Rewriting

I thought I'd just summarize posts. Wrong.

Short-form narration needs:

- A hook in the first 3 seconds
- Smooth spoken pacing
- No Reddit artifacts (TL;DR, edit notes, nested quotes)
- First-person voice

The rewrite step outputs:

- **Hook line** (punchy, under 10 words)
- **Narration script** (designed to be spoken in ~60s)

Prompt engineering mattered way more than I expected. "Technically correct" and "actually watchable" are different problems.

---

## Voice

TTS converts the narration to audio.

I'm not aiming for Oscar-worthy acting. I need:

- Consistency
- Clear diction
- Fast generation
- Minimal weird artifacts

Once TTS is reliable, you can generate dozens of reels in a batch without touching a microphone.

---

## Captions

Captions are the retention mechanic.

But you can't just dump the script on screen. You need **timing**.

The pipeline:

- Transcribes the audio
- Extracts word-level timestamps
- Outputs styled subtitle files
- Adjusts timing for readability

Getting captions readable on mobile is its own entire project.

---

## Rendering

This is where it becomes real.

Steps:

- Grab a segment from background gameplay
- Crop to 9:16
- Burn in captions
- Mix narration + background audio
- Export upload-ready MP4

FFmpeg is incredibly powerful and incredibly unforgiving. You'll learn this the hard way.

The goal: repeatable output. Same pipeline, consistent visuals, predictable filenames.

---

## Publishing (the hard part)

I wanted one-click: generate + publish.

Meta's Instagram Graph API *technically* supports this for professional accounts. Reality:

- You need the right account type
- Often need a connected Facebook Page
- App setup, roles, permissions, tokens
- Opaque errors like **"Insufficient Developer Role"**
- Token refresh, review processes, webhook hell

I made publishing **optional**:

- Core factory generates assets
- Publishing is documented but not required

*Important: This is a technical experiment. Review outputs for rights/attribution and platform compliance before publishing.*

This kept the project shippable. Platform integration is slow and bureaucratic; don't let it block the main pipeline.

---

## What works now

The repo can:

- Scrape and cache stories
- Rewrite them into Reel-ready narration
- Generate TTS
- Generate timed captions
- Render vertical videos with burnt-in subs
- Output ready-to-post MP4s
- Document the publishing extension (and its headaches)

**Repo:** https://github.com/Razpines/reels_factory

---

## What I learned

**Automation means handling ugly inputs**  
The hard part isn't generating a video. It's handling edge cases without manual intervention.

**Captions aren't a feature—they're the product**  
Great voiceover + bad captions = dead retention.

**Modular pipelines beat monoliths**  
When something breaks, rerun one stage, inspect the artifact, keep moving.

**Publishing is a separate engineering problem**  
Platform APIs, permissions, compliance—it's not "the last step." It's its own system.

---

## Next steps

- **No-API demo mode** (ship with example story + background clip)
- **Content ranking** (not just filtering—predict what'll perform)
- **More caption styles** (themes, per-word highlights, dynamic pacing)
- **Robust publishing service** (token refresh, scheduling, retries)

---

## Try it yourself

Start by generating one reel end-to-end, then scale to batch mode.

**Repo:** https://github.com/Razpines/reels_factory

---

If you're building something similar, tell me what hurt most: sourcing? rewriting? captions? rendering? publishing?

The AI part is easy. The infrastructure around it will humble you.
