# zama-fhevm-tooling

An agent skill for designing and building confidential tools on Zama's fhEVM.

It teaches an AI coding agent how to build verifiers, gates, meters, and
scorers: contracts that evaluate encrypted data and reveal only a small
result, never the inputs. It is scoped to tooling, not confidential token
transfers, payments, or auctions.

## What it does

AI coding agents have no built-in knowledge of fhEVM. They do not know the
encrypted types, the ACL rules, the async decryption flow, or the cost
limits. The result is plausible-looking code that fails silently or leaks
data.

This skill closes that gap for one specific class of work: confidential
tooling. After loading it, an agent can take a plain-English request like
"build a tool that checks an encrypted credential" and produce a correct
fhEVM contract that keeps the input private.

## The four patterns

Every tool in scope is one of these, or a composition of them:

- **Verifier** checks whether a submitted encrypted value matches a stored
  encrypted reference. Key verification, allowlist and membership checks.
- **Gate** decides pass or fail by comparing an encrypted value to a
  threshold. Eligibility gating, access control on an encrypted attribute.
- **Meter** tracks a running encrypted count against an encrypted limit.
  Usage metering, encrypted rate limiting, quota enforcement.
- **Scorer** computes a score from encrypted inputs, then checks it against
  a threshold. Private scoring, confidential risk assessment.

## What is inside

zama-fhevm-tooling/
├── SKILL.md                      The decision framework and the four patterns
└── references/
├── capabilities.md           Operations on encrypted types and hard limits
├── acl-rules.md              allow, allowThis, allowTransient, isSenderAllowed
├── revealing-results.md      The async public decryption flow
├── client-integration.md     Off-chain encryption, the SDK, the ZKPoK proof
├── cost-and-limits.md        HCU metering, budgets, expensive operations
├── anti-patterns.md          Common mistakes and their symptoms
└── network-setup.md          Networks, tooling versions, licensing

The skill loads progressively. The agent reads SKILL.md first and pulls a
reference file only when the task reaches that step.

## Installation

### Claude Code

Clone the repo into your skills directory:
git clone https://github.com/ronniethedevv/zama-fhevm-tooling.git 
~/.claude/skills/zama-fhevm-tooling

Claude Code reads the frontmatter description and loads the skill
automatically when a task matches.

### Cursor

Place the `zama-fhevm-tooling` folder inside your project. Either reference
`@SKILL.md` directly in the chat when you need it, or add a project rule
that points the agent to `zama-fhevm-tooling/SKILL.md` for confidential
fhEVM work so it loads automatically.

### Other agents

The skill is a plain folder of Markdown files. Load `SKILL.md` into the
agent's context as a system prompt or instruction file. It works with any
agent that accepts instruction files.

## Scope and honest limits

This skill is deliberately narrow. It does not cover confidential token
transfers, payments, or auctions, and it will tell the agent to stop and
reconsider if a task needs real Solidity control flow driven by an
encrypted value, since that is not possible on fhEVM.

It is also honest about the boundary of FHE itself: encryption happens
client-side, so whoever runs the tool that encrypts the data sees the
plaintext first. fhEVM hides the value from the contract, the chain, and
every other party, but not from the encrypting client. The skill carries
this framing so the agent does not oversell the privacy guarantee.

## Sources

The technical content is drawn from the official Zama documentation and
the Zama documentation assistant. Cost figures and network details change
over time. Verify the HCU figures in cost-and-limits.md and the network
details in network-setup.md against current Zama docs before any
production deployment.

## License

See the LICENSE file.

## Contributing

Issues and pull requests are welcome. Useful additions include new tool
patterns, more worked examples, and corrections as the Zama protocol
evolves.