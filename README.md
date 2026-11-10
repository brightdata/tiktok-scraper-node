# TikTok scraper for Node.js and JavaScript

Scrape TikTok profiles, videos, comments and TikTok Shop products to JSON with Node.js and JavaScript. No TikTok login, no browser.
Built on the [Bright Data TikTok Scraper API](https://brightdata.com/products/web-scraper/tiktok?utm_source=github&utm_medium=referral&utm_content=tiktok-scraper-node). New accounts get 5,000 free credits a month.

[![Live check](https://github.com/brightdata/tiktok-scraper-node/actions/workflows/live.yml/badge.svg)](https://github.com/brightdata/tiktok-scraper-node/actions/workflows/live.yml)
[![last verified](https://img.shields.io/badge/last%20verified-10%20Nov%202026-brightgreen)](https://github.com/brightdata/tiktok-scraper-node/actions/workflows/live.yml) <!-- verified: rewritten by the daily run -->

[Quickstart](#how-to-scrape-tiktok-data-with-nodejs) · [No login](#scrape-tiktok-without-login-or-an-access-token) · [CLI](#bulk-tiktok-scraper-cli) · [Endpoints](#scrape-tiktok-posts-profiles-comments-and-tiktok-shop) · [Data](#tiktok-post-data-fields-json) · [Errors](#tiktok-scraper-errors) · [MCP and Claude Code](#tiktok-mcp-server-and-claude-code-skill) · [Free credits](#is-this-tiktok-scraper-free) · [Docs](https://docs.brightdata.com/products/scrapers/tiktok/introduction) · [Support](#support)

TikTok profiles, posts, comments and TikTok Shop products as JSON, in
JavaScript. No TikTok login, no browser. Built on the
[Bright Data TikTok Scraper API](https://brightdata.com/products/web-scraper/tiktok?utm_source=github&utm_medium=referral&utm_content=tiktok-scraper-node).

Uses the [Bright Data JavaScript SDK](https://github.com/brightdata/sdk-js).
Full API docs:
[TikTok Scraper API](https://docs.brightdata.com/products/scrapers/tiktok/introduction).

Also here: a one-command CLI for recent posts, and the
[Bright Data CLI](#tiktok-mcp-server-and-claude-code-skill), which needs no JavaScript at all.

## How to scrape TikTok data with Node.js

Node 20 or newer. This package is ESM, so use `import`, not `require`.

```bash
npm install @brightdata/sdk
export BRIGHTDATA_API_TOKEN=YOUR_API_KEY
```

Get a token from the
[Bright Data control panel](https://brightdata.com/cp/setting/users). This SDK
does not read a `.env` file; Node loads one for it with
`node --env-file=.env yourscript.mjs`.

Or skip the token. Run `npx -p @brightdata/cli bdata login` once: it opens a
browser, and from then on the SDK finds the stored credentials on its own, for
you and for any coding agent working in that terminal. Agents cannot click
through the login, so do it yourself first. The same CLI scrapes TikTok
directly; see [TikTok MCP server and Claude Code skill](#tiktok-mcp-server-and-claude-code-skill).

No account yet? [Create one](https://brightdata.com/cp/start); new accounts get
[5,000 free credits a month](https://docs.brightdata.com/general/account/billing-and-pricing/free-tier).

```javascript
import { bdclient } from "@brightdata/sdk";

const client = new bdclient({ autoCreateZones: false });
const job = await client.scrape.tiktok.collectPostsByProfileFast(
  [{ url: "https://www.tiktok.com/@tiktok", num_of_posts: 5 }],
  { async: true, includeErrors: true },
);
const result = await job.toResult({ pollTimeout: 900_000 });
if (!result.success) throw new Error(`${result.status}: ${result.error}`);
for (const post of result.data) {
  console.log(post.digg_count, post.comment_count, post.url);
}
await client.close();
```

```
12000 5090 https://www.tiktok.com/@tiktok/video/7694797676306615583
14800 3612 https://www.tiktok.com/@tiktok/video/7693184538704416031
12300 3222 https://www.tiktok.com/@tiktok/video/7692114317151423775
11200 1821 https://www.tiktok.com/@tiktok/video/7691450773619625247
9428 1391 https://www.tiktok.com/@tiktok/video/7691013160425327903
```

Expect one to two minutes: the API runs a job and `toResult` waits for it. One
[credit](https://brightdata.com/pricing/web-scraper) per post. `digg_count` is
TikTok's name for likes.

Four things in that snippet are not optional.

`autoCreateZones: false` stops the SDK creating zones on startup. The zones are
for Web Unlocker and SERP, two other Bright Data products this scraper never
touches. Creating one fails on accounts without a payment method.

`num_of_posts` caps the run. Without it the Fast API returns, and bills, every
post the account has. The orchestrated `postsByProfileFast([url])` cannot pass
it, so use the `collect` form.

`async: true` makes the call hand back a job. `includeErrors: true` makes the
API report a dead account as a row. Without it the row is dropped and you get
nothing.

`pollTimeout` is milliseconds, not seconds. A number copied from a Python
example expires before the first status check.

## Scrape TikTok without login or an access token

The TikTok Scraper API needs no TikTok account, no TikTok login and no TikTok
Research API access. The only credential is a Bright Data API key, set as
`BRIGHTDATA_API_TOKEN` or stored by `bdata login`.

## Bulk TikTok scraper CLI

The command in this repo runs the Node.js quickstart for several accounts at
once and writes one JSON file.

```bash
npm install -g github:brightdata/tiktok-scraper-node
tiktok-scraper tiktok khaby.lame
```

```
Fetching up to 5 recent posts per account, for: tiktok, khaby.lame
Usually one to three minutes each. One credit per post, 5,000 free per month.
asking  @tiktok...
got     @tiktok: 5 posts
asking  @khaby.lame...
got     @khaby.lame: 5 posts

Saved 10 posts as JSON to tiktok.json (36 fields per post)
```

In a terminal the `asking` lines are replaced by a spinner and a running
clock, updating in place, so you can see it is working.

```
--limit N    posts per account, default 5, minimum 1
--out PATH   output file, default tiktok.json
```

Import it instead of running it, for `ok`, `note` and `error` per account
instead of raw rows. `scrape` never rejects for one bad account; check `ok`
before reading `posts`:

```javascript
import { scrape } from "@brightdata/tiktok-scraper-node";

for (const outcome of await scrape(["tiktok", "zz_not_a_real_account_zz"], { limit: 1 })) {
  if (outcome.ok) {
    console.log(`${outcome.handle}: ${outcome.posts.length} posts`);
  } else {
    console.log(`${outcome.handle} failed: ${outcome.error}`);
  }
}
```

```
tiktok: 1 posts
zz_not_a_real_account_zz failed: User is banned
```

An account that does not exist comes back as "User is banned". An account with
nothing recent is a success with no posts.

## Scrape TikTok posts, profiles, comments and TikTok Shop

The Bright Data TikTok Scraper API has 12 endpoints, each with its own page in
the [API reference](https://docs.brightdata.com/api-reference/scrapers/social-media-apis/tiktok-profiles-collect-by-url).
This SDK wraps four of them, on `client.scrape.tiktok`. The other eight,
discovery and TikTok Shop, are raw HTTP; [the last section](#tiktok-shop-scraper-and-discovery-over-raw-http)
shows the one call they all share.

| you have | want | how |
| --- | --- | --- |
| a profile URL | recent posts, 36 fields | `collectPostsByProfileFast([{ url, num_of_posts: 5 }], { async: true, includeErrors: true })` |
| a profile URL | recent posts, 44 fields | raw HTTP, Posts `discover_by=profile_url` |
| a profile URL | the profile | `profiles([url])` |
| a TikTok search URL | matching profiles | raw HTTP, Profiles `discover_by=search_url` |
| post URLs | those posts, 44 fields | `posts([url, url])` |
| a keyword | matching posts | raw HTTP, Posts `discover_by=keyword` |
| a tag, music or explore URL | posts on that page | raw HTTP, Posts `discover_by=url` |
| a post URL | its comments | `comments([url])`, one credit per comment |
| a TikTok Shop product URL | the product | raw HTTP, Shop collect |
| a Shop category URL, keyword or store URL | products | raw HTTP, Shop `discover_by=category`, `keyword` or `shop` |

Every one of these is an asynchronous job. The API triggers it, the SDK polls,
and the call returns when it is ready. Every snippet below is complete and
needs only `@brightdata/sdk`, or nothing at all for raw HTTP: paste it as is.
Every one of them runs in Actions each Monday, a smaller check runs every
other day, and the badge at the top is the latest result.

The three short names `profiles`, `posts` and `comments` trigger, poll and
fetch in one call and hand back a `ScrapeResult`. The `collect` names hand back
a `ScrapeJob` you poll yourself when you pass `async: true`. Only that second
group can pass `includeErrors`.

### Scrape multiple TikTok accounts in one job

An array of inputs is one job, not one per account.

```javascript
import { bdclient } from "@brightdata/sdk";

const client = new bdclient({ autoCreateZones: false });
const job = await client.scrape.tiktok.collectPostsByProfileFast(
  [
    { url: "https://www.tiktok.com/@tiktok", num_of_posts: 1 },
    { url: "https://www.tiktok.com/@khaby.lame", num_of_posts: 1 },
  ],
  { async: true, includeErrors: true },
);
const result = await job.toResult({ pollTimeout: 900_000 });
if (!result.success) throw new Error(`${result.status}: ${result.error}`);
for (const post of result.data) {
  console.log(post.account_id, post.url);
}
await client.close();
```

```
khaby.lame https://www.tiktok.com/@khaby.lame/video/7697307919943060767
tiktok https://www.tiktok.com/@tiktok/video/7694797676306615583
```

`account_id` is the handle. `profile_username` is the display name, "TikTok"
or "Khabane lame", not the handle.

### Trigger now, fetch later

For anything bigger than a few accounts, do not block a process for an hour.
Trigger, keep the snapshot id, fetch when ready. Snapshots stay downloadable
for 30 days.

```javascript
import { bdclient } from "@brightdata/sdk";

const client = new bdclient({ autoCreateZones: false });
const job = await client.scrape.tiktok.collectPosts(
  ["https://www.tiktok.com/@tiktok/video/7680721699171601694"],
  { async: true, includeErrors: true },
);
console.log("snapshot:", job.snapshotId);
await job.wait({ pollInterval: 5_000, pollTimeout: 900_000 });
console.log("status:", await job.status());
const [record] = await job.fetch();
console.log("fetched:", record.url, "likes:", record.digg_count);
await client.close();
```

```
snapshot: sd_mvgd8dubxoh0ti15z
status: ready
fetched: https://www.tiktok.com/@tiktok/video/7680721699171601694 likes: 22900
```

A `ScrapeJob` also has `download()` to write the snapshot to disk and
`cancel()` to stop it.

### Filter TikTok posts by date

```javascript
import { bdclient } from "@brightdata/sdk";

const client = new bdclient({ autoCreateZones: false });
const job = await client.scrape.tiktok.collectPostsByProfileFast(
  [
    {
      url: "https://www.tiktok.com/@tiktok",
      num_of_posts: 3,
      start_date: "09-01-2026", // MM-DD-YYYY
      end_date: "09-30-2026",
    },
  ],
  { async: true, includeErrors: true },
);
const result = await job.toResult({ pollTimeout: 900_000 });
if (!result.success) throw new Error(`${result.status}: ${result.error}`);
for (const post of result.data) {
  console.log(post.create_time, post.play_count, post.url);
}
await client.close();
```

```
2026-09-30T21:57:30.000Z 346500 https://www.tiktok.com/@tiktok/video/7691450773619625247
2026-09-29T17:39:33.000Z 264700 https://www.tiktok.com/@tiktok/video/7691013160425327903
2026-09-25T20:13:19.000Z 449600 https://www.tiktok.com/@tiktok/video/7689568478570843422
```

### Scrape a TikTok profile by URL

```javascript
import { bdclient } from "@brightdata/sdk";

const client = new bdclient({ autoCreateZones: false });
const result = await client.scrape.tiktok.profiles(["https://www.tiktok.com/@tiktok"]);
if (!result.success) throw new Error(`${result.status}: ${result.error}`);
const [profile] = result.data;
console.log(profile.account_id, "followers:", profile.followers, "videos:", profile.videos_count);
await client.close();
```

```
tiktok followers: 96400000 videos: 1510
```

### Scrape comments from a TikTok video

One credit per comment, and `comments()` has no way to ask for fewer. Check
`comment_count` on the post first: the TikTok post used elsewhere in this
README has more than 5,000. The post below has two. To cap a big one, use raw
HTTP with `limit_per_input`; 3 on a 344-comment post returned 3 rows.

```javascript
import { bdclient } from "@brightdata/sdk";

const client = new bdclient({ autoCreateZones: false });
const result = await client.scrape.tiktok.comments([
  "https://www.tiktok.com/@cooking__alone/video/7565981443512339742",
]);
if (!result.success) throw new Error(`${result.status}: ${result.error}`);
console.log(result.data.length, "comments; first:", JSON.stringify(result.data[0].comment_text.slice(0, 60)));
await client.close();
```

```
2 comments; first: "GfcgUZxolnnnmmmmmmmm mmm."
```

### TikTok Shop scraper and discovery over raw HTTP

The SDK has no TikTok Shop methods and no TikTok discovery, so these go
straight to the API. Nothing to install, but this path reads
`BRIGHTDATA_API_TOKEN` only, not the CLI login.

```javascript
const API = "https://api.brightdata.com/datasets/v3";
const headers = {
  Authorization: `Bearer ${process.env.BRIGHTDATA_API_TOKEN}`,
  "Content-Type": "application/json",
};

async function call(path, body) {
  const init = body === undefined ? { headers } : { method: "POST", headers, body: JSON.stringify(body) };
  const response = await fetch(API + path, init);
  if (!response.ok) throw new Error(`${response.status}: ${await response.text()}`);
  return response.json();
}

const product =
  "https://shop.tiktok.com/us/pdp/wireless-bluetooth-earphones-low-latency-gaming-gift/1732339489808880487";
const job = await call("/trigger?dataset_id=gd_m45m1u911dsa4274pi&include_errors=true", [{ url: product }]);
while (!["ready", "failed"].includes((await call(`/progress/${job.snapshot_id}`)).status)) {
  await new Promise((resolve) => setTimeout(resolve, 10_000));
}
const [record] = await call(`/snapshot/${job.snapshot_id}?format=json`);
if (record.error) throw new Error(record.error);
console.log(record.title, record.final_price, record.currency, record.store_name);
```

```
Wireless Bluetooth Earphones, Black TWS Gaming Headset 13.99 USD Saeme
```

Every other endpoint is the same `/trigger` call with a different dataset, a
`type=discover_new&discover_by=...` pair, and a different input. Add
`&limit_per_input=N` to cap a discovery: without it a keyword or a category
returns, and bills, every match. Each input below was accepted by the live API
on 2026-10-20. Several differ from the API reference pages, and the live API
wins:

| endpoint | dataset_id | discover_by | input |
| --- | --- | --- | --- |
| Shop product | `gd_m45m1u911dsa4274pi` | none | `{ url }`, a `shop.tiktok.com/us/pdp/...` URL |
| Shop by keyword | `gd_m45m1u911dsa4274pi` | `keyword` | `{ keyword }`. `num_of_products` is rejected |
| Shop by category | `gd_m45m1u911dsa4274pi` | `category`, not `category_url` | `{ category_url }` |
| Shop by store | `gd_m45m1u911dsa4274pi` | `shop`, not `shop_url` | `{ url }`, not `shop_url` |
| Posts by profile, 44 fields | `gd_lu702nij2f790tmv9h` | `profile_url` | `{ url, num_of_posts }` |
| Posts by keyword | `gd_lu702nij2f790tmv9h` | `keyword` | `{ search_keyword, num_of_posts }` |
| Posts on a tag or music page | `gd_lu702nij2f790tmv9h` | `url` | `{ URL }`, upper case. `url` is rejected |
| Profiles from a search | `gd_l1villgoiiidt09ci` | `search_url` | `{ search_url, country }`, both required |

`bdata pipelines info <dataset_id> <method>` prints the input fields the API
validates, which is how the table above was checked. Discovery is slow: shop
discovery by keyword and by category took 18 to 20 minutes for two products.

## TikTok post data fields (JSON)

The fields most people want:

```
url  create_time  description  hashtags  digg_count  comment_count  play_count  account_id
```

`digg_count` is likes, `collect_count` is saves, and `share_count` arrives as a
string. The code hardcodes no field list. Whatever the API returns lands in
`result.data`, and in the command's file.

<!-- fields:start -->
<details>
<summary>All 44 fields, with type and description</summary>

Regenerated every day from the Posts dataset schema, via
`client.datasets.tiktokPosts.getMetadata()`, so it cannot go stale. The
Fast API the command calls returns a subset; the sample file has 36
of these 44.

| field | type | description |
| --- | --- | --- |
| `url` | url | Unique post link |
| `post_id` | text | Identification for each post |
| `description` | text | Text accompanying the post |
| `create_time` | date | Post creation timestamp |
| `digg_count` | number | Likes received |
| `share_count` | text | Number of shares |
| `collect_count` | number | Number of times saved |
| `comment_count` | number | Number of comments |
| `play_count` | number | Video play count |
| `video_duration` | number | Duration of the video |
| `hashtags` | array | Included hashtags |
| `original_sound` | text | Sound used, if any |
| `profile_id` | text | Unique user identifier |
| `profile_username` | text | User's username |
| `profile_url` | url | User's profile link |
| `profile_avatar` | url | User's profile picture |
| `profile_biography` | text | User's bio |
| `preview_image` | url | Post preview image |
| `post_type` | text | Type of post (video, image, text) |
| `discovery_input` | object | Discovery input value |
| `offical_item` | boolean | Indicates if the video is an official item |
| `secu_id` | text | Security identifier for the profile |
| `original_item` | boolean | Indicates if the video is original |
| `shortcode` | text | Shortcode for the video |
| `width` | number | Width of the video |
| `ratio` | text | Aspect ratio of the video |
| `video_url` | url | URL of the video, URLs that are generated are session-based only |
| `music` | object | Information about the music used in the video |
| `cdn_url` | url | URL link to media |
| `is_verified` | boolean | Indicates whether the TikTok account is verified (boolean: True/False) |
| `account_id` | text | Account ID |
| `carousel_images` | array | List of images for 'carousel'-type TikTok posts |
| `tagged_user` | array | Tagged user |
| `profile_followers` | number | Number of followers of the post Author |
| `tt_chain_token` | text | A session-related token used by TikTok to track and validate user requests |
| `region` | text | The region column indicates the video's country of origin (e.g., US, UK) |
| `commerce_info` | text | Information indicating if the post includes shopping, promotional, or affiliate content |
| `subtitle_url` | url | URL for the subtitles |
| `subtitle_format` | text | Format of the subtitles |
| `country` | text | Some profiles are restricted by location. Please select a country to continue(from discovery) |
| `subtitle_info` | array | Subtitle info |
| `cdn_link` | text | URL link to media |
| `num_share_count` | number | Number of shares |
| `comments` | array | Comments of the post |

</details>
<!-- fields:end -->

<details>
<summary>The start of a real output file, from <code>tiktok-scraper tiktok --limit 1</code></summary>

```json
{
  "generated_at": "2026-10-20T07:33:23.874Z",
  "handles": [
    {
      "handle": "tiktok",
      "posts": [
        {
          "url": "https://www.tiktok.com/@tiktok/video/7694797676306615583",
          "post_id": "7694797676306615583",
          "description": "this summer gave us pop songs, glam routines, and a few main-character habits. @McCall Mirabella breaks down this summer's souvenirs 🐬☀️",
          "create_time": "2026-10-09T22:25:11.000Z",
          "digg_count": 12000,
          "share_count": "1260",
          "collect_count": 1228,
          "comment_count": 5088,
          "play_count": 1400000,
          "video_duration": 49,
          "original_sound": "TikTok: original sound",
          "profile_id": "107955",
  ...
```

The whole file, one post with every field, is
[examples/sample_output.json](examples/sample_output.json).

</details>

## TikTok scraper errors

| you see | what it means |
| --- | --- |
| `API token required but not found.` | Exit 2, before any request. Set the token. |
| `failed  @name: User is banned` | Exit 1. No such account, usually a typo. TikTok says "banned" for an account that does not exist. |
| `got     @name: 0 posts` | Exit 0, and correct. Nothing public in the period searched. |
| `failed  @name: Polling timed out after ...` | Exit 1. A request gives up after 600 seconds. Run it again. |

Any failure exits 1, so a run is safe to gate a script on.

From the SDK, the same conditions look like this:

| you see | what it means |
| --- | --- |
| `AuthenticationError: No API token found.` | No token anywhere: not in the options, the environment, or the CLI login. |
| `APIError` with status 401 | The token is set but wrong. |
| `result.success` is `false`, `result.status` is `"timeout"` | The SDK gave up waiting. Raise `pollTimeout`, in milliseconds, or run it again. |
| a row in `result.data` with an `error` key | The API's answer for one input, when you asked for `includeErrors`. The other rows are fine. |
| zero rows where you expected an error | You did not pass `includeErrors: true`, so the API dropped the row that would have explained it. |
| HTTP 400 `validation_error` on raw HTTP | An input field the API does not accept. The message names it; compare with the table above. |

## TikTok MCP server and Claude Code skill

No JavaScript, nothing installed. Paste both lines; the first opens a browser
once, or use `bdata login --device` over SSH and in CI:

```bash
npx -p @brightdata/cli bdata login
npx -p @brightdata/cli bdata pipelines run gd_m7n5v2gq296pex2f5m collect_by_url --url "https://www.tiktok.com/@tiktok" --num_of_posts 2 --pretty
```

The CLI runs a scraper by its dataset ID, every TikTok one including Shop and
discovery. `bdata pipelines search tiktok` lists their IDs and methods, and
`--limit-per-input N` caps a discovery, 10 by default.

`npx skills add brightdata/skills` teaches Claude Code, Cursor and Codex these
commands and the docs, so plain language works afterwards. Full guide:
[Bright Data for your coding agent](https://docs.brightdata.com/quickstart-coding-agent).

No terminal, for a hosted assistant? The
[Bright Data MCP server](https://github.com/brightdata/brightdata-mcp#which-tool-to-use)
runs every Bright Data scraper, TikTok included, through six tools such as
`search_scrapers` and `run_scraper`. They are on by default; to load only
them, add the `scrapers` group:

    https://mcp.brightdata.com/mcp?token=YOUR_API_TOKEN&groups=scrapers

An agent can also open the account itself, no signup form:
[agent registration](https://brightdata.com/auth.md). Everything else Bright
Data connects to, from LangChain to Zapier and n8n:
[integrations](https://docs.brightdata.com/integrations/introduction).

<details>
<summary>TikTok commands for the CLI, each run on 20 Oct 2026</summary>

Prefix each command with `npx -p @brightdata/cli`, or install once with
`npm i -g @brightdata/cli`. Each prints JSON and costs one credit per record.

Five scrapers, each run with `bdata pipelines run <dataset_id> <method>`:
profiles `gd_l1villgoiiidt09ci`, posts `gd_lu702nij2f790tmv9h`, comments
`gd_lkf2st302ap89utw5k`, the Posts by Profile Fast API `gd_m7n5v2gq296pex2f5m`
and TikTok Shop `gd_m45m1u911dsa4274pi`.

```bash
bdata pipelines run gd_l1villgoiiidt09ci collect_by_url "https://www.tiktok.com/@tiktok" --pretty
```

1 record, 35 fields, 33 seconds. `"account_id": "tiktok"`, `"followers": 96300000`.

```bash
bdata pipelines run gd_m7n5v2gq296pex2f5m collect_by_url --url "https://www.tiktok.com/@tiktok" --num_of_posts 2 --pretty
```

2 records, 34 fields, 70 seconds. Without `--num_of_posts` it returns, and
bills, every post the account has.

```bash
bdata pipelines run gd_lu702nij2f790tmv9h collect_by_url "https://www.tiktok.com/@tiktok/video/7680721699171601694" --format csv -o posts.csv
```

11 minutes. The CSV header carries the same field names as the table above.

```bash
bdata pipelines run gd_lu702nij2f790tmv9h discover_by_keyword --search_keyword "street food" --limit-per-input 2 --pretty
```

2 records, 41 fields, 6.5 minutes.

```bash
bdata pipelines run gd_lkf2st302ap89utw5k collect_by_url "https://www.tiktok.com/@cooking__alone/video/7565981443512339742" --timeout 900 --pretty
```

2 records, 16 fields, 2 minutes. One credit per comment, so check the post's
`comment_count` first. The same command without `--timeout 900` once gave up
at the CLI's 600-second default.

```bash
bdata pipelines run gd_m45m1u911dsa4274pi collect_by_url "https://shop.tiktok.com/us/pdp/wireless-bluetooth-earphones-low-latency-gaming-gift/1732339489808880487" --pretty
```

1 product, 44 fields, 2 minutes. Shop discovery works the same way with
`discover_by_keyword --keyword`, `discover_by_category --category_url` or
`discover_by_shop`, but took 18 to 20 minutes for two products, past the
600-second default: add `--timeout 1500`, or `--async` and fetch later with
`bdata pipelines results <snapshot_id>`.

`bdata pipelines info <dataset_id> <method>` shows a scraper's input fields.
`bdata pipelines run --help` shows `--format json|csv|ndjson|jsonl`, `-o FILE`,
`--timeout` and `--async`.

</details>

## Is this TikTok scraper free?

New Bright Data accounts get
[5,000 free credits a month](https://docs.brightdata.com/general/account/billing-and-pricing/free-tier).
Posts, profiles and TikTok Shop products cost one credit per record, and
comments one credit per comment, so the free credits cover 5,000 posts a
month. [Pricing](https://brightdata.com/pricing/web-scraper) lists the rates
after that.

## Support

Bugs in this repo:
[open an issue](https://github.com/brightdata/tiktok-scraper-node/issues), and
[CONTRIBUTING.md](CONTRIBUTING.md) says what to put in it.
Anything about the API, your account or your credits:
[Bright Data support](https://brightdata.zendesk.com/hc/en-us/requests/new).

## License

[Apache 2.0](LICENSE).

## TikTok scraper for Python

The same TikTok scraper is available for Python: [brightdata/tiktok-scraper](https://github.com/brightdata/tiktok-scraper).

## More open source scrapers from Bright Data

Other Node.js scrapers built the same way, with live-verified examples:

- [Instagram scraper for Node.js](https://github.com/brightdata/instagram-scraper-node)
- [YouTube scraper for Node.js](https://github.com/brightdata/youtube-scraper-node)
- [Facebook scraper for Node.js](https://github.com/brightdata/facebook-scraper-node)
