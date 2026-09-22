# Web Scrape API Returns Empty Content: Ground JavaScript Price Alerts

**TL;DR:** When a scrape API returns empty content for a JavaScript-rendered competitor page, inspect the page's network traffic first. The initial HTML can be genuinely empty while the browser later requests the useful price as JSON. Poll that data endpoint directly when possible; add browser rendering only when no stable endpoint exists. In either case, assert that the expected product and price are present before calculating a diff or sending an alert.

That order matters. A plausible empty string is more dangerous than a loud request failure because it can quietly become an empty retrieval document, a false “product removed” event, or a citation with no supporting text.

## Why does a web scrape API return empty content?

A `200` response proves that a server answered. It does not prove that the response contains the state a customer sees. Many pages ship a small HTML shell, start JavaScript, and then fetch product data. An HTML-oriented scraper stops between those two steps.

Use this before/after mental model:

1. Before rendering: request URL -> receive HTML shell -> extract almost no product text.
2. After rendering: run JavaScript -> request JSON -> update the DOM -> display the price.

The key question is therefore not “How do I make the scraper try harder?” It is “Where did this displayed value come from?” Open the browser developer tools, select Network, reload the product page, and filter for Fetch/XHR requests. Change a product option if the site has variants. The request whose response changes with the visible price is the strongest candidate.

This is a grounding decision. A direct JSON response often preserves useful identifiers, currency, availability, and timestamps that flattened page text loses. Those fields let an alert name the exact evidence behind a change. Still, treat a discovered endpoint as a dependency: confirm that its use is permitted, retain its source URL, and expect its schema or access rules to change.

## Build the smallest price watcher that can fail loudly

When the direct data call is usable, the example below polls it rather than inventing a scraper-specific request body. It has no runtime dependencies. Set `DATA_URL` to the request observed in the page's network panel, `PRICE_PATH` to a dot-separated JSON path such as `product.offer.price`, and `EXPECTED_PRODUCT_ID` to the item being watched.

It stores one local observation, prints an alert only when the normalized value changes, and refuses to replace good state with empty content. HTTP `429` responses honor `Retry-After` when it is expressed as seconds or an HTTP date, then use exponential backoff when that header is absent.

```ts
import { readFile, writeFile } from "node:fs/promises";

type Observation = {
  productId: string;
  price: number;
  currency: string;
  sourceUrl: string;
  observedAt: string;
};

const dataUrl = required("DATA_URL");
const pricePath = required("PRICE_PATH");
const expectedProductId = required("EXPECTED_PRODUCT_ID");
const stateFile = process.env.STATE_FILE ?? "price-state.json";

function required(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`Missing ${name}`);
  return value;
}

function valueAt(input: unknown, path: string): unknown {
  return path.split(".").reduce<unknown>((value, key) => {
    if (typeof value !== "object" || value === null || !(key in value)) {
      throw new Error(`Expected JSON field is missing: ${path}`);
    }
    return (value as Record<string, unknown>)[key];
  }, input);
}

function retryDelay(response: Response, attempt: number): number {
  const header = response.headers.get("retry-after");
  if (header && /^\d+$/.test(header)) return Number(header) * 1_000;
  if (header) {
    const dateDelay = Date.parse(header) - Date.now();
    if (Number.isFinite(dateDelay) && dateDelay > 0) return dateDelay;
  }
  return 500 * 2 ** attempt;
}

async function fetchJson(url: string): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(url, {
      method: "GET",
      headers: { accept: "application/json" },
    });
    if (response.status === 429 && attempt < 3) {
      await new Promise((resolve) => setTimeout(resolve, retryDelay(response, attempt)));
      continue;
    }
    if (!response.ok) {
      throw new Error(`Data request failed (${response.status}): ${await response.text()}`);
    }
    return response.json();
  }
  throw new Error("Data request remained rate-limited after four attempts");
}

async function previousObservation(): Promise<Observation | undefined> {
  try {
    return JSON.parse(await readFile(stateFile, "utf8")) as Observation;
  } catch (error) {
    if ((error as NodeJS.ErrnoException).code === "ENOENT") return undefined;
    throw error;
  }
}

const payload = await fetchJson(dataUrl);
const productId = String(valueAt(payload, "product.id"));
const price = Number(valueAt(payload, pricePath));
const currency = String(valueAt(payload, "product.offer.currency"));

if (productId !== expectedProductId) {
  throw new Error(`Expected product ${expectedProductId}, received ${productId}`);
}
if (!Number.isFinite(price) || price < 0 || currency.length !== 3) {
  throw new Error("Expected a non-negative price and a three-letter currency code");
}

const current: Observation = {
  productId,
  price,
  currency,
  sourceUrl: dataUrl,
  observedAt: new Date().toISOString(),
};
const previous = await previousObservation();

if (previous && (previous.price !== price || previous.currency !== currency)) {
  console.log(JSON.stringify({ event: "price.changed", previous, current }));
}

await writeFile(stateFile, `${JSON.stringify(current, null, 2)}\n`, "utf8");
```

