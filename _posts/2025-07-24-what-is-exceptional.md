---
layout: post
title: "On Doing Exceptional Work: A Grit-Fueled Journey Through Tech's Trenches"
date: 2025-07-24
categories: [nlp]
---
Hey folks, I've been knee-deep in job hunting lately, and one question keeps hitting me like a ton of bricks: "Prove to me you've done exceptional work." As someone who's not a natural self-promoter—hell, bragging feels alien to me—it's a tough one to answer without sounding forced. But after some introspection, I realized this post could serve as a note to myself and anyone else in the same boat: the quiet grinders in a world full of hype. Exceptional work isn't about landing a NeurIPS paper (though I'd love that someday, lol) or flashing a PhD for instant nods of approval. To me, it's about grit—the raw persistence to push through obstacles, learn on the fly, and deliver when it counts.

I'm happy for folks with fancy credentials; they've earned them. But credentials often miss the full picture. I've spent years overcoming hurdles, and it makes me wonder: If I ever start my own company, how do I spot talent like mine? The self-starters who don't shine on paper but crush it in the real world? This post is my attempt to unpack that, blending my career story with concrete examples. It's self-congratulatory in spots, sure, but it's born from genuine reflection on what "exceptional" really looks like.

## My Rocky Start: From Analyst Misery to Betting on Myself

I always dreamed of comp sci, but with an engineering background, I landed as an analyst post-grad. At first, it felt grown-up—big boy pants and all. But the politics, hierarchical blame games, and half-baked decisions from higher-ups (5 minutes of thought, tops) wore me down. I naively thought execs were smarter, reasoning deeper. Nope. It was a world of quick calls without follow-through, lacking that "mia culpa" attitude to scrap bad ideas and iterate. I was miserable for reasons I'll save for another post.

Then the pandemic hit: Jobless, relationship crumbled. I said, "Bet it on red." Landed a startup role because the hiring manager saw his younger self in me. "The government's paying half your salary—if you fly good, great; if not, we'll part ways." Bet. In a year, I went from sweating over cloning the main branch (fearing I'd nuke it) to brainstorming and delivering proof-of-concept designs with the CTO.

Exceptional? Not flashy, but gritty. I spotted a lag in feedback loops—CEO's schedule bottlenecked customer insights during feature dev. I volunteered to drive to clients, conduct user interviews, listen deeply, and iterate like a madman with my CTO and manager's blessing. I led engineers, mentored interns, trained customers, shaped product direction, tested code (diving into lines to verify hunches), and chased rabbit holes—like when demos crashed Chrome. Turns out, uncompressed audio overflowed tab memory. Fixed it through experiments.

Company sold the product to a better-funded competitor. Verbal share promises? Vanished. Asked to take over two engineers' roles while severely underpaid (plus legal fees from the ex), I was honest: Requested a modest raise. Fired a week later. Back to unemployment, I hustled at Starbucks with a young daughter to support.

## Diving into NLP: From Side Hustles to "Show, Don't Tell"

My NLP entry was serendipitous. At the startup, as the "prototype guy," my CTO/manager tasked me with Azure NLP APIs for a medication dosage feature. Limitations? Downloaded GPT-Neo and prompted it. Chatted with Cohere's founder Nick (brilliant support guy) about their model—he taught me personas. That sparked my dive.

But rewind: Years earlier, inspired by Lambda Labs' early days (no Canada plans), I pitched a buddy on a Canadian GPU hardware company. Back when 4x12GB cards were SOTA. On my 3-hour daily commute (miserable analyst job weekdays + restaurant weekends), I built our website productively. Specced servers, cold-emailed researchers. Intuitively, on-prem appealed for data privacy. Landed a researcher unfazed by our $40k rig—margins so fat, we could've rebuilt one on failure and still profited. She wanted support; we were two full-timers. I said, "Whatever it takes—sleeping bags in the server room if needed." Buddy got scared; I preserved the friendship over steaming ahead. Mistake—I wouldn't now. Starting a company needs activation energy, like a chemical reaction. Anecdotally, that's the vibe. No more customers; it tanked.

Fast-forward post-startup: Jobless again, I spot an NLP posting—from that same researcher I'd pitched! Perfect role, but "No way I get it." Applied anyway. Interview: "Here's data—talk about building a classifier." Me vs. 10 others theorizing. I thought, "Fuck it, show." Used FastAI with an LSTM backbone; hit ~78% accuracy. Presented the working model. Landed the job—my current one.

Lesson: Exceptional work is action over talk. Grit means building when others speculate.

## Recent Wins: Grit in Action on ML Projects

This mindset fuels my work today. Here are snippets showing grit as exceptional:

- **PII Annotation Automation:** Humans (including me) took a week for 1,000 samples—inconsistent labels tanked training. Tested Qwen3:30B (~90% score), exposed our errors. Pivoted to automate NER. SLURM (80GB) failed on GGUF; hacked Ollama to run 30B on my 8GB GPU. Synthetic pipeline with ChatGPT placeholders, SQL-queried FHIR samples, regex NER. Scaled to 7,700 in a week. Model at 75%, fixing labels.
- **Query Expansion Revival:** Flopped JaColBERT in December; revisited for Shopify prep after their AI head's paper nod. Blended hard negatives (10+ / 25-) for semantic boost. Post-rejection? Kept tinkering.
- **Chart Reading Fix:** Cohere interview—CoT logic solid, but chart interp trash. Proposed markdown + images on ChartQA. Log probs flagged lows; curated ~95, spotted annotation BS. POC in LabelStudio for ~200 samples.
- **KD Self-Study:** Shopify asked KD—clueless. 2.5 days: Hinton's paper, simulated log probs (Jeremy Howard inspo), Mistral PII consolidation, hybrid loss (alpha tweaks). Out-of-pocket compute 'cause SLURM sucked.

(Me + Grok/ChatGPT = fire for these.)

## Wrapping Up: A Note on Recognition and Spotting Grit

I'm tired of flying under the radar amid self-promoters. Exceptional work feels ordinary to the doer—it's just grinding. But in a credential-obsessed world, share your story: Blog, tweet, build publicly. It's how you'll connect and get recognized.

If I start a company, I'll hunt for grit signals: Folks who build classifiers instead of talking, drive to customers, hack hardware on commutes. What's your exceptional? Drop thoughts.
