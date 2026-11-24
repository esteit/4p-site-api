# 4Partners Site API v2

Site API v2 is the API of a 4Partners partner site. Use it to build your own storefront or app on top of the site
and to sync the site with your systems: the product catalog, the cart and checkout of customers, guest checkout,
orders of the site, content and settings.

- API: `https://api.4partners.io/v2`
- Interactive documentation: <https://api.4partners.io/v2>
- Documentation for language models: <https://api.4partners.io/v2/llms.txt>,
  every operation in detail: <https://api.4partners.io/v2/llms-full.txt>
- Error types: <https://api.4partners.io/api/problems>

This repository is updated automatically after each release of the API. AI agents that work with the API for you
start with [`AGENTS.md`](AGENTS.md).

This README and the Bruno collection describe the scenarios and the rules shared by all operations. The full
contract of each operation — request and response models, enums and the error types it can return (`x-problem-types`
of its responses) — is in the OpenAPI specification: `spec/openapi.json` and `spec/llms-full.txt` of this repository,
and the interactive documentation at <https://api.4partners.io/v2>.

## Requests and responses

Every operation is `POST https://api.4partners.io/v2/<group>/<entity>/<action>` with a JSON body
(`Content-Type: application/json`). An operation without parameters takes `{}`. Ids go in the body, keys are
`snake_case`, and an unknown key gives 422.

```sh
curl -X POST https://api.4partners.io/v2/stock/product/show \
  -H 'Content-Type: application/json' \
  -H "X-Auth-Token: $SITE_TOKEN" \
  -d '{"product_id": 123}'
```

- **Envelope.** A successful response is `{timestamp, error, data, meta}`: `error` is `false`, `data` is an object
  or an array for lists, `meta` has `request_id` and `usage`. A field without a value comes as `null`, it is never
  left out. A 204 response has no body; its request id is in the `X-Request-Id` header. 204 comes from
  `rubric/delete`, `content/article/delete`, `content/doc-template/delete`, `settings/theme/delete`,
  `customer-account/address/delete`, `customer-account/profile/delete`, `customer-auth/sign-out` and
  `customer-auth/password/reset`; `customer-cart/item/delete` and `customer-cart/promo-code/delete` return the cart
  and the checkout respectively.
- **Money** is a decimal string with two digits: `"1990.00"`. Country codes are ISO 3166-1 alpha-2 in upper case
  (`RU`).
- **Pages.** Lists for display take `page` (from 1) and `per_page` and return `meta.pager` with `page`,
  `per_page`, `total` and `pages`. The limit and the default of `per_page` are in the specification of each
  operation (for example `rubric/list` 1–1000, 500 by default; `stock/product/search` 1–100, 20 by default); `0`
  gives 422. A full export (`stock/product/id/list`) takes `cursor` and `limit` (1–10 000, 1000 by default) and
  returns `data.ids` in ascending order and `data.next_cursor`: pass `next_cursor` as `cursor` until it is `null`.
  An invalid cursor gives 422.
- **Errors** are `application/problem+json`: `type` is a stable identifier to branch on, `detail` says what is wrong
  and what to do, `pointer` points to the wrong value of the body (`#/items/0/quantity`). A 422 `validation-error`
  lists every wrong value in `errors` with `pointer` and `detail`. `type` is a link: open
  `https://api.4partners.io/api/problems/<slug>` for its meaning and the fix. Quote `request_id` (also in the
  `X-Request-Id` header) when you contact 4Partners support. Examples are under «Errors».
- **Rate limit.** 600 requests per minute, sliding window, for each operation group. With a server key the limit is
  counted per site, and all server keys of the site share one counter. With an app key it is counted per key and
  client IP address. In the customer groups it is counted per customer or guest. The state is in
  `meta.usage` (`limit`, `used`, `remaining`, `retry_after`) and in the `X-RateLimit-Limit` and
  `X-RateLimit-Remaining` headers. Over the limit the answer is 429 with `Retry-After`. Batch operations
  (`stock/product/batch-show`, `stock/list`, `order/batch-show`) save requests. Other limits are under «Limits».
