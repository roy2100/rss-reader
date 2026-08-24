# Plan: move title translation off DeepSeek onto a local Ollama runtime

## Goal

Run title translation entirely on this Mac. Install Ollama, serve **Hunyuan-MT-7B (Q4_K_M)**
through its OpenAI-compatible endpoint, and repoint `llm_config` at
`http://localhost:11434/v1`. DeepSeek's price rise is the trigger; the local runtime makes the
per-title cost zero and takes the feature off the network entirely.

Nothing in `internal/translate` should need a provider branch: the package already speaks plain
`POST {base_url}/chat/completions` with a bearer token, and `ValidateBaseURL` already allows a
loopback `http://` base specifically for Ollama/vLLM. That is the hypothesis this plan tests.

## Scope

In:
- Homebrew install of `ollama`, run as a background service (`brew services`), bound to loopback.
- Pull `hf.co/Mungert/Hunyuan-MT-7B-GGUF:Q4_K_M` (~4.7 GB) and confirm it loads on 16 GB.
- Measure it against the **real** request the worker sends (system prompt + two demonstration
  turns + a 来源/摘要/标题 user message), not a hand-written toy prompt.
- Repoint `llm_config` via `PATCH /api/llm/config` on the loopback API (127.0.0.1:4002) and verify
  with `POST /api/llm/config/test`.
- Whatever prompt-path change the measurement turns out to require (see Risks).

Out:
- Any provider abstraction in `internal/translate`. Still one client, one endpoint shape.
- Per-feed model selection. Still one global config row.
- Keeping DeepSeek as a fallback. One endpoint at a time; the config row is a single row and
  switching back is a `PATCH`.
- Translating anything beyond the existing 24h pending window. Switching endpoints does not
  backfill history, by design.

## Steps

1. `brew install ollama`, `brew services start ollama`; confirm `127.0.0.1:11434` answers and that
   the listener is loopback-only (it must not become a second public surface next to the tunnel).
2. `ollama pull hf.co/Mungert/Hunyuan-MT-7B-GGUF:Q4_K_M`; `ollama list` to confirm size on disk.
3. Smoke-test the OpenAI shim: `POST /v1/chat/completions` with a bare title, `stream:false`.
4. Replay the **exact** wire format `client.go` builds — system + `exampleTurns` + one
   `userMessage` — and read the answer through the same lens `clean()` applies. Record latency and
   what the raw content looks like before cleaning.
5. Decide from step 4 whether the existing prompt path is usable as-is (see Risks).
6. `PATCH /api/llm/config` → `base_url=http://localhost:11434/v1`, `model=<tag>`, `api_key=<any
   non-empty>` (Ollama ignores it, but `Config.Ready()` requires it), `enabled=true`.
7. `POST /api/llm/config/test`, then watch `translate usage` lines in the app log for real rows.

## Risks / open questions

- **Model/prompt mismatch — the main risk.** Hunyuan-MT-7B is a *dedicated translation* model,
  trained on a single fixed instruction (`把下面的文本翻译成<target_language>，不要额外解释。\n\n
  <source_text>`), with no system prompt and no few-shot turns. This app's prompt is the opposite:
  a persona ("科技资讯编辑"), six register rules, five arrow examples, two demonstration turns, and
  a three-line labelled user message whose 来源/摘要 lines are explicitly *not* to be translated.
  The likely failure is that the model translates the whole form — the exact
  `来源：… 标题：…` echo that `stripEchoedLabels` was written to bound — or that it renders a
  faithful but literal headline, losing the 翻译腔-avoidance the prompt exists to buy.
  Step 4 measures this rather than assuming it. If it fails, the options in preference order are:
  (a) keep the prompt, accept a more literal register; (b) send Hunyuan's native template for this
  model only — a per-config prompt shape, which is the provider branch this codebase has so far
  refused; (c) run a general instruct model (Qwen3-8B et al.) that follows the existing prompt, and
  keep Hunyuan-MT out of the loop.
- **Memory.** 16 GB shared, with the app, Xcode-less but browser-heavy desktop. Q4_K_M ≈ 4.7 GB
  resident while loaded. `OLLAMA_KEEP_ALIVE` decides whether that stays resident between the 30s
  worker ticks; keeping it loaded costs RAM permanently, unloading costs a multi-second reload on
  every tick that has work.
- **Latency.** The worker's client timeout is 30s (`translate.New`). A cold load plus generation on
  an M2 could approach that on the first call after an unload.
- **Newer model exists.** Tencent released HY-MT1.5-7B (2025-12-30), available as
  `demonbyron/HY-MT1.5-7B:Q4_K_M` on the Ollama registry. Not what was asked for; noted here so the
  choice is deliberate.

## Complexity

Medium — the install and the config swap are trivial; the prompt/model fit is the real work.

## Outcome

Done. Translation now runs entirely on this Mac; `internal/translate` was not touched.

**Deployed:** Homebrew `ollama` 0.32.15, `brew services start ollama`, listening on
`127.0.0.1:11434` only — verified with `lsof`, so it adds no public surface next to the tunnel.
`hf.co/Mungert/Hunyuan-MT-7B-GGUF:Q4_K_M` pulled (4.7 GB) as asked, plus `qwen3:8b` (5.2 GB).

