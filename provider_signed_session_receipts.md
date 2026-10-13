# Provider-Signed Session Receipts for Claude

**Proposal draft, 2026-10-13.** Idea by scsnake. Drafted with Claude Fable 5.1 from my idea.

## Summary

Today a person can claim "this PR was written by Claude Fable" and nobody can check it. A person can also edit Claude's output and still present it as pure model output. This proposal asks Anthropic to sign a small receipt for each session. The receipt proves two things: the output is the exact bytes that a named Claude model emitted, and no turn was changed, added, or removed after the fact. Verification works offline with a public key.

This is a positive certificate of origin. It is not a detector. Watermarks answer "did an AI touch this?" A receipt answers "did this model, with these settings, produce exactly this?"

## Why it is needed

1. **Model tier is a quality signal.** A PR made by Fable at high effort with tests is a different object from a PR made by Haiku at low effort. Reviewers, CI gates, and downstream agents can set trust by tier. Today the model name is a free-text trailer that anyone can type.
2. **Human edits hide inside model output.** A `Co-Authored-By` line survives a hand edit. A reviewer cannot tell which lines the model wrote and which lines a person changed afterward.
3. **Watermarks do not solve this.** Anthropic's SynthID-based text watermark is detect-only, limited to eligible parties, weak on short code, and not tied to a model tier. The watermark is still useful. The receipt is a different layer.
4. **The pieces already exist.** Each API call has a request ID. Claude Code already has the full transcript. GitHub already shows a "Verified" badge for app-signed commits. What is missing is a signature, a chain link, and a public key directory.

## Design

The diagram below shows the end-to-end lifecycle from prompt to verified PR check:

```mermaid
sequenceDiagram
    autonumber
    participant Human as Human author
    participant Harness as Harness (Claude Code)
    participant Prov as Model Provider (Anthropic)
    participant Repo as GitHub (PR and tree)
    participant Verifier as Verifier (CI or reviewer)

    Note over Human,Harness: Local side. The certificate does not trust it.
    Note over Prov: Only the provider computes hashes and signs.

    Human->>Harness: Give the task
    Harness->>Prov: Send prompt and base commit hash
    loop Session turns
        Prov->>Harness: Return output, tool calls, and step hash
        alt Local mode
            Harness->>Prov: Return tool results (file reads, command outputs)
        else Hosted mode
            Prov->>Prov: Run tools in the provider sandbox, record the tree hash
        end
    end
    Prov->>Harness: Issue signed receipt (chain head, closed marker, overlap score)
    Harness->>Repo: Push PR with receipt trailer or git note
    opt Public key not in cache
        Verifier->>Prov: Fetch public key from JWKS directory (one time, then cached)
    end
    Verifier->>Repo: Read receipt and PR tree
    Verifier->>Verifier: Check signature, closed marker, and code tree (offline)
    Verifier-->>Repo: Publish check (model, effort, overlap score, tree match)
```

### Chain

For each step `i` in a session, the provider computes:

```
h_i = H( h_(i-1) || H(salt_i || input_i) || H(output_i) || spec_i || t_i )
```

- `input_i` is everything the model saw at that step: user text, system prompt, tool results, attached files. In an agent session most input is tool output. If tool output is not hashed, a person can smuggle hand-written code in through a file read.
- `output_i` is the exact bytes the model emitted, including tool calls. Hidden reasoning is committed as a separate hash and stays private.
- `spec_i` holds the model ID, model snapshot, reasoning effort, sampling settings, tool list, and harness version.
- `t_i` is the provider timestamp.
- `salt_i` is a random value per step. Short inputs like "yes" are easy to guess from a bare hash.

A regenerated turn creates a fork. Each step records its parent, so a verifier can see that a branch was not the first attempt.