- **Language and region.** `site_variation` in the body sets the language, country and currency of data: product
  names, prices, rubrics, articles. Take the codes from `settings/site-variation/list`; without it the default
  variation is used. Pass it only to operations whose specification has it: an unknown key gives 422. `stock/*`
  operations except `stock/product/id/list` also take `country`. The cart, checkout and `customer-order/create` take
  `site_variation`, `country`, `currency` and `locale`; `country` overrides the country of the variation and
  `currency` must be one of that country, but once the checkout has an address in a country the site delivers to,
  the country and currency of the address win. `dictionary/country/detect` only suggests a country and changes
  nothing. `Accept-Language: en` or `ru` changes only the language of error texts, English by default.

## Errors

A refused request is `application/problem+json`. `type` is the value to branch on; the short name of a type
(`validation-error`) is the last part of `type`. A 422 `validation-error` lists every wrong value in `errors`:

```json
{
  "error": true,
  "type": "/api/problems/validation-error",
  "title": "Request validation failed",
  "status": 422,
  "detail": "1 value in the request is invalid.",
  "instance": "/v2/customer-account/address/create",
  "request_id": "01J8Z6Q4X2N7V3K9B5T1R0M8HD",
  "errors": [
    {
      "pointer": "#/address/phone",
      "detail": "The phone \"123\" is invalid: pass 10 to 15 digits with the country code, an optional \"+\" in front."
    }
  ]
}
```

A business refusal has its own `type`, and `pointer` when it is about one value of the body:

```json
{
  "error": true,
  "type": "/api/problems/deposit-insufficient",
  "title": "Not enough deposit",
  "status": 409,
  "detail": "Not enough deposit to pay order 123: 803.08 USD required, 0.00 USD available. Top up the deposit by at least 803.08 USD and repeat `order/pay`.",
  "instance": "/v2/order/pay",
  "request_id": "01J8Z6Q4X2N7V3K9B5T1R0M8HD"
}
```

Every 401 has the type `/api/problems/unauthorized`: a missing, expired or revoked token, a wrong email or password,
a wrong code or an unknown `confirmation_id`. What to do depends on the request and on the session the app holds:

- `customer-auth/sign-in` or `customer-auth/confirm`: the email, password or code is wrong, or `confirmation_id` is
  unknown. Let the customer fix it; do not renew tokens.
- `customer-auth/refresh`: the refresh token expired, was used or the sign-in ended. Sign the customer in again; do
  not call `refresh` again.
- A customer request with a customer token: call `customer-auth/refresh` once (one call for all parallel requests,
  since a refresh token works once) and repeat the request with the new token. If it gives 401 again, sign the
  customer in again.
- A customer request with a guest token: a guest has no refresh token. The guest token works until it expires
  (`expires_in`); renew it before that with `customer-auth/guest` and `guest_token`. A guest token that expired gives
  401 and cannot be renewed: start with a new guest token. After the guest signs in, use the customer token.
- A site key that is not accepted: the partner asks their 4Partners manager.

502 `code-not-sent` comes from `customer-auth/sign-in` by phone and `customer-auth/sign-up`: the request is valid,
but the SMS gateway or the mail service did not send the code. The attempt still counts: for a phone it lengthens the
pause before the next code and uses one of the codes of the day, for an email the new code replaces the previous
one. A repeat too soon gives 422 `business-rule`.

## Access

**Site key**, header `X-Auth-Token`. A site has two kinds of keys. Ask your 4Partners manager for a server key or an
app key.

- **Server key.** It opens every operation except the customer groups: orders, the deposit, products, content and
  settings.
- **App key.** It opens only the storefront and customer sign-in: reading `stock/*`, rubrics, brands, aspects,
  content blocks, articles, tags and article types, `content/link/resolve`, `settings/site-variation/list` and
  `show`, `dictionary/*`, `delivery/calculate`, `delivery/pick-up-point/list` and `customer-auth/*`. Other site
  operations give 403 `site-token-required`; its `detail` says that the operation needs a server key. The limit of
  requests is counted per key and client IP address. An app key sends at most 10 SMS codes, sign-up codes and password
  reset links in total per hour from one IP address; over the limit the answer is 429 `sign-in-locked` with `Retry-After`.

