# Google Scholar (SerpApi) → SQLite Integration (Challenge 3)
A simple Java MVC application that runs **API consumption + database storage (2 researchers × 3 articles each)**, verification, basic error handling, and delivery.
The project demonstrates a clean separation of concerns with **Model–View–Controller** architecture.

![MVC Diagram](./images/MVC_diagram.png)

## 📖 Project Purpose
The goal of this project is to **explore and document the use of the Google Scholar API through SerpApi & manage it with a MDB like SQLite**. It includes a technical report summarizing essential API details and a functional GitHub repository with clear documentation for collaboration.

---

## 🚀 Key Functionalities
- **SerpApi Setup**
    - Created a SerpApi account under the free plan.
    - Obtained an API key to authenticate requests.

- **Technical Documentation**
    - Endpoints (URLs used to query the API).
    - Authentication methods.
    - Query parameters for filtering and customization.
    - Response formats (JSON).
    - Usage limits under the free plan.
    - Code examples in multiple programming languages.

- **A runnable Java app that:**
    - Calls **SerpApi’s Google Scholar Author** endpoint for **two** author IDs.
    - Stores **up to 3 articles per author** into a local **SQLite** database file `scholar.db`.
    - Handles common errors (network/HTTP 4xx–5xx, 429 rate limit, missing fields).
    - Avoids duplicates using `UNIQUE(researcher_id, title)` with UPSERT.
- Evidence the integration works:
    - SQL queries (or the included `Verify.java`) showing **3 rows per author**.
    - Optional “ID cleanup” migration if you ever pasted IDs with `&hl=...`.
- Repo updated and accessible to the **Digital NAO** team.

- **GitHub Repository**
    - Contains this README (project overview + full technical report).
    - Configured access for the **Digital NAO team**.

---

## 🧱 Tech stack

- **Java 17+**, **Maven**
- **SerpApi** – Google Scholar Author API
- **Jackson** (JSON), `java.net.http` (HTTP client)
- **SQLite** via `org.xerial:sqlite-jdbc`
- (Optional) **DB Navigator** plugin for IntelliJ Community to view/run SQL
---


## 2. Authentication Methods
- Sign up at [https://serpapi.com](https://serpapi.com).
- Retrieve API key from your dashboard.
- Add it to each request:

```
https://serpapi.com/search?engine=google_scholar&q=machine+learning&api_key=YOUR_API_KEY
```

---

## 🗄️3. Database schema (SQLite)

Table: `articles`

| column            | type     | notes |
|-------------------|----------|------|
| `id`              | INTEGER  | primary key, autoincrement |
| `researcher_id`   | TEXT     | author’s Google Scholar ID (`user=` in profile URL) |
| `researcher_name` | TEXT     | resolved from API |
| `title`           | TEXT     | article title |
| `authors`         | TEXT     | comma-separated |
| `publication_date`| TEXT     | store `YYYY-01-01` if you only have a year |
| `abstract`        | TEXT     | may be null |
| `link`            | TEXT     | article/citation link |
| `keywords`        | TEXT     | simple keywords derived from the title |
| `cited_by`        | INTEGER  | citation count (if available) |
| `created_at`      | TEXT     | defaults to `datetime('now')` |

**Constraint:** `UNIQUE(researcher_id, title)` — prevents duplicates on re-import.

---



## 5. Usage Limits
- **Free Plan**: 100 searches/month.
- **Rate Limit**: 1 request/second.
- Higher usage requires a paid plan.

---
## 🔑 SerpApi key

In **IntelliJ → Run → Edit Configurations…**, open the run config for `Main` and add:

- **Environment variables** → `SERPAPI_API_KEY=your_key_here`

(Alternatively, pass `-DSERPAPI_API_KEY=...` in VM options.)

---

## 👤 Get the two `author_id`s

1) Open each researcher’s **Google Scholar profile**.
2) Copy the value of `user` from the URL (e.g. `...citations?user=FyYiDG0AAAAJ&hl=es` → **`FyYiDG0AAAAJ`**).
3) You can also paste the **full profile URL**; the app will extract/clean the ID for you.

