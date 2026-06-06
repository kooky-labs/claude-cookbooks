# Recipe 02: Conta Azul skill (accounting + the fiscal-emission boundary)

The second LATAM adaptation. A Conta Azul skill that records sales, reads cash flow, and tracks the issuance of Brazilian fiscal documents (NFS-e and NF-e). The headline finding is not a clever endpoint. It is a boundary: the one step that legally belongs to a human stays with the human, and the skill is designed to wrap tightly around it.

This recipe walks through the pattern shapes that came out of building it. The skill itself lives in a private repo (`kooky-os-skills/skills/os-conta-azul/`); this recipe extracts the patterns that generalize.

A few terms, glossed once for readers outside Brazil:

- **NFe** (Nota Fiscal Eletrônica): Brazil's mandatory electronic invoice for goods.
- **NFS-e** (Nota Fiscal de Serviço eletrônica): the municipal variant for services. KOOKY sells services, so NFS-e is the primary path here. NF-e is the same shape.
- **SEFAZ** (Secretaria da Fazenda): the state tax authority that authorizes an NF-e. For NFS-e, the município (city) authorizes.
- **ICP-Brasil**: Brazil's national public-key certificate chain. An **e-CNPJ** certificate (a company-level cert) is what signs a fiscal document.
- **contribuinte**: the registered taxpayer (the business issuing the invoice).
- **venda**: a sale. In Conta Azul this is the system-of-record entity a fiscal document is later emitted from.
- **pessoa** / **tomador**: a person-or-entity record, and the party receiving the invoice.

## What this skill does

The skill exposes accounting capabilities to callers, plus one bridge back to the payments skill:

- `accounting.customer.*`: create, get, and list customers (clientes), including foreign tomadores.
- `accounting.invoice.*`: create, get, and list sales (vendas), the system-of-record entity.
- `accounting.financial_event.list`: list receivables or payables by due date (the cash-flow read).
- `accounting.account.balance`: real-time cash on hand across financial accounts.
- `accounting.nfse.status`: list and poll emitted NFS-e and their authorization state.
- `payments.reconcile`: a bridge capability that matches a payment back to a sale by `external_reference`.

There is one capability the skill deliberately does **not** implement: `accounting.nfe.issue`. The reason is the first pattern.

## Pattern 1: prepare and verify around a step that belongs to a human

A fiscal document in Brazil is not optional metadata. It is a legal act. Issuing one requires the contribuinte to sign the document with an ICP-Brasil certificate and obtain authorization from SEFAZ (for NF-e) or the município (for NFS-e). That much is set by law.

What is *not* set by law is who clicks the button. Conta Azul, as the ERP, keeps emission inside its own web UI: the operator imports the e-CNPJ certificate once, then clicks "Emitir" on each sale. Conta Azul's public v2 API exposes consultation of fiscal documents but no emission endpoint (the services NFS-e emission API is listed as "em breve", coming soon). This is a product-architecture choice, not a legal one. Other Brazilian fiscal platforms (for example PlugNotas or Focus NFe) expose programmatic emission and are equally compliant. The law mandates the signing and the authorization, not the integration surface.

So if you are integrating against Conta Azul specifically, you cannot emit through the API. The naive reactions are both wrong:

1. Block the whole flow until emission is automatable. It never will be on this platform.
2. Pretend to emit (fake the step, or screen-drive the UI). That fakes a legal act, which is exactly the thing you must not automate away.

The honest design is to do everything up to the boundary, hand the one reserved act to the human cleanly, then resume on the other side by verifying the result:

```bash
# 1. The skill prepares the sale.
./scripts/create-customer --name "ACME Ltda" --document "12345678000199" --type Jurídica
./scripts/create-invoice  --customer_id <id> --items '[...]' --external_reference "<payment_id>"

# 2. The human emits. The operator opens the sale in the Conta Azul UI and clicks "Emitir".
#    The certificate signs it; SEFAZ or the município authorizes it. The skill does not touch this.

# 3. The skill verifies.
./scripts/get-nfse-status --start_date 2026-06-01 --end_date 2026-06-15 --numero_venda <n>
```

**The design lesson:** when a step is reserved to a human, by law or by the platform, design the automation to wrap tightly around it rather than through it. Do everything up to the boundary, hand off explicitly, then pick the result back up by polling. The connector that prepares and verifies is more useful (and more honest) than one that pretends the boundary is not there.

## Pattern 2: read server-owned state before you write it

Conta Azul's sale-creation has two fields you cannot construct on the client side. You have to read them from the server first.

- **The sale number (`numero`) is required and collision-prone.** Guessing it (last number plus one) races against any other actor creating sales. Conta Azul exposes the next safe value directly: `GET /v1/venda/proximo-numero` returns a bare integer. Read it, then use it.
- **Line items must reference existing inventory.** A sale item is not free text. Each item's `id` must be a pre-existing serviço or produto UUID from the catalog (`GET /v1/servico`). The código de serviço (the LC-116 service classification) and the ISS tax rate live on the serviço record, not on the sale. The sale just points at it.

