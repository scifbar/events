# Scientific Friday Bar #2
**Event summary - [GEUS](https://www.geus.dk/), Friday 30 October 2026**

*By Nikolai Andrianov, GEUS (nikolai@scifbar.dk). Reviewed by the participants.*

## About

[Scientific Friday Bar](https://scifbar.dk/) is a recurring, informal session where people discuss real AI use cases. Each session is hosted by one organization, a company or a research institution, on its own premises, so the discussion stays close to the host's actual work and data. The host presents one concrete case: a working AI workflow that needs scrutiny, or a problem still looking for a solution. The room brings together the host's technical and domain staff with external data scientists. The host provides the venue and the drinks; attending is free. Sessions are capped at around 20 participants, invited from the registration pool.

This second session followed directly from the [first one](https://github.com/scifbar/events/blob/main/2026-09-geus/scifbar_20260918_geus_summary.md). In September the room told us to put a harness around the model instead of improving retrieval, and to decide what a correct answer is and check for it. This time we came back with numbers on both.

## This session's host
- Geological Survey of Denmark and Greenland ([GEUS](https://www.geus.dk/))
- A research institute within the [Danish Ministry of Climate, Energy and Utilities](https://www.kefm.dk/)
- Exploitation and protection of geological resources
- Mapping, compilation and storage of data, research, monitoring and consultancy within water, energy, minerals, climate and environment
- About 200 scientists in Copenhagen, Aarhus and Nuuk

## Who was in the room

| Participant category | Number |
|---|---|
| Data scientist | 6 |
| Host domain expert | 1 |
| Other organization / potential host | 3 |
| Second speaker | 1 |
| Organizer and speaker | 1 |
| **TOTAL** | **12** |

## The talks

### Harness vs naive RAG on confidential subsurface data

*Nikolai Andrianov, GEUS. Slides: [PDF](https://github.com/scifbar/events/blob/main/2026-10-geus/scifbar_20261030_geus_slides_andrianov.pdf).*

The case is the same as in September: a local chatbot over 276 scanned reports from Greenlandic mineral expeditions, built in the project Exploring AI for Data Mining and Knowledge Extraction (Ministry of Mineral Resources of Greenland), on the same NVIDIA DGX Spark that serves the OMGCCS work. The reports are converted to markdown with [DeepSeek-OCR](https://arxiv.org/abs/2510.18234), which also extracts the images, and everything runs on premises, so the documents stay on site. The model is Qwen3.6-35B-A3B-FP8 at about 50 tokens per second.

To compare the two approaches we first needed a definition of a correct answer. The reference set has 21 questions, each with the report that holds the answer, a regular expression for the expected value (so that "1870 ppm" and "1,870" both count) and the exact lines in the markdown file where a human verified it. Some questions name the report ("in report 22602, what is the highest copper concentration in the eastern breccia zone, Klitdal N?"), some do not. Since an LLM is a probabilistic model and can give two different answers to the same question, every question was asked three times.

Naive RAG cuts the corpus into chunks of 900 characters, turns the question and every chunk into vectors of 1024 numbers, and sends the 10 chunks with the highest cosine similarity to the model together with the question: retrieve once, answer once. Looking at the chunks retrieved for the copper question shows the weakness. Most came from the right report, but only some contained the number. A chunk that talks about the copper concentration without giving the value is very similar to the question, and still useless.

The harness is [OpenCode](https://opencode.ai/), an open-source agentic coding tool, pointed at the markdown files. An LLM is like a draft horse: smart and strong, but useless without a harness to steer it. In pseudocode the harness is just a loop. The model gets the question, a long system prompt and three standard tools (search text in files, read a file, find a file by name), and the loop runs until the model stops calling tools. There are no chunks: the model opens the actual reports and searches them. The price is that the message grows with every call, so it needs guardrails, and it takes longer.

| Check | What it means | Naive RAG | Harness |
|---|---|---|---|
| Answer | correct value or fact in the answer | 48/63 | 57/63 |
| Retrieval | right report retrieved | 52/63 | 49/63 |
| Form | report ID up front, no tool scaffolding in the answer | 27/63 | 57/63 |
| IDs | no wrong report ID cited | 62/63 | 62/63 |
| Median time | seconds per question | 5.1 s | 10.2 s |

The harness gives the right answer in 57 of 63 runs against 48 for naive RAG, at twice the median time; on broad questions it can take up to a minute. The conclusion that surprised me most was not in the table. Before a visit to a GEUS' counterpart in Norway we had positive feedback and opened user accounts there; when I checked the usage before this talk, the last login was the day we left. Early users are satisfied, but few go beyond testing the functionality. The real question is what we actually want to ask it.

### Evaluation results on regulatory texts

*Eike F. Rades, DemAI (volunteer). Slides: [PDF](https://github.com/scifbar/events/blob/main/2026-10-geus/scifbar_20261030_geus_slides_rades.pdf). A personal side project, not a DemAI product.*

Eike built a RAG system for radiation-safety regulation after being made radiation protection officer in a previous job and having to know all the rules at once. The talk was mostly about measuring where it fails. It opened with a real question: a company holds a sealed Cs-137 source of 0.5 TBq; which security group is it in, and how often must it be inspected? The system answered that it could not tell, because the context did not contain Annex 6 of the order. An honest refusal, and still a failure, since the answer is in the law.

The corpus is about 20 IAEA publications in English (chunked with Docling) and five Danish executive orders as XML from retsinformation.dk, chunked along paragraph, subsection, item and annex row, plus two Danish guidance PDFs (chunked with Docling). The answering agent (LangGraph, Gemma 4 26B-A4B on Scaleway in the EU) retrieves 5 chunks per collection, checks whether the context is enough and searches again if not, then checks whether its own answer is supported by quotes that code verifies are really in the context, and retries if not. Underneath sits an evaluation harness: 145 questions with the facts a good answer must contain ("vital nuggets") and the verbatim source passages, two LLM judges (Qwen 3.8 27B and Mistral Medium 3.5, three votes each, never the answering model), 95% Wilson intervals, paired sign tests and an adoption rule written down before each measurement. One run of 53 questions costs about one euro.

The talk was built as a sequence of beliefs, each tested and corrected:

- *"Danish is the problem."* On the 39 development questions every retrieval miss was a Danish-law question. Switching to a multilingual embedder (bge-multilingual-gemma2) closed most of the language gap, and structure helped further: chunking along paragraphs and table rows raised recall@5 from 0.90 to 0.97. With the right embedder the misses concentrate in tables, cross-references and multi-part questions, in both languages. A BM25 hybrid made things worse and was dropped.
- *"We are at 97%."* Stricter prompt rules took the development set to 97% (with web search on; 92% without). On 106 fresh questions written before any answer was seen, the pass rate was 75%. A development score measures how well you tuned, not how good the system is.
- *"The self-correction loop will close the gap."* A bug had tied both correction loops to the web-search switch, so when web search was turned off for the fresh runs, the loops stopped. Once fixed, the loop found missing passages (retrieval misses 12 to 7 on one set) but the gain in pass rate was within the noise.
- *"We need a better, closed model."* On the same 53 questions Gemini 3.8 Flash passed 85-89%, the open Gemma 4 26B 83%, at $4.40 against €0.72 per run: no clear difference. A small model on a laptop with 6 GB of video RAM passed 60%; it also used a different, local embedder, so part of that gap is retrieval.

The Cs-137 question can only be answered by following the cross-reference: § 80 says group C sources are inspected every 25 months and points to Annex 6, whose table puts 0.5 TBq of Cs-137 in group C. Two chunks, one cross-reference, and the law is only 3% of the index. Of the 13 failed answers on the tuning set, 11 were incomplete with nothing false, one was wrong and one timed out. But none of them carried a warning. The goal is not 100%, but every claim tied to a visible source passage and what is missing reported as missing.

## What came out of it

The two talks attacked the same problem from opposite ends: a harness without much measurement, and a careful measurement of a pipeline without a harness. The discussion afterwards boiled down to the following five items worth trying.

1. **Build one shared evaluation for both systems.** One suggestion was a common eval, possibly as a public repository under [scifbar on GitHub](https://github.com/scifbar): one golden-set format, both systems run on both corpora (geology reports and radiation-safety law), the same judges and the same statistics. Both corpora are public, so questions, golden answers and outputs can be published. This answers Eike's own open question, a harness against his pipeline on the same golden set, and we are now discussing with Eike how to set it up.

2. **Test whether agentic search beats better retrieval on tables and cross-references.** This is where both systems fail, so it may be beneficial to run the OpenCode harness over the legal corpus, and turn cross-references into links at ingestion on the RAG side, then compare on the questions of that type. The impact of OCR errors on tables in the geology reports has not been studied yet and belongs in the same test.

3. **Run the 26B-class model locally.** The open Gemma 4 26B showed no clear gap to Gemini Flash, but it was measured in an EU cloud, while the laptop model dropped to 60% (a much smaller model, with a different embedder). The DGX Spark in the first talk is exactly the hardware class where it was not yet measured. A short experiment with a clear number at the end, and the answer to whether a fully local setup can match the cloud one.

4. **Make incomplete answers say so.** None of the 13 failed answers carried a warning, e.g. that some of the requested values were not found. This is the same point as calibrated uncertainty from the first session, now with a measurement behind it.

5. **Ask the users what they actually want to ask.** Low usage after a positive first reaction says the bottleneck is the question, not the model. Collecting real questions from geologists both tells us what the tool is for and gives the golden set questions nobody tuned on.

## Feedback from the room

The participants made the following points during and after the talks. The numbers in brackets refer to the actions listed above.

1. Every answer needs an evaluation, and an evaluation needs a definition of a correct answer that a machine can check. Two systems measured on different questions with different rules cannot be compared, so the comparison has to happen on a shared set, in the open. (Action 1.)

2. The right answer often fails to come back because a report holds a lot of irrelevant text: the model is handed a haystack and asked for a needle. Chunk size is a trade-off; small chunks lose context, large ones dilute it, and cutting along paragraphs or whole tables instead of a fixed number of characters is usually better. Tables are a problem for everyone, and OCR errors come on top. Cosine similarity also measures the wrong thing for this purpose: a chunk that names exactly what is being searched for, but without the value, scores high and is useless. (Action 2.)

3. The harness is slower, and on one GPU that matters. With more hardware the loop can be split into sub-agents that work in parallel, or the same task can be given to many agents and the majority answer taken, which raises reliability at the cost of compute. Easy questions should pass without the loop; capping the loops (three searches, two retries in Eike's system) keeps the cost bounded. Speed is also energy: a minute per answer instead of five seconds adds up. (Action 3.)

4. Open models are close enough for text analysis, and the decision is economic: count the users, then compare buying hardware against paying an EU host. The price of EU hosting against the cheapest global providers (OpenRouter-type prices of about 10 cents per million tokens for recent Qwen and DeepSeek models) would be worth quantifying as the price of privacy. A local setup also removes the dependency on a provider who can withdraw a model at short notice. For coding the closed models are still ahead. (Action 3.)

5. OpenCode can run with a local model, but in practice the integration with some hosted open models is fragile; one participant switched to [Goose](https://github.com/block/goose), another open-source harness, for that reason. Another noticed that OpenCode used more tokens than Claude Code for the same tasks with the same model. Adding a memory of earlier questions, so that a repeated question does not trigger the whole loop again, was recommended for any system that goes into production. (Actions 2 and 3.)

6. The practical value of a system like Eike's is in places like hospitals, where staff need to know the rules on X-ray equipment or radiotherapy. Training the model on the behaviour of a narrow domain, rather than on its knowledge, may help more than another retrieval trick. (Action 4.)

## Also at the bar

- **Satellite data are moving from pixels to products.** In remote sensing for land use and land-use emissions, the work is shifting from processing raw imagery to analytics on pre-trained models, embeddings and global map products from large providers, and towards asking a model in plain language how much of a crop was planted in a given year. Uncertainty in the underlying products still needs an expert. Applications range from crop forecasts and insurance to land affected by the war in Ukraine; in Denmark one company maps flood and storm risk for coastal buildings, which can end up uninsurable.

- **Build the bridge from both sides.** On the energy transition: make demand intelligent (an electric car should charge when energy is available, or support the grid, not just start charging when plugged in), and on the supply side use statistics and AI to decide where energy storage actually belongs, instead of everyone building their own.



*This session was funded by the OMGCCS project (INNO-CCUS) and the project Exploring AI for Data Mining and Knowledge Extraction (Ministry of Mineral Resources of Greenland).*
