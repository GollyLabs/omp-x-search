# Design notes and findings

This file records why `x_search` is built the way it is, what live testing showed, and what to watch when OMP or xAI change. Research and testing happened on 2026-09-28 and 2026-09-29 against OMP 18.4.2.

Evidence labels:

- **Code**: read in source pinned to a commit.
- **Docs**: stated in official documentation.
- **Live**: observed in a real request.

## Why an OMP custom tool

OMP offers three places this could live.

| Option | Verdict | Reason |
|---|---|---|
| Custom tool | Chosen | Documented ([custom-tools.md][omp-doc-ct]). One file. Gets `ctx.modelRegistry` for OMP's stored credentials. Works under any coding model. |
| Extension with `registerTool` | Not needed | Also documented ([extensions.md][omp-doc-ext]), but its lifecycle and event machinery add nothing for one tool. |
| `web_search` provider | Not possible | The provider list is fixed inside OMP ([provider.ts][omp-search-registry]). It is chosen by the `web` model role rather than the caller, and `web_search` has no handle or date filters. |

Custom tools are discoverable by default. The model sees a one-line entry for `xd://x_search`, reads it for the schema, and calls it by writing JSON. OMP validates the arguments against the schema first and passes cancellation through ([xdev.ts][omp-xdev]). **Code**

Two existing projects were reviewed and not adopted. Both are MIT licensed, and this tool copies no code from either.

- [omp-extended-search][ext-xsearch] at `e7c4952`: its schema omits `query`, its timeout stops before the response body is read, it retries requests that may already have been billed, it falls back from the OAuth login to an API key, and its optional Firecrawl capture accepts non-X URLs. **Code**
- [pi-xai-oauth][pi-routing] v1.6.0: a full provider stack for Pi. Its tools only run when the session model is Grok ([model-scope.ts][pi-scope]), and its X search returns plain text without structured citations. **Code**

OMP's own xAI `web_search` provider ([xai.ts][omp-xai-provider]) was the model for credential handling and citation parsing. It uses `type: "web_search"`, not `x_search`.

## Authentication and endpoint

| Claim | Evidence |
|---|---|
| xAI documents `api.x.ai` for API keys. | Docs ([inference auth][xai-inference]) |
| xAI's Grok Build CLI signs in at `auth.x.ai` and sends inference through `cli-chat-proxy.grok.com`. `api.x.ai` is "only needed when using `api_key` auth". | Docs ([Grok Build hosts][xai-build-ent]) |
| OMP's `xai-oauth` provider sends the SuperGrok login to `https://api.x.ai/v1` for chat and web search. | Code ([xai-oauth.kdl][omp-kdl]) |
| OMP lets `xai-oauth` fall back to `XAI_API_KEY`. | Code ([xai-oauth.kdl][omp-kdl], [env docs][omp-env-doc]) |
| pi-xai-oauth copies Grok Build's route and headers, and calls that a compatibility audit, not a public API. | Code ([routing.ts][pi-routing], [compat doc][pi-compat]) |
| A SuperGrok login is authorized for `x_search` at `api.x.ai/v1/responses`. | Not documented. Live: it works. |

Decisions:

1. Use `https://api.x.ai/v1/responses`, the same host OMP already uses for this login. The endpoint is a constant, so a model base-URL override can never redirect the token.
2. Do not use `cli-chat-proxy.grok.com`. It belongs to Grok Build, and its header contract is reverse-engineered.
3. Use the OAuth login only. The tool requires `authStorage.keys.source("xai-oauth", { env: "none" }).kind === "oauth"` and refuses any token starting with `xai-`. That closes OMP's fallback to a paid API key.
4. Get tokens through `ctx.modelRegistry.resolver(model, sessionId)`. The tool never reads OMP's credential database.
5. Default to `grok-4.7`, xAI's primary `x_search` example. `OMP_X_SEARCH_MODEL` overrides it.

## Live findings

Eleven live calls were made, all with the SuperGrok login and `grok-4.7`. They included direct calls through the tool code, calls from Claude Haiku and Claude Opus sessions inside OMP, a call from a resumed session, and a call through the installed tool.

