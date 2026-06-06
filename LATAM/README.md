# LATAM patterns

LATAM-specific recipes paced behind the [KOOKY OS](https://github.com/kooky-labs) build cycle. This directory is the public companion to a LinkedIn series on adapting Anthropic's *Claude for Small Business* for the LATAM stack.

**Start here:** [00-the-idea.md](./00-the-idea.md) for what this directory is, what makes a good LATAM recipe, and how it connects to the underlying build cycle.

Each recipe walks through one shipped LATAM connector or workflow as a Cookbook-shaped pattern: capability surface, integration shape, sandbox setup, error handling, and the design lessons that aren't obvious from the official docs.

## Recipes

| # | Title | Status | Article |
|---|---|---|---|
| 00 | [The idea](./00-the-idea.md) | live | [Article #1 — Why Claude for Small Business might face slower adoption in LATAM](https://www.linkedin.com/pulse/why-claude-small-business-might-face-slower-adoption-latam-goulart-u1ywe/) |
| 01 | [MercadoPago skill (cards + Pix)](./01-mercadopago-skill.md) | recipe live, skill shipped | [Article #2 — Localizing Claude for Small Business for LATAM](https://www.linkedin.com/pulse/localizing-claude-small-business-latam-gap-isnt-language-goulart-geuye) |
| 02 | [Conta Azul skill (accounting + the fiscal-emission boundary)](./02-conta-azul-nfe.md) | recipe live, skill shipped | [Article #3 — My Claude for Small Business got a NO from ContaAzul this week](https://www.linkedin.com/pulse/my-claude-small-business-got-from-contaazul-week-renato-goulart-ohiff) |
| 03 | [WhatsApp for a sandboxed agent (the host MCP pattern)](./03-whatsapp-business.md) | recipe live, skill shipped | [Article #4 — AI can read your books. Make sure it can't accidentally damage your business](https://www.linkedin.com/pulse/ai-can-read-your-books-make-sure-cant-accidentally-damage-goulart-5eref/) |
| 04 | Pix Direct (rail/BCB, vendor-neutral) | planned (June 2026) | |

## Why this fork

Two reasons:

1. **Public code receipts.** The LinkedIn series tells the operator-level story. This directory is where the technical patterns live, indexed for engineers and partnership evaluators.
2. **Upstream pattern candidates.** Some patterns generalize beyond LATAM. Where they do, the plan is to contribute them back to [anthropics/claude-cookbooks](https://github.com/anthropics/claude-cookbooks) via PR, selectively, when the pattern earns it.

## Cadence

Each recipe lands after the corresponding skill ships. This directory is the slower-paced, second-pass companion to the build, not active build work. Better to ship one strong recipe than four thin ones.

---

Maintained by [Renato Goulart](https://www.linkedin.com/in/renatogoulart) at [KOOKY AI Exchange](https://github.com/kooky-labs). Independent commentary, not affiliated with Anthropic.
