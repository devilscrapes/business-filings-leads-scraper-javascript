<p align="center"><a href="https://apify.com/DevilScrapes/new-business-filings-leads-scraper"><img src="assets/banner.svg" alt="New Business Filings — US State Registry Leads Feed — JavaScript example" width="100%"></a></p>

# Business Filings Leads Scraper for JavaScript and Node.js: a new-company feed for your stack

<p align="center">
<a href="https://apify.com/DevilScrapes/new-business-filings-leads-scraper"><img alt="Run on Apify" src="https://img.shields.io/badge/Run%20on-Apify-DC2626?logo=apify&logoColor=white"></a>
<img alt="JavaScript" src="https://img.shields.io/badge/Node.js-20%2B-339933?logo=nodedotjs&logoColor=white">
<a href="LICENSE"><img alt="MIT license" src="https://img.shields.io/badge/license-MIT-0A0606"></a>
<a href="https://github.com/devilscrapes/business-filings-leads-scraper-javascript/actions/workflows/ci.yml"><img alt="CI" src="https://github.com/devilscrapes/business-filings-leads-scraper-javascript/actions/workflows/ci.yml/badge.svg"></a>
</p>

Use this repository to wire the **[New Business Filings Leads Scraper](https://apify.com/DevilScrapes/new-business-filings-leads-scraper)** into a Node.js service. Your code sends a formation date range and a list of state codes; it receives one object per newly registered New York, Colorado, Connecticut or Oregon company, including entity type, address, registered agent where published, and a URL that resolves to the underlying state record. The heavy lifting runs on Apify, so a serverless cron, a queue worker or an n8n step only triggers the run and consumes rows. For a quick look, `node index.js` drops the dataset into a JSON file and a CSV file beside the script.

> **Actor:** [New Business Filings — US State Registry Leads Feed](https://apify.com/DevilScrapes/new-business-filings-leads-scraper) · **Publisher:** [Devil Scrapes](https://apify.com/DevilScrapes) · **Also in Python:** [business-filings-leads-scraper-python](https://github.com/devilscrapes/business-filings-leads-scraper-python)

## 🔥 What you get

- **Query by formation date, not by name**: you supply a window with `dateFrom` and an optional `dateTo`, and the Actor asks each state dataset for every entity whose filing date falls inside it, newest first.
- **Four registries, one set of keys**: New York, Colorado, Connecticut and Oregon all label their columns differently; each row comes back with the same twelve fields regardless of state.
- **Oregon rows merged per company**: Oregon publishes one line per associated name, so the Actor groups them by registry number and lifts the principal (or mailing) address and the registered agent into a single record.
- **New York address fallback**: when a New York filer leaves the location columns blank, `principal_address` is filled from the service-of-process address instead of coming back empty.
- **Exact entity-type filter**: `entityTypes` is passed to each state as written, so `DLLC` narrows Colorado and `DOMESTIC LIMITED LIABILITY COMPANY` narrows New York.
- **A state that misbehaves is skipped, not fatal**: throttling and server errors are retried with growing pauses (honouring `Retry-After`), and a state that still errors is logged while the others finish.
- **Provenance on every record**: `source_record_url` queries the exact raw row and `registry_url` opens the state's dataset page.

## 🎯 Use cases

### Push new companies into a CRM every morning

A scheduled function asks for the last two days, skips any `jurisdiction` and `entity_id` pair it has already synced and creates lead records for the rest. Mapping the fields onto a HubSpot, Salesforce or Pipedrive lead is a short function.

### Trigger onboarding outreach through a webhook

Post each unseen company to Zapier, Make or your own endpoint as soon as the run finishes. Downstream steps can geocode the address, look for a website and queue a welcome letter or call.

### Build a territory alert for field reps

Parse the city and ZIP out of `principal_address`, match them to sales territories and message the owning rep in Slack. New storefronts and service businesses in their area reach them while the filing is still days old.

### Back a 'recently formed' search in your product

Store the rows in Postgres keyed by state and id, and expose a filter by state, legal form and formation date. Fintech and SaaS teams can drive activation campaigns from that list.

## ⚡ Quick start

You need an Apify account and an API token from [Console → Settings → Integrations](https://console.apify.com/settings/integrations). Apify's free plan includes monthly usage credit; this Actor's event prices are listed under Pricing below.

```bash
git clone https://github.com/devilscrapes/business-filings-leads-scraper-javascript.git
cd business-filings-leads-scraper-javascript
npm install
export APIFY_TOKEN=your_token_here
node index.js                       # uses input.json
node index.js my-input.json --out data --max-charge 2
```

The script starts the [New Business Filings — US State Registry Leads Feed](https://apify.com/DevilScrapes/new-business-filings-leads-scraper), waits for it to finish, prints a link to the run in Apify Console and saves every result to `results.json` and `results.csv`. It exits with code 1 and the run's status message if the run fails.

## 🧩 The core call

Here is the minimal version, without CLI flags or exports. The promise from `call()` settles when the run stops, and `listItems()` fetches the stored rows.

```javascript
import { ApifyClient } from 'apify-client';

const client = new ApifyClient({ token: 'YOUR_APIFY_TOKEN' });
const run = await client.actor('DevilScrapes/new-business-filings-leads-scraper').call({"dateFrom": "2026-10-01", "jurisdictions": ["NY", "CO"], "maxResultsPerJurisdiction": 5, "proxyConfiguration": {"useApifyProxy": false}});
const { items } = await client.dataset(run.defaultDatasetId).listItems();
console.log(items);
```

## 📥 Input

`index.js` passes `input.json` through as written. It requests filings from New York and Colorado dated 1 October 2026 or later, five per state, so one run costs about a quarter. Leave `dateTo` out for an open-ended window ending today, or set both dates for a fixed backfill.

```json
{
  "dateFrom": "2026-10-01",
  "jurisdictions": [
    "NY",
    "CO"
  ],
  "maxResultsPerJurisdiction": 5,
  "proxyConfiguration": {
    "useApifyProxy": false
  }
}
```

| Field | Type | Required | Default | Allowed values |
|---|---|---|---|---|
| `dateFrom` | string | yes | `"2026-07-01"` | — |
| `dateTo` | string | no | — | — |
| `jurisdictions` | array | no | `["NY", "CO", "CT", "OR"]` | — |
| `entityTypes` | array | no | — | — |
| `maxResultsPerJurisdiction` | integer | no | `1000` | — |
| `proxyConfiguration` | object | no | `{"useApifyProxy": false}` | — |

## 📤 Output

You get one plain object per company. All twelve keys are always present, though several can be `null` depending on the state, so guard address and agent access with `??`.

| Field | Type | Always present |
|---|---|---|
| `entity_id` | string | yes |
| `entity_name` | string | yes |
| `jurisdiction` | string | yes |
| `entity_type` | string or null | no |
| `formation_date` | string | yes |
| `status` | string or null | no |
| `principal_address` | string or null | no |
| `registered_agent_name` | string or null | no |
| `registered_agent_address` | string or null | no |
| `source_record_url` | string | yes |
| `registry_url` | string | yes |
| `scraped_at` | string | yes |

## 💰 Pricing

| Event | Price (USD) |
|---|---|
| Actor start | $0.2 |
| Result row emitted | $0.005 |

Prices as configured on 2026-08-20. The [Actor page](https://apify.com/DevilScrapes/new-business-filings-leads-scraper) always shows the current price. Runs are billed to the Apify account that owns the token. Use `--max-charge` to cap what a single run can cost.

## 🛠️ Working with the data in JavaScript

This Node.js snippet makes repeated runs incremental. It remembers every `jurisdiction:entity_id` key in `seen.json`, writes only unseen companies to `new-leads.ndjson` (one JSON object per line, easy to stream to a webhook) and prints a count per state. It relies on built-ins only; `Object.groupBy` needs Node 21 or later.

```javascript
import { readFile, writeFile } from 'node:fs/promises';

const rows = JSON.parse(await readFile('results.json', 'utf8'));
const seen = new Set(JSON.parse(await readFile('seen.json', 'utf8').catch(() => '[]')));

const fresh = [];
for (const row of rows) {
    const key = `${row.jurisdiction}:${row.entity_id}`;
    if (seen.has(key)) continue;
    seen.add(key);
    fresh.push({
        key,
        company: row.entity_name,
        state: row.jurisdiction,
        formed: row.formation_date,
        type: row.entity_type ?? 'unknown',
        address: row.principal_address ?? row.registered_agent_address ?? null,
        evidence: row.source_record_url,
    });
}

await writeFile('new-leads.ndjson', fresh.map((lead) => JSON.stringify(lead)).join('\n') + (fresh.length ? '\n' : ''));
await writeFile('seen.json', JSON.stringify([...seen]));
const byState = Object.groupBy(fresh, (lead) => lead.state);
console.log(Object.fromEntries(Object.entries(byState).map(([state, list]) => [state, list.length])));
```

## ❓ FAQ

### Should I use a fixed or a rolling date window?

For a recurring feed, compute `dateFrom` as two or three days ago on every run and dedupe on state plus id. Registries publish with a short delay, and the overlap catches late arrivals without paying twice for rows you keep.

### Can I run it from n8n or Make?

Yes. Both have Apify integrations that start an Actor with a JSON input and read its dataset. This repository is for when you would rather own the code.

### Is an entity's status kept up to date?

`status` is the registry's label at the moment of the run. The Actor is a formation feed, so recheck older companies separately if standing matters to you.

### How do I cap spend while testing?

Two knobs: a low `maxResultsPerJurisdiction` in the input, and the script's spending ceiling flag, such as `--max-charge 0.3`, which Apify enforces on the run itself.

## 🧯 Troubleshooting

- **`Set APIFY_TOKEN first`**: export the token in the same shell, or put it in a local `.env` file your shell loads. Never commit it.
- **Run finished with `FAILED` or `ABORTED`**: open the printed Console link; the run log says why. Input validation errors name the field.
- **Fewer rows than expected**: a `--max-charge` cap also limits how many results can be charged. Raise the cap or lower the limits in your input.
- **Something looks wrong in the data**: open an issue on the [Actor page](https://apify.com/DevilScrapes/new-business-filings-leads-scraper) so the maintainers see it.

## 🔗 More from Devil Scrapes

- [New Business Filings — US State Registry Leads Feed on Apify Store](https://apify.com/DevilScrapes/new-business-filings-leads-scraper)
- [Python version of this example](https://github.com/devilscrapes/business-filings-leads-scraper-python)
- [All Devil Scrapes Actors](https://apify.com/DevilScrapes)
- [Apify web scraping cookbook](https://github.com/devilscrapes/apify-web-scraping-cookbook)

Related Actors:

- [UCC Lien Filing Scraper — CT + CO State Portals](https://apify.com/DevilScrapes/ucc-lien-filing-scraper) — Connecticut and Colorado lien filings, a signal that a young company has taken on secured debt.
- [US Building Permit Leads Scraper (Socrata)](https://apify.com/DevilScrapes/us-building-permit-leads-scraper) — permit records for spotting new businesses fitting out premises.
- [Socrata Open Data API Scraper](https://apify.com/DevilScrapes/socrata-open-data-scraper) — any other dataset on these and hundreds more Socrata portals.

---

MIT licensed example code. The Actor itself runs on [Apify](https://apify.com/DevilScrapes/new-business-filings-leads-scraper) and is maintained by [Devil Scrapes](https://apify.com/DevilScrapes). Not affiliated with the New York, Colorado, Connecticut or Oregon Secretary of State offices and their open-data portals.
