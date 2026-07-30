---
name: Run a LiquidM Visual Report
description: >-
  Authenticate against the LiquidM platform and pull programmatic advertising
  performance data from the Reporting API, split by any combination of campaign,
  placement, geo, device or audience dimensions.
api: openapi/liquid-m-reporting-openapi.yml
operations:
  - createAuthToken
  - getVisualReport
---

# Run a LiquidM Visual Report

The LiquidM Reporting API returns the same data as the Visual Reports screen in
the LiquidM platform. It is a single read operation with a rich query vocabulary.

## 1. Get an auth token

Call `createAuthToken` — `GET /api/auth` on `https://platform.liquidm.com` — with
`email`, `password` and `api=true`.

**Before you do this, stop and check whether a token already exists.** Issuing a
new AUTH_TOKEN immediately invalidates the previous one. If a scheduled job or
another integration is using the account's token, requesting a fresh one will
break it. Reuse a stored token whenever you have one.

## 2. Build the query

Call `getVisualReport` — `GET /visual_reports.json` — with `auth_token` and:

- `start_date` / `end_date` — date strings. Defaults are seven days ago and today.
- `granularity` — `all`, `day` or `hour`. Default `all`.
- `currency` — an ISO 4217 code. Default `EUR`.
- `time_zone` — a Rails TimeZone constant. Only whole-hour offsets from UTC work;
  a 30-minute offset zone is not supported.
- `dimensions` — comma-separated. What to split the result by.
- `filters` — comma-separated, each `<dimension>-<value>`, e.g. `campaign-99999`.
  Every dimension is also usable as a filter.
- `metrics` — comma-separated. Default `ais,cost`.

Use only ids that appear in `vocabulary/liquid-m-vocabulary.yml`. The most
commonly needed ones: dimensions `campaign`, `ad`, `site`, `domain`, `country`,
`devicetype`, `os_name`, `media_type`; metrics `ais` (impressions), `bids`,
`bid_requests`, `clicks`, `ctr`, `win_rate`, `e_cpm`, `earnings` (cost),
`conversion_rate`.

## 3. Read the response

The response has three arrays:

- `dimensions` — the dimensions used.
- `columns` — each with an `id` and a display `name`.
- `rows` — one **flat** list. It is never nested, even when several dimensions
  were requested.

Each cell carries `value` and `formatted_value`. **Always show
`formatted_value` to a human.** LiquidM states that `formatted_value` is
persistent while `value` may change representation — for example from microcents
to cents — so arithmetic on `value` across time is unsafe. Entity cells (such as
`country`) carry `name` instead of `formatted_value`.

## 4. Handle errors

The API signals with HTTP status only; there is no structured error body. `200`
is success. `401` means the token was wrong, missing, or superseded by a newer
token — re-issue via step 1 and retry once.

## Notes

- The token travels as a **query parameter**, so it will appear in proxy and
  access logs. Do not paste report URLs into shared channels.
- There is no pagination. A wide date range crossed with several high-cardinality
  dimensions returns one large response; narrow with `filters` instead.
- No rate limits are documented. Be conservative with polling.
