# Demo script: the beats and the talk track

Run the beats from the **control panel** (`PANEL=true` in `.env`, reached through `scripts/tunnel.sh`),
or type the equivalent `kubectl` from the workshop modules. Keep the Lynx console open beside it so the
audience sees each click land live. Every button maps to a real change on the cluster.

The arc: **Discover, Register, Default-deny, Allow, Least privilege, Block a lane, Contain, (MCP), Reset.**

---

## 0. Set the scene (before you click anything)
> "This is NorthBridge, a financial company. It already runs AI agents in its cluster: a Concierge that
> fields customer requests and hands off to Retail, Payments, Fraud, Wealth, and so on, plus a platform
> engineering team with agents of its own. Nobody signed off on all of them. Let's see what Lynx does with that."

Open the console on **Discover**. Point at the shadow workloads the eBPF detector already flagged
(five in the estate; the two to name on stage are `dev-test-agent`, which someone spun up to test and
forgot, and `app-connect-agent`, which an app scaffolded and nobody registered).

## 1. Discover (eBPF)
- On **Discover**, open a shadow's card. It was found at the kernel by its **SNI** to a model endpoint,
  by the SDKs loaded in its process, and by its environment variable names. No sidecar, no cooperation
  from the workload. `dev-test-agent` talks to Gemini, `app-connect-agent` to OpenAI, so the audience
  sees two distinct destinations.
> "Lynx didn't need the app's help. It saw the TLS handshakes to the model API and the libraries in
> the process and said: that's an agent, and it's not on the books."

## 2. Register, then default-deny
- Panel: **Register agents**. The 14 registered agents (8 bank, 6 platform) land in **Register > Agents**.
- Panel: **Default-deny** (removes the base policy).
- In the Concierge chat, ask for anything cross-agent. It cannot even start: with the base policy gone,
  nothing permits a human to reach the Concierge.
> "Registered is not trusted. With no policy, Lynx denies everything. This is the safe default."

## 3. Blanket allow (the naive step)
- Panel: **Blanket allow**.
- Re-run the request. Now the full chain works: Concierge, Retail, Payments, Ledger.
> "Open the floodgates and it works, but now any agent can call any agent. That's not what we want."

## 4. Least privilege (the point)
- Panel: **Least-privilege**. This applies the real base policy: only the lanes each department needs
  (Concierge to Retail, Fraud and Wealth; Retail to Payments and KYC; Payments and Fraud to Ledger).
- Re-run the payment chain: still works. Try a lane that is not permitted: it does not.
> "Same demo, but now every lane is explicit and testable. Least privilege for agents."

## 5. Block a lane (read this, it is subtle)
- Panel: **Block Retail→Payments**. Wait a minute (agents refresh their view of the registry every
  60 seconds), then re-run the payment request.
- **What happens:** the chain stops at Retail. Retail cannot complete the payment.
- **What does NOT happen:** you will *not* see a red "denied" hop mid-chain.

**Why:** Lynx enforces at **discovery**, not only at call time. The `forbid` removes Payments from
what Retail can even *see*. Retail's peer list no longer contains Payments, so its model never
attempts the call, so there is no 403 to render. The security story is *stronger* than a denial:
> "Watch: I block Retail to Payments and re-run. Retail goes to move the funds and it can't, because
> Payments just disappeared from Retail's world. Lynx didn't only block the call, it removed the
> target from discovery. An agent can't attack what it can't see."

Show the edge vanishing in **Observe > Access Map**; that is the visible artifact for this beat.

## 6. Contain (quarantine), the dramatic moment
- Panel: **Quarantine app-connect**. This labels `app-connect-agent`'s workload (the OpenAI-calling
  shadow nobody registered). On Lynx 1.4.x quarantine is a built-in deny-all **sandbox**: the eBPF
  layer blocks **process execution and file writes** in the kernel ("operation not permitted").
  Network is a separate sandbox policy (**Govern > Sandbox**), so say "it cannot run or write", not
  "it cannot talk".
- Show the workload's sandbox reading `builtin::quarantine` on **Discover**.
> "This one's rogue, an agent an app stood up that nobody signed off on. One click. It cannot run a
> process or write a file any more, enforced in the kernel, not with a rule it might dodge. That's containment."
- Panel: **Release app-connect** to undo.

## 7. MCP governance (optional)
- Panel: **Approve Hugging Face MCP** / **Send back to review**. Flips an MCP server between usable
  and blocked by changing its `reviewStatus` (the review gate forbids anything pending) and its
  `readOnly` attribute (the base policy permits read-only servers). No agent is touched.

## 8. Reset
- Panel: **Reset** returns everything to the clean pre-demo state (release quarantine, revoke the
  MCP approval, clear the block lanes). Safe to click any time you get lost.

---

## Good prompts to run in the Concierge chat (rich multi-hop trails)
1. *"A retail customer wants to send a $2,000 payment from their chequing account. Coordinate it end to end."*
   Concierge, Retail, Payments, Ledger.
2. *"A new customer is opening an account and needs identity verification."* Concierge, KYC, Retail.
3. *"Flag any suspicious activity on this account and log the review."* Concierge, Fraud, Ledger.

## Identity cheat-sheet (for the curious exec)
- **Agent identity = Kubernetes service-account workload identity.** In an agent's token:
  `iss = https://kubernetes.default.svc.cluster.local`, `sub = system:serviceaccount:<ns>:<agent>`.
- **Human login = Keycloak/OIDC** (`iss = <console origin>/realms/lynx-demo`).
- **Service identities** are non-AI callers (a Slack bot, a batch job) governed by the same rules.
- **Discovery is policy-filtered:** an agent asks the registry for peers and only gets back the ones
  policy permits, as gateway proxy URLs. That is why beat 5 behaves the way it does.