> [!WARNING]
> The server key gives full access to the site: orders, the deposit, products, content and settings. Keep it on
> your server. Do not put it into a mobile app, browser code or any other client: your app calls your server, and
> your server calls the API. A mobile app or browser code that calls the API directly uses an app key.

**Customer token**, header `X-Customer-Auth-Token`. It opens `customer-account/*`, `customer-cart/*` and
`customer-order/*`. Your server or app gets it from `customer-auth/*` with a site key, server or app:

- `customer-auth/sign-in` by `email` and `password` returns the tokens at once. By `phone` it sends an SMS code and
  returns `confirmation_id`; `customer-auth/confirm` exchanges it and the code for the tokens.
- `customer-auth/sign-up` registers a customer by email: the code goes to the email, then `customer-auth/confirm`.
- Sign-in by SMS accepts only phones with the country code +7 and only on sites that allow sign-in by phone: otherwise
  422 `business-rule` with `pointer` `#/phone`, sign such a customer in by email. The phone of a delivery address is not limited to +7.
- Codes are 4 digits; `code_length` of the answer says so. A code works until it is used, a new code is sent to the
  same phone or email, or it was checked 5 times.
- **A new code.** `code_lifetime` of the answer is the seconds to wait before asking for the next code. For an SMS it
  grows with the number of codes sent to the same phone: 3 minutes after the first and the second code, then 8
  minutes, 30 minutes and an hour; after the 10th code within 24 hours the next one waits up to 24 hours. For an email
  it is 55 seconds. A request before the pause ends gives 422 `business-rule`: send the same request again after
  the pause. A new SMS code keeps `confirmation_id` and replaces the code. Each sign-up returns its own
  `confirmation_id`, and a new email code replaces the previous code of that email. If the SMS gateway or the mail
  service fails, the answer is 502 `code-not-sent`: repeat later or offer the other way to sign in.
- **Confirm.** A wrong code or an unknown `confirmation_id` gives 401. Every try of `customer-auth/confirm`, right
  or wrong, counts: 5 tries in 15 minutes for one email or phone, a success resets the count. The 6th try gives 429
  `sign-in-locked` with `Retry-After` up to the end of the 15 minutes, even with the right code. A code checked 5
  times no longer works: `confirm` with it gives 422 `business-rule` until a new code is sent. After 5 wrong codes
  ask for a new code and confirm it once `Retry-After` has passed.
- The customer token is valid for an hour. `customer-auth/refresh` exchanges the refresh token (one-time, 30 days)
  for a new pair. A refresh token used twice ends the whole sign-in, so do not share it between parallel requests.
- `customer-auth/sign-out` with the refresh token ends the sign-in. A password change or reset and disabling the
  site key that issued the customer token also end it.
- `customer-auth/password/reset` sends a link to a page of the site, valid for a day; the customer finishes the reset
  there, not in the API. The answer is 204 for an unknown email too. `customer-account/password/update` needs
  `old_password`, so a customer who signed up by phone has no password to change: such a customer sets one with the
  reset link.
- `customer-account/profile/update` with `email` sets an email only when the customer has none. The email becomes
  the login at once, and a confirmation link goes to it; `email_confirmed` of the profile shows whether it was
  confirmed.

**Guest token**, the same header `X-Customer-Auth-Token`. `customer-auth/guest` issues it for a customer without
an account. It opens the cart, checkout, addresses and orders: `customer-cart/*`, `customer-account/address/*`,
`customer-order/create`, `customer-order/list` and `customer-order/show`. The profile, the password and order cancel
(`customer-account/profile/*`, `customer-account/password/*`, `customer-order/cancel`) give 403
`customer-token-required`: they need a signed-in customer. To cancel an order, the guest signs in with `guest_token`
first. The token lives 90 days: call `customer-auth/guest` with `{"guest_token": "<the token>"}` before it expires to
renew it and keep the cart and orders. The answer has a new `token` and `expires_in` in seconds; the previous token
keeps working until its own expiry. An expired, unknown or closed `guest_token` gives 422: a guest who lost the token
or let it expire cannot open that cart and those orders again, so keep the token in the app and renew it in time.

