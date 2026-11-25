# ScraperAPI webhook setup: how do you get results POSTed to your endpoint, what happens when delivery fails, and when should you use a callback instead of polling?

Add a `callback` object to the JSON body you send to `https://async.scraperapi.com/jobs`, and ScraperAPI will POST the finished job to your endpoint:

json
{
  "apiKey": "YOUR_KEY",
  "url": "https://example.com/product/123",
  "callback": {
    "type": "webhook",
    "url": "https://yourapp.com/scraperapi/callback"
  }
}


Three things decide whether this works in production. Only successful jobs reach your URL by default. Delivery is attempted three times, and after the third failure the job is cancelled. Results stay retrievable for 72 hours at most, with only the first 24 guaranteed.

The rest of this page covers the exact payload, the failure-report parameter and a spelling conflict between two official pages, batch behaviour up to 50,000 URLs, the credit costs that apply, and what to use instead when the scrape needs to run on a schedule.

> **Quick pick:** for a single URL where you want the HTML back in the same response, the Standard API is the simpler call. The Async API plus a webhook earns its keep once you're pushing 20 or more URLs, the target is slow or flaky, or you need PDFs and images. Any paid tier can send callbacks; the trial gives you 5,000 credits over 7 days with no card. 👉 [Check the current plans and start the trial](https://www.scraperapi.com/?fp_ref=coupons)

## What the callback object does, and what it doesn't

`callback.type` accepts one value today: `webhook`. ScraperAPI has said database callbacks and S3 delivery are planned, but neither is live, so don't design around them yet.

Once a callback is attached you no longer need to poll `statusUrl`. It keeps working, and a GET on it returns the same content the webhook receives, which is handy for debugging a callback that never arrived.

The API key field is `apiKey` in camelCase. The Standard API uses `api_key`. Mixing them up produces a 401 that looks like a bad key.

Submitting a job returns a small acknowledgement, not the scrape:

json
{"id": "...", "status": "running", "statusUrl": "https://async.scraperapi.com/jobs/...", "url": "https://example.com/product/123"}


Store that `id` and `statusUrl` somewhere durable. They're your fallback if the callback is lost.

## Which plan the webhook work actually needs

Callbacks aren't a paid add-on. What your plan determines is how many jobs you can have in flight and how many credits you burn per request.

