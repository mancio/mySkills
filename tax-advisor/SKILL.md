---
name: personal-tax-advisor
description: Build from zero a Python app that acts as a personal tax advisor specialised in Polish tax law (PIT, CIT, VAT, ZUS, JDG, sp. z o.o., ryczałt, IP Box, etc.) but extensible to any other country. The app researches the user's question live on the open internet — restricted to sources hosted in the target country and written in that country's language — saves every fact it discovers as structured JSON, weighs the risk of the proposed action, and emits a PDF report. Use when the user asks to "build the tax advisor app", "set up the personal tax advisor", "create the Poland tax helper", "ask my tax advisor about X", or similar. The skill prescribes the source tiers, country-scoped search strategy, knowledge-base format, risk-scoring rubric, and PDF layout. It MUST NOT invent legal references — every citation comes from a real fetched URL. It MUST ask clarifying questions before answering anything substantive.
---

# Personal Tax Advisor — Country-Scoped Research & PDF Report Builder

## Goal

Stand up a Python CLI that, given a natural-language tax question and a target country (default: **Poland**), will:

1. Interview the user with follow-up questions until the situation is unambiguous.
2. Search the live internet **only on sources hosted in the target country**, in three priority tiers.
3. Save every retrieved fact (URL, source tier, publication date, language, extracted text, embedding-ready summary) into a versioned JSON knowledge base under `kb/{country}/`.
4. Synthesise an answer that explicitly states the **deductibility / legality border**, the **grey zones**, and the **practical risk** of the action the user is contemplating.
5. Render a PDF report (`reports/{country}/{YYYY-MM-DD}_{slug}.pdf`) with citations and a risk score (Low / Medium / High / Very High).

No mock data. No invented statute numbers. If a required secret (LLM key, search API key) is missing, **stop and ask the user** to provide it.

## Hard rules (read first)

- **Country-locking is mandatory.** Every HTTP request that hits a search engine MUST include the country/language filter. Every result whose hostname does not resolve to a domain registered in / clearly serving the target country MUST be discarded. For Poland this means: TLD `.pl`, OR the page's `<html lang>` is `pl`, OR the host is on the curated allow-list (see below). No `.com` blogs slip through unless they are on the allow-list.
- **Recency-first, fallback to older.** Sort results by publication date desc; only fall back to older material when nothing from the last 24 months is found. Always store the publication date in the JSON record and surface "stale source" warnings in the PDF when the newest citation is > 24 months old.
- **Never fabricate a legal article number, ustawa, rozporządzenie, interpretacja indywidualna sygnatura, or KIS ruling ID.** If you cannot retrieve the exact reference from a fetched page, write `"reference": null` and flag it.
- **Always ask clarifying questions first.** Tax answers depend on form of business (JDG vs. sp. z o.o. vs. UoP vs. ryczałt vs. karta podatkowa), VAT status, residency, tax year, and the concrete amount. Do not produce a PDF until those are pinned down.
- The tool is a **research assistant, not a licensed doradca podatkowy**. Every PDF must carry the disclaimer on the cover page (see template).
- Do not commit `.env`, `kb/` raw HTML dumps, or `reports/`.

## Source tiers (priority order)

The crawler tries tier 1 first, then tier 2, then tier 3. It keeps results from every tier but weights citations by tier in the final answer (tier 1 = authoritative, tier 3 = anecdotal).

### Tier 1 — Official government & courts
For **Poland**, the curated allow-list:

```
podatki.gov.pl                # MF — main tax portal
www.gov.pl/web/finanse        # Ministerstwo Finansów
www.gov.pl/web/kas            # Krajowa Administracja Skarbowa
eureka.mf.gov.pl              # interpretacje indywidualne (KIS) — ESKP database
sip.lex.pl                    # only public preview pages
isap.sejm.gov.pl              # Internetowy System Aktów Prawnych — ustawy & rozporządzenia
orzeczenia.nsa.gov.pl         # NSA / WSA judgments
www.zus.pl                    # ZUS for składki
www.nfz.gov.pl                # NFZ for składka zdrowotna
```

For other countries the skill must build the equivalent allow-list at first run by asking the user to confirm 3–5 official domains; cache it under `kb/{country}/_sources.json`.

### Tier 2 — Tax accountants, doradcy podatkowi, Big-4 / mid-tier consulting
For Poland, examples (extend as discovered, all `.pl`):

```
www.pwc.pl  www.ey.com/pl_pl  www2.deloitte.com/pl  www.kpmg.com/pl
www.mddp.pl  www.crido.pl  www.gtkonsulting.pl  www.taxa.pl
poradnikprzedsiebiorcy.pl  ifirma.pl/blog  ksiegowosc.infor.pl
podatki.biz  gofin.pl  rp.pl/podatki
```

### Tier 3 — Forums / Reddit / community Q&A (anecdotal, "what really happens")
For Poland:

