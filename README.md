# bangla-news-intelligence-system
An event-centric NLP pipeline for Bangla news — clusters same-day articles into events, tracks emerging topics, extracts entities, analyzes sentiment, and compares how different sources frame the same story.

An event-centric intelligence pipeline over Bangla news. Instead of just scraping and listing headlines, this project tries to answer harder, more useful questions about the Bangla news landscape:

What is happening right now?
What topics are emerging?
Who/which organizations are involved?
Which articles from different outlets describe the same event?
How does each source frame the same event differently?
How does an event evolve over time?
What unusual changes are happening in the news landscape?

Status: Working end-to-end prototype / portfolio project. Not production-grade — see Limitations below.

Table of Contents
Overview
Architecture
Design Decisions Worth Knowing About
Getting Started
Pipeline Walkthrough
Sample Output
Known Limitations & Honest Caveats
Roadmap
Tech Stack
License
Overview

Bangla-language news is fragmented across dozens of outlets with no unified way to see the bigger picture: which stories are actually the same event, how coverage differs by source, and what's spiking versus what's just background noise. This project builds that layer on top of raw scraped articles.

The pipeline runs as a single Google Colab notebook (Bangla_News_Intelligence_System.ipynb) — no server or database setup required to get started, though it's structured so pieces can be lifted out into a standalone service later.

Architecture
RSS feeds (discovery)
        │
        ▼
trafilatura (article extraction — content-density heuristics, not per-site selectors)
        │
        ▼
Bangla text cleaning (Unicode NFC normalization, digit normalization, boilerplate stripping)
        │
        ▼
Multilingual sentence embeddings (paraphrase-multilingual-mpnet-base-v2)
        │
        ├──► Time-windowed event clustering  ──► "same event?" / "what's happening?"
        ├──► NER (BanglaBERT)                ──► "who/what is involved?"
        ├──► Sentiment analysis              ──► per-article, per-source tone
        ├──► TF-IDF emerging-topic detection ──► "what's trending?"
        ├──► Event timelines                 ──► "how did this evolve?"
        ├──► Robust anomaly detection        ──► "what's unusual?"
        └──► Cross-source framing comparison ──► "how does coverage differ?"
                │
                ▼
        Daily digest / chatbot query interface + dashboard visualizations
Design Decisions Worth Knowing About

A few choices in this project were deliberate, not defaults — worth calling out if you're reviewing the code:

RSS for discovery, trafilatura for extraction — not hardcoded CSS selectors. Per-site selectors break the moment a news outlet redesigns its pages. trafilatura uses content-density heuristics to find the article body/title/author/date, which works across differently-designed sites without per-source maintenance. RSS feeds are used purely for discovering fresh article URLs, since feed structure is far more stable than page markup.

A custom Bangla-aware tokenizer for TF-IDF (bug found and fixed during development). scikit-learn's default tokenizer regex (\b\w\w+\b) splits Bangla words at vowel signs and the virama/hasanta character, because those combining Unicode marks aren't matched by its definition of a "word character." For example, "বন্যা" (flood) would silently fragment into "বন" + "য" + "া" — three meaningless pieces instead of one real word. This was caught with a synthetic test (a news corpus where "flood" should be the clear emerging topic) before it could silently corrupt topic detection on real data. Fixed with a custom token_pattern matching whole runs of Bengali Unicode block codepoints.

Time-windowed clustering, not pure similarity clustering. Event clusters are formed using both embedding similarity and recency — two unrelated stories that happen to share vocabulary but are months apart won't merge into one "event." This was verified with a synthetic test: three distinct synthetic events across different sources and time gaps were correctly separated, including one case designed specifically to try to trick the clustering into merging unrelated stories.

Median/MAD-based anomaly detection, not mean/std. A single large spike (e.g., a major breaking news day) would otherwise inflate the standard deviation of every subsequent rolling window and mask smaller anomalies that follow it. A robust median/absolute-deviation z-score is far less sensitive to that one outlier. Verified with synthetic data containing both an injected volume spike and an injected drop — both were correctly flagged with zero false positives.