- **Orders without sign-in.** A guest places an order without a code or password. The order belongs to this guest
  token only. It is never linked to an existing customer with the same phone or email. The guest sees the order
  while the token lives.
- **Address email.** An address of a guest needs `email`: without it `customer-account/address/create` and
  `update` give 422 with `pointer` `#/address/email`. For a signed-in customer an omitted `email` is the email of the
  customer.
- **Notifications.** A guest gets no order emails. SMS go to the phone of the delivery address.
- **Limits.** With an app key one IP address may start checkout for 20 new guests and place 10 guest orders an hour
  (see «Limits»). A server key has no such limits.
- **Promo codes.** A code for the first order or for one use per customer is not applied to a guest (422): the
  customer signs in first.
- **Sign-in of a guest.** When the guest signs in or up with `guest_token` (`customer-auth/sign-in` by email and
  password, or `customer-auth/confirm` after a code), the cart, orders and addresses move to the customer, and the
  guest token stops working (401): use the customer token from then on. The cart of the guest becomes the cart of the
  customer, and the items of the cart the customer had are added to it as separate lines; read `customer-cart/show`
  after the sign-in. Sign-in by phone takes `guest_token` in `customer-auth/confirm`, not in `sign-in`.

Dictionaries (`dictionary/*`) accept a site key or a customer or guest token. A token of the wrong kind gives 403
`customer-token-required` or `site-token-required`.

- **Address suggestions.** `dictionary/address/suggest` takes what the customer typed (`query`, 3 to 255
  characters) and returns addresses with `value`, the whole address as one line to show in the list, and the
  fields of `customer-account/address/create`: `country`, `region`, `city_id` (KLADR of the settlement, Russia only),
  `city_name`, `street` with its type, `house`, `flat`, `zip`; any of them except `value` and `country` may be
  `null`. Russia, Belarus, Kazakhstan and Uzbekistan get full addresses down to the house. Other countries get
  mostly settlements and streets, with the house number inside `street` and `house: null`. «Addresses» below says
  how to turn a suggestion into an address. `country` is the main country of the site by default; a country the site does not deliver to gives 422
  `country-not-delivered`. If the suggestion service fails, the list is empty: let the customer type the address.
- **Country of the customer.** `dictionary/country/detect` returns the country to preselect and `source`, the way
  that gave it: `coordinates` by `latitude` and `longitude` of the device, `ip` by the IP of the device, `default`
  for the main country of the site. The IP is used only with an app key or a customer or guest token received with
  an app key: with a server key the IP is your server. Only a country the site delivers to is returned.

## Limits

Every limit answers 429 with `Retry-After` in seconds. A request refused for a missing or not accepted site or
customer token is not counted; wrong passwords and codes are counted as below.

| What is counted | Counted per | Window | Limit | Answer |
| --- | --- | --- | --- | --- |
| Requests of one operation group (`stock`, `order`, `customer-cart` and others) | site (server keys), app key and IP, customer or guest | 1 minute, sliding | 600 | `too-many-requests` |
| Failed and successful attempts of `customer-auth/sign-in` by email, `customer-auth/confirm`, `customer-account/password/update`; a success resets it | email, phone or customer of the site | 15 minutes | 5 | `sign-in-locked` |
| Wrong passwords and codes | site key | 15 minutes | 300 | `sign-in-locked` |
| SMS codes, sign-up codes and password reset links sent with an app key, refused sends too | IP | 1 hour | 10 | `sign-in-locked` |
| New guests with an app key, counted at the first checkout request of a guest token: `customer-cart/checkout/show` or `update`, `delivery-option/list`, `pick-up-point/list`, `promo-code/apply` or `delete`, `customer-account/address/create`, `customer-order/create`; a refused request counts too | IP | 1 hour | 20 | `too-many-requests` |
| Guest orders placed with an app key; a refused order and a repeat with the same `idempotency_key` do not count | IP | 1 hour | 10 | `too-many-requests` |