```
www.reddit.com/r/Polska
www.reddit.com/r/PolskaPolityka
www.reddit.com/r/ksiegowosc
www.reddit.com/r/programowanie  (for IP Box / B2B threads)
forum.infor.pl
forum.gazetaprawna.pl
www.wykop.pl
```

Reddit results MUST be filtered to threads in Polish (detect with `langdetect`); English r/Poland threads are dropped unless the OP's situation is identical (same `country` filter still applies — language check, not just subreddit).

## Search strategy

Use **one** of the following back-ends, asked from the user at first run:

| Back-end | Free? | Country filter |
|---|---|---|
| Tavily Search API (`api.tavily.com`) | free tier 1k/mo | `country="poland"` + `include_domains=[...tier1+tier2+tier3]` |
| Brave Search API | free tier 2k/mo | `country=PL` + `search_lang=pl` |
| SerpAPI Google | paid | `gl=pl&hl=pl&google_domain=google.pl` |
| DuckDuckGo HTML (`duckduckgo.com/html/`) | free, no key | `kl=pl-pl`, then post-filter by hostname |

Fallback chain: if the chosen back-end returns < 3 hits, retry with broader query, then with `include_domains` removed (still post-filter by TLD/lang).

For every query the agent issues **three** variants:
1. Verbatim user question translated into the target language.
2. Statute-style query (e.g. `"ustawa o PIT" art. 22 koszty uzyskania przychodu reklama`).
3. Practical query (e.g. `"czy mogę odliczyć" laptop JDG VAT 50%`).

Then merge & dedupe by URL, sort by date desc.

## Borderline / "grey zone" handling

When the question is of the form *"can I deduct X / treat Y as koszt uzyskania przychodu / claim VAT on Z"*:

1. From tier 1, extract the exact statute text and any KIS interpretacja indywidualna that mentions the same expense category. Save sygnatura + date.
2. From tier 2, collect at least 2 advisor write-ups summarising the practical line.
3. From tier 3, collect at least 3 first-person reports of what actually happened (audit, no audit, challenged, accepted). Tag each with `outcome: accepted | challenged | unknown`.
4. Compute a **risk score** with this rubric:

| Signal | Weight |
|---|---|
| Tier-1 statute clearly permits → -2 |
| KIS interpretacja in user's favour, < 24 mo old → -2 |
| KIS interpretacja against, < 24 mo old → +3 |
| NSA judgment in user's favour → -3 |
| NSA judgment against → +4 |
| Tier-2 consensus "permitted" → -1 |
| Tier-2 consensus "risky" → +1 |
| Tier-3 majority `accepted` (n ≥ 3) → -1 |
| Tier-3 majority `challenged` → +2 |
| No tier-1 source found → +2 |
| Newest source > 24 mo old → +1 |

Map score → label: `≤ -3 Low`, `-2..0 Medium`, `1..3 High`, `≥ 4 Very High`.

## Knowledge base format

Every fetched & accepted source becomes one JSON file under `kb/{country}/{tier}/{sha1(url)}.json`:

```json
{
  "url": "https://...",
  "host": "podatki.gov.pl",
  "tier": 1,
  "country": "PL",
  "lang": "pl",
  "title": "...",
  "published_at": "2025-09-14",
  "fetched_at": "2026-05-06T10:22:00Z",
  "query": "czy mogę odliczyć VAT od samochodu osobowego JDG",
  "summary": "≤ 800 chars, in Polish",
  "summary_en": "≤ 800 chars, English mirror for the user",
  "key_facts": [
    {"claim": "...", "reference": "art. 86a ust. 1 ustawy o VAT", "stance": "permits|forbids|conditional"}
  ],
  "raw_text_path": "kb/PL/1/{sha1}.txt"
}
```

A per-question **answer bundle** is saved to `kb/{country}/answers/{YYYY-MM-DD}_{slug}.json` containing the question, the clarifying-question transcript, the list of cited source SHA-1s, the computed risk score, and the final synthesised answer.

## Project layout