| Plan | API credits | Concurrent threads | Geotargeting | Pricing |
| --- | --- | --- | --- | --- |
| Free trial | 5,000 (7 days) | — | US & EU | [ Check trial terms](https://www.scraperapi.com/?fp_ref=coupons) |
| Hobby | 100,000 | 20 | US & EU | [ See Hobby pricing](https://www.scraperapi.com/?fp_ref=coupons) |
| Startup | 1,000,000 | 50 | US & EU | [ See Startup pricing](https://www.scraperapi.com/?fp_ref=coupons) |
| Business | 3,000,000 | 100 | Country-level | [ See Business pricing](https://www.scraperapi.com/?fp_ref=coupons) |
| Scaling | 5,000,000 | 200 | Country-level | [ See Scaling pricing](https://www.scraperapi.com/?fp_ref=coupons) |
| Professional | 10,500,000 | 300 | Country-level | [ See Professional pricing](https://www.scraperapi.com/?fp_ref=coupons) |
| Advanced | 21,500,000 | 500 | Country-level | [ See Advanced pricing](https://www.scraperapi.com/?fp_ref=coupons) |
| Enterprise | 22,000,000+ | 500+ | Country-level | [ See Enterprise terms](https://www.scraperapi.com/?fp_ref=coupons) |

Analytics history is 30 days on Hobby and Startup and unlimited from Business upward. ScraperAPI quotes the dollar figure per plan on its pricing page rather than in the API documentation, and the billing term changes it, so confirm the current number on the plan page before you commit.

Two things sit outside the table. Pay-as-you-go, which keeps requests flowing past your credit cap at a fixed per-credit rate up to a monthly limit you set, is available on Scaling, Professional, Advanced and Enterprise only. Auto-upgrade to the next tier when credits run out is opt-in on Hobby through Professional. If you expect uneven monthly volume, that difference matters more than the credit totals.

## The payload that lands on your endpoint

A finished job arrives in this shape:

json
{
  "id": "...",
  "status": "finished",
  "statusUrl": "https://async.scraperapi.com/jobs/...",
  "url": "https://example.com/product/123",
  "response": {
    "headers": {
      "content-type": "text/html",
      "sa-final-url": "...",
      "sa-statuscode": "200"
    },
    "body": "<!doctype html>...",
    "statusCode": 200
  }
}


Read the HTML from `response.body`. `response.statusCode` is what the target site returned. The `sa-statuscode` and `sa-final-url` headers tell you the same thing after redirects, which is useful when the site bounces you to a consent page or a locale variant.

Binary targets work differently. For PDFs and images the bytes arrive in `response.base64EncodedBody` instead of `body`, so decode and write to disk:

python
pdf_bytes = base64.b64decode(payload["response"]["base64EncodedBody"])
open("report.pdf", "wb").write(pdf_bytes)


A job can be `finished` and still be useless. If the target blocked you, `status` is `finished` with `statusCode: 403`. Retry that URL with `premium: true` in `apiParams` before you conclude the URL is dead.

## Getting failed jobs delivered too

By default only successful jobs trigger a callback. To receive failed ones, add `expectUnsuccessReport: true` to the request body.

Two official pages disagree on that name. The complete request-body reference and the callbacks-and-params page both spell it `expectUnsuccessReport`, and the first of those lists it twice, in the full parameter block and in the Async-exclusive parameter table. A separate, older callbacks page spells it `expectsUnsuccessReport` with an s. The parameter block version has more support across the docs, so start there, and if the API complains about an unknown field, try the other spelling before assuming the feature is broken.

The failure payload is short:

json
{
  "id": "db726d84-2609-4709-93cb-88c4a9fa4288",
  "attempts": 50,
  "status": "failed",
  "statusUrl": "https://async.scraperapi.com/jobs/db726d84-...",
  "failReason": "failed_due_to_timeout",
  "url": "https://example.com"
}


`failed_due_to_timeout` means the job exhausted the internal retry window, which runs up to 24 hours, without a usable response. There's no `response` object in a failure payload, so your handler needs a branch that logs the reason and moves on rather than trying to parse HTML.

## What happens after three failed deliveries

ScraperAPI tries your webhook URL three times, then cancels the job. Your endpoint has to be reachable from the public internet, and it has to absorb whatever volume your batch produces. An endpoint behind a VPN, a staging host with basic auth at the proxy level, or a serverless function with a 3-second timeout are the usual culprits.

If you need authentication, embed it in the URL:


https://user:password@yourcompany.com/scraperapi


For a first test without standing up a server, point the callback at webhook.site and watch the request arrive.

A cancelled job is the expensive failure mode. The scrape may have succeeded and the credits may already be spent, but the result is gone once the three delivery attempts fail. Two habits reduce the damage: persist the `statusUrl` from the submission response and reconcile unfinished jobs on a schedule, and keep your handler's first action cheap, such as writing the raw body to a queue and returning 200 before doing any parsing.

## Batch jobs up to 50,000 URLs

The batch endpoint is `POST https://async.scraperapi.com/batchjobs`. It takes a `urls` array and returns one `{id, status, statusUrl, url}` object per submitted URL. The ceiling is 50,000 URLs per request; beyond that, split into multiple batch calls.

Attach the same `callback` object to a batch request and every job in it reports back to that URL. Large batches are exactly where callbacks earn their place: polling 10,000 status URLs one at a time is slow, and that's the official recommendation, not a preference.

Because all those callbacks arrive at one endpoint with no obvious grouping, use `meta` to tag the run:

json
{"meta": {"batchId": "run-2026-11", "sourceFile": "urls.csv"}}


Whatever you put in `meta` comes back unchanged in every response and callback, which turns an anonymous stream of payloads into something you can route and count. `meta` and `timeoutSec`, which overrides the default job timeout in seconds, are Async-only parameters alongside `expectUnsuccessReport`.

## The retention window people miss

Job results are stored for a maximum of 72 hours, and only the first 24 are guaranteed. After that the data is deleted and the job has to be resubmitted, at full credit cost.

This is the single strongest argument for webhooks over polling on anything long-running. A pipeline that submits a batch on Friday and reads the results on Monday will find empty responses. If your workflow can't consume results within a day, push them into your own storage the moment the callback arrives.

## Credits, caps and submission errors

Async jobs use the same rates as the Standard API. Current per-request costs on the pricing page:

| Request type | Credits |
| --- | --- |
| Standard page | 1 |
| E-commerce (Amazon, Walmart, eBay) | 5 |
| SERP (Google, Bing) | 25 |
| Social (LinkedIn) | 30 |
| Bot-protection bypass | 10 |
| JS rendering (`render: true`) | 10 |
| Premium proxy (`premium: true`) | 10 |
| Screenshot | 10 |
| Premium proxy + rendering | 25 |
| Ultra premium (`ultra_premium: true`) | 30 |
| Ultra premium + rendering | 75 |

Geotargeting, sessions, device type and output format add nothing. Failed requests cost 0, and async jobs that fail after exhausting every retry aren't charged, so a `failed_due_to_timeout` callback doesn't drain your balance.

Set `max_cost` inside `apiParams` to cap a single request. Anything that would exceed the cap returns a 403 instead of quietly spending more. For a batch of 50,000 URLs where a few hundred hit bot protection, that's the difference between a predictable bill and a surprise.

Submission errors are worth knowing by heart:

- **401**: the API key is wrong. Check for `api_key` where `apiKey` belongs.
- **403**: out of credits or over a plan limit.
- **429**: too many concurrent submissions. Back off and resubmit in smaller batches; the request is refused rather than queued, and there's no extra charge for hitting the limit.

👉 [Compare the credit allowances against your batch size](https://www.scraperapi.com/?fp_ref=coupons)

## When the scrape needs to run on a schedule

Everything above is one-off work: you know the URLs, you submit them, you get results. Recurring scrapes belong to DataPipeline, a separate product with its own API base and its own webhook settings.

A DataPipeline project takes a `webhookOutput` object:

json
"webhookOutput": {
  "url": "https://yourapp.com/pipeline-results",
  "webhookEncoding": "multipart_form_data_encoding"
}


Results POST to that URL as each run completes. Leave `webhookOutput` out and results sit in the dashboard for download for 30 days, then get deleted automatically.

Input can be a webhook as well. With `"projectInput": {"type": "webhook", "webhookUrl": "..."}`, ScraperAPI polls your URL for the item list when the job starts, one item per line with no commas, which suits lists generated on the fly.

Scheduling options are `once`, `hourly`, `daily`, `weekly`, `monthly` and `cron`. Every recurring option requires a paid plan, and the free tier is limited to one-time runs with 5 concurrent connections. A single job accepts up to 100,000 input items, a direct list input caps at 500, and CSV uploads up to 100,000 rows are dashboard-only because the API takes list inputs. Only one job can run per project at a time; cancelling lets in-flight requests finish before the status settles on "Cancelled". Notifications are configurable through `notificationConfig` with `never`, `with_every_run`, `daily` or `weekly` for success and failure separately.

DataPipeline bills the same per-request credit rates, counted across every request in a run, and the dashboard previews the estimated cost before you launch. It charges for 200 and 404 responses; other failures are free.

## Picking the right endpoint

Match the tool to the shape of the job:

- One URL, result needed in the same response: Standard API.
- A known list of URLs, one-off, with or without a webhook: Async API, batch endpoint above 1 URL.
- Recurring schedule, results pushed to your endpoint: DataPipeline.
- Unknown URLs that require crawling a site's links: Crawler.

Before you hand over a card, confirm two numbers: the concurrency limit on the tier you're choosing, since 429s under load are the most common complaint in async work, and whether pay-as-you-go is available on that tier, since it isn't on Hobby, Startup or Business. Both are on the plan pages.

👉 [Start with the 5,000-credit trial and test your callback endpoint first](https://www.scraperapi.com/?fp_ref=coupons)