**Live config:** `base_url=http://localhost:11434/v1`, `model=feedoverflow-translate`,
`api_key=ollama-local`, `enabled=true`. `POST /api/llm/config/test` passes; one row forced back to
NULL was re-translated by the worker within 15s (`ESPN streaming plans are getting more expensive`
→ `ESPN 流媒体订阅服务价格上涨`).

### Deviation: Hunyuan-MT-7B is deployed but not in use

Step 4's measurement — the real wire format against 10 real titles from the DB — killed option (a)
and (b) both, and the ask was rerouted to option (c) after the numbers were on the table.

| Configuration | Correct | Failure |
|---|---|---|
| Hunyuan-MT-7B, app prompt | **1/10** | translated the 摘要, not the 标题 (6/10); stored `新闻来源：Reuters` as a headline (1/10) |
| Hunyuan-MT-7B, native template | 10/10 literal | sentence register, 句号, inline glosses, `——路透社` kept; no place for the summary |
| `qwen3:8b`, app prompt | **0/10** | every response empty |
| `feedoverflow-translate`, app prompt | **10/10** | one name error (`Total CEO` → `总装CEO`) |

Two mechanisms, both now recorded in CLAUDE.md:

1. **Hunyuan-MT has no instruction layer.** Its chat template is `{{.System}}…{{.Prompt}}` with no
   multi-turn support, so `exampleTurns` is dropped before the model ever sees it, and nothing acts
   on 「只有「标题：」后面那一行需要翻译」. It translates the text in front of it, and the summary is
   most of that text. `maxGrowth` cannot catch this: a translated summary is well inside 4x. The
   model is not defective — it is a translator, and this app wants an editor with context.
2. **A thinking model empties every row, silently.** `thinking:{"type":"disabled"}` is DeepSeek's
   field; Ollama ignores it *without* a 400, so client.go's 400-retry never fires. Thinking then
   runs into `maxCompletionTokens = 200`, truncates mid-thought, and returns empty content — which
   `translatePending` settles as `''`, "no translation, do not retry". Permanent data loss, no error
   anywhere. Note also that Ollama's field is `reasoning`, not DeepSeek's `reasoning_content`, so
   `logUsage`'s `reasoningRunes` reads 0 and the log would not have shown it either.

### The fix lives in the model, not the request

`scripts/ollama/Modelfile.translate` derives `feedoverflow-translate` from `qwen3:8b`, patching the
stock template so the last user turn always carries `/no_think` and the assistant turn is always
prefilled with an empty `<think></think>` block. Sampling params (Qwen's non-thinking recommendation)
sit there too, which keeps the app's "send no sampling params" rule intact.

Ollama's `reasoning_effort:"none"` also works and was rejected: it would put a second
provider-specific field in `chatRequest`, and the model name is already a config value, so the fix
costs nothing to carry there.

**Reconciled against the plan's risks:** memory is fine (~5.2 GB resident, one model at a time);
latency is 1.2–2.0 s warm and ~20 s on a cold load, both inside the 30 s client timeout, so
`OLLAMA_KEEP_ALIVE` was left at its default rather than pinning 5.2 GB permanently on a 16 GB
machine. Throughput ceiling is `translateBatch=20` × ~1.5 s ≈ 30 s, exactly one tick — adequate at
the observed ~370 translations/day, but it is the number to watch if feeds are added in bulk.

**Loose ends for the operator:** the DeepSeek API key was overwritten by the `PATCH` and is not
recoverable from the DB — switching back means re-entering it in SettingsModal. Hunyuan-MT-7B is
still installed and unused; `ollama rm hf.co/Mungert/Hunyuan-MT-7B-GGUF:Q4_K_M` reclaims 4.7 GB.
A temporary in-package probe harness (`wire_probe_test.go`) produced the table above and was deleted;
`make check` passes.

### Follow-up: temperature 0.3 was tried and rejected

`feedoverflow-translate` keeps Qwen's recommended `temperature 0.7`. Lowering it to 0.3 was
proposed to fix two things and fixed neither — 10 titles, two runs per temperature:

- **Proper nouns got worse, not better.** `Total CEO Says` (correct: TotalEnergies / 道达尔) came
  back as `总持CEO` and `总装能源CEO` at 0.3 — the first is an invented word — while the only
  *correct* rendering across all four runs, `Total首席执行官称`, came from 0.7. The failure is not
  sampling: the summary says `TotalEnergies SE` in plain text and the model is not reading it, which
  no temperature reaches.
- **Run-to-run stability barely moved**: 2/10 identical lines at 0.3 vs 1/10 at 0.7. Each title is
  translated once and the result is permanent, so there is no variance to converge anyway — the same
  reasoning that keeps sampling params out of the request in the first place.

The experiment did surface one real regression against DeepSeek, at **both** temperatures and so not
a tuning matter: roughly 1 in 10 titles has a fact lifted out of the 摘要 and folded into the
headline. `US-Canada Tariff War Escalates After Collapse of Talks` became
`…贸易谈判破裂后美方加征 50% 关税`, where the 50% exists only in the summary. This is exactly what
`examplePairs`' second demonstration exists to prevent, and `clean()` cannot catch it — it checks
growth and echoed labels, not added facts. Left as observed behaviour rather than chased: the fix
would be prompt or model surgery, and the rate is worth watching for a while first.