---

## ▶️ Run the importer

**IntelliJ (recommended):**

- **Run → Edit Configurations…** → select the config for `Main`
- **Program arguments**:
  ```
  <AUTHOR_ID_1> <AUTHOR_ID_2> 3
  ```
  Example:
  ```
  FyYiDG0AAAAJ Mxgb_LUAAAAJ 3
  ```
  (Pasting full profile URLs also works.)

Run it. The app creates/updates **`scholar.db`** in your project root.

**Maven (CLI) alternative:**
```bash
mvn -q -DskipTests exec:java \
  -Dexec.mainClass=org.example.scholar.Main \
  -Dexec.args="FyYiDG0AAAAJ Mxgb_LUAAAAJ 3"
```

---

## 🔍 Verify the data

### Option A — Use the provided verifier
Run `org.example.scholar.Verify` (optionally pass an author ID as an argument).

Expected console output:
```
== Count by researcher_id ==
FyYiDG0AAAAJ -> 3 rows
Mxgb_LUAAAAJ -> 3 rows

== Articles for FyYiDG0AAAAJ ==
01) ...
02) ...
03) ...
```

### Option B — SQL (DB Navigator or any SQLite client)
```sql
-- 0) List tables (sanity check)
SELECT name FROM sqlite_master WHERE type='table';

-- 1) Any “dirty” IDs (should be 0)
SELECT COUNT(*) AS dirty FROM articles WHERE researcher_id LIKE '%&%';

-- 2) Count per author (should be 3 and 3 if available)
SELECT researcher_id, COUNT(*) AS n_rows
FROM articles
GROUP BY researcher_id;

-- 3) The 3 rows of a given author
SELECT title, COALESCE(cited_by,0) AS cited_by, link
FROM articles
WHERE researcher_id = 'FyYiDG0AAAAJ'
ORDER BY cited_by DESC, title;
```

---

## 🧽 Optional: normalize old data (if IDs were stored with `&hl=...`)

Run this **safe migration** once to clean `researcher_id` and consolidate duplicates by `(researcher_id, title)` keeping the highest `cited_by`:

```sql
CREATE TABLE articles_new (
  id               INTEGER PRIMARY KEY AUTOINCREMENT,
  researcher_id    TEXT    NOT NULL,
  researcher_name  TEXT,
  title            TEXT    NOT NULL,
  authors          TEXT,
  publication_date TEXT,
  abstract         TEXT,
  link             TEXT,
  keywords         TEXT,
  cited_by         INTEGER,
  created_at       TEXT DEFAULT (datetime('now')),
  UNIQUE(researcher_id, title)
);

INSERT INTO articles_new (
  researcher_id, researcher_name, title, authors, publication_date,
  abstract, link, keywords, cited_by, created_at
)
SELECT
  CASE
    WHEN instr(researcher_id, '&') = 0 THEN researcher_id
    ELSE substr(researcher_id, 1, instr(researcher_id, '&') - 1)
  END AS researcher_id_clean,
  researcher_name,
  title,
  authors,
  publication_date,
  abstract,
  link,
  keywords,
  MAX(COALESCE(cited_by, 0)) AS cited_by,
  MIN(created_at)
FROM articles
GROUP BY researcher_id_clean, title;

DROP TABLE articles;
ALTER TABLE articles_new RENAME TO articles;
```

---

## 🛡️ Error handling (what you’ll see)

- **HTTP 429** (rate limit): clear message; retry later.
- **HTTP 4xx/5xx**: status code and response body are printed.
- **Missing fields**: `abstract` may be null; year-only dates stored as `YYYY-01-01`.
- **Idempotency**: re-running upserts (no duplicates) via unique constraint.

---

## 🧩 Troubleshooting

- **“Define SERPAPI_API_KEY…”** → set the env var in the Run Configuration.
- **No `articles` table** → run the importer once; it creates the schema.
- **IntelliJ Community has no Database tool** → install **DB Navigator** or run `Verify.java`.
- **DB Navigator shows a filter error** when opening the table → use **Remove Filter** (right-click table → Data → Filter → Remove).