```bash
# Read the next sale number from the server, never compute it locally.
numero=$(./scripts/_lib/ca_request GET /v1/venda/proximo-numero)

# Resolve item IDs against the live catalog before building the sale.
service_id=$(./scripts/_lib/ca_request GET "/v1/servico?busca=Consultoria" | jq -r '.itens[0].id')
```

**The design lesson:** some fields are owned by the server, not the caller. Sequence numbers, valid inventory IDs, anything the backend assigns or validates against its own state. Read them first; do not synthesize them. This is the inverse of the auto-detect pattern from Recipe 01: there the input carried its own meaning, so the tool decided; here the backend owns the meaning, so the backend decides.

## Pattern 3: an async regulatory result is terminal and authoritative

After the human emits, authorization is asynchronous. The document passes through processing states (`EMITINDO`, `AGUARDANDO_RETORNO`) before reaching a terminal one: `EMITIDA` (authorized) or `FALHA` (rejected). Two properties of the lookup shape the integration.

First, **the status read is windowed**. `GET /v1/notas-fiscais-servico` requires a competência (accrual) date range, and that range is capped at 15 days. You cannot ask "what is the status of this one document" by ID alone; you ask "what NFS-e exist in this window" and filter. So the poll is always scoped to a date window, not a single key.

Second, **the terminal state is the regulator's answer, not a transient error**. A `FALHA` is not a 5xx to retry. It means the document was rejected (wrong tax setup, an invalid tomador field, a município rule), and a human has to fix the underlying data before re-emitting.

```bash
# Poll within the required competência window until the document is terminal.
./scripts/get-nfse-status \
  --start_date 2026-06-01 --end_date 2026-06-15 \
  --numero_venda 41 --status EMITIDA,FALHA
```

The skill's escalation rules follow from this: a document stuck in `EMITINDO` past a few minutes means the authorization queue is backed up (wait, do not resubmit), and a `FALHA` surfaces the rejection reason verbatim and stops (no auto-retry, because retrying bad data just fails again).

**The design lesson:** treat a regulatory or otherwise authoritative async result as terminal. Poll inside the window the API gives you, and when the answer is "rejected", route it to a human with the reason intact. Automatic retries belong to transport failures, not to a tax authority saying no.

## A foreign-customer gotcha worth its own line

Conta Azul validates a customer's cidade and estado against the Brazilian município table. A foreign tomador (customer type `Estrangeira`) has neither, and sending them returns a 400. The fix is counterintuitive: for foreign customers, **omit** cidade and estado entirely and carry the location through `país` and `logradouro` instead. Also note that `document` (CPF or CNPJ) is not required for foreign customers, because they have no Brazilian tax ID. A skill that assumes every customer has a document and a Brazilian city will reject every foreign one.

## Setup and testing notes

A few non-obvious gotchas from the build:

- **There is no sandbox.** Conta Azul has no isolated test environment. The closest thing is a 30-day Development-type app that runs against the same API on live data. Plan testing accordingly: you are always touching a real account, so use a dedicated test company and fictitious-but-valid data.
- **OAuth scope is a single fixed string, not granular.** Conta Azul's auth is built on AWS Cognito and grants `openid+profile+aws.cognito.signin.user.admin`: full admin on whatever account authorizes the app. There are no per-resource scopes. Least-privilege is therefore your skill's responsibility to enforce, not something you can lean on OAuth to provide.
- **The refresh token rotates on every renewal.** It has a long life (years), but every refresh call returns a new refresh token and invalidates the old one. Persist the new token atomically inside the refresh call, or the next call fails.
- **Read the portal values, not the docs.** Two drifts cost time: the live API base is `api-v2.contaazul.com` (several doc pages still say `api.contaazul.com`), and a Development app's redirect URI is auto-set to `https://contaazul.com` without `www` (the docs say `www`). The portal is authoritative; the docs lag.
- **Credential leaks delete the app.** Conta Azul's stated policy is that a detected credential leak deletes the application immediately and without notice. Never commit the client secret or token store. Treat the secret as the highest-risk value in the integration.

## What's not in this recipe

- The OAuth bootstrap flow (manual authorization-code to token exchange). It is a one-time per-credential step documented separately.
- Production NF-e for goods. The same prepare-and-verify shape applies, but with SEFAZ rather than the município as the authorizer.
- NFS-e cancellation and correction (carta de correção). Later build.
- The reconciliation bridge internals (`payments.reconcile`), which match a payment back to a sale by `external_reference`. That pairs with Recipe 01.

## Where this came from

Built during the LATAM adaptation sprint, after Recipe 01 (MercadoPago). The companion LinkedIn article tells the operator-level story of discovering the emission boundary the hard way: [My Claude for Small Business got a NO from ContaAzul this week. The why was more interesting than I expected](https://www.linkedin.com/pulse/my-claude-small-business-got-from-contaazul-week-renato-goulart-ohiff).

Next recipe lands when the WhatsApp Business skill ships.
