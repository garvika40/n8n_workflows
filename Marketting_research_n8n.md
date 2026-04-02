# Topic to Article Research and Generation Pipeline

## Overview

This n8n workflow takes a topic as input and fully automates the process of researching, scraping, cleaning, summarising, and writing a high-quality 1000–1500 word blog article. It uses a chain of AI agents and web scraping to produce original, human-like content grounded in real source material.

---

## What It Does





```
Input topic
  └──→ Expand into keywords and questions
         └──→ Build optimised search queries
                └──→ Search Google for relevant articles
                       └──→ Scrape top article URLs
                              └──→ Extract and clean content
                                     └──→ Aggregate all articles
                                            └──→ Summarise insights and themes
                                                   └──→ Generate final article
                                                          └──→ Format and output
```

---

## Node Reference

### 1. Input Topic

| Property | Value |
|---|---|
| Type | Manual Trigger |
| Input field | `topic` |

**Purpose:** Starting point of the workflow. Provide a topic string to kick off the pipeline.

**Example input:**
```json
{
  "topic": "The future of AI in healthcare"
}
```

---

### 2. Keyword Expansion Agent

| Property | Value |
|---|---|
| Type | LangChain Agent |
| Model | `gpt-4o-mini` (temp: 0.2) |
| Output parser | JSON — keywords, long_tail, questions |

**Purpose:** Acts as an SEO strategist. Expands the input topic into structured keyword data.

**System prompt:**
```
You are an SEO strategist. Generate 10 high-intent search keywords,
5 long-tail keywords, and 5 "People Also Ask" style questions based
on the input topic.
```

**Output schema:**
```json
{
  "keywords":  ["keyword1", "keyword2", "..."],
  "long_tail": ["long tail phrase 1", "..."],
  "questions": ["What is ...?", "How does ...?", "..."]
}
```

---

### 3. Search Query Builder Agent

| Property | Value |
|---|---|
| Type | LangChain Agent |
| Model | `gpt-4o-mini` (temp: 0.2) |
| Output parser | JSON — search_queries array |

**Purpose:** Takes the expanded keywords and selects the 3 best search queries likely to return high-quality blog and article results.

**System prompt:**
```
You are a search optimization assistant. Select the best 3 search
queries that will return high-quality blog/article results.
```

**Output schema:**
```json
{
  "search_queries": ["query 1", "query 2", "query 3"]
}
```

---

### 4. Split Search Queries

| Property | Value |
|---|---|
| Type | SplitOut |
| Field to split | `output.search_queries` |

**Purpose:** Splits the 3 search queries into individual items so each one is processed separately by the Search Agent.

---

### 5. Search Agent

| Property | Value |
|---|---|
| Type | LangChain Agent |
| Model | `gpt-4o-mini` |
| Tool | SerpAPI (Google Search) |

**Purpose:** Uses the SerpAPI tool to search Google for each query and returns the top 3 most relevant article URLs.

**System prompt:**
```
You are a search assistant. Use the search tool to find the top 3 most
relevant article URLs for the given query. Return only the URLs.
```

---

### 6. Extract Top 3 URLs

| Property | Value |
|---|---|
| Type | Set node |
| Output field | `urls` |

**Purpose:** Maps the Search Agent's output into a clean `urls` field for the next step.

---

### 7. Split URLs

| Property | Value |
|---|---|
| Type | SplitOut |
| Field to split | `urls` |

**Purpose:** Splits the URLs array so each URL is scraped individually.

---

### 8. Scrape Article Content

| Property | Value |
|---|---|
| Type | HTTP Request |
| URL | `{{ $json.urls }}` |
| On error | Continue to error output |

**Purpose:** Fetches the raw HTML of each article URL. The error output routes to the Handle Scraping Errors node so failed URLs don't stop the pipeline.

---

### 9. Extract Main Content

| Property | Value |
|---|---|
| Type | HTML node |
| Operation | Extract HTML content |
| Source | Binary |

**Purpose:** Parses the raw HTML and extracts the readable text content from the page body.

---

### 10. Handle Scraping Errors

| Property | Value |
|---|---|
| Type | Set node |
| Output fields | `error_message`, `failed_url` |

**Purpose:** Captures failed scrape attempts gracefully. Logs the error message and the URL that failed for debugging without halting the workflow.

---

### 11. Content Cleaner Agent

| Property | Value |
|---|---|
| Type | LangChain Agent |
| Model | `gpt-5-mini` |
| Input | `{{ $json.markdown }}` |

**Purpose:** Strips navigation menus, ads, footers, cookie banners, and unrelated links from the scraped content. Returns clean, readable article text only.

**System prompt:**
```
You are a content cleaner. Extract the main article content from the
following text. Remove navigation, ads, footers, and unrelated links.
Return clean readable text only.
```

---

### 12. Combine All Articles

| Property | Value |
|---|---|
| Type | Aggregate |
| Mode | Aggregate all item data |
| Output field | `articles` |

**Purpose:** Waits for all scraped and cleaned articles to finish processing, then combines them into a single `articles` array for the Summarization Agent.

---

### 13. Summarization Agent

| Property | Value |
|---|---|
| Type | LangChain Agent |
| Model | `gpt-4o-mini` (temp: 0.2) |
| Output parser | JSON — insights, themes, unique_points, statistics, gaps |

**Purpose:** Acts as a research analyst. Reads all collected articles and extracts structured intelligence across five dimensions.