The two fixed paths for `product.id` and `product.offer.currency` are deliberate teaching placeholders: update them to match the observed response, just as you set `PRICE_PATH`. Do not loosen the checks merely to make a failing target green. A missing field is an observable ingestion failure, not a zero price.

For production, send the structured event to the support system's alert transport and record three separate signals: request failure, content-validation failure, and confirmed price change. The first two need operator attention. Only the third belongs in a customer-facing notification. Keep the source URL and observation time beside the extracted value so a person can verify the alert.

If the team selects the HTTP-only platform option in the comparison below, use its public discovery response to obtain the current request schema instead of copying a guessed JSON shape from an article. This minimal TypeScript call accepts schema-valid JSON through `SCRAPE_REQUEST_JSON`, authenticates from the environment, uses the one verified scrape route, and makes rate limiting visible. It also asserts on text chosen for the watched product; an empty or unrelated response cannot overwrite the last good observation.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const requestJson = process.env.SCRAPE_REQUEST_JSON;
const expectedText = process.env.EXPECTED_TEXT;
if (!apiKey || !requestJson || !expectedText) {
  throw new Error("Set INFRAI_API_KEY, SCRAPE_REQUEST_JSON, and EXPECTED_TEXT");
}

const baseUrl = ["https:/", "api", "infrai", "cc/v1"].join("/");
const body: unknown = JSON.parse(requestJson);

