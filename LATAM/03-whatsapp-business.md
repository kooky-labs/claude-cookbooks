# Recipe 03: WhatsApp for a sandboxed agent (the host MCP pattern)

The third LATAM adaptation. WhatsApp is the customer channel in Latin America: for a PME (pequena e média empresa, a small or medium business) it is often the only channel customers actually use. So a Claude-powered operations agent has to be able to read and send WhatsApp messages.

The interesting problem turned out not to be WhatsApp at all. It was *where the connector runs*. When the agent lives inside a sandbox (as Claude in Cowork does), it cannot reach host secrets or a private internal network. This recipe is about the pattern that solves that: run the connector as a host MCP server, outside the sandbox, and expose it to the agent as tools. WhatsApp is the concrete example.

A note on the backend, up front, because it matters. For **production**, the compliant path is Meta's official **WhatsApp Cloud API** (part of the WhatsApp Business Platform), with the Business Verification and message-template approval that entails. The implementation behind this recipe used a **self-hosted WhatsApp gateway** for development and internal dogfooding, which is faster to stand up but is not the official API. The pattern below is deliberately written so the backend is swappable: the agent-facing capability does not change when you move from a development gateway to the official Cloud API. That swappability is Pattern 3, and it is the point.

## What this skill does

The connector exposes four capabilities to the agent, as MCP tools:

- `whatsapp_list_instances`: list connected WhatsApp instances and their connection state. Read-only, no arguments.
- `whatsapp_instance_status`: live connection state of one instance. Read-only.
- `whatsapp_send_text`: send a text message to a number. **Write: this sends a real message.**
- `whatsapp_send_media`: send media (image, document, etc.) to a number. **Write: sends a real message.**

Once registered, the agent sees these as `mcp__os-whatsapp__whatsapp_list_instances` and so on, in both Claude Desktop chat and Cowork.

## Pattern 1: when the agent is sandboxed, run the connector as a host MCP server

Claude in Cowork runs in a sandbox. That sandbox is a security feature, and it has two consequences for any integration:

1. It cannot read host secrets. The API credential in the host's 1Password or macOS Keychain is not reachable from inside.
2. It cannot reach a private internal network. If the WhatsApp gateway lives on an internal network (a VPN or overlay network, not the public internet), the sandbox's egress rules block it.

A connector that needs both a host secret and internal-network egress therefore cannot run inside the sandbox at all. The resolution is to invert where it runs: the connector is a small **MCP server launched by the host** as a local stdio subprocess (Claude Desktop starts it), living **outside** the sandbox. It holds the host's trust: it can read the Keychain secret and reach the internal gateway. The sandboxed agent never touches either; it just calls the tool, and the host process does the privileged work and returns the result.

Registration is a few lines in the host's MCP config:

```json
"mcpServers": {
  "os-whatsapp": {
    "command": "/usr/local/bin/node",
    "args": ["/abs/path/to/os-whatsapp/mcp/dist/index.js"]
  }
}
```

After a restart, the tools appear to both Desktop and Cowork. The privileged work (secret access, internal egress) happens in the host subprocess; the agent stays sandboxed.

**The design lesson:** a sandbox boundary is also a capability boundary. When an agent is sandboxed but a task needs host trust (secrets) or network reach the sandbox denies, do not try to widen the sandbox. Move that work to a host-side MCP server and let the agent call into it. The agent stays contained; the privileged surface stays small, explicit, and auditable.

## Pattern 2: separate read tools from write tools when writes cannot be undone

A sent WhatsApp message cannot be unsent. That makes the blast radius of these tools uneven: listing instances is harmless, sending a message is irreversible and customer-facing. The connector makes that asymmetry explicit rather than leaving it implicit:

- The **read** tools (`list_instances`, `instance_status`) take no destructive arguments and are safe to call freely, including during exploration.
- The **write** tools (`send_text`, `send_media`) are documented as sending a real message, in the tool description the model sees, so the agent treats them with appropriate caution.

The credential is resolved **once** at server startup and cached for the process lifetime: at most one secret-store unlock per launch, never one per message. On a client machine where the backend is the OS keychain with a pre-granted ACL, it never prompts at all.

```text
[host MCP server starts]
   -> resolve the WhatsApp gateway credential once from Keychain/1Password
   -> cache for process lifetime
[agent calls whatsapp_send_text]
   -> reuse cached credential, call the gateway, return the result
```

**The design lesson:** label irreversible, outward-facing tools as such where the model can see it, and keep read and write on clearly different footings. Resolve secrets once at startup for a long-running connector, so the security cost (an unlock prompt) is paid per launch, not per call.

## Pattern 3: keep the capability stable so the backend can change

The agent-facing capability is "send a WhatsApp message to this number." Nothing in that contract says *how*. That is deliberate. Behind `whatsapp_send_text` sits one transport today (a self-hosted gateway, for development and internal use) and a different, official one tomorrow (Meta's WhatsApp Cloud API, for production, with its Business Verification and approved message templates). The migration from the first to the second should not change a single tool name or argument the agent reasons about.

So the connector keeps the transport details (base URL, auth header, the gateway's specific endpoints and message shapes) entirely inside the server, behind a stable tool surface. Swapping the backend is a change to the server's internals, not to the agent's tools.

**The design lesson:** the right seam between an agent and an external service is the *capability*, not the vendor. Define tools around what the user wants done, hide the transport behind them, and you can change or upgrade the backend (here, move from a development gateway to the official, compliant API) without retraining the agent or rewriting the workflows that call it.

## Setup notes

- **The host must have the reach the sandbox lacks.** The MCP server runs on the machine running Claude Desktop / Cowork, and that machine must be able to reach the WhatsApp gateway (for an internal-network gateway, that means being joined to the same private network) and to read the credential from its local secret store.
- **It is a local stdio subprocess, not a hosted service.** Claude Desktop launches it; it is not the public internet's business. MCP config loads at launch, so fully quit and reopen the host app after registering.
- **Verify read-only first.** Drive a `tools/list` and a `list_instances` call over stdio before ever invoking a write tool, so you confirm wiring without sending anyone a message.

## What's not in this recipe

- The official Meta WhatsApp Cloud API onboarding: Business Verification, phone-number registration, and message-template approval. That is the production path and deserves its own write-up.
- The pairing and session mechanics of the development gateway.
- Multi-tenant credential isolation (one client, one instance, one credential), which is a later build.

## Where this came from

Built during the LATAM adaptation sprint, after Recipe 02 (Conta Azul). WhatsApp is the channel that makes a LATAM operations agent useful at all, and getting it to work from inside a sandboxed agent is what surfaced the host-MCP pattern. The companion LinkedIn article on the WhatsApp build follows in the series.

Next recipe lands when the Pix Direct skill ships.