**System prompt:**
```
You are a research analyst. Extract key insights (bullet points),
common themes across sources, unique ideas from each source,
important data/statistics, and content gaps or missing angles
from the provided articles.
```

**Output schema:**
```json
{
  "insights":      ["insight 1", "..."],
  "themes":        ["theme 1", "..."],
  "unique_points": ["unique point 1", "..."],
  "statistics":    ["stat 1", "..."],
  "gaps":          ["gap 1", "..."]
}
```

---

### 14. Article Generation Agent

| Property | Value |
|---|---|
| Type | LangChain Agent |
| Model | `gpt-5-mini` |
| Input | Topic + Research Summary |

**Purpose:** Writes the final 1000–1500 word blog article using the research summary. Produces original, human-like content with H2/H3 subheadings, examples, and a strong intro and conclusion.

**System prompt:**
```
You are an expert blog writer. Write a high-quality, engaging article
(1000-1500 words) based on the research provided. Use a clear, engaging
tone. Include strong introduction and conclusion. Use subheadings (H2,
H3). Add examples where useful. Fill content gaps creatively. Do NOT
mention sources explicitly. Make it original and human-like.
```

---

### 15. Format Final Article

| Property | Value |
|---|---|
| Type | Set node |
| Output fields | `topic`, `article`, `word_count` |

**Purpose:** Packages the final article into a clean output object. Calculates approximate word count from the article text.

**Output:**
```json
{
  "topic":      "The future of AI in healthcare",
  "article":    "## Introduction\n\n...",
  "word_count": 1243
}
```

---

## Full Node Connection Map

```
Input Topic
  └──→ Keyword Expansion Agent ←── OpenAI Model - Keywords
  │                            ←── JSON Output Parser - Keywords
  └──→ Search Query Builder Agent ←── OpenAI Model - Search
  │                               ←── JSON Output Parser - Search
  └──→ Split Search Queries
         └──→ Search Agent ←── OpenAI Model - Search Agent
         │                 ←── Search Google (SerpAPI)
         └──→ Extract Top 3 URLs
                └──→ Split URLs
                       └──→ Scrape Article Content
                              ├── success ──→ Extract Main Content
                              │                └──→ Content Cleaner Agent ←── OpenAI Model - Cleaner
                              │                       └──→ Combine All Articles
                              │                              └──→ Summarization Agent ←── OpenAI Model - Summarizer
                              │                              │                        ←── JSON Output Parser - Summary
                              │                              └──→ Article Generation Agent ←── OpenAI Chat Model (gpt-5-mini)
                              │                                     └──→ Format Final Article
                              └── error ──→ Handle Scraping Errors
```

---

## Credentials Required

| Service | Credential type | Where to get it |
|---|---|---|
| OpenAI | API Key (`openAiApi`) | [platform.openai.com](https://platform.openai.com) → API Keys |
| SerpAPI | API Key (`serpApi`) | [serpapi.com](https://serpapi.com) → Dashboard |

> Both credentials are referenced by name in the workflow. Create them in n8n under **Settings → Credentials** before running.

---

## Models Used

| Node | Model | Why |
|---|---|---|
| Keyword Expansion | `gpt-4o-mini` | Structured JSON output, low cost |
| Search Query Builder | `gpt-4o-mini` | Structured JSON output, low cost |
| Search Agent | `gpt-4o-mini` | Tool use (SerpAPI), fast |
| Content Cleaner | `gpt-5-mini` | Better at stripping noise from messy HTML |
| Summarizer | `gpt-4o-mini` | Structured JSON output, cost efficient |
| Article Generator | `gpt-5-mini` | Best writing quality for final output |

---

## How to Run

1. Import the workflow JSON into n8n
2. Add your **OpenAI** and **SerpAPI** credentials
3. Open the **Input Topic** node and click **Test workflow**
4. Enter your topic in the input panel:
   ```json
   { "topic": "Your topic here" }
   ```
5. Click **Execute workflow**
6. Find the final article in the **Format Final Article** node output under `article`

---

## Typical Runtime

| Stage | Approximate time |
|---|---|
| Keyword + query generation | 5–10 seconds |
| Google searches (3 queries) | 10–20 seconds |
| Scraping + cleaning (up to 9 URLs) | 30–60 seconds |
| Summarisation | 10–20 seconds |
| Article generation | 15–30 seconds |
| **Total** | **~2–3 minutes** |

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Scraping node fails on all URLs | Site blocks bots | Add a User-Agent header to the HTTP Request node |
| Search Agent returns no URLs | SerpAPI quota exceeded | Check usage at serpapi.com dashboard |
| Article is under 1000 words | Model hit max token limit | Increase `max_tokens` in Article Generation Agent options |
| `gpt-5-mini` model not found | Model name unavailable in your region | Replace with `gpt-4o` in Content Cleaner and Article Generator nodes |
| JSON parser errors | Agent returned malformed JSON | Lower temperature to `0` on the affected agent |
| Workflow stops at Combine All Articles | Content Cleaner Agent failed silently | Add "Continue on Fail" to the Content Cleaner Agent node |

---

## Extending the Workflow

- **Publish to CMS** — add a WordPress or Webflow node after Format Final Article to auto-publish
- **SEO scoring** — add a node to score the article against the original keywords
- **Save to Notion** — pipe the final article into a Notion database for editorial review
- **Multiple topics** — replace the Manual Trigger with a Google Sheets trigger to process a list of topics in batch
