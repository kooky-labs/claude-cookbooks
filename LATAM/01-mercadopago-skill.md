# Recipe 01: MercadoPago skill (cards + Pix)

The first LATAM adaptation. A MercadoPago payments skill that exposes a single capability surface (`payments.charge`) hiding the two-product split underneath. Cards run through Checkout Pro. Pix runs through the Orders API. Same merchant, different APIs, single caller-facing interface.

This recipe walks through the pattern shapes that came out of building it. The skill itself lives in a private repo (`kooky-os-skills/skills/os-mercadopago/`); this recipe extracts the patterns that generalize.

## What this skill does

Three capabilities, exposed to callers as a single contract:

- `payments.charge(method=cards|pix, amount, customer, ...)`: initiates a payment, returns a hosted checkout URL (cards) or a Pix QR code/payload (Pix).
- `payments.status(id)`: looks up the status of any payment by any of three possible ID types (Preference ID, Order ID, Payment ID).
- `payments.refund(payment_id, amount)`: issues a refund against a completed payment.

The caller passes `method` once and never sees the underlying split. The skill routes internally.

## Pattern 1: hide the two-product split behind one capability

MercadoPago is two products with two completely different API shapes. Cards live behind Checkout Pro (preference creation, hosted checkout, payment-on-completion). Pix lives behind the Orders API (order creation with a Pix transaction object). The endpoints, request shapes, and response shapes don't overlap.

A naïve connector exposes both: `mp.checkoutPro.create(...)` and `mp.orders.create(...)`. Callers learn which one to call. The skill knows the difference; the caller has to know it too.

The cleaner shape is to hide the difference entirely. The skill declares a single capability:

```yaml
exposes:
  capability: payments.charge
  inputs:
    method: { enum: [cards, pix] }
    amount: { type: number }
    customer: { type: object, properties: { email: string } }
  returns:
    type: object
    properties:
      payment_id: string
      checkout_url: string   # cards only
      pix_payload: string    # Pix only
```

The script reads `method`, routes to the right endpoint, normalizes the response shape, returns one object regardless. Callers never branch on `method` outside the skill.

**The design lesson:** if your integration target has two product families that look like one product from the outside, hide them as one product from the inside too. The complexity belongs in the skill, not in every caller.

## Pattern 2: auto-detect ID type for status lookups

Status lookup has the inverse problem. Three possible ID types, one capability:

- A **Preference ID** identifies a Checkout Pro session (cards). Format: `<alphanumeric>-<alphanumeric>`.
- An **Order ID** identifies a Pix order. Format: starts with `ORD-`.
- A **Payment ID** identifies a completed payment (cards, post-checkout). Format: numeric only.

Three options for handling this:

1. Make the caller declare the ID type. Pushes complexity onto the caller (and the LLM, if the caller is an LLM).
2. Try all three endpoints in sequence until one works. Wasteful: two of three calls return 4xx per lookup.
3. Auto-detect by ID format.

We chose option 3. The skill reads the ID's structural signature:

```bash
case "$id" in
  ORD-*)
    endpoint="/v1/orders/$id"
    ;;
  [0-9]*)
    endpoint="/v1/payments/$id"
    ;;
  *-*)
    endpoint="/checkout_pro/preferences/$id"
    ;;
  *)
    echo "Unrecognized ID format: $id" >&2
    exit 1
    ;;
esac
```

The caller passes any of the three; the skill routes correctly. This pattern generalizes well. Any API where a single conceptual entity has multiple ID spaces (orders vs payments vs sessions vs receipts) can use this shape. It's a strong upstream PR candidate.

**The design lesson:** when input shapes carry their own type signature, let the receiver detect the type. Don't push detection onto the caller.

## Pattern 3: surface raw response bodies on every 4xx

MercadoPago's error vocabulary is unstable. The same "this resource isn't yours" failure returned HTTP 401 in one test run and HTTP 404 in the next. Different label, same root cause.

If you branch logic on the error code, you'll write code that handles 401 correctly and silently fails on the same scenario when it shows up as 404. The fix isn't to handle both. It's to stop branching on the code at all.

The skill's pattern:

```bash
mp_request() {
  local method="$1" path="$2" body="$3"
  local tmp status

  tmp=$(mktemp)
  status=$(curl -sS -o "$tmp" -w '%{http_code}' \
    -X "$method" \
    -H "Authorization: Bearer $MP_TOKEN" \
    -H "Content-Type: application/json" \
    -d "$body" \
    "https://api.mercadopago.com$path")

  if [ "$status" -ge 400 ]; then
    echo "MercadoPago $method $path returned HTTP $status:" >&2
    cat "$tmp" >&2
    rm -f "$tmp"
    exit 1
  fi

  cat "$tmp"
  rm -f "$tmp"
}
```

Every 4xx gets surfaced verbatim to stderr. The caller (or the calling LLM) reads what the API actually said, not what the docs predicted. This pattern is also a strong upstream PR candidate. It generalizes to any API with inconsistent error vocabularies, which is most of them.

**The design lesson:** error codes are a translation layer that can break. Raw response bodies are ground truth.

## Sandbox setup notes

A few non-obvious gotchas that surfaced during the build:

- **MercadoPago sandbox is country-locked at app creation time.** When you create a test app in the MP developer console, the country selector is permanent. If you pick the wrong country, you create a new app. For BR-scoped testing, choose Brazil at app creation. Documented but easy to miss.
- **Sandbox tokens look like production tokens.** Both start with `APP_USR-`. The only way to tell them apart is which credentials panel section they came from. Don't pattern-match `APP_USR-` to assume production.
- **Pix in sandbox returns mock QR codes that won't pay.** You can test the API path end-to-end, but you can't simulate the customer-side payment flow naturally. Status stays `pending` indefinitely unless you trigger the MP sandbox's "simulate Pix payment" admin action.

## What's not in this recipe

- The live OAuth flow. The skill assumes a long-lived sandbox token; production OAuth lands separately.
- Client-specific business logic from KOOKY production deployments.
- The webhook receiver that consumes payment-completion notifications. That ships as part of a later build (a unified payments-rail-wide receiver covering MP + Pix Direct).

## Where this came from

Built during the first week of a 6-week LATAM adaptation sprint. The companion LinkedIn article tells the operator-level story: [Localizing Claude for Small Business for LATAM: the gap isn't language, it's how the local stack actually works](https://www.linkedin.com/pulse/localizing-claude-small-business-latam-gap-isnt-language-goulart-geuye).

Next recipe lands when the Conta Azul skill ships, around mid-June 2026.
