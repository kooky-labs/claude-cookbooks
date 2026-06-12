# Recipe 04: vendor-neutral Pix (the swappable-backend pattern)

The fourth and final LATAM adaptation. Pix is Brazil's instant payment system, created and operated by the Central Bank (Banco Central do Brasil, BCB): free for individuals, near-free for businesses, settled in seconds, around the clock. For a Brazilian small business it is the cash register, so a Claude-powered operations agent has to be able to create a Pix charge (a *cobrança*) and know when it was paid.

The interesting problem was not connecting to Pix. The first recipe in this series already did that, through MercadoPago. The problem was what that first integration quietly created: the agent could only ask for Pix in one vendor's dialect, which means a public rail had become a private dependency. This recipe is about the pattern that undoes that: one stable capability surface for the agent, with the payment processor as a swappable backend behind it. Two backends are live, MercadoPago and PagSeguro (PagBank), selected by a single environment variable.

A note on "rail-level," up front, because precision matters here. Direct participation in Pix's settlement infrastructure is reserved for institutions authorized by the Central Bank. A small-business stack reaches the rail through licensed payment service providers, and that is what both backends are. "Rail-level" in this recipe means the *design* treats the rail as the product and every processor as an interchangeable adapter, so the integration never hands the rail to any one of them.

## What this skill does

Three capabilities, exposed to the agent as MCP tools and to scripts as executables. The three-way name mapping is the contract:

| Capability | MCP tool | Script | Backends |
|---|---|---|---|
| `payments.pix_charge` | `pix_create_charge` | `scripts/create-charge` | both |
| `payments.pix_status` | `pix_get_charge` | `scripts/get-charge` | both |
| `payments.pix_list` | `pix_list_charges` | `scripts/list-charges` | mercadopago only |

Create returns the dynamic QR in both forms a Brazilian customer expects: the *copia e cola* text (the code you paste into a banking app) and an image link. Status returns a canonical state: `pending | paid | declined | canceled | expired | refunded | error`. The backend is chosen once, in configuration:

```bash
PIX_BACKEND=pagseguro    # or mercadopago. The agent never sees this.
```

## Pattern 1: the seam between agent and vendor is the capability, not the API

Recipe 03 argued that backends should be swappable in principle. This build is the principle running in production with two live backends, and the mechanics are worth spelling out.

The agent learns `payments.pix_charge` once. Everything vendor-specific lives behind the dispatch: which endpoint to call, how the request body is shaped, which fields hold the QR, what the provider calls a paid charge. A single dispatch module owns the backend allowlist, the credential preflight, the error typing, and the status maps, so there is exactly one place where "vendor" exists as a concept. Swapping processors is a one-variable configuration change. Nothing the agent reasons about moves.

That last part is the payoff in Claude Cowork: the business owner sees one connector named Pix. There is no vendor name on the tool surface at all, because the vendor is an implementation detail, deliberately kept invisible.

**The design lesson:** define the agent-facing contract around what the business needs done (charge, check, list) and force every vendor difference down into one dispatch layer. If "which vendor" appears anywhere the agent can see, the lock-in has already happened.

## Pattern 2: vendor-neutral does not mean vendors are identical

The two processors do not offer the same surface. PagBank's API has no endpoint to list past charges, so `payments.pix_list` simply does not exist on that backend. The tempting move is to fake it: cache charges locally, emulate the listing, present two vendors as one smooth surface. This build refuses. The interface declares, per capability and per backend, what is actually supported:

```yaml
capability_flags:
  payments.pix_list:
    backends: [mercadopago]
    pagseguro: unsupported_backend_capability
```

Calling list on the wrong backend fails immediately with a typed, machine-readable error (`unsupported_backend_capability`), not a silent empty result. The same flags declare the other honest limits: no recurring charges (Pix Automático is out of scope for v0), no refund execution (the `refunded` status is read-mapped only), sandbox environment only.

