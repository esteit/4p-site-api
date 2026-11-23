# AGENTS.md

Instructions for an AI agent that calls 4Partners Site API v2 on behalf of a partner. People start with
[`README.md`](README.md).

## Start

1. Read `spec/llms.txt` or <https://api.4partners.io/v2/llms.txt>: the groups and every operation, one line each.
2. Before calling an operation, read its entry in `spec/llms-full.txt` (<https://api.4partners.io/v2/llms-full.txt>):
   body fields, response fields, error types. For exact schemas use `spec/openapi.json`
   (<https://api.4partners.io/v2.jsonopenapi>).
3. The snapshots in `spec/` match the last release. If they disagree with the live documentation, trust the live one.

## Calling

- Every operation is `POST https://api.4partners.io/v2/<group>/<entity>/<action>` with a JSON body and
  `Content-Type: application/json`. Send `{}` when there are no parameters. Ids go in the body, never in the path.
- Send the site token in `X-Auth-Token`. The `customer-account`, `customer-cart` and `customer-order` groups take
  the customer token in `X-Customer-Auth-Token`; get it with `customer-auth/*`. A guest token opens only
  `customer-cart/show` and `customer-cart/item/*`.
- Send only the fields of the specification: an unknown key gives 422.
- Pass `site_variation` from `settings/site-variation/list` to get data in the language, country and currency of
  the partner's region.
- Read the result from `data`; `null` means no value. Money is a decimal string: do not convert it to a float.

## Ids

- Do not guess or construct ids. Take them from the list or search operations of the same site: `stock/product/id/list`,
  `stock/product/search`, `rubric/list`, `order/list`, `customer-account/address/list` and others.
- On 404 or 422, `detail` usually names the operation that returns valid values. Call it and use an id from its answer.

## Errors

- Branch on `type` and `pointer`, not on the text of `detail`.
- Fix the request as `detail` says, then send the corrected request. Do not repeat the same request: it gets the
  same error.
- 422 `validation-error`: fix every value listed in `errors`.
- 401: the token is missing, expired or revoked. Renew the customer token with `customer-auth/refresh`; if refresh
  also gives 401, the customer must sign in again. A site token that is not accepted is a matter for the partner
  and 4Partners support.
- `GET https://api.4partners.io/api/problems/<slug>` describes a `type` and its fix.

## Limits

- Watch `meta.usage`: `remaining` requests in the sliding one-minute window, `retry_after` seconds to wait. All
  tokens of the site share one counter per operation group; in the customer groups the counter is per customer.
- On 429, wait `Retry-After` seconds before the next request. Do not retry in a loop.
- Prefer batch operations (`stock/product/batch-show`, `stock/list`, `order/batch-show`) to many single calls.

## Safety

- Do not repeat a write operation blindly after a timeout or a 500: read the state first (`order/show`,
  `order/list`, `customer-order/list`). Call `order/create` with `external_id`: a repeat with the same `external_id`
  and body returns the existing order with `replayed: true` instead of placing a new one. `customer-order/create`
  has no such key.
- Call these operations only when the user explicitly asks for them:
  - operations that send an SMS or an email: `customer-auth/sign-in` by phone, `customer-auth/sign-up`,
    `customer-auth/password/reset`, `customer-account/profile/update` with an `email`;
  - operations that delete data or cannot be undone: every `*/delete`, `settings/variable/reset`,
    `customer-account/profile/delete`, `order/cancel`, `customer-order/cancel`;
  - operations that spend money or place orders: `order/create`, `order/pay`, `customer-order/create`.
- Never print, log or send tokens to the user or anywhere else: the site token, customer, guest and refresh tokens.
  The site token gives full access to the site.
