---
name: Launch a LiquidM campaign
description: >-
  Create a budget, a campaign and an ad in the LiquidM demand-side platform in
  the correct order, then verify the ad can actually deliver.
api: openapi/liquid-m-management-openapi.yml
operations:
  - createBudget
  - createCampaign
  - createAd
  - listCampaigns
  - listAds
---

# Launch a LiquidM campaign

These operations commit advertising spend. Confirm intent with a human before
running any of the create steps.

LiquidM's own README marks the Management API "Under development" — only the
campaign, budget and ad surface below is documented.

## Ordering matters

The budget is created **first**, and the campaign is created with the resulting
budget id. This is the order LiquidM's own client uses.

## 1. Create the budget

Call `createBudget` — `POST /budgets` on `https://platform.liquidm.com/api/v1`,
with `auth_token` in the query string and a form-encoded body wrapping the
resource in a `budget` key.

Fields: `start_date`, `end_date` (ISO 8601 date-times), `unit_price_cents`,
`overall_cents`, `overall_units`, `daily_cents`, `daily_units`, and a pacing mode
for each cap — `overall_units_pacing`, `overall_cents_pacing`,
`daily_units_pacing`, `daily_cents_pacing` (for example `optimized`).

Keep the created `budget.id`.

## 2. Create the campaign

Call `createCampaign` — `POST /campaigns` — with the body wrapped in a `campaign`
key. Set `account_id` and put the budget id from step 1 into `budget_ids`.

Fields: `name`, `unit_type` (for example `cpm`), `currency` (ISO 4217),
`advertiser_domain`, `price_optimization_enabled`, `timezone`, `category_id`
(IAB category).

Keep the created `campaign.id`.

## 3. Create the ad

Call `createAd` — `POST /ads` — with the body wrapped in an `ad` key, setting
`campaign_id`. An ad binds four configuration sections — creative, supply,
targeting and setting — referenced by `creative_id`, `supply_id`, `targeting_id`
and `setting_id`, with `section_order` governing the configuration sequence
(default `["Targeting", "Supply", "Creative", "Setting"]`).

## 4. Verify the ad can deliver

**A `200` does not mean the ad is live.** A newly created ad comes back with
`state: incomplete`. Check three things:

- `meta.errors` — per-section validation failures, keyed by `creative`,
  `setting`, `targeting`.
- `delivery_errors` — for example "Your ad cannot deliver. Some required inputs
  are missing."
- `delivery_warnings` — for example a creative served over a non-secure
  connection.

Also check `appnexus_approval_info` for the exchange approval state; a creative
pending approval will not serve.

## Retry rule

**There is no idempotency key on any of these operations.** If a create times out
or fails ambiguously, do **not** blindly retry — a duplicate budget doubles the
spend envelope. Instead call `listCampaigns` (`GET /campaigns`, with `account_id`)
or `listAds` (`GET /ads`, with `campaign_id` and an optional `embed` list) to see
whether the resource already exists, and only then retry.

The dry-run mode in LiquidM's JavaScript client is entirely client-side — it
substitutes local response emulators and never contacts the API. There is no
server-side preview.
