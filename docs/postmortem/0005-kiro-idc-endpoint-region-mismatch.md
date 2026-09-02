# Post-mortem 0005: `dsh-kiro` sent IdC-authenticated requests to a region and hostname AWS never deployed

English | [中文](0005-kiro-idc-endpoint-region-mismatch.zh.md)

Status: resolved (fix applied to the community `dsh-kiro` plugin; upstream issue filed)

## Executive summary

The community `dsh-kiro` provider chose its model-generation hostname by `authMethod` (`codewhisperer.<region>` for IAM Identity Center, `q.<region>` otherwise) and filled `<region>` from the sign-in token's region. For an IdC deployment whose token region was `ap-southeast-1`, that produced `codewhisperer.ap-southeast-1.amazonaws.com` — a hostname with no public DNS record in any resolver, because AWS never deployed CodeWhisperer's streaming API there. The token's region is the IAM Identity Center instance's region, not the region the model API is deployed to; every `authMethod` must call the one hostname AWS actually ships (`q.us-east-1.amazonaws.com`), confirmed against the vendor's own CLI and an independent open-source reverse-engineering of the same API. The fix hardcodes `us-east-1` and drops the hostname branch by `authMethod`.

## Impact

Every chat turn against the Kiro provider failed. The user-facing symptom was `Invalid model. Please select a different model to continue.` (`INVALID_MODEL_ID`), which reads as a model-catalog problem and sent debugging in exactly the wrong direction: the response comes from a real AWS endpoint that authenticates the request but rejects the model id, because that endpoint has never heard of any Kiro model id — it is the wrong service. The account's own CLI (`kiro-cli`) worked throughout, which is what eventually falsified the "account misconfigured" theory.

## Timeline

- User reports `dsh-kiro`'s usage-refresh button and chat turns both failing after an unrelated harness upgrade/rollback cycle.
- First hypothesis: the harness upgrade changed something. Ruled out — the failure reproduced on the pre-upgrade harness commit too.
- Second hypothesis: the account's AWS IAM Identity Center region (`ap-southeast-1`, read from the stored sign-in token) needs to be threaded through as the plugin's `region` config. Applied; usage-refresh started working (that endpoint tolerates either region), but chat turns still failed, now with `codewhisperer.ap-southeast-1.amazonaws.com` unreachable — `getent hosts` returned no record, confirmed independently against Google's public resolver (not a local DNS filter).
- Third hypothesis (this one held): the network failure was a symptom, not the cause. `kiro-cli` — the vendor's own client, using the same signed-in account — worked the entire time. `strace -f -e trace=network` on a live `kiro-cli` chat call showed every DNS query and connection went to `*.us-east-1.amazonaws.com`, never `ap-southeast-1`. Extracting readable strings from the `kiro-cli` binary surfaced its baked-in endpoint table — `q.eu-central-1`, `q.us-gov-east-1`, `q.us-gov-west-1`, `codewhisperer.us-east-1` — with no `ap-southeast-1` entry at all.
- Cross-checked against an independently maintained open-source Kiro-to-OpenAI/Anthropic proxy (unrelated codebase, same reverse-engineered wire protocol): its provider hardcodes `KIRO_API_URL = "https://q.us-east-1.amazonaws.com/generateAssistantResponse"` — no per-account region, no `authMethod` branch.
- Isolated the two remaining variables with direct `curl` calls holding everything else constant: `q.us-east-1` returned `200` with a real streamed answer; `codewhisperer.us-east-1` — same account, same bearer token, same request body — returned `400 INVALID_MODEL_ID`, the exact error the user had reported. That reproduced the original symptom from a hostname choice alone.

## Root cause

`dsh-kiro`'s `kiroRequestEndpoint(token, region)` picked the model-generation hostname from `token.authMethod`:

```ts ignore-check
function kiroRequestEndpoint(token: KiroToken, region: string): string {
  return token.authMethod === 'idc' || token.authMethod === 'external_idp'
    ? `https://codewhisperer.${region}.amazonaws.com/generateAssistantResponse`
    : `https://q.${region}.amazonaws.com/generateAssistantResponse`
}
```

`region` came from `connection.region ?? token.region`, and `token.region` is read straight from the persisted sign-in token — the region of the caller's IAM Identity Center instance, chosen when the organization set up SSO. That value has no connection to where AWS deployed the CodeWhisperer/Q Developer streaming API: AWS ships that service in a small, fixed set of regions (`us-east-1`, plus a few gov/China partitions per the vendor CLI's own endpoint table), completely independent of where any customer's Identity Center instance lives. Two more choices compounded the mismatch:

- **Wrong hostname family for IdC.** Even pinned to `us-east-1`, `codewhisperer.us-east-1.amazonaws.com` is a distinct, real, authenticating endpoint from `q.us-east-1.amazonaws.com` — and it rejects every Kiro model id with `400 INVALID_MODEL_ID`, because IdC accounts are served by the `q.` endpoint regardless of `authMethod`.
- **A per-account IAM Identity Center profile ARN, once discovered, silently overrides any configured region.** `dsh-kiro`'s profile-auto-discovery flow (triggered whenever the token has no `profileArn` yet) calls `ListAvailableProfiles`, and when more than one profile matches, prefers the one whose ARN region equals `token.region` over the first (or configured-region) result. Once that profile is cached, `profileRegion(profileArn)` — not the `region` config — decides every later request's hostname. A configuration fix that only sets `region` in `cordis.yml` therefore does not survive profile discovery; it must also account for what the discovered profile ARN carries.

## Guardrails

- **Never trust a credential's own metadata for a provider's fixed service topology.** A sign-in token's `region` describes where *authentication* is anchored (the Identity Center instance, an OAuth issuer, an SSO tenant) — not where the *product API* the token authorizes is deployed. Read the vendor's own client for the real endpoint table before wiring an account-derived value into a request URL; multi-region services are the exception a provider states explicitly, not the default to assume.
- **Reverse-engineered protocols need two independent confirmations before trusting the derived contract.** The vendor's own CLI (traced with `strace -f -e trace=network`, then binary `strings` for its baked-in endpoint table) and an unrelated open-source client targeting the same wire protocol agreed on one hardcoded endpoint with no region parameter at all — that agreement, not either source alone, is what justified hardcoding `us-east-1` in the fix.
- **Isolate one variable per request when a symptom could have more than one cause.** `curl` with identical headers, body, and bearer token, varying only the hostname, reproduced the exact reported error from the hostname alone and ruled out account/model/content-type theories in one step — cheaper and more conclusive than adding another config knob and re-running the full harness.
- A HTTP 400 from a real, authenticating endpoint is not evidence the request reached the right *service* — only that it reached *a* service that understood enough of the request to reject a field. Confirming DNS existence (a public resolver, not just the local one) is a prerequisite check before trusting either a successful or a failing response as diagnostic signal.