```
tax/
├── SKILL.md                  # this file
├── .env.example              # LLM_API_KEY=, SEARCH_BACKEND=tavily|brave|serpapi|ddg, SEARCH_API_KEY=
├── requirements.txt          # requests, beautifulsoup4, lxml, python-dotenv, jinja2,
│                             # langdetect, tldextract, pydantic, rich, tqdm,
│                             # tavily-python (optional), reportlab, pdfkit OR weasyprint,
│                             # readability-lxml, dateparser, openai (or anthropic)
├── src/
│   ├── __init__.py
│   ├── config.py             # loads .env, exposes constants & per-country source allow-lists
│   ├── countries/
│   │   ├── __init__.py
│   │   ├── base.py           # CountryProfile dataclass
│   │   └── poland.py         # PL allow-lists, language="pl", currency="PLN", tax-year rules
│   ├── interview.py          # asks clarifying questions until profile is complete
│   ├── search.py             # pluggable search back-ends + country/language filter
│   ├── fetch.py              # polite fetch (UA, 15 s timeout, 1 retry, robots.txt respected)
│   ├── extract.py            # readability + bs4 → clean text, detect lang, parse pub date
│   ├── kb.py                 # read/write JSON knowledge base, sha1 keys, dedupe
│   ├── analyze.py            # LLM call: summarise, extract key_facts, compute risk score
│   ├── report.py             # renders templates/report.html.j2 → PDF via weasyprint
│   ├── advisor.py            # orchestrates: interview → search → fetch → extract → kb → analyze → report
│   └── main.py               # CLI entry: `python -m src.main --country PL --question "..."`
├── templates/
│   └── report.html.j2        # cover page, executive summary, risk badge, citations, disclaimer
├── kb/                       # generated, gitignored
└── reports/                  # generated, gitignored
```

## Build steps (execute in order)

1. **Ask the user, before writing any code:**
   - Which LLM provider & API key (OpenAI / Anthropic / local Ollama)?
   - Which search back-end & API key (Tavily recommended for free + good country filter)?
   - Confirm the default country (Poland) and the user's situation skeleton: residency, form of business (JDG / sp. z o.o. / UoP / B2B / ryczałt / karta), VAT registered (TAK/NIE), tax year.
   - Confirm the Polish tier-1 allow-list above is complete; offer to add/remove.
2. Write `.env` from answers, create `requirements.txt`, set up venv:
   `python -m venv .venv ; .venv\Scripts\pip install -r requirements.txt` (Windows / PowerShell).
3. Implement modules in the order: `config → countries/poland → interview → search → fetch → extract → kb → analyze → report → advisor → main`.
4. Each module gets a `__main__` smoke test (e.g. `python -m src.search "VAT samochód osobowy" --country PL` prints filtered hits).
5. `advisor.py` flow per question:
   1. `interview.run(question, profile)` → loops until `profile.is_complete()`.
   2. Build 3 query variants in target language.
   3. `search.run(...)` → list of hits, country-filtered.
   4. `fetch.get(url)` for the top N per tier (default 5/5/5), respect robots.txt.
   5. `extract.parse(html)` → text + lang + date; drop if lang ≠ target.
   6. `analyze.summarise(text)` (LLM, structured output via pydantic) → `key_facts`.
   7. `kb.save(record)` for each accepted source.
   8. `analyze.score(records)` → risk score + label.
   9. `analyze.synthesise(question, records, score)` → final markdown answer with inline `[1][2]` citations.
   10. `report.render(...)` → PDF.
6. Print: `Wrote reports/PL/2026-05-06_vat-samochod.pdf  (risk: Medium, 14 sources cited)`.

## PDF report requirements

`templates/report.html.j2` rendered through **weasyprint** (preferred — pure Python on Windows is fiddly; if weasyprint install fails on Windows, fall back to `pdfkit` + wkhtmltopdf and tell the user to install it).

Layout:

1. **Cover page**
   - Title: the user's question, verbatim.
   - Country flag emoji + country name + tax year.
   - Generation timestamp (UTC).
   - Big risk badge (color-coded): Low `#0a7d28` · Medium `#d4a106` · High `#d97706` · Very High `#b91c1c`.
   - Disclaimer block: *"Niniejszy dokument ma charakter informacyjny i nie stanowi porady prawnej ani podatkowej w rozumieniu ustawy o doradztwie podatkowym. Przed podjęciem decyzji skonsultuj się z licencjonowanym doradcą podatkowym."* (translate for other countries.)
2. **User profile** — bullet list of the answers gathered in the interview.
3. **Executive answer** — 5–10 sentences, plain language, in the user's preferred language (ask).
4. **Where the line is** — explicit "what is clearly OK / clearly NOT OK / grey zone" three-column table.
5. **Risk analysis** — the rubric table with which signals fired and why.
6. **Citations** — numbered list grouped by tier, each entry: title, host, date, URL, 1-line takeaway. Tier 1 first.
7. **Stale-source warning** (only if newest tier-1 source > 24 mo old).
8. **Appendix** — full list of clarifying Q&A from the interview.

## Done =

Running `python -m src.main --country PL --question "Czy mogę zaliczyć w koszty zakup roweru elektrycznego dla JDG?"` from a clean checkout (after `pip install -r requirements.txt` and a populated `.env`) will:

1. Interactively ask any missing profile questions.
2. Produce 5–15 JSON files under `kb/PL/{1,2,3}/`.
3. Produce one answer bundle under `kb/PL/answers/`.
4. Produce one PDF under `reports/PL/` whose cover page shows a real risk score and whose Citations section lists only `.pl` (or allow-listed) URLs in Polish, sorted newest first.