| Finding | Evidence |
|---|---|
| The SuperGrok login runs real X searches at `api.x.ai`. Every call that searched returned real posts. [Hermes #88040][hermes-88040] reports citation-free answers with OAuth, but that did not happen here. | Live |
| Grok's internal tools are `x_keyword_search`, `x_semantic_search`, `x_user_search`, and `x_thread_fetch`. A system prompt that said "use the x_search tool" once made Grok answer without searching. The prompt now names the real tools. | Live |
| A bare query such as `Grok` was once read as a message to the model, and Grok searched for "user request posts on X". Queries are now sent as `Find X posts for this search request: <query>`. | Live |
| xAI's `url_citation` annotations list every post the search touched, not only the posts the answer uses. One call had 33 sources for 5 answered posts. The tool therefore lists the posts the answer names and checks them against the sources. | Live |
| Grok sometimes writes a post URL that is not among xAI's sources. The tool flags those as unverified. | Live |
| xAI rejects a bad token with HTTP 400 `Incorrect API key provided`, not 401. | Live |
| `max_turns: 2` with `parallel_tool_calls: false` does not cap searches. Calls ran 1 to 7 searches. | Live |
| The REST API rejects `include` values `x_search_call_output`, `x_search_call.results`, and `x_search_call.action.sources` with HTTP 400. There is no way to get structured post objects. | Live |
| Every response carries `usage.cost_in_usd_ticks` (1 USD = 10^10 ticks) and `server_side_tool_usage_details`, even with OAuth. | Live, Docs ([cost tracking][xai-cost]) |
| After about $0.85 of API-equivalent searches, the SuperGrok weekly meter still read 0% used. Whether and where these searches are charged is unknown. | Live |
| Latency was 15 to 58 seconds per call. | Live |

xAI pricing for API keys, for reference: $5 per 1,000 posts fetched and $10 per 1,000 profiles fetched, plus tokens. Thread fetches count every post in the thread, and repeated posts count again. **Docs** ([pricing][xai-pricing], [X Search][xai-xsearch])

## OMP behavior

- OMP loads custom tools only when a session is created ([sdk.ts][omp-sdk-load]). `/reload-plugins` does not load, update, or remove them. **Live, Code**
- Resuming an older conversation inside a newly started session keeps the new process's tools. **Live**
- Subagents reuse their parent session's tool scan. **Code**
- Plugins can declare tools in `package.json` under `omp.tools` ([plugin types][omp-plugin-types]). `omp plugin install <path>` links a local checkout, and `omp plugin install github:GollyLabs/omp-x-search` installs the published repo. The tool loads from both, and `omp plugin uninstall omp-x-search` removes it. **Live**
- OMP's argument validation keeps unknown top-level fields ([validation.ts][omp-validation]), so a schema that omits a field does not always drop it at dispatch. **Code**
- `ctx.fetch` and `ctx.settings` are declared on the custom-tool context but are not passed through ([sdk.ts][omp-sdk-bridge]). The tool uses the global `fetch`. **Code**
- A thrown error becomes an error result that the model sees. **Live**

## Reliability decisions

- **No retries after a request may have been accepted.** xAI documents no idempotency key for `POST /v1/responses`, so a retry after a timeout, network error, 429, or 5xx could be billed twice. The single exception is a token rejection (401, or 400 "Incorrect API key"), which happens before any work.
- **One deadline for everything.** A 120-second timeout joins the caller's cancel signal. It covers the token lookup, the request, and the body read. OMP ignores the signal on the first token lookup ([api-key-resolver.ts][omp-resolver]), so the tool races that wait against the signal itself. A cancelled body read ends like a finished one, so the tool checks the signal before using the text. The timeout message appears only when the tool's own deadline fired.
- **Bounded output.** Responses over 2 MiB are refused. Grok's answer is capped at 12,000 characters, and post URLs are only extracted from the part that is shown. At most `limit` posts and `limit` unverified URLs are listed.
- **Honest post lists.** When the answer names posts but none appear in xAI's sources, the tool says so and lists nothing else. It falls back to the first sources only when the answer names no posts at all.
- **Credentials stay out of errors.** HTTP error bodies and network error messages are scrubbed of the token and anything shaped like a credential.
- **No answer without a search.** If `x_search_calls` is 0 or missing, the answer is withheld.
- **Untrusted content.** The answer is wrapped in a delimited block, with terminal escapes and control characters removed.

## Not tested

- A login that expires in the middle of a session, and the refresh-and-replay against a real expired token.
- Long-term rate limits with the OAuth login.
- Logins other than SuperGrok, such as X Premium+.

## Maintenance

After each OMP release, run `bun test` and a live headless check (see the README). Diff these OMP files against the last tested tag:

- `packages/coding-agent/src/extensibility/custom-tools/{types,loader}.ts`
- `packages/coding-agent/src/tools/xdev.ts`
- `packages/coding-agent/src/sdk.ts` (custom-tool bridge)
- `packages/ai/src/auth-retry.ts` and `packages/coding-agent/src/config/api-key-resolver.ts`
- `packages/ai/src/auth/cascade.ts` (`keys.source`)
- `packages/catalog/src/compat/rules/providers/xai-oauth.kdl`
- `packages/coding-agent/src/web/search/providers/xai.ts`

Also watch these:

- xAI's [X Search][xai-xsearch], [pricing][xai-pricing], [citations][xai-citations], and [Responses reference][xai-responses] pages, plus the [release notes](https://docs.x.ai/developers/release-notes).
- [OMP #3341][omp-3341], the open request for native xAI X search. If OMP ships it, compare the two and consider retiring this tool.
- [Hermes #88040][hermes-88040], the report of degraded OAuth X search.

When you bump the targeted OMP version, update the `@oh-my-pi/omptype` pin in `package.json` to match.

## Sources

OMP v18.4.2 (`4620bb8338e0ecace7ea237da9d5088d16068617`):

- [Custom tools docs][omp-doc-ct], [extensions docs][omp-doc-ext], [xd:// devices][omp-xdev], [validation][omp-validation]
- [Custom-tool loading][omp-sdk-load], [custom-tool bridge][omp-sdk-bridge], [plugin manifest types][omp-plugin-types]
- [xAI web_search provider][omp-xai-provider], [xai-oauth catalog][omp-kdl], [environment variables][omp-env-doc], [search provider registry][omp-search-registry]
- [Issue #3341][omp-3341]

xAI documentation, retrieved 2026-09-28:

- [X Search][xai-xsearch], [pricing][xai-pricing], [citations][xai-citations], [Responses reference][xai-responses], [cost tracking][xai-cost], [inference auth][xai-inference], [Grok Build hosts][xai-build-ent]

Other projects:

- [omp-extended-search @ e7c4952][ext-xsearch]
- [pi-xai-oauth v1.6.0 routing][pi-routing], [compatibility audit][pi-compat], [model scope][pi-scope]
- [Hermes #88040][hermes-88040]

[omp-doc-ct]: https://github.com/can1357/oh-my-pi/blob/4620bb8338e0ecace7ea237da9d5088d16068617/docs/custom-tools.md
[omp-doc-ext]: https://github.com/can1357/oh-my-pi/blob/4620bb8338e0ecace7ea237da9d5088d16068617/docs/extensions.md
[omp-xdev]: https://github.com/can1357/oh-my-pi/blob/4620bb8338e0ecace7ea237da9d5088d16068617/packages/coding-agent/src/tools/xdev.ts
[omp-validation]: https://github.com/can1357/oh-my-pi/blob/4620bb8338e0ecace7ea237da9d5088d16068617/packages/ai/src/utils/validation.ts#L1741-L1793
[omp-sdk-load]: https://github.com/can1357/oh-my-pi/blob/4620bb8338e0ecace7ea237da9d5088d16068617/packages/coding-agent/src/sdk.ts#L2491
[omp-sdk-bridge]: https://github.com/can1357/oh-my-pi/blob/4620bb8338e0ecace7ea237da9d5088d16068617/packages/coding-agent/src/sdk.ts#L1260-L1359
[omp-plugin-types]: https://github.com/can1357/oh-my-pi/blob/4620bb8338e0ecace7ea237da9d5088d16068617/packages/coding-agent/src/extensibility/plugins/types.ts#L27-L45
[omp-xai-provider]: https://github.com/can1357/oh-my-pi/blob/4620bb8338e0ecace7ea237da9d5088d16068617/packages/coding-agent/src/web/search/providers/xai.ts
[omp-kdl]: https://github.com/can1357/oh-my-pi/blob/4620bb8338e0ecace7ea237da9d5088d16068617/packages/catalog/src/compat/rules/providers/xai-oauth.kdl
[omp-env-doc]: https://github.com/can1357/oh-my-pi/blob/4620bb8338e0ecace7ea237da9d5088d16068617/docs/environment-variables.md#L69-L70
[omp-search-registry]: https://github.com/can1357/oh-my-pi/blob/4620bb8338e0ecace7ea237da9d5088d16068617/packages/coding-agent/src/web/search/provider.ts
[omp-3341]: https://github.com/can1357/oh-my-pi/issues/3341
[xai-xsearch]: https://docs.x.ai/developers/tools/x-search
[xai-pricing]: https://docs.x.ai/developers/pricing
[xai-citations]: https://docs.x.ai/developers/tools/citations
[xai-responses]: https://docs.x.ai/developers/rest-api-reference/inference/responses
[xai-cost]: https://docs.x.ai/developers/cost-tracking
[xai-inference]: https://docs.x.ai/developers/rest-api-reference/inference
[xai-build-ent]: https://docs.x.ai/build/enterprise
[ext-xsearch]: https://github.com/bnivanov/omp-extended-search/blob/e7c49526aac481cfac33b5626aa4c8ce9f20065c/tools/x_search.ts
[pi-routing]: https://github.com/BlockedPath/pi-xai-oauth/blob/f2408b0dc108103e0568e6ab02ee0b30644254bb/extensions/xai/routing.ts#L31-L48
[pi-compat]: https://github.com/BlockedPath/pi-xai-oauth/blob/f2408b0dc108103e0568e6ab02ee0b30644254bb/compatibility/grok-build-wire-protocol.md
[pi-scope]: https://github.com/BlockedPath/pi-xai-oauth/blob/f2408b0dc108103e0568e6ab02ee0b30644254bb/extensions/xai/tools/model-scope.ts#L44-L79
[hermes-88040]: https://github.com/NousResearch/hermes-agent/issues/88040
[omp-resolver]: https://github.com/can1357/oh-my-pi/blob/4620bb8338e0ecace7ea237da9d5088d16068617/packages/coding-agent/src/config/api-key-resolver.ts#L52-L98
