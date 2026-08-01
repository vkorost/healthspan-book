# Extraction Pipeline

Companion document to the [README](./README.md). This describes how the source corpus for [*Healthspan, Annotated*](./book/Healthspan-Annotated.pdf) was built.

## What it is

A four-stage pipeline that converts raw podcast transcripts into structured, sourced claims organized by topic, then consolidates those claims into per-topic reference documents that serve as the grounding material for each chapter.

## Why it exists

Roughly 8,900 videos across thirteen channels is nearly 72,000 segments of text. No human reader could process that volume and retain a reliable picture of what the field actually says. The pipeline exists to reduce that corpus to a set of discrete, attributed claims, each carrying metadata about who said it, where, when, and how strong the evidence behind it appears to be.

## Corpus composition

| Channel | Videos |
|---------|--------|
| [Huberman Lab](https://www.youtube.com/@hubermanlab) (Andrew Huberman) | 397 |
| [The Drive](https://www.youtube.com/@PeterAttiaMD) (Peter Attia) | 1,342 |
| [The Doctor's Farmacy](https://www.youtube.com/@drmarkhyman) (Mark Hyman) | 1,233 |
| [FoundMyFitness](https://www.youtube.com/@FoundMyFitness) (Rhonda Patrick) | 136 |
| [Ben Greenfield Life](https://www.youtube.com/@bengreenfieldlife) (Ben Greenfield) | 1,125 |
| [The Human Upgrade](https://www.youtube.com/@DaveAspreyBPR) (Dave Asprey) | 1,037 |
| [The Dr. Gundry Podcast](https://www.youtube.com/@GundryMDYT) (Steven Gundry) | 829 |
| [The Ultimate Human](https://www.youtube.com/@ultimatehumanpodcast) (Gary Brecka) | 264 |
| [The Genius Life](https://www.youtube.com/@maxlugavere) (Max Lugavere) | 855 |
| [Andy Galpin](https://www.youtube.com/@drandygalpin) | 135 |
| [Gabrielle Lyon](https://www.youtube.com/@DrGabrielleLyon) | 389 |
| [Chris Masterjohn](https://www.youtube.com/@chrismasterjohn) | 549 |
| [Physionic](https://www.youtube.com/@Physionic) | 609 |

**Total: 8,900 videos, 71,825 unique text segments after chunking.**

These are counts of subtitle files, one per video, not counts of podcast episodes. No video was downloaded; only the subtitle track. Several of these channels publish clips, excerpts, and short-form videos alongside full episodes, and nothing in the subtitle metadata distinguishes one from another, so a long interview and a two-minute cut from that interview each contribute one file. Channels that clip aggressively are therefore weighted more heavily by file count than by hours of distinct material. Where a video had no subtitle track available, it is simply absent from the corpus and no record of the attempt was kept. The claim extraction stage samples per topic rather than per file, which limits but does not eliminate these effects.

Publication dates in the corpus run from February 2007 to 2 May 2026.

Subtitles were downloaded using [YTubeFetch](https://github.com/vkorost/ytubefetch), preserving each video's title, description, and publication date.

## Pipeline stages

### Stage 0: Chunking

Raw transcripts were split into segments targeting 500 to 1,000 words each; actual segment lengths range from 121 to 1,100 words. The result was 72,470 chunk records, 71,825 unique after deduplication. Output: one JSONL file with all chunks.

### Stage 1: Tagging

Each unique chunk was tagged against 103 topics using Claude Haiku. Topics span 13 categories:

| Prefix | Category | Topics |
|--------|----------|--------|
| MOV | Movement & Exercise | 14 |
| EAT | Nutrition | 18 |
| SLP | Sleep | 8 |
| HBL | Heat, Breath, Light | 5 |
| CVM | Cardiovascular & Metabolic | 7 |
| GUT | Gut Health | 6 |
| BRN | Brain & Cognition | 8 |
| HRM | Hormones | 6 |
| CMP | Compounds & Supplements | 15 |
| MSR | Measurement & Testing | 7 |
| SHA | Skin, Hair, Oral | 3 |
| ENV | Environment | 5 |
| AIH | AI for Health | 1 |

Across the 71,825 unique chunks, the average was 0.70 topics per chunk. 44,083 chunks (61.4%) matched no topic and were not used further.

### Stage 2: Claim extraction

Tagged chunks were sampled per topic with caps by relevance tier (full-match: 50, explain-mechanism: 40, passing: 15, vocabulary: 10). Newer videos were prioritized through recency-preferred round-robin sampling. Near-duplicate detection via MD5 of the first 40 words, lowercased and whitespace-normalized, prevented redundant extraction.

Each sampled chunk was processed by Claude Sonnet to extract discrete claims. Each claim carries structured metadata:

- `claim_text`: the claim itself
- `speaker`: who said it
- `source`: channel, video title, date
- `claim_type`: benefit, risk, protocol, mechanism, study finding, dose, etc.
- `evidence_signal`: RCT, observational, mechanistic, anecdote, expert opinion, etc.
- `contested`: whether other speakers in the corpus disagree
- `contested_note`: the nature of the disagreement
- `caution_flags`: replication concerns, animal-only data, single-study basis, etc.
- `primary_study`: named study if cited

**Result: 5,090 claims across 103 topics.**

### Stage 3: Consolidation

Claims were consolidated into structured reference documents, one per topic. Each document contains:

- Consensus claims (what most speakers agree on)
- Protocols and doses (specific actionable recommendations with attribution)
- Mechanisms (how the intervention is thought to work)
- Disagreements (where speakers contradict each other)
- Named studies (cited but unverified against primary literature)
- Caution flags (where evidence is thin or contested)
- Prevalence counts (how many videos and channels mention the topic)

**Result: 103 reference documents, 1.1 MB total.** These are the grounding material the book's [chapters](./book/chapters/) were written from. The `contested`, `evidence_signal`, and `caution_flags` fields are what the finished prose reads as evidence strength; see [A Note on the Voice](./book/chapters/00b-note-on-the-voice.md).

## Book assembly

The book was assembled using a separate multi-agent pipeline. A Style Agent established the prose voice. A Grounding Agent read the reference documents for each chapter and produced a factual scaffold. A Registry Agent resolved cross-chapter overlaps. A Writer Agent drafted each chapter from the grounding document. A Fact-Trace Agent verified every empirical claim back to the extracted sources. An Endnotes Compiler assembled the source citations now in [`ENDNOTES.md`](./book/ENDNOTES.md). A Deduplication pass, two Reviewer Agents, an Editor Agent, and a Revision Agent completed the quality loop. Bolded terms were collected into [`GLOSSARY.md`](./book/GLOSSARY.md).

The pipeline is described in [weekend-diy-book](https://github.com/vkorost/weekend-diy-book). The specific skill used for orchestration is `book-pipeline-runner`, a Claude Code skill that runs the full sequence and gates each step.

## What is not published

The raw subtitles, the chunked corpus, the tagging output, the extracted claims database, the consolidated reference documents, and the pipeline scripts are not published. What is published is the finished book ([PDF](./book/Healthspan-Annotated.pdf), [EPUB](./book/Healthspan-Annotated.epub), [Markdown chapters](./book/chapters/), [glossary](./book/GLOSSARY.md), [endnotes](./book/ENDNOTES.md)) and this description of the approach.