`customer-auth/guest` itself, `customer-cart/item/*` and `customer-cart/show` do not count a new guest.

## Scenarios

**Catalog export.** `stock/product/id/list` page by page with `cursor` → `stock/product/batch-show` for up to 100
products at a time → `stock/list` for frequent polling of prices and stock (up to 100 products or 500 variations).

1. Go through `stock/product/id/list` to `next_cursor: null`. There is no snapshot: a product added during the pass
   with an id above the cursor comes on a later page.
2. Read the cards with `stock/product/batch-show` and compare them with your catalog by the table below.
3. Take off sale the ids of your catalog that step 1 or step 2 did not return and the closed products, but only
   after a pass that reached `next_cursor: null`.
4. Between passes poll `stock/list`. Stop selling a product or variation that is left out of its answer or comes
   with `available: false`, until it comes back available. Leave deleting cards to the next full pass.

| State | `stock/product/batch-show` | `stock/list` | What to do |
| --- | --- | --- | --- |
| The product is not on the site: an unknown id or a product outside the rubrics of the site | left out | left out | Take it off sale after a full pass |
| The product is closed for the site | `closed: true`, `variations: []` | left out | Take it off sale |
| A variation the card does not show | not in `variations` | not in `variations`; the product may come with empty `variations` | Do not sell the variation |
| The variation cannot be ordered now | in `variations` with `available: false`, `quantity: 0` and `unavailable_reason` | the same | Keep it, do not sell it until `available` is `true` |
| No offer of the site for the country | `price: null`, `unavailable_reason: no_offer` | the same | As the row above |

`unavailable_reason` is `no_offer`, `out_of_stock`, `price_over_limit` or `weight_over_limit`; only `no_offer`
comes with `price: null`.

Stock `99` in `stock/*` and in the cart means an offer whose supplier does not pass stock: the quantity is unknown,
not 99 pieces. `order/create` does not limit such an offer; the customer cart takes up to 99 of it.

**Storefront.** `settings/site-variation/list` for the region → `content/block/list` for the banners and product
sliders of the home page → `rubric/list` for the menu → `stock/product/search` with `stock/facet/list` for the
filters of the same selection → `stock/product/suggest` for the search box → `stock/product/show` for a product card →
`stock/product/similar` for the «you may also like» block.

- **Menu.** `rubric/list` without `parent_id` returns the whole tree, page by page, with the menu flags
  `in_top_menu`, `in_all_menu`, `in_footer_menu`, `unfold` and `unfold_count`. `active` is the own flag of a rubric:
  show a rubric only when it and every rubric of its `ancestors` are `active` (find the ancestors by `id` in the
  same list).
- **Links.** `link` of a banner, `products_url` of a product slider, a link in an article text and `url` of products,
  rubrics, brands and articles are links of the storefront. Do not parse them in the app: `content/link/resolve` takes
  the link and returns the screen to open in `type`. `product` gives `product_id` and `product_variation_id`: pass
  both to `stock/product/show`, which returns the card of the product with that variation to preselect (a variation
  of another product is refused). `products` gives `search`, a ready body for `stock/product/search`; `article` gives
  `article_id` for `content/article/show`; `home` is the home screen; `page` and `external` open `url` in a browser.
  `site_variation` of the answer is the code of the site variation when the link is on its folder or subdomain, and
  `null` otherwise: pass a code as `site_variation` to the next request (`stock/product/search`,
  `stock/product/show`, `content/article/show`); with `null` keep the region of the app.

**Cart and checkout of a customer.**

1. `customer-auth/guest` → `customer-cart/item/create`, `customer-cart/show`. A repeated `item/create` for the same
   variation sets the quantity (1–1000, at most `available_quantity`), it does not add to it. `customer-cart/show`
   updates prices and drops items that are no longer sold without a separate notice.
2. To check out as a signed-in customer: `customer-auth/sign-in` by phone or `customer-auth/sign-up` by email →
   `customer-auth/confirm` with the code and `guest_token`. A guest skips this step and goes on with the guest token.
3. `customer-account/address/create` (a guest passes `email` in the address) → `customer-cart/checkout/update` with
   `address_id`.