Getting Started
Open Bangla_News_Intelligence_System.ipynb in Google Colab.
Run Section 0 to install dependencies.
Run the feed-verification cell in Section 1 — this checks which configured RSS feeds are actually alive from wherever you're running the notebook, before you rely on them.
Run Section 2–4 to scrape and extract real articles.
Continue top-to-bottom through embeddings, clustering, NER, sentiment, topic detection, timelines, and anomaly detection.
Use Section 15 (get_daily_digest() / news_chatbot()) to query any date's news interactively.

No API keys are required for the core pipeline. An optional ANTHROPIC_API_KEY unlocks two extra features (see below).

Pipeline Walkthrough
Section	What it does	Answers
0–1	Setup, config, feed verification	—
2–4	RSS discovery → article extraction → ingestion	—
3	Bangla-specific text cleaning	—
5	Multilingual sentence embeddings	(foundation for everything below)
6	Time-windowed event clustering	Which articles describe the same event? / What's happening?
7	Named Entity Recognition (BanglaBERT)	Who/what organizations are involved?
8	Sentiment analysis	(input to framing comparison)
9	TF-IDF emerging-topic detection	What topics are emerging?
10	Event timeline builder	How does an event evolve over time?
11	Robust anomaly detection	What unusual changes are happening?
12	Cross-source framing comparison (statistical + optional LLM-assisted)	How is the same event framed differently?
13	Dashboard (matplotlib)	Visual summary of everything above
14	Save to CSV / Google Drive	—
15	Daily digest + interactive chatbot query	"Show me everything that happened on [date]"

Note: This version covers recent news only (whatever each source's RSS feed exposes at the time you run it — typically the last few days). Pulling articles for arbitrary past dates would require a sitemap-based historical backfill, which isn't implemented in this version — see Roadmap.

Sample Output

A single day's digest looks like this:

📅 2026-09-20 — 3 articles, 2 distinct events

🔹 Severe flooding in Dhaka
   Sources: prothomalo, bdnews24_bangla (2 articles)
   Entities: Sylhet, Government, Army
   Overall tone: negative (score: -0.30)

🔹 Bangladesh wins cricket match
   Sources: kalerkantho (1 article)
   Entities: Bangladesh
   Overall tone: positive (score: 0.80)
Known Limitations & Honest Caveats

This is a working prototype, not a production monitoring system. Specifically:

Clustering and anomaly thresholds are untuned on real data. The values used (sim_threshold=0.55, z_threshold=3.0) are reasonable starting points, validated only against synthetic test cases — not calibrated against a real Bangla news corpus yet.
NER and sentiment models are small, community-maintained models, not production-grade or benchmarked at scale. Verify their output against real examples before trusting the numbers.
Only a handful of sources are configured, and RSS feed URLs should be re-verified periodically — outlets occasionally rename or retire feeds.
Only covers recent news, not historical archives. RSS feeds only expose recent articles (typically the last few days), so the dataset only grows from whenever you start running ingestion — it can't retroactively pull older news. A sitemap-based backfill would solve this but isn't implemented in this version (see Roadmap).
Cross-source framing comparison is the least mature piece — there's no established benchmark for "framing" in Bangla NLP, so the current approach (sentiment differentials + entity emphasis, with an optional LLM-assisted layer) should be treated as a first attempt, not a validated method.
Roadmap
 Sitemap-based historical backfill, so arbitrary past dates (not just recent RSS-window news) can be queried
 Scheduled/automated ingestion (e.g., GitHub Actions) for true near-real-time updates
 Expand source list, including news websites of TV channels (Somoy TV, Channel 24, etc.)
 Natural-language date queries ("yesterday," "last week") in the digest/chatbot interface
 Replace TF-IDF topic extraction with BERTopic for richer topic-cluster labels
 Move from an in-memory embedding comparison to a proper vector database (Qdrant/FAISS) at larger scale
 Calibrate clustering/anomaly thresholds against a real, larger Bangla news corpus
Tech Stack
Ingestion: feedparser, trafilatura, requests
NLP: sentence-transformers (multilingual embeddings), Hugging Face transformers (BanglaBERT-based NER & sentiment)
Data: pandas, numpy, scikit-learn (TF-IDF)
Visualization: matplotlib
Optional: Anthropic API (Claude) for LLM-assisted framing comparison and narrative daily summaries
License

This project doesn't currently specify a license. Consider adding one (e.g., MIT) if you intend for others to reuse or build on this code.
