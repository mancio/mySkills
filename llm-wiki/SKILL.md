---
name: llm-wiki
description: Bootstrap an Obsidian-like LLM wiki (knowledge graph + Flask
  viewer + ingestion workflow) inside any project, OR ingest raw source
  material from `<wiki>/raw/` into curated Markdown pages under
  `<wiki>/ingested/`. SCAFFOLD MODE — USE WHEN the user says "scaffold a new
  wiki", "bootstrap an llm wiki here", "create the wiki structure in this
  project", "set up an empty wiki", "I need a wiki in this repo". INGEST
  MODE — USE WHEN the user says "ingest the wiki", "process raw files",
  "update the llm wiki", "add this to the wiki", or drops new files into
  `raw/` and asks the agent to incorporate them. This skill is fully
  self-contained: every file required to scaffold a working wiki is
  embedded below as a verbatim code block.
---

# llm-wiki skill (self-contained)

Two modes share the same on-disk structure. Section A scaffolds it from
scratch using only the file contents embedded in this document. Section B
populates `ingested/` from `raw/`.

---

## Mode A — Scaffold (bootstrap a new wiki)

### Triggers

"scaffold", "bootstrap", "create the wiki structure", "set up an empty
wiki", "I need a wiki in this repo", "build the structure for an llm
wiki".

### Target structure

Create the following layout at `<target-root>/<wiki-name>/` (default
`<wiki-name>` is `llm-wiki`):

~~~text
llm-wiki/
├── README.md
├── SKILL.md             # this file (copied verbatim so the new wiki can re-scaffold itself)
├── requirements.txt
├── src/
│   ├── __init__.py      # empty
│   ├── wiki.py
│   ├── render.py
│   └── app.py
├── templates/
│   ├── base.html
│   ├── home.html
│   ├── page.html
│   ├── tag.html
│   ├── search.html
│   ├── graph.html
│   ├── raw.html
│   └── new.html
├── static/
│   └── style.css
├── raw/
│   └── .gitkeep         # empty marker file
└── ingested/
    └── .gitkeep         # empty marker file
~~~

The wiki starts **empty of knowledge**. Do not write any files into
`ingested/` (other than `.gitkeep`).

### Procedure