4. `customer-cart/checkout/show`. For every package with `available: false`: a `courier` method of its
   `delivery_options` goes to `customer-cart/checkout/update` with `package_id` and `delivery_method_id`; a
   `pick_up_point` method also needs `pick_up_point_id` from `customer-cart/pick-up-point/list`. One call sets one
   package; repeat until `deliverable` is `true`. A package with `unavailable_reason` `out_of_stock` needs a lower
   quantity; with `no_delivery_method` unselect its items (`selected: false` in `customer-cart/item/update`) or
   choose another address.
5. `payment_methods` of the checkout has a preselected method (`selected: true`). To change it, pass its `id` as
   `payment_method_id` to `customer-cart/checkout/update`.
6. `customer-cart/promo-code/apply`, if the customer has a code.
7. `customer-order/create` with `return_url` and `idempotency_key` → open `payment_url` for the customer. The
   selected items leave the cart for the order.
8. There is no payment notification: poll `is_paid` of `customer-order/show`.

Read `customer-cart/checkout/show` again after every change of items, quantities, `selected`, the address, the promo
code or a sign-in. `package_id` changes with the set of items of a package: take it from the latest checkout. A change
of the country or currency resets the delivery choice. Place the order when `deliverable` is `true` and `total` is the
amount the customer saw.

**Addresses.** `customer-account/address/create` takes the fields in the `address` object and requires `first_name`,
`last_name`, `phone` (10 to 15 digits with the country code, an optional `+` in front) and `street`. `country` is `RU`
by default and must be a country the site delivers to.

- Russia: `house` is required, and `city_id` (KLADR from `dictionary/address/suggest` or `dictionary/city/search`) or
  `city_name`. When both are passed, `city_name` is used and `city_id` is dropped: pass one of them.
- Other countries: `city_name` is required, `city_id` is not allowed, `region` is text. `street` is the address
  line; `house` and `flat`, when passed, are added to it.
- `zip` is up to 16 characters, `email` is a valid email; a guest must pass `email`.
- `customer-account/address/update` replaces the address: a field left out becomes empty, except `zip`, which stays.

From a suggestion of `dictionary/address/suggest` take only these fields:

| Address field | From the suggestion |
| --- | --- |
| `country` | `country` |
| `city_id` | `city_id`, in Russia, when it is not `null`; then leave out `city_name` |
| `city_name` | `city_name`, when there is no `city_id`; outside Russia always |
| `region` | `region`, outside Russia |
| `street`, `house`, `flat`, `zip` | the same fields, when they are not `null` |
| `first_name`, `last_name`, `phone`, `email`, `middle_name`, `note` | not in the suggestion: the customer types them |

`value` is only for the list. When a required field is `null` (`street`; `house` in Russia; `city_name` outside
Russia), let the customer type it. A suggestion in Russia
`{"value": "г Москва, ул Тверская, д 7", "country": "RU", "region": "г Москва", "city_id": "7700000000000",
"city_name": "Москва", "street": "ул Тверская", "house": "7", "flat": null, "zip": "125009"}` gives:

```json
{
  "address": {
    "first_name": "Ivan",
    "last_name": "Petrov",
    "phone": "79001234567",
    "country": "RU",
    "city_id": "7700000000000",
    "street": "ул Тверская",
    "house": "7",
    "zip": "125009"
  }
}
```

A suggestion in Kazakhstan `{"value": "Казахстан, г Алматы, пр-кт Абая, д 10", "country": "KZ", "region":
"г Алматы", "city_id": null, "city_name": "Алматы", "street": "пр-кт Абая", "house": "10", "flat": null, "zip":
"050000"}` gives:

```json
{
  "address": {
    "first_name": "Aigerim",
    "last_name": "Nurlanova",
    "phone": "77011234567",
    "country": "KZ",
    "region": "г Алматы",
    "city_name": "Алматы",
    "street": "пр-кт Абая",
    "house": "10",
    "zip": "050000"
  }
}
```

A guest adds `email` to both.

