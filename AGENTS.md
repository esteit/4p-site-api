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
- Send the site key in `X-Auth-Token`. A server key opens every operation except the customer groups; use it only
  in code that runs on a server. An app key opens only reading the storefront (`stock/*`, rubrics, brands, aspects,
  content blocks, articles and `content/link/resolve`, site variations), `dictionary/*`, `delivery/calculate`,
  `delivery/pick-up-point/list` and `customer-auth/*`; it is for code that runs in a mobile app or a browser. For
  orders of the site, the deposit and any write outside `customer-auth/*` and the customer groups ask the partner for
  a server key: an app key gives 403 `site-token-required` there.
- The `customer-account`, `customer-cart` and `customer-order` groups take the customer token in
  `X-Customer-Auth-Token`; get it with `customer-auth/*`. A guest token opens `customer-cart/*`,
  `customer-account/address/*`, `customer-order/create`, `list` and `show`, so a guest can order without signing in;
  the profile, the password and `customer-order/cancel` need a signed-in customer. A guest address needs `email`.
- Send only the fields of the specification: an unknown key gives 422.
- Pass `site_variation` from `settings/site-variation/list` to get data in the language, country and currency of
  the partner's region.
- Read the result from `data`; `null` means no value. Money is a decimal string: do not convert it to a float.
- For an address form call `dictionary/address/suggest` and map the chosen suggestion to
  `customer-account/address/create` as the table in «Addresses» of `README.md` says: never pass `value`; in Russia
  pass `city_id` when it is not `null`, else `city_name`, never both; the recipient (`first_name`, `last_name`,
  `phone`, `email`) comes from the user. To preselect a country call `dictionary/country/detect`. Product sets of a
  product card come from `stock/product/similar`: there is no other method for them.
- Do not parse links of the storefront (`link` of banners, `products_url` of sliders, links in articles, `url` of
  products, rubrics, brands and articles): pass the link to `content/link/resolve` and open the screen its `type`
  names. For `product` call `stock/product/show` with `product_id` and `product_variation_id` and preselect that
  variation; for `products` send `search` of the answer as the body of `stock/product/search`; for `page` and
  `external` open `url` in a browser. When `site_variation` of the answer is not `null`, pass it as `site_variation`
  to that next request.
- Build a menu from `rubric/list` and its menu flags. Show a rubric only when it and every rubric of its
  `ancestors` are `active`; find the ancestors by `id` in the same list.

## Ids

- Do not guess or construct ids. Take them from the list or search operations of the same site: `stock/product/id/list`,
  `stock/product/search`, `rubric/list`, `order/list`, `customer-account/address/list` and others.
- On 404 or 422, `detail` usually names the operation that returns valid values. Call it and use an id from its answer.

## Errors

- Branch on `type` and `pointer`, not on the text of `detail`.
- Fix the request as `detail` says, then send the corrected request. Do not repeat an unchanged request that got 400,
  403, 404 or 422: it gets the same error. 429 is repeated after `Retry-After`; 409 `conflict` of
  `customer-order/create` is repeated with the same `idempotency_key` after a few seconds. One 422 is repeated
  unchanged: `business-rule` of `customer-auth/sign-in` by phone or `customer-auth/sign-up` when a new code was asked
  too soon; send the same request once `code_lifetime` seconds of the previous answer have passed.
- 422 `validation-error`: fix every value listed in `errors`.
- 401 (`unauthorized`): act by the request and the kind of session.
  - `customer-auth/sign-in` or `confirm`: a wrong password, a wrong code or an unknown `confirmation_id`. Ask the
    user again, do not refresh.
  - `customer-auth/refresh`: the refresh token no longer works. The customer signs in again; do not refresh again.
  - A customer request with a customer token: refresh once (one call for all parallel requests) and repeat the
    request with the new token; on a second 401 the customer signs in again.
  - A customer request with a guest token: there is no refresh. Renew the guest token with `customer-auth/guest`
    and `guest_token` before it expires; an expired guest token cannot be renewed, start with a new guest token.
  - A site key that is not accepted is a matter for the partner and their 4Partners manager.
- 502 `code-not-sent` on `customer-auth/sign-in` by phone or `customer-auth/sign-up`: the code was not sent, but the
  attempt counts toward the code limits. Do not repeat at once; tell the user to try later or to sign in the other
  way.
- 403 `site-token-required` with a site key: the key is an app key and the operation needs a server key. Do not
  retry; tell the user that the operation needs a server key.
- `GET https://api.4partners.io/api/problems/<slug>` describes a `type` and its fix.

## Limits

- Watch `meta.usage`: `remaining` requests in the sliding one-minute window, `retry_after` seconds to wait. All
  server keys of the site share one counter per operation group; an app key has a counter per key and client IP
  address; in the customer groups the counter is per customer or guest. Sign-in, codes and guest checkout have their
  own limits: see «Limits» in `README.md`.
- On 429, wait `Retry-After` seconds before the next request. Do not retry in a loop.
- Prefer batch operations (`stock/product/batch-show`, `stock/list`, `order/batch-show`) to many single calls.

## Safety

- Do not repeat a write operation blindly after a timeout or a 500: read the state first (`order/show`,
  `order/list`, `customer-order/list`). Call `order/create` with `external_id`: a repeat with the same `external_id`
  and the same items, delivery method, pick-up point and currency returns the existing order with `replayed: true`
  instead of placing a new one; if that order is canceled, a new order is placed. Call `customer-order/create` with
  a new `idempotency_key` for each order and repeat a lost request with the same key: it returns the same order with
  `replayed: true` for the same site and customer or guest, within a day of the first order with this key. A day
  after the first try, or after a guest signed in, read `customer-order/list` instead of repeating the key.
- Call these operations only when the user explicitly asks for them:
  - operations that send an SMS or an email: `customer-auth/sign-in` by phone, `customer-auth/sign-up`,
    `customer-auth/password/reset`, `customer-account/profile/update` with an `email`;
  - operations that delete data or cannot be undone: every `*/delete`, `settings/variable/reset`,
    `customer-account/profile/delete`, `order/cancel`, `customer-order/cancel`;
  - operations that spend money or place orders: `order/create`, `order/pay`, `customer-order/create`.
- Never print, log or send keys and tokens to the user or anywhere else: site keys, customer, guest and refresh
  tokens. The server key gives full access to the site: never put it into app or browser code.