**The design lesson:** an abstraction that hides real vendor differences is a debt that comes due in production. Declare the differences in the contract itself, where both the agent and the human can plan around them, and make the unsupported path fail loudly with a typed error instead of quietly with wrong data.

## Pattern 3: mint your own correlation key, keep the raw values

Normalizing two vendors into one contract creates an identity problem: provider IDs are not stable across operations. On MercadoPago, for example, the search endpoint returns payment IDs while create returns an order ID, and the two do not link back to each other cleanly. Building on any one provider ID couples you to that provider's quirks.

The skill solves it by minting its own key. Every charge gets a skill-generated `external_reference`, passed to whichever backend is active, and that key is the stable handle across create, status, and list, on both backends. Alongside it, every response carries the raw material: the provider's own status string in `provider_status` and the full set of provider IDs, so nothing is lost in normalization and any dispute can be traced in the vendor's own terms.

```json
{
  "txid": "ORDE_...",
  "backend": "pagseguro",
  "status": "paid",
  "provider_status": "PAID",
  "external_reference": "os-pix-1760000000-12345",
  "qr_text": "00020126...",
  "expires_at": "2026-06-12T21:00:00-03:00"
}
```

**The design lesson:** when one contract fronts many vendors, identity has to belong to the contract, not to any vendor. Generate your own correlation key, thread it through every backend, and always return the raw provider values next to the normalized ones.

## Pattern 4: components that move money get adversarial review and hard safety rails

This connector creates real payment charges, so the build ran through two independent adversarial review gates: one attacking the plan before any code, one attacking the code before merge. The merge gate caught a real flaw the build had missed: a charge ID flowing into a URL path without sanitization, an injection vector in a component that talks to payment APIs. It was fixed and re-verified before the connector ever touched a charge.

The same posture shows up as hard rails in the runtime. v0 is sandbox-only, and that is enforced, not documented: pointing the PagSeguro backend at the production environment fails hard. The sandbox payer fallback (placeholder customer data, which PagBank order creation requires) is explicitly forbidden from ever applying in production. Errors are a typed contract, `{error, code, retryable, backend}`, so an agent can distinguish a bad input from an expired credential from a provider outage and react accordingly instead of parsing prose.

**The design lesson:** for connectors that can move money, an independent adversarial pass before merge is the cheapest insurance you will ever buy. And encode environment limits as hard failures in code; a "sandbox only" line in the README protects no one.

## Setup notes

- **Sandbox credentials are the fast part.** PagBank sandbox onboarding takes minutes; MercadoPago sandbox setup is covered in Recipe 01. Both tokens live in the OS keychain or 1Password, resolved at runtime by the scripts themselves.
- **Keep only the active backend's token line in `.env`.** The secret resolver expands the whole env file, so a reference to an unprovisioned secret for the unused backend fails the run even though that backend was never called. One backend, one token line.
- **The MCP server connects first, warms secrets second.** Cold secret-store resolution can take a minute, far past the host's MCP handshake timeout, so the server establishes its stdio connection immediately and resolves credentials in the background. Expect the first tool call of a cold session to be slow and everything after it to be fast.
- **Verify with a full round trip before wiring it to an agent.** Create a charge in sandbox, read it back by its ID, and (on MercadoPago) confirm it appears in list. The PagSeguro sandbox auto-pays charges, which conveniently exercises the `pending` to `paid` transition end to end.

## What's not in this recipe

- The webhook receiver. Payment confirmation here is polling; the webhook normalization contract is documented for a separate, later build.
- Pix Automático (recurring charges) and refund execution.
- Direct participation in the Central Bank's settlement infrastructure, which is a licensed-institution path, not a small-business integration.

## Where this came from

Built as the closer of the LATAM adaptation arc: payments, fiscal documents, customer channel, and now the payment rail treated as what it is, public infrastructure. Four recipes, four authorities shaping the design. The companion LinkedIn article on this build: [Brazil made payments free. I almost added the cost back without noticing.](https://www.linkedin.com/pulse/brazil-made-payments-free-i-almost-added-cost-back-without-goulart-xeh8f)