`dictionary/city/search` returns `id: null` outside Russia: pass the name as `city_name`. `delivery/calculate` and
`delivery/pick-up-point/list` take a settlement only as `city_id` (KLADR), so outside Russia methods that need a
settlement come in `omitted`. In the cart the country of the address selects the delivery methods; a package that no
method of the site carries has `unavailable_reason: no_delivery_method`. `dictionary/country/detect` takes
`latitude` and `longitude` together or neither (else 422); when they give no country, it falls back to the IP and then
to the main country of the site.

**Payment and return.** `payment_url` is a page of the site: open it in a browser. It does not expire: open it again to
repeat a failed or abandoned payment (`customer-order/show` also returns it until the order is paid or canceled).
`return_url` is an `https` address or a deep link of your app (`myapp://orders`), up to 2000 characters; `http`,
`javascript`, `data`, `file` and similar schemes give 422. After the bank the payment page adds `order_id` and
`payment_status` to the query of `return_url`, before its `#fragment`:
`myapp://orders?order_id=123&payment_status=paid`. `payment_status` is `paid` (the order is paid), `pending` (the bank
reported success, the confirmation is on its way) or `failed`. Not every bank sends the customer back on a failure, and
a customer may close the browser: treat `payment_status` as a hint and show the result from `is_paid` of
`customer-order/show`.

**A lost answer of `customer-order/create`.** Send `idempotency_key`, 8 to 64 characters `A-Z a-z 0-9 . _ : -` (a
UUID fits), new for each order the customer places:

1. Make the key before the first try. Keep it with the checkout attempt, the customer or guest it belongs to and the
   time of the first try, in storage that survives a restart of the app, until the answer is known.
2. On a timeout or a network error send the same request with the same key. It returns the order already placed with
   `replayed: true` (status 201, as a new one), and nothing new is placed.
3. 409 `conflict`: the first request is still running. Wait a few seconds and repeat with the same key.
4. 422: nothing was placed and the key is free: fix the checkout and repeat.
5. After 201 make a new key for the next order. A repeat returns the order of the key without comparing the body.

Where a key works:

- **Site and customer.** A key finds an order of the same site and the same customer, signed in or guest. The same
  key from another customer or guest places a new order.
- **A day.** The day starts when the first order with this key is placed. Later the same key places a new order.
  Do not repeat a key automatically a day or more after its first try: read `customer-order/list` first.
- **A guest who signed in.** The orders of the guest move to the customer, and the key of the guest no longer finds
  them. Look for the order of a lost answer in `customer-order/list` of the customer; a new key is needed only for a
  new order.

Without a key, or when the key no longer works, a lost answer cannot be told from a request that was not accepted:
find the order in `customer-order/list` before you place it again.

**Orders of the site.** The partner orders for its own recipients and pays from the deposit:
`delivery/calculate` (and `delivery/pick-up-point/list` for pick-up methods) → `order/create` with `external_id` →
`order/pay` → `order/show`, `order/list`. `balance/show` shows the deposit in each currency (`RUB`, `USD`, `EUR`).
`balance/operation/list` takes `date_from` and `date_to` as `YYYY-MM-DD`; both days are included, and days are
counted in Moscow time (UTC+3).

- `order/create` saves the prices of the lines and of the delivery and returns `deposit_due`, what `order/pay` will
  charge, in `deposit_currency`, the currency of the order. Stock is checked when the order is placed, but it is not
  reserved, and `order/pay` does not check it again.
- An unpaid order is canceled automatically, after 30 days on most sites; ask your 4Partners manager for the term of
  your site. Until then it can be paid. The same applies to an unpaid order of a customer.
- The prices of a partner order stay as placed until it is paid. An unpaid order of a customer keeps its prices for an
  hour after `customer-order/create`; later, if a supplier price changes, its line prices and total are recalculated:
  show the customer the amount of `customer-order/show` before paying again.
- `order/pay` charges in the currency of the order. With `deposit_currency` of another currency it converts the
  amount at the rate of the moment of payment and checks the balance in that currency.