1. Confirm or default the target root (the user's workspace root) and the
   folder name (`llm-wiki`).
2. Create every directory in the tree above.
3. Create every file with the **exact verbatim content** embedded in
   "Embedded files" below. Match whitespace and indentation precisely.
4. Create `raw/.gitkeep` and `ingested/.gitkeep` as empty files (so the
   directories survive in git).
5. Print the run instructions:

   ~~~powershell
   cd <target>/llm-wiki
   python -m venv .venv
   .\.venv\Scripts\Activate.ps1            # PowerShell
   # or: source .venv/bin/activate         # bash/zsh
   python -m pip install -r requirements.txt
   python -m src.app
   # then open http://127.0.0.1:5057
   ~~~

6. Stop. Do **not** populate `ingested/` during scaffold mode.

### Anti-patterns (scaffold)

- Embedding ingested knowledge from another wiki into the new one.
- Renaming `raw/`, `ingested/`, `src/`, `templates/`, or `static/` — the
  viewer hardcodes those names.
- Skipping `.gitkeep` markers (empty dirs disappear under git).
- Creating files at the workspace root as bare files — always nest the
  wiki inside its own folder.

---

## Mode B — Ingest (turn `raw/` into curated pages)

### Triggers

"ingest", "process raw files", "update the llm wiki", "add this to the
wiki".

### Inputs

- Source files in `<wiki>/raw/` (any format: `.md`, `.txt`, `.pdf`,
  `.html`, `.json`, code, transcripts, screenshots, spreadsheets, etc.).
- Existing pages in `<wiki>/ingested/`.

### Output contract

Each ingested page is a Markdown file in `ingested/` with:

1. YAML frontmatter:

   ~~~yaml
   ---
   title: <Human-readable topic title>
   slug: <kebab-case-id>            # matches filename without .md
   sources:                          # list of raw files this page draws from
     - raw/<relative-path>
   tags: [<topic>, <topic>, ...]
   created: YYYY-MM-DD
   updated: YYYY-MM-DD
   ---
   ~~~

2. **Summary** — one paragraph (3–6 sentences, self-contained).
3. **Key points** — bulleted list of atomic facts.
4. **Details** — sub-topic sections in prose.
5. **Related** — links to other pages in `ingested/` (prefer
   `[[Other Page Title]]` wikilinks; relative `[text](other-page.md)`
   links also work).
6. **Sources** — list of raw files that fed this page.

Filenames use kebab-case slugs (e.g. `ingested/transformer-attention.md`).

### Procedure

1. **Inventory `raw/`** recursively. Compare against the `sources:`
   frontmatter of every file in `ingested/` to identify new or changed
   raw files.
2. **Cluster by topic.** One topic ≈ one page. Prefer updating an
   existing page over creating a near-duplicate.
3. **Draft / update.** Preserve `created`; refresh `updated`. Merge new
   sources into the `sources:` list (deduplicated). Rewrite Summary and
   Key points to reflect the merged content — do not just append.
4. **Cross-link** related pages.
5. **Regenerate `ingested/INDEX.md`** with all pages grouped by tag plus
   an alphabetized full list.
6. **Never modify `raw/`.**
7. **Report** at the end: pages created, pages updated, raw files newly
   covered, raw files still unprocessed (and why).

### Style rules

- Short sentences, explicit subjects, no dangling pronouns.
- Bullets for enumerable facts; prose for reasoning.
- Quote verbatim only when wording matters; otherwise paraphrase.
- Never invent facts. Flag ambiguity explicitly.
- Keep each page under ~500 lines; split large topics and link them.

### Anti-patterns (ingest)

- Dumping raw text into `ingested/` without restructuring.
- Creating one page per raw file (group by topic instead).
- Editing files in `raw/`.
- Forgetting to update `INDEX.md` or the `updated:` date.

---

## Viewer awareness

The Flask viewer (`src/app.py`) renders `ingested/` like a mini-Obsidian:
sidebar with all pages and tags, full-text search, tag pages, backlinks,
graph view, and `[[wikilink]]` resolution. Run with:

~~~bash
python -m src.app          # http://127.0.0.1:5057
~~~

URLs are `/page/<slug>` — keep `slug:` stable in frontmatter.

---

## Embedded files

Every file below is written **verbatim** to the indicated path. Preserve
whitespace and the exact content between the fence lines. Files use 4-tick
fences here only because some embedded files contain 3-tick fences; the
fence delimiter itself is **not** part of the file content.

### `requirements.txt`

````text
Flask>=3.0
markdown>=3.6
PyYAML>=6.0
Pygments>=2.17
````

### `README.md`

`````markdown
# llm-wiki

A personal LLM-curated wiki.

## Layout

- `raw/` — drop zone for unprocessed source files (PDFs, docs, web exports,
  notes, transcripts, code dumps, etc.). Anything goes here as-is.
- `ingested/` — clean, structured wiki pages produced by the ingestion skill.
  One topic per Markdown file, cross-linked, summarized for LLM consumption.
- `SKILL.md` — the ingestion workflow an agent follows to turn `raw/` into
  `ingested/`.

## Workflow

1. Drop new material into `raw/`.
2. Ask the agent to "ingest" it (see `SKILL.md`).
3. Query the wiki by reading files in `ingested/` — or browse it visually.

## Local viewer (Obsidian-like)

A small Flask app renders `ingested/` with a sidebar, wikilinks, backlinks,
tag index, full-text search, and an interactive graph view.

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
python -m src.app
```

Then open <http://127.0.0.1:5057>.

Supported syntax inside pages:

- Standard Markdown (tables, fenced code, TOC, code highlighting).
- `[[Page Title]]`, `[[slug]]`, `[[slug#anchor]]`, `[[slug|alias]]` wikilinks.
- Relative `[text](other-page.md)` Markdown links (auto-rewritten).
- YAML frontmatter (`title`, `slug`, `tags`, `sources`, `created`, `updated`).

The app auto-reloads when files in `ingested/` change.
`````

### `src/__init__.py`

````python
````

(Empty file — create it with zero bytes.)

### `src/wiki.py`

````python
"""Wiki engine: scan ingested/, parse frontmatter + links, build graph."""
from __future__ import annotations

import re
from dataclasses import dataclass, field
from pathlib import Path
from typing import Dict, Iterable, List, Optional, Set, Tuple

import yaml

WIKI_LINK_RE = re.compile(r"\[\[([^\[\]|#]+)(?:#([^\[\]|]+))?(?:\|([^\[\]]+))?\]\]")
MD_LINK_RE = re.compile(r"\[([^\]]+)\]\(([^)]+)\)")
FRONTMATTER_RE = re.compile(r"^---\s*\n(.*?)\n---\s*\n?", re.DOTALL)


@dataclass
class Page:
    slug: str                       # filename without .md, lowercased
    path: Path                      # absolute path
    rel: str                        # path relative to ingested root, posix style
    title: str
    tags: List[str] = field(default_factory=list)
    sources: List[str] = field(default_factory=list)
    created: Optional[str] = None
    updated: Optional[str] = None
    frontmatter: dict = field(default_factory=dict)
    body: str = ""                  # markdown body without frontmatter
    raw: str = ""                   # full file content
    out_links: Set[str] = field(default_factory=set)   # slugs this page links to


@dataclass
class Wiki:
    root: Path
    pages: Dict[str, Page] = field(default_factory=dict)        # slug -> Page
    by_rel: Dict[str, str] = field(default_factory=dict)        # rel path -> slug
    backlinks: Dict[str, Set[str]] = field(default_factory=dict) # slug -> slugs that link to it
    tags: Dict[str, Set[str]] = field(default_factory=dict)     # tag -> slugs

    def reload(self) -> None:
        self.pages.clear()
        self.by_rel.clear()
        self.backlinks.clear()
        self.tags.clear()
        if not self.root.exists():
            return
        for md_path in sorted(self.root.rglob("*.md")):
            page = _load_page(md_path, self.root)
            self.pages[page.slug] = page
            self.by_rel[page.rel] = page.slug
        # second pass: resolve links
        for page in self.pages.values():
            page.out_links = _extract_links(page, self)
        # backlinks
        for slug, page in self.pages.items():
            for target in page.out_links:
                self.backlinks.setdefault(target, set()).add(slug)
        # tag index
        for slug, page in self.pages.items():
            for tag in page.tags:
                self.tags.setdefault(tag, set()).add(slug)

    def get(self, slug: str) -> Optional[Page]:
        return self.pages.get(slug.lower())

    def all_pages(self) -> List[Page]:
        return sorted(self.pages.values(), key=lambda p: p.title.lower())

    def search(self, query: str, limit: int = 50) -> List[Tuple[Page, str]]:
        """Naive case-insensitive substring search; returns (page, snippet)."""
        q = query.strip().lower()
        if not q:
            return []
        hits: List[Tuple[Page, str]] = []
        for page in self.pages.values():
            hay = (page.title + "\n" + page.body).lower()
            idx = hay.find(q)
            if idx == -1:
                continue
            # snippet around match from raw body (preserve case)
            body = page.body
            start = max(0, idx - 60)
            end = min(len(body), idx + len(q) + 80)
            snippet = body[start:end].replace("\n", " ")
            if start > 0:
                snippet = "…" + snippet
            if end < len(body):
                snippet = snippet + "…"
            hits.append((page, snippet))
        hits.sort(key=lambda t: t[0].title.lower())
        return hits[:limit]

    def graph(self) -> dict:
        """Return nodes/edges suitable for vis-network."""
        nodes = []
        for slug, page in self.pages.items():
            degree = len(page.out_links) + len(self.backlinks.get(slug, set()))
            nodes.append({
                "id": slug,
                "label": page.title,
                "value": max(1, degree),
                "tags": page.tags,
            })
        edges = []
        for slug, page in self.pages.items():
            for target in page.out_links:
                if target in self.pages:
                    edges.append({"from": slug, "to": target})
        return {"nodes": nodes, "edges": edges}


def _load_page(path: Path, root: Path) -> Page:
    raw = path.read_text(encoding="utf-8")
    fm: dict = {}
    body = raw
    m = FRONTMATTER_RE.match(raw)
    if m:
        try:
            fm = yaml.safe_load(m.group(1)) or {}
        except yaml.YAMLError:
            fm = {}
        body = raw[m.end():]
    slug = (fm.get("slug") or path.stem).lower()
    title = fm.get("title") or path.stem.replace("-", " ").replace("_", " ").title()
    tags = fm.get("tags") or []
    if isinstance(tags, str):
        tags = [t.strip() for t in tags.split(",") if t.strip()]
    sources = fm.get("sources") or []
    if isinstance(sources, str):
        sources = [sources]
    rel = path.relative_to(root).as_posix()
    return Page(
        slug=slug,
        path=path,
        rel=rel,
        title=str(title),
        tags=[str(t) for t in tags],
        sources=[str(s) for s in sources],
        created=str(fm["created"]) if fm.get("created") else None,
        updated=str(fm["updated"]) if fm.get("updated") else None,
        frontmatter=fm,
        body=body,
        raw=raw,
    )


def _extract_links(page: Page, wiki: "Wiki") -> Set[str]:
    found: Set[str] = set()
    # [[wikilinks]]
    for m in WIKI_LINK_RE.finditer(page.body):
        target = m.group(1).strip()
        slug = _resolve_target(target, wiki)
        if slug:
            found.add(slug)
    # [text](other.md) — relative markdown links to other ingested pages
    for m in MD_LINK_RE.finditer(page.body):
        href = m.group(2).strip()
        if href.startswith(("http://", "https://", "mailto:", "#")):
            continue
        # strip fragment
        href_clean = href.split("#", 1)[0]
        if not href_clean.endswith(".md"):
            continue
        slug = _resolve_target(href_clean, wiki)
        if slug:
            found.add(slug)
    found.discard(page.slug)
    return found


def _resolve_target(target: str, wiki: "Wiki") -> Optional[str]:
    """Resolve a wikilink/relative-md target to a slug if it exists."""
    t = target.strip().lower()
    if not t:
        return None
    # exact slug
    if t in wiki.pages:
        return t
    # filename match
    if t.endswith(".md"):
        stem = t[:-3]
        if stem in wiki.pages:
            return stem
        # rel path match
        if t in wiki.by_rel:
            return wiki.by_rel[t]
    # title match (case-insensitive)
    for slug, page in wiki.pages.items():
        if page.title.lower() == t:
            return slug
    # slugified title match
    slugified = re.sub(r"[^a-z0-9]+", "-", t).strip("-")
    if slugified in wiki.pages:
        return slugified
    return None
````

### `src/render.py`

````python
"""Markdown rendering with wikilink + relative-link rewriting."""
from __future__ import annotations

import re
from typing import TYPE_CHECKING

import markdown as md

if TYPE_CHECKING:
    from .wiki import Wiki

from .wiki import WIKI_LINK_RE, _resolve_target


def render(body: str, wiki: "Wiki") -> str:
    body = _rewrite_wikilinks(body, wiki)
    body = _rewrite_relative_md_links(body, wiki)
    html = md.markdown(
        body,
        extensions=[
            "fenced_code",
            "tables",
            "toc",
            "sane_lists",
            "codehilite",
        ],
        extension_configs={
            "codehilite": {"guess_lang": False, "css_class": "codehilite"},
        },
        output_format="html5",
    )
    return html


def _rewrite_wikilinks(body: str, wiki: "Wiki") -> str:
    def repl(m: re.Match) -> str:
        target = m.group(1).strip()
        anchor = m.group(2)
        label = (m.group(3) or target).strip()
        slug = _resolve_target(target, wiki)
        if slug:
            href = f"/page/{slug}"
            if anchor:
                href += f"#{anchor.strip()}"
            return f'<a class="wikilink" href="{href}">{label}</a>'
        return f'<a class="wikilink missing" href="/new?title={target}">{label}</a>'
    return WIKI_LINK_RE.sub(repl, body)


_MD_LINK = re.compile(r"\[([^\]]+)\]\(([^)]+)\)")


def _rewrite_relative_md_links(body: str, wiki: "Wiki") -> str:
    def repl(m: re.Match) -> str:
        text = m.group(1)
        href = m.group(2).strip()
        if href.startswith(("http://", "https://", "mailto:", "#", "/")):
            return m.group(0)
        href_clean, _, frag = href.partition("#")
        if not href_clean.endswith(".md"):
            return m.group(0)
        slug = _resolve_target(href_clean, wiki)
        if slug:
            new = f"/page/{slug}"
            if frag:
                new += f"#{frag}"
            return f"[{text}]({new})"
        return m.group(0)
    return _MD_LINK.sub(repl, body)
````

### `src/app.py`

````python
"""Flask app: Obsidian-like local viewer for llm-wiki/ingested/."""
from __future__ import annotations

from pathlib import Path
from typing import Optional

from flask import Flask, abort, jsonify, redirect, render_template, request, url_for

from .render import render as render_md
from .wiki import Wiki

REPO_ROOT = Path(__file__).resolve().parent.parent
INGESTED = REPO_ROOT / "ingested"
RAW = REPO_ROOT / "raw"


def create_app() -> Flask:
    app = Flask(__name__, template_folder=str(REPO_ROOT / "templates"),
                static_folder=str(REPO_ROOT / "static"))
    wiki = Wiki(root=INGESTED)
    wiki.reload()

    @app.before_request
    def _maybe_reload():
        # cheap auto-reload in dev: re-scan if any file mtime changed
        if not INGESTED.exists():
            return
        latest = max((p.stat().st_mtime for p in INGESTED.rglob("*.md")), default=0)
        if latest > getattr(app, "_wiki_mtime", 0):
            wiki.reload()
            app._wiki_mtime = latest  # type: ignore[attr-defined]

    @app.context_processor
    def _ctx():
        return {
            "all_pages": wiki.all_pages(),
            "all_tags": sorted(wiki.tags.keys()),
            "wiki_root_name": INGESTED.name,
        }

    @app.route("/")
    def index():
        # prefer INDEX.md if present
        idx = wiki.get("index")
        if idx:
            return redirect(url_for("page", slug="index"))
        return render_template("home.html", pages=wiki.all_pages(), tags=sorted(wiki.tags.items()))

    @app.route("/page/<path:slug>")
    def page(slug: str):
        p = wiki.get(slug)
        if not p:
            abort(404)
        html = render_md(p.body, wiki)
        backlinks = sorted(wiki.backlinks.get(p.slug, set()))
        backlink_pages = [wiki.get(s) for s in backlinks if wiki.get(s)]
        outlinks = [wiki.get(s) for s in sorted(p.out_links) if wiki.get(s)]
        return render_template(
            "page.html",
            page=p,
            html=html,
            backlinks=backlink_pages,
            outlinks=outlinks,
        )

    @app.route("/tag/<path:tag>")
    def tag(tag: str):
        slugs = sorted(wiki.tags.get(tag, set()))
        pages = [wiki.get(s) for s in slugs if wiki.get(s)]
        return render_template("tag.html", tag=tag, pages=pages)

    @app.route("/search")
    def search():
        q = request.args.get("q", "").strip()
        results = wiki.search(q) if q else []
        return render_template("search.html", q=q, results=results)

    @app.route("/graph")
    def graph_view():
        return render_template("graph.html")

    @app.route("/api/graph")
    def api_graph():
        return jsonify(wiki.graph())

    @app.route("/raw-files")
    def raw_files():
        items = []
        if RAW.exists():
            for p in sorted(RAW.rglob("*")):
                if p.is_file() and not p.name.startswith("."):
                    items.append({
                        "name": p.name,
                        "rel": p.relative_to(RAW).as_posix(),
                        "size": p.stat().st_size,
                    })
        return render_template("raw.html", items=items)

    @app.route("/new")
    def new_page():
        title = request.args.get("title", "")
        return render_template("new.html", title=title)

    return app


app = create_app()


if __name__ == "__main__":
    app.run(host="127.0.0.1", port=5057, debug=True)
````

### `templates/base.html`

````html
<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8">
<title>{% block title %}llm-wiki{% endblock %}</title>
<link rel="stylesheet" href="{{ url_for('static', filename='style.css') }}">
</head>
<body>
<div class="layout">
  <aside class="sidebar">
    <header class="sidebar-header">
      <a href="{{ url_for('index') }}" class="brand">📒 llm-wiki</a>
    </header>
    <form action="{{ url_for('search') }}" method="get" class="search-form">
      <input type="search" name="q" placeholder="Search…" autocomplete="off">
    </form>
    <nav class="sidebar-nav">
      <a href="{{ url_for('graph_view') }}">🕸 Graph</a>
      <a href="{{ url_for('raw_files') }}">📂 Raw files</a>
    </nav>
    <section class="sidebar-section">
      <h3>Pages</h3>
      <ul class="page-list">
        {% for p in all_pages %}
          <li><a href="{{ url_for('page', slug=p.slug) }}">{{ p.title }}</a></li>
        {% endfor %}
      </ul>
    </section>
    <section class="sidebar-section">
      <h3>Tags</h3>
      <ul class="tag-list">
        {% for t in all_tags %}
          <li><a href="{{ url_for('tag', tag=t) }}">#{{ t }}</a></li>
        {% endfor %}
      </ul>
    </section>
  </aside>
  <main class="content">
    {% block content %}{% endblock %}
  </main>
</div>
</body>
</html>
````

### `templates/home.html`

````html
{% extends "base.html" %}
{% block title %}llm-wiki — Home{% endblock %}
{% block content %}
<h1>llm-wiki</h1>
<p>{{ pages|length }} page{{ '' if pages|length == 1 else 's' }} indexed.</p>

<h2>All pages</h2>
<ul>
  {% for p in pages %}
    <li>
      <a href="{{ url_for('page', slug=p.slug) }}">{{ p.title }}</a>
      {% if p.tags %}
        <span class="muted">— {% for t in p.tags %}<a class="tag" href="{{ url_for('tag', tag=t) }}">#{{ t }}</a> {% endfor %}</span>
      {% endif %}
    </li>
  {% endfor %}
</ul>

<h2>Tags</h2>
<ul>
  {% for tag, slugs in tags %}
    <li><a href="{{ url_for('tag', tag=tag) }}">#{{ tag }}</a> <span class="muted">({{ slugs|length }})</span></li>
  {% endfor %}
</ul>
{% endblock %}
````

### `templates/page.html`

````html
{% extends "base.html" %}
{% block title %}{{ page.title }} — llm-wiki{% endblock %}
{% block content %}
<article class="page">
  <header class="page-header">
    <h1>{{ page.title }}</h1>
    <div class="meta">
      {% if page.updated %}<span>updated {{ page.updated }}</span>{% endif %}
      {% if page.created %}<span>· created {{ page.created }}</span>{% endif %}
      {% for t in page.tags %}
        <a class="tag" href="{{ url_for('tag', tag=t) }}">#{{ t }}</a>
      {% endfor %}
    </div>
  </header>

  <div class="markdown">
    {{ html|safe }}
  </div>

  <section class="panel">
    <h3>Backlinks ({{ backlinks|length }})</h3>
    {% if backlinks %}
      <ul>{% for b in backlinks %}<li><a href="{{ url_for('page', slug=b.slug) }}">{{ b.title }}</a></li>{% endfor %}</ul>
    {% else %}<p class="muted">No pages link here yet.</p>{% endif %}
  </section>

  <section class="panel">
    <h3>Outgoing links ({{ outlinks|length }})</h3>
    {% if outlinks %}
      <ul>{% for o in outlinks %}<li><a href="{{ url_for('page', slug=o.slug) }}">{{ o.title }}</a></li>{% endfor %}</ul>
    {% else %}<p class="muted">No outgoing links.</p>{% endif %}
  </section>

  {% if page.sources %}
  <section class="panel">
    <h3>Sources</h3>
    <ul>{% for s in page.sources %}<li><code>{{ s }}</code></li>{% endfor %}</ul>
  </section>
  {% endif %}
</article>
{% endblock %}
````

### `templates/tag.html`

````html
{% extends "base.html" %}
{% block title %}#{{ tag }} — llm-wiki{% endblock %}
{% block content %}
<h1>#{{ tag }}</h1>
<p>{{ pages|length }} page{{ '' if pages|length == 1 else 's' }}.</p>
<ul>
  {% for p in pages %}
    <li><a href="{{ url_for('page', slug=p.slug) }}">{{ p.title }}</a></li>
  {% endfor %}
</ul>
{% endblock %}
````

### `templates/search.html`

````html
{% extends "base.html" %}
{% block title %}Search — llm-wiki{% endblock %}
{% block content %}
<h1>Search</h1>
<form action="{{ url_for('search') }}" method="get">
  <input type="search" name="q" value="{{ q }}" autofocus style="width:60%">
  <button type="submit">Search</button>
</form>
{% if q %}
  <p class="muted">{{ results|length }} result(s) for “{{ q }}”.</p>
  <ul class="results">
    {% for page, snippet in results %}
      <li>
        <a href="{{ url_for('page', slug=page.slug) }}"><strong>{{ page.title }}</strong></a>
        <div class="snippet">{{ snippet }}</div>
      </li>
    {% endfor %}
  </ul>
{% endif %}
{% endblock %}
````

### `templates/graph.html`

````html
{% extends "base.html" %}
{% block title %}Graph — llm-wiki{% endblock %}
{% block content %}
<h1>Graph</h1>
<div id="graph" style="height: 80vh; border:1px solid var(--border); border-radius:6px;"></div>
<script src="https://unpkg.com/vis-network/standalone/umd/vis-network.min.js"></script>
<script>
fetch('/api/graph').then(r => r.json()).then(data => {
  const container = document.getElementById('graph');
  const network = new vis.Network(container, data, {
    nodes: { shape: 'dot', scaling: { min: 6, max: 32 }, font: { color: '#ddd' } },
    edges: { arrows: 'to', color: { color: '#666' }, smooth: true },
    physics: { stabilization: true, barnesHut: { gravitationalConstant: -3000 } },
  });
  network.on('click', function (params) {
    if (params.nodes.length) window.location = '/page/' + params.nodes[0];
  });
});
</script>
{% endblock %}
````

### `templates/raw.html`

````html
{% extends "base.html" %}
{% block title %}Raw files — llm-wiki{% endblock %}
{% block content %}
<h1>Raw files</h1>
<p class="muted">Source material in <code>raw/</code>. Drop new files here, then re-run the ingestion skill.</p>
<table class="raw-table">
  <thead><tr><th>File</th><th>Size</th></tr></thead>
  <tbody>
    {% for it in items %}
      <tr><td><code>{{ it.rel }}</code></td><td>{{ '%.1f' % (it.size/1024) }} KB</td></tr>
    {% endfor %}
  </tbody>
</table>
{% endblock %}
````

### `templates/new.html`

````html
{% extends "base.html" %}
{% block title %}New page — llm-wiki{% endblock %}
{% block content %}
<h1>Page not found</h1>
<p>No page named <strong>“{{ title }}”</strong> exists yet.</p>
<p>Create it by adding a Markdown file under <code>ingested/</code> with frontmatter:</p>
<pre><code>---
title: {{ title }}
slug: {{ title.lower().replace(' ', '-') }}
sources: []
tags: []
created: today
updated: today
---

## Summary

…
</code></pre>
{% endblock %}
````

### `static/style.css`

````css
:root {
  --bg: #1e1e22;
  --panel: #25262b;
  --sidebar: #1a1b1e;
  --text: #e4e4e7;
  --muted: #8a8a93;
  --accent: #7aa2f7;
  --accent-hover: #a4c2ff;
  --border: #2e2f36;
  --tag-bg: #2d3748;
  --missing: #f7768e;
  --code-bg: #15161a;
}

* { box-sizing: border-box; }
html, body { margin: 0; padding: 0; background: var(--bg); color: var(--text);
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
  font-size: 15px; line-height: 1.55; }

a { color: var(--accent); text-decoration: none; }
a:hover { color: var(--accent-hover); text-decoration: underline; }

.layout { display: grid; grid-template-columns: 280px 1fr; min-height: 100vh; }

/* Sidebar */
.sidebar { background: var(--sidebar); border-right: 1px solid var(--border);
  padding: 16px; overflow-y: auto; height: 100vh; position: sticky; top: 0; }
.sidebar-header { margin-bottom: 12px; }
.brand { font-size: 18px; font-weight: 600; color: var(--text); }
.brand:hover { text-decoration: none; color: var(--accent); }
.search-form input { width: 100%; padding: 6px 10px; background: var(--bg);
  border: 1px solid var(--border); border-radius: 4px; color: var(--text); }
.sidebar-nav { display: flex; flex-direction: column; gap: 4px; margin: 12px 0; }
.sidebar-nav a { padding: 4px 6px; border-radius: 4px; }
.sidebar-nav a:hover { background: var(--panel); text-decoration: none; }
.sidebar-section { margin-top: 16px; }
.sidebar-section h3 { font-size: 11px; text-transform: uppercase; letter-spacing: 0.05em;
  color: var(--muted); margin: 0 0 6px; }
.page-list, .tag-list { list-style: none; padding: 0; margin: 0; }
.page-list li, .tag-list li { padding: 2px 0; font-size: 13px; }

/* Content */
.content { padding: 32px 48px; max-width: 920px; }
.content h1 { margin-top: 0; }

/* Page */
.page-header .meta { color: var(--muted); font-size: 13px; display: flex;
  gap: 8px; flex-wrap: wrap; align-items: center; margin-top: -8px; margin-bottom: 18px; }
.tag { background: var(--tag-bg); color: var(--text); padding: 1px 8px;
  border-radius: 10px; font-size: 12px; }
.tag:hover { text-decoration: none; background: var(--accent); color: var(--bg); }

.markdown { padding-bottom: 24px; }
.markdown h2 { border-bottom: 1px solid var(--border); padding-bottom: 4px; margin-top: 1.8em; }
.markdown h3 { margin-top: 1.6em; }
.markdown code { background: var(--code-bg); padding: 1px 5px; border-radius: 3px;
  font-size: 0.92em; }
.markdown pre { background: var(--code-bg); padding: 12px; border-radius: 6px;
  overflow-x: auto; border: 1px solid var(--border); }
.markdown pre code { background: transparent; padding: 0; }
.markdown table { border-collapse: collapse; margin: 12px 0; }
.markdown table th, .markdown table td { border: 1px solid var(--border); padding: 6px 10px; }
.markdown table th { background: var(--panel); }
.markdown blockquote { border-left: 3px solid var(--accent); margin: 0;
  padding: 4px 12px; color: var(--muted); }

.wikilink { background: rgba(122,162,247,0.08); padding: 0 3px; border-radius: 3px; }
.wikilink.missing { color: var(--missing); background: rgba(247,118,142,0.08); }

/* Panels */
.panel { margin-top: 28px; padding: 12px 16px; background: var(--panel);
  border: 1px solid var(--border); border-radius: 6px; }
.panel h3 { margin: 0 0 8px; font-size: 13px; text-transform: uppercase;
  letter-spacing: 0.05em; color: var(--muted); }
.panel ul { margin: 0; padding-left: 18px; }

.muted { color: var(--muted); font-size: 13px; }

/* Search */
.results { list-style: none; padding: 0; }
.results li { padding: 10px 0; border-bottom: 1px solid var(--border); }
.snippet { color: var(--muted); font-size: 13px; margin-top: 4px; }

/* Raw */
.raw-table { width: 100%; border-collapse: collapse; }
.raw-table th, .raw-table td { padding: 6px 10px; border-bottom: 1px solid var(--border); text-align: left; }

/* Pygments minimal dark */
.codehilite .k, .codehilite .kd { color: #bb9af7; }
.codehilite .s, .codehilite .s1, .codehilite .s2 { color: #9ece6a; }
.codehilite .c, .codehilite .c1 { color: #565f89; font-style: italic; }
.codehilite .nb { color: #7dcfff; }
.codehilite .mi, .codehilite .mf { color: #ff9e64; }
````

### `raw/.gitkeep`, `ingested/.gitkeep`

Empty zero-byte files. Create them so the directories survive in git.

### `SKILL.md`

When scaffolding a new wiki, copy **this entire file (including the YAML
frontmatter at the top)** to `<target>/llm-wiki/SKILL.md` so the new wiki
remains self-bootstrapping.

---

## Verification checklist (after scaffold)

After creating all files, the agent should verify:

1. The directory tree matches the structure above.
2. Running `python -m src.app` from inside the new wiki folder serves
   HTTP 200 on `/`, `/graph`, `/api/graph` (returns
   `{"nodes": [], "edges": []}`), `/raw-files`, and `/search?q=anything`.
3. `ingested/` contains only `.gitkeep` (no leaked knowledge from another
   wiki).