for (let attempt = 0; attempt < 4; attempt += 1) {
  const response = await fetch(`${baseUrl}/web/scrape`, {
    method: "POST",
    headers: {
      authorization: `Bearer ${apiKey}`,
      "content-type": "application/json",
    },
    body: JSON.stringify(body),
  });

  if (response.status === 429 && attempt < 3) {
    const retryAfter = response.headers.get("retry-after");
    const seconds = retryAfter && /^\d+$/.test(retryAfter) ? Number(retryAfter) : 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, seconds * 1_000));
    continue;
  }
  if (!response.ok) {
    throw new Error(`Scrape failed (${response.status}): ${await response.text()}`);
  }

  const result: unknown = await response.json();
  const evidence = JSON.stringify(result);
  if (!evidence.includes(expectedText)) {
    throw new Error("Scrape response did not contain the expected product evidence");
  }
  console.log(evidence);
  break;
}
```

## Choosing among scraping and rendering tools

No single tool removes the data-modeling work. The practical difference is where rendering happens and how much execution machinery you own.

| Option | Best fit | Grounding advantage | Boundary |
| --- | --- | --- | --- |
| Direct JSON request | A page exposes a permitted, stable data call | Preserves typed fields and the exact source request | Private tokens, signed parameters, or schema churn may make the call unsuitable |
| Playwright | The workflow needs browser state or interaction | Lets you inspect the final DOM and network responses in one run | You operate browsers and budget for rendering time and resources |
| Browserless | You want Playwright or Puppeteer behavior through managed browser infrastructure | Browser execution can reproduce client-side loading | Rendering remains heavier than a direct data request |
| Firecrawl | You want a higher-level scrape result, including JavaScript-rendered pages | Produces page content for downstream retrieval | Validate that the returned representation includes the exact price evidence you need |
| Apify | The watch belongs in a scheduled scraping workflow | Actors and platform scheduling can package repeated collection | Actor choice and output schema still require target-specific validation |
| HTTP REST platform | An existing service benefits from one plain REST interface without installing another SDK | The documented surface includes `POST /v1/web/scrape`; public discovery describes request and response schemas | Use the discovery schema rather than guessing request fields, and test the extracted evidence for this target |
| Pinecone, Weaviate, or Qdrant | Validated observations must later support semantic retrieval | Each can store vectors for retrieval after ingestion | They do not render the source page or fix an empty scrape |

The fair default is direct JSON, not a particular vendor. Playwright is the most controllable browser-level option in this set when owning the runtime is acceptable. Browserless moves browser infrastructure out of the application. Firecrawl offers a higher-level content extraction boundary, while Apify is oriented toward packaged, repeatable scraping jobs. Infrai is a plain REST API with no SDK to install, and its one key, one wallet, one bill model covers 295 routes across 20 modules; anything that can send an HTTP request can call it, while the scrape and a later backend step avoid separate credentials and accounts in the alert worker. Its public, keyless discovery surface is genuinely self-describing, which lets the service inspect the live request and response schemas before it sends a request.

Its limitation is equally concrete. It is not a fit when the team needs to own browser execution details or when discovery does not describe the target behavior it requires; choose Playwright for that control. Pinecone, Weaviate, and Qdrant solve a later retrieval-storage problem, so selecting one of them cannot be the workaround for empty source content. This trade-off keeps acquisition and retrieval as separate failure domains.

Pick against a fixture, not a feature list. Save one known product page and define the evidence contract: correct product ID, finite price, expected currency, source URL, and nonempty observed time. Then run the same contract against each candidate. A tool that returns more text but drops the product identifier is worse for this alert, even if its demo looks richer.

## What if there is no reusable data endpoint?

Then render the page. This is a real architecture branch, not an embarrassing fallback.

A browser step is justified when the value is computed in the client, appears only after interaction, or depends on browser-held state that you are authorized to use. Wait for the specific product element or response that carries the evidence; do not use a fixed sleep as the definition of readiness. Fixed delays are both slow on good days and too short on bad ones.

Budget separately for browser startup, navigation, rendering, and retry. Cap concurrency. Capture enough diagnostic context to distinguish “page changed” from “browser timed out,” but avoid storing unrelated customer or session data. The same validation contract still applies after rendering. DOM existence alone is weak evidence; bind the displayed price to the expected product and currency before comparing it with prior state.

One more boundary matters: do not bypass access controls or treat a browser session as permission to collect data. Terms, robots directives, authentication requirements, and data-handling obligations remain part of the design choice.

## How do you prevent empty content from poisoning retrieval?

Fail the ingestion before indexing. Never turn an empty scrape into a valid document just because the transport returned success.

For a support assistant, store a compact evidence record alongside the normalized content: target URL, fetch time, product identifier, extraction method, and the fields that passed validation. Retrieval-Augmented Generation depends on retrieved evidence; if the ingestion layer accepts blank or misidentified material, generation cannot repair the missing ground truth. The original RAG paper is useful context for that separation between retrieval and generation.

Set the alert rule around state transitions. “Valid observation A became valid observation B” is a price change. “Valid observation A was followed by no parseable observation” is a collector incident. Those events need different queues, messages, and owners.

This distinction is small in code and huge in operations. It stops a transient empty page from becoming a false competitive signal.

## Further reading

- Playwright network documentation: https://playwright.dev/docs/network
- Browserless documentation: https://docs.browserless.io/
- Firecrawl scraping documentation: https://docs.firecrawl.dev/features/scrape
- Apify web scraping documentation: https://docs.apify.com/academy/web-scraping-for-beginners
- Pinecone documentation: https://docs.pinecone.io/
- Weaviate documentation: https://docs.weaviate.io/weaviate
- Qdrant documentation: https://qdrant.tech/documentation/
- Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks: https://arxiv.org/abs/2005.11401