```mermaid
flowchart TD
    subgraph Step1["Step 1"]
        direction TB
        In1["Input 1 + salt 1<br/>(User prompt, base commit)"]
        Out1["Output 1<br/>(Model text, tool calls)"]
        Spec1["Spec 1<br/>(Model ID, effort, tools)"]
        T1["Timestamp t_1<br/>(Provider clock)"]
        H1["Step Hash: h_1"]
        In1 & Out1 & Spec1 & T1 --> H1
    end

    subgraph Step2["Step 2, first attempt (abandoned)"]
        direction TB
        In2["Input 2 + salt 2<br/>(Tool results, file reads)"]
        Out2["Output 2<br/>(Code edits, tool calls)"]
        Spec2["Spec 2<br/>(Model ID, effort, tools)"]
        T2["Timestamp t_2"]
        H2["Step Hash: h_2"]
        In2 & Out2 & Spec2 & T2 --> H2
    end

    subgraph Step2R["Step 2, regenerated (fork)"]
        direction TB
        In2R["Input 2 + new salt<br/>(Same tool results)"]
        Out2R["Output 2b<br/>(New code edits, tool calls)"]
        Spec2R["Spec 2b<br/>(Same settings, regeneration count 1)"]
        T2R["Timestamp t_2b"]
        H2R["Step Hash: h_2b"]
        In2R & Out2R & Spec2R & T2R --> H2R
    end

    H1 -->|"parent"| H2
    H1 -->|"parent"| H2R

    subgraph StepN["Step N (Session End)"]
        direction TB
        HN["Final Step Hash: h_n"]
        Closed["'closed' marker"]
        Overlap["Overlap score"]
        Payload["Receipt Payload<br/>(h_n + closed + overlap)"]
        HN & Closed & Overlap --> Payload
    end

    H2R -.->|"later steps"| HN

    Key[("Provider Private Key")] --> Sign["Provider signs payload"]
    Payload --> Sign
    Sign --> Receipt["Provider-Signed Receipt"]
```

### Signature

The provider signs the chain head together with a `closed` marker when the session ends. Keys are published as a JWKS at a well-known URL, in the same way as the IETF Web Bot Auth draft. A verifier checks the signature offline. No call to Anthropic is needed. Anthropic keeps the chain heads for disputes and key revocation. An optional append to a public transparency log stops anyone from backdating a receipt later.

### Selective disclosure

The receipt holds only hashes. The author reveals the full transcript only when a verifier asks. The verifier checks each revealed message against its hash. Private prompts stay private until the author decides.

## Overlap score

A valid chain does not prove authorship. A person can write a PR by hand, give it to Fable, and say "output this verbatim." The chain is honest, and the quality signal is gone.

To close this gap, the provider computes an **overlap score** at the end of the session and puts it in the signed spec. The score is the fraction of output tokens that also appear in the session inputs, measured on the final output only. A PR with overlap near 0% was composed by the model. A PR with overlap near 100% was copied. The score is cheap to compute, needs no judgment, and the provider already holds both sides.

A verifier who sees a high score can ask for the transcript. The echo prompt is then visible.

```mermaid
flowchart LR
    Inputs["Session Inputs<br/>(Prompts, files, tool outputs)"] --> Compare["Token Overlap Calculation"]
    Outputs["Final Output Code"] --> Compare
    Compare --> Score["Overlap Score<br/>(Shared tokens / Total output tokens)"]

    Score -->|Near 0%| LowScore["Low overlap<br/>Consistent with model-composed work<br/>(not proof, a rewritten echo also scores low)"]
    Score -->|Near 100%| HighScore["High overlap<br/>Cue to ask for the transcript<br/>(not proof, an honest edit of an existing file also scores high)"]
```

## Binding the receipt to code

The model emits edits, not a final file. Two ways to bind the receipt to a PR:

1. **Replay.** The base commit hash goes into the first input. The verifier replays the tool calls from the chain over that base commit and compares the result to the PR tree. If a person changed one line after the session, the replay does not match. This works with a local harness, because the harness cannot change what the provider signed.
2. **Hosted execution.** When the provider runs the sandbox, it signs the final tree hash directly. No replay is needed.