- `status` of an order is one of: `new` (placed, waits for payment and confirmation), `confirmed` (confirmed, ready
  to be bought), `wait_in_store` (all goods are bought, on the way to the warehouse), `in_store` (all goods
  are at the warehouse), `packed` (packed), `sent` (shipped), `delivered`, `canceled`. A status can move back, for
  example from `in_store` to `wait_in_store`: show the current one, and `status_history`
  for the path. Old orders may have `sent_msk` (shipped within Russia): show it as `sent`. Show any other value as
  it is, without failing.
- A repeated `order/create` with the `external_id` of an order that is not canceled returns that order with
  `replayed: true` (200) when the items (in any order), the delivery method, and the pick-up point and currency when
  passed are the same; otherwise it gives 409 `external-id-conflict`. The recipient is not compared. When the order
  with this `external_id` is canceled, a new order is placed. `external_id` is up to 512 bytes and does not expire.

**Assortment and catalog of the site** (server key). `source/store/list` → `source/store-rubric/list` →
`stock/product/id/list` with `store_id` and `rubric_id` → `product/rubric/link` to rubrics of the site tree
(`rubric/create`, `rubric/update`, `rubric/move`, `rubric/activate`) → `product/primary-rubric/update`.
`product/rubric/link` and `unlink` answer 202 and finish in the background within minutes, with no task id or status
request. A link is done when the product comes in `stock/product/id/list` with that `rubric_id` or the rubric is in
`rubric_ids` of `stock/product/show`; both also count links to subrubrics. An unlink is done when the product is gone
from that list. `stock/product/search` shows the change after the next indexing. The API does not show whether a
product is linked to a rubric itself or only to its subrubric: for a product already in a subrubric the link changes
nothing you can see. A failed background link is not reported. If after an hour the product is still missing from
`stock/product/id/list` with that `rubric_id`, send the same `product/rubric/link` again; if it is still missing an
hour later, send the `request_id` of both 202 answers to your 4Partners manager. Texts:
`product/update`, `product/variation/update`, `brand/update`, `rubric/brand/update`; `brand/deactivate` hides a brand
with its products.

**Content and settings** (server key). Articles and page templates: `content/article/*`, `content/doc-template/*`.
Settings and storefront themes: `settings/variable/*`, `settings/theme/*`.

**App with an app key.** The app reads the storefront, gets a guest token at the first start, and checks out as a
guest or after sign-in, as in «Cart and checkout of a customer». `dictionary/country/detect` preselects the country,
and `dictionary/address/suggest` fills the address form. Links of banners, sliders and articles open through
`content/link/resolve` (see «Storefront»). Product sets under a product card come from
`stock/product/similar`: there is no separate method for them. Everything else goes through your server with the
server key.

The Bruno collection lists these scenarios step by step with its requests (see «This repository»).

## This repository

- `spec/` — snapshots of the released version: `openapi.json` (the OpenAPI specification), `llms.txt` and
  `llms-full.txt`.
- `bruno/` — a [Bruno](https://www.usebruno.com/) collection with a request for every operation, grouped as in the
  documentation. The documentation of the collection lists the scenarios with the requests to run, and the folders
  of the main groups describe their steps. The collection shows working requests, not the full models: take those
  from `spec/`.

To use the collection:

1. Open `bruno/` in Bruno and select the `prod` environment. Its `baseUrl` is `https://api.4partners.io`, without
   `/v2`: the requests add `/v2/...` themselves.
2. Enter your server key into the secret variable `siteToken` (an app key runs only the requests open to
   it). Bruno keeps secret values on your computer, not in
   the environment file.
3. Replace the zero ids of the environment (`productId`, `productVariationId`, `rubricId`, `deliveryMethodId`,
   `pickUpDeliveryMethodId`) with ids of your site from the list operations. Other ids are saved by the requests
   that return them; the documentation of the collection lists them and the few ids to put into a body by hand.
4. `customerToken` is filled by the `Customer_Auth` sign-in, confirm and refresh requests and `Customer_Account`
   password change. Enter it by hand only to run the customer folders as a guest: set it to the value of
   `guestToken`.

Requests of the collection run against the live site: placing, paying and canceling orders, deleting objects and
sending codes really happen.
