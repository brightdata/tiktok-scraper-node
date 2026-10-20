# For coding agents working in this repository

Read this before changing anything. Every line below was verified against the
live API, the installed SDK (`@brightdata/sdk` 1.2.0) or the Bright Data CLI
0.4.0 on 2026-10-20. Where the API reference pages disagree with the live API,
the live API is what is written here.

## Auth

- The SDK reads `apiKey`, then `BRIGHTDATA_API_TOKEN` or `BRIGHTDATA_API_KEY`
  from the environment, then the credentials `bdata login` stored. Setting the
  variable, or running the login once, is the one step a human must do; never
  paste a key into code.
- The JavaScript SDK does not read a `.env` file. Node 20 loads one for it:
  `node --env-file=.env`.
- Always construct the client as `new bdclient({ autoCreateZones: false })`.
  Left on, the SDK creates Web Unlocker and SERP zones on the first request.
  This repository never uses a zone, and zone creation fails on accounts
  without a payment method.
- The raw HTTP snippets in the README read `BRIGHTDATA_API_TOKEN` only. They
  do not see the CLI login.

## How the API behaves

- TikTok lives on `client.scrape.tiktok`. It has `profiles`, `posts`,
  `comments` and three Fast API calls. It has no discovery method and no TikTok
  Shop method; those are raw HTTP.
- Recent posts come from `collectPostsByProfileFast([{ url, num_of_posts }],
  { async: true, includeErrors: true })`. Without `async: true` it waits on the
  sync endpoint instead of returning a `ScrapeJob`. The orchestrated
  `postsByProfileFast([url])` cannot pass `num_of_posts`, so it bills every
  post the account has. It returns 36 fields; Posts discovery and
  `posts([url])` return 44.
- Pass `includeErrors: true` on every collect call. Unset, a dead account
  comes back as zero rows. A handle that does not exist comes back as a row
  whose error is "User is banned".
- Poll options are milliseconds, not seconds.
- Many inputs in one call is one job. Prefer that over a loop. For large runs
  keep the `snapshotId` and fetch later; snapshots stay downloadable 30 days.
- One credit per record. Comments cost one per comment, and `comments()`
  cannot cap them: check `comment_count` on the post first. Raw HTTP with
  `limit_per_input` does cap them; 3 on a 344-comment post returned 3 rows.
- `account_id` is the handle. `profile_username` is the display name.
- Input names and `discover_by` values, as the API validates them. Several API
  reference pages are wrong:
  - Posts discover by URL takes `URL`, upper case. `url` is a 400:
    "This input should not contain a url field".
  - Shop discovery is `discover_by=category`, `keyword` or `shop`. The pages'
    `category_url` and `shop_url` are a 400: "Incorrect discovery collector
    id Available types: category, keyword, shop".
  - Shop by store takes `url`, not `shop_url`. Shop by category takes
    `category_url`. Shop by keyword takes `keyword`; `num_of_products` is a 400.
  - Profiles discover by search URL requires `country`.
  - `bdata pipelines info <dataset_id> <method>` prints the live input schema.
- Shop discovery is slow: 18 to 20 minutes for two products by keyword or by
  category.
- The schema changes without notice. Never hardcode a field list. Read it with
  `client.datasets.tiktokPosts.getMetadata()`. It returns `{ id, fields }`,
  and `fields` is an object keyed by field name, 44 of them on 2026-10-20.
- The full documentation index, one `.md` page per entry:
  https://docs.brightdata.com/llms.txt

## Working here

- `npm test` runs offline and needs no token. `npm run lint` must pass.
- CI installs from the README's own commands on an empty machine. A weekly
  workflow executes every fenced block in the README against the real API, and
  a daily one runs a smaller live check.
  A snippet must be complete and paste-able on its own, and its shown output
  must be real.
- The field table in the README sits between `<!-- fields:start -->` and
  `<!-- fields:end -->` and is regenerated daily. Do not edit it by hand.
- The "last verified" badge line at the top of the README is rewritten by the
  daily run. Do not edit it by hand.
- Keep it small. Do not add retries, deduplication, scheduling, databases or
  concurrency.