---

## 📦 Deliverables checklist

- [ ] Source code (importer, verifier, optional migrator).
- [ ] `scholar.db` with **2 authors × 3 articles** (if available from API).
- [ ] Basic error handling in place.
- [ ] Repo pushed to GitHub and **Digital NAO** has access.

---

## 💰 Cost estimate (ballpark)

**Assumptions (adjust as needed):**
- Each run makes **2 requests** (one per author) and fetches 3 articles per request.
- You will execute **N runs per month** for testing/demos.
- Pricing changes; always check SerpApi’s current plans.

**1) API usage (SerpApi):**
- Requests per run: ~2
- Example monthly usage:
    - ~**50 runs/month** → ~**100 requests** → often fits **free tier (~100 req/mo)**.
    - **200–500 runs/month** → **400–1,000 requests** → likely **entry paid tier**.
- Ballpark **paid tier**: ≈ **USD $50/month** (few thousand requests).

**2) Developer time (one-time):**
- Setup + coding + verification: **~6–10 hours**.
- At **USD $15–$40/hour** → **USD $90–$400** one-time.

**3) Hosting/DB:**
- **SQLite** file → **$0**.
- If migrating to managed MySQL/PostgreSQL later: **$5–$15/month** typical.

**Summary:**
- **Recurring:** **$0/month** (free tier) to **~$50/month** depending on runs.
- **One-time:** **$90–$400** for development/packaging (varies by rates and what’s already coded).

---

## 📜 License & Terms

Educational use. Review SerpApi’s Terms of Service and rate-limit policies before production use.

---

## ✅ Conclusion

This sprint proves end-to-end integration from **Google Scholar (via SerpApi)** to **SQLite**, with clear run/verify steps, duplicate protection, and error handling. It can be extended to other DBMS or scaled to fetch more fields as your project grows.


---
## 📌 Utility and Scope Scenarios
- **Academic Research**: Helps researchers, students, and administrators quickly obtain citation metrics and top publications from Google Scholar profiles.
- **Institutional Reports**: Can be integrated into university systems to auto-generate performance metrics of faculty members.
- **Data Analysis**: Provides structured data that can later be extended into dashboards or research analytics platforms.

---

## 🌱 Sustainability
- The project leverages **SerpApi**, an established API provider, reducing the need for custom scraping (which may break frequently).
- Code is modular (MVC pattern), making it easy to maintain and extend over time.
- Open-source approach allows community contributions to ensure long-term project health.

---

## ⚙️ Applicability of Technical Aspects
- Demonstrates the use of **MVC in Java**, a widely adopted design pattern.
- Shows integration with an **external REST API** using Apache HttpClient and JSON parsing with Jackson.
- Can be extended into GUI applications or web backends with minimal changes.

---

## 📈 Scalability, Economic Viability, and Impact
- **Scalability**: The architecture allows integration with multiple APIs (e.g., OpenAlex, Scopus, PubMed) to broaden coverage beyond Google Scholar.
- **Economic Viability**: SerpApi has free and paid tiers, making it cost-effective for small projects and scalable for enterprise-level usage.
- **Impact**: Facilitates data-driven decisions in academia and research policy by providing transparent and structured author metrics.

---

## ⚠️ Notes
- The **Google Scholar Profiles API** has been discontinued. This project uses the **Google Scholar Author API** via SerpApi.
- Free SerpApi accounts have **request limits**. If you hit errors like `HTTP 403`, check your quota.

---

## 📜 License
This project is for **educational purposes** (demonstrating MVC in Java).  
Check SerpApi’s [Terms of Service](https://serpapi.com/legal) before production use.


## Conclusion
The **Google Scholar API via SerpApi** provides developers with a structured way to query authors, articles, and citation data.

By following this documentation, you can:
- Retrieve research results and author data.
- Integrate results into applications or databases.
- Handle authentication, query customization, and usage limits effectively.  