```mermaid
flowchart TD
    subgraph Execution["1. Session Execution"]
        direction TB
        Hosted["Hosted Cloud Sandbox<br/>(Provider runs environment)"]
        Local["Local Harness (Claude Code)<br/>(Base commit in input_1)"]
        Hosted -->|"Provider signs tree hash directly"| ReceiptHosted["Receipt with Tree Hash"]
        Local -->|"Provider signs tool calls in chain"| ReceiptLocal["Receipt with Step Chain<br/>(hashes only)"]
        Local -->|"Author discloses on request"| Disclosed["Disclosed Tool Calls<br/>(from the session transcript)"]
    end

    subgraph Verification["2. Offline Verification (GitHub Check / CI)"]
        direction TB
        JWKS["Provider JWKS<br/>(Public Key Directory, cached)"] --> CheckSig{"Verify Signature"}
        ReceiptHosted --> CheckSig
        ReceiptLocal --> CheckSig

        CheckSig -->|Invalid| FailSig["Fail: Invalid Signature"]
        CheckSig -->|Valid| CheckClosed{"'closed' marker present?"}
        CheckClosed -->|No| FailOpen["Fail: Session not closed<br/>(chain may be truncated)"]
        CheckClosed -->|Yes| MatchCheck{"Verify Code Integrity"}

        MatchCheck -->|"Hosted Mode"| CompareHosted{"Tree Hash == PR Tree?"}
        MatchCheck -->|"Local Mode"| CheckCalls{"Disclosed tool calls<br/>match chain hashes?"}
        Disclosed --> CheckCalls
        CheckCalls -->|No| FailCalls["Fail: Tool calls do not match the chain"]
        CheckCalls -->|Yes| Replay["Replay tool calls over Base Commit"]
        Replay --> CompareLocal{"Replayed Tree == PR Tree?"}

        CompareHosted -->|Match| Pass["Verified PR Check<br/>• Model and effort confirmed<br/>• Session closed<br/>• PR tree equals session result<br/>• Overlap score shown"]
        CompareLocal -->|Match| Pass

        CompareHosted -->|Mismatch| FailEdit["Fail: PR tree differs from session result<br/>(hand edit, rebase, or format hook)"]
        CompareLocal -->|Mismatch| FailEdit
    end
```

## Fit with Claude Code and Managed Agents

- **Managed Agents and cloud sessions.** This is the easiest place to start. The provider controls the model, the sandbox, and the clock. One signed receipt per session, with the final tree hash, covers the full chain. Ship this first.
- **Claude Code, local.** The API returns a signed step hash with each response. Claude Code appends the chain to the session log and writes the final receipt next to the commit, for example as a git note or a `Receipt:` trailer that points to a receipt file. Replay verification covers the local harness.
- **GitHub.** A GitHub App or Action verifies the receipt and shows a check: model, effort, number of turns, overlap score, replay match. Reviewers and merge rules can act on it. A downstream agent that reads the PR can set its own trust from the same check.
- **API.** Expose one field per response, `receipt_hash`, and one endpoint, `GET /receipts/{session_id}`, that returns the signed head. Everything else is client-side.

## What the receipt does not prove

- It does not prove the model authored the ideas. The overlap score and transcript disclosure help, and are not a full fix.
- It does not stop a person from running many sessions and keeping the best one. This is the same as a human who writes ten drafts. Accept it.
- It trusts Anthropic's key. A compromised key makes receipts from that key doubtful. Key rotation and a transparency log limit the damage. TEE attestation can reduce this trust later. The IETF SPICE inference-chain draft already combines the two.

## Next steps

1. Publish a receipt format. Reuse the in-toto statement shape with Anthropic as the "builder" and the session as the "build."
2. Add `receipt_hash` to API responses and a signed head at session close. Start with Managed Agents.
3. Add receipt writing and replay verification to Claude Code.
4. Publish a reference GitHub check.
5. Propose the format to C2PA and the IETF SPICE group so other providers can issue the same receipt.

## Prior art

- IETF draft, SPICE inference chain (March 2026): ordered inference logs with a Merkle root in a token.
- IETF draft, HTTP Message Signatures for Bots (September 2026): operator keys at a well-known URL.
- C2PA 2.4 `c2pa.ai-disclosure` assertion: model and oversight fields for files.
- in-toto and SLSA provenance: statement format for signed build claims.
- Anthropic text watermark (August 2026): detect-only layer that this proposal complements.

---

License: [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/)
