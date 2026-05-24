# The idea

LATAM-specific recipes paced behind the [KOOKY OS](https://github.com/kooky-labs) build cycle. This directory tracks patterns that surface when you bring Anthropic's *Claude for Small Business* (a Cowork plugin with 31 pre-built Skills and 12 default Connectors) into a regional context the default Connectors don't cover.

Each recipe here is post-hoc. The pattern earns its slot by emerging from a working build, not from a thought experiment. Recipes land after the corresponding skill ships in production-shaped form. No speculation, no "we should probably try."

## Why a separate LATAM section

The Cookbook upstream is excellent for general-purpose patterns. It's structured around the patterns that work everywhere: prompt design, tool use, structured outputs, RAG, MCP integration. What it doesn't cover, by design, is what happens when the rails change underneath the same workflow.

A `/cash-flow-snapshot` Skill from *Claude for Small Business* is a clean example. The Skill works anywhere. The Connector beneath it doesn't. In Brazil, Stripe and QuickBooks aren't the defaults. MercadoPago and Conta Azul are. The Skill keeps working; the integration shape underneath has to change.

This directory documents what changes, and what stays the same. Regional integration knowledge expressed as Cookbook-shaped recipes.

## What makes a good LATAM pattern recipe

Three tests:

1. **It came from a working build.** Every pattern here came out of the build cycle, after the skill shipped against a real (sandbox or production) API. Not from documentation. Not from architectural speculation.
2. **It has a design lesson, not just an implementation.** The recipe explains why the pattern looks the way it does. What the alternative was, what broke when you tried it, what makes the working version the right shape.
3. **It transfers.** Some patterns are MP-specific. Others apply to any regional payment processor. Or to any heterogeneous integration. The recipe should be explicit about which.

## Connection to the LinkedIn series

There's a LinkedIn series running in parallel that tells the operator-level story of each build. It opened with [Why Claude for Small Business might face slower adoption in LATAM](https://www.linkedin.com/pulse/why-claude-small-business-might-face-slower-adoption-latam-goulart-u1ywe/), which sets up the gap-map this directory is designed to close.

Different audience (founders, partnership leads, strategy folks), different pacing (one Article per shipped adaptation), different framing (build narrative, market context, vendor-vs-platform-partner thesis).

The two are complementary. The LinkedIn pieces explain why a build matters. The recipes here explain how the build actually works. If you want the strategic frame, read the article. If you want the integration code shape, read the recipe.

## Selective upstream PRs

Some patterns here will earn upstream PRs to [anthropics/claude-cookbooks](https://github.com/anthropics/claude-cookbooks). The auto-detect-by-ID pattern, the SPEC-driven error-handling rules: these aren't LATAM-specific. They emerged from a LATAM build but apply to any heterogeneous-ID, multi-product-family integration.

The bar for a contributed PR is high: the pattern has to generalize cleanly, the example has to stand on its own without LATAM context, and the value-add has to be obvious. Better to contribute two strong patterns by the end of the run than ten weak ones across it.

## What's NOT here

- Credentials, sandbox tokens, or any live API keys. Recipes describe shape and pattern; they don't ship working secrets.
- Client-specific code from KOOKY production deployments. The recipes are derivative of the public-shaped patterns, not the live business logic.
- Speculative integrations. If a recipe shows up before its corresponding skill ships, that's a bug in the cadence.

---

Start with [01-mercadopago-skill.md](./01-mercadopago-skill.md): the first concrete recipe. The os-mercadopago skill ships cards + Pix payments through a single capability surface, with three reusable patterns underneath.
