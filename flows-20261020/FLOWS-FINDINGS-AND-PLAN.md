# Flows: findings, plan and evaluation (2026-10-20, handover to the Flows owner)

**Status, 2026-10-20.**
- The owner has paused Flow AI Builder work. A colleague now owns Flows.
- The Builder may be **replaced**. So every rule below is placed in **Flows core**: publish, run, package import and the editor. Then the Builder, its replacement, or a person in the editor all get the same rule.
- This file is the Flows-only counterpart to `/home/ccimen/dev/eneo/demo-stack/specs/builder-arch-20261020/FINDINGS-AND-PLAN.md`. Builder planning, proposal and compile internals stay in that file.
- Code reference: integration tip **044ec559c** on `feature/flows-and-builder`, checked out read-only at `/tmp/flows-tip-read`. Path roots: a path with a folder but no root prefix (for example `runtime/executor.py`) is relative to `backend/src/eneo/flows/`. Test paths start with `backend/tests/`. A path that starts with `backend/`, `frontend/` or `docs/` is relative to the repository root (the tip checkout root). Sibling packages are written in full (`backend/src/eneo/flow_packages/...`, `backend/src/eneo/files/...`, `backend/src/eneo/server/...`).
- Evidence marks:
  - **V** means checked in `/tmp/flows-tip-read` while writing this file.
  - **S** means taken from a source document and not re-read at the tip.
  - **UNKNOWN** means not established.
  - Bead status comes from `/home/ccimen/dev/eneo/eneo/.beads/issues.jsonl`, read 2026-10-20.

## 0. Index

| What | Where (absolute) |
|---|---|
| This file | /home/ccimen/dev/eneo/demo-stack/specs/flows-20261020/FLOWS-FINDINGS-AND-PLAN.md |
| Validated Flows beads (102 beads, one verdict each); published in this folder with this plan | /home/ccimen/dev/eneo/demo-stack/specs/flows-20261020/FLOWS-BEADS-VALIDATED.md |
| Machine-readable form of the same bead validation | /home/ccimen/dev/eneo/demo-stack/specs/flows-20261020/flows-beads-validated.json |
| Builder findings and plan (Builder details, live meeting case, evaluation design) | /home/ccimen/dev/eneo/demo-stack/specs/builder-arch-20261020/FINDINGS-AND-PLAN.md |
| Architecture appendix (VAL-*, DR-* items cited below) | /home/ccimen/dev/eneo/demo-stack/specs/builder-arch-20261020/APPENDIX-A-architecture.md |
| Flows quality program (Waves A-E, delivery log) | /home/ccimen/dev/eneo/demo-stack/specs/flows-mcp-tools/quality-program-20261020.md |
| Validation ladder (T0 replay, live-run rules) | /home/ccimen/dev/eneo/demo-stack/specs/flows-mcp-tools/validation-ladder-20261020.md |
| Bugs found outside a lane's scope | /home/ccimen/dev/eneo/demo-stack/specs/found-outside-scope.md |
| Lane reservations, landing notes | /home/ccimen/dev/eneo/demo-stack/specs/sessions/log.md, /home/ccimen/dev/eneo/demo-stack/specs/sessions/coordination.md |
| Unfinished eaar.1 patch (relates to section 4) | /home/ccimen/dev/eneo/demo-stack/specs/builder-arch-20261020/eaar1-partial/ (patch, state.md, test_output_modes_content_part.py, lane-receipts/) |
| Hand-corrected meeting packages (positive controls) | /home/ccimen/dev/eneo/demo-stack/specs/builder-cases/motesanteckningar-20261020/fixed-package/ (2.0.0, 2.1.0, långa-möten 1.0.0, unpacked-langa-moten/). The run exports in the parent folder hold personal data: never use them as fixtures |
| Read-only code snapshot | /tmp/flows-tip-read (044ec559c) |
| Landing tools | ~/.claude/skills/eneo-slice-landing/scripts/ and /home/ccimen/dev/eneo/demo-stack/builder-guards/ |

## 1. Flows capabilities today

| Area | Capability | Status | Evidence |
|---|---|---|---|
| Step input sources | flow_input, previous_step, all_previous_steps, http_get, tool | works. The authoring subset has no http_get | V enums.py:20-25, :28-40 |
| Input types | text, json, image, audio, document, file, any | partial: image input gets a typed refusal | V enums.py:49-58; S type_policies.py:36-43 |
| Output types | text, json, pdf, docx | works | V enums.py:62-66 |
| Output modes | pass_through, compose_text, copy_input, http_post, transcribe_only, template_fill, render_verbatim, speaker_mapping | works. http_post cannot be authored by spec, package or Builder, because it needs credentials | V enums.py:69-83, :170-181; runtime/step_handlers/ |
| Run form and run input | form fields, upload (runtime_input), single_call or per_source | works. A step that reads run free text (`indata_text`, `flow_input.text`) on a flow whose run form declares fields or uploads is refused at publish (flow_input_alias_not_received). It is a publish rule, not a run refusal | V domain/flow.py:52, :153-161; flow_validators.py:2398-2470 (strict-only call :623-629); flow_variable_definitions.py:120-168 |
| Section processing | text_processing: process_each_section, summarize | works. Refuses a structured source (section 4) | V domain/text_processing.py:30-45; flow_validators.py:2469-2543; runtime/step_input_resolution.py:1444-1455 |
| Item map | item_map with a required max_items ceiling | works | V domain/step_item_map.py:10-40; runtime/step_handlers/per_item_map.py |
| Review checkpoints | view and edit, expiry, edits checked against the contract; speaker_mapping requires edit | partial: the edit check runs in-process on the API loop (eneo-8han) | V application/flow_run_review_checkpoint_service.py:361-365; flow_review_policy.py:80-82, :107-111 |
| Transcription | transcribe_only, speaker_mapping, corrections with speaker names, regeneration, live socket, remote service | works. Outbound policy gaps: eneo-plsj (service endpoint), the live relay (not filed) | V runtime/step_handlers/speaker_mapping.py; S api/flow_live_transcription_socket_router.py:190-201 (found-outside-scope.md:61) |
| DOCX templates | template_fill, template assets, readiness in the run contract, package template uploads, three standard documents | partial: the `output_config.bindings` shape and the `step_input.*` syntax are not documented (S found-outside-scope 2026-09-30) | runtime/template_fill_runtime.py; runtime/templates/standard/ |
| Whole-flow templates | none in the backend | missing | V: flows/ has no flow template catalog |
| Tools / MCP | one tool per step through the assistants' chain, publish pinning, argument check in the killable child | partial: eneo-v7xp, epim.21, epim.25, epim.15 | V runtime/tool_argument_check.py; application/flow_tool_publication.py |
| HTTP | http_get input (egress policy, SSRF block, secrets codec), http_post output | partial: a 3xx answer counts as success (section 3, F9) | V runtime/egress/http.py:237; runtime/http_orchestration.py:257, :428 |
| Webhooks | outbox, claim waves, backoff, dead letter, one job hard stop | works after A2. Follow-ups: eneo-7cet, -5gen, -olj8, -cb1j | V application/flow_webhook_delivery_policy.py:55-77; runtime/tasks.py:918-963 |
| Evidence and classification | step levels, write-down check, monotonic run level, evidence bundle, raw export and passage gating, retention holds | works (A1) | V infrastructure/flow_run_repo.py:1654-1688; application/flow_run_access_policy.py:368-387 |
| Packages (.eneopkg) | export, import planner in publish mode, install, template uploads, model matching | partial: export refuses a flow with stale http auth config (eneo-jijy residual, V backend/src/eneo/flow_packages/application/flow_package_export_service.py:625-633) | S backend/src/eneo/flow_packages/application/*; V backend/src/eneo/flow_packages/application/flow_package_import_planner.py:240-250 |
| Run lifecycle | queued, running, awaiting_review, completed, failed, cancelled; retry as a new run; heartbeat reconciler | partial: A4a, A4b and A5 are open | V runtime/tasks.py:694; infrastructure/flow_run_repo.py:215-224 |
| Run history | list, detail, steps, evidence, access policy | partial: one unreadable version aborts the list (eneo-eaar.9) | V application/flow_run_service.py:1087-1119 |
| Authoring and publish | draft with a revision fence, immutable versioned snapshots with checksum | partial: no validity phases; a per-source step can make publish answer 500 | V application/flow_service.py:1352-1358; runtime/step_definition_parser.py:696-706 |
| Read-only "would this publish?" check | exists for an uploaded package only: `POST /import-plan/` is a side-effect-free checklist that reports whether the imported draft could be published. It runs `collect_step_graph_issues(require_complete_template_fill_config=True)`, skips the audio-model-required issue and reports the first issue. Missing for a stored editor flow: no route runs the strict check without publishing | partial | V backend/src/eneo/flow_packages/api/flow_package_router.py:216-232; backend/src/eneo/flow_packages/application/flow_package_import_planner.py:240-250; `POST /validate/` (:167) checks package structure only. `api/flow_authoring_router.py` has 8 routes: create, list, get, published runtime view (:474), patch, delete, publish, unpublish; none validates |
| Definition migration | none; the parser accepts only the current schema_version, and the current version is 1 | missing | V published_definition.py:33, :382-395 (parser gate). "Has never left 1": S APPENDIX-A DR-1 (not shown by a tip check) |

## 2. What landed 2026-10-19..20

All commits are on `feature/flows-and-builder` (`git -C /tmp/flows-tip-read log --since=2026-10-19`). Commits that touch only Builder code are not listed here.

| sha | What | Follow-up beads |
|---|---|---|
| 044ec559c | A3: every step contract check (input, LLM output, copy_input) runs in the killable child. Every contract check goes through the child (V runtime/contract_check.py:36; runtime/step_input_validation.py:116, :146; runtime/step_handlers/copy_input.py:80). The time limit counts queue wait (V runtime/tool_argument_check.py:11-18, :280-303). Gate GREEN 8 | eneo-8han, eneo-v7xp, eneo-nwvw, eneo-sdhq |
| 42f51c157 | A1: the run evidence level only rises (`greatest`, V infrastructure/flow_run_repo.py:1673). Raw export and passages use max(space, stored). An unknown run with evidence is held to class 3 | none |
| d112b62ad | A2: webhook delivery budget with a hard stop, a singleton cron, and unsent claims released without a charged attempt. Gate it3 GREEN 8 | eneo-hud2 (A4b), eneo-5gen, eneo-7cet, eneo-cb1j, eneo-olj8 |
| 3704a79fd | E2E: the Word-template save poll gets 15 s and the package import 120 s. Tests only | eneo-5uce is still open. It is still a wall-clock poll, not a wait on the save response |
| 348931fe7, 3a815a8f2, 154c7db18, caa8ded9f, 85ebada6d | Wave C dead code and wire-code fixes (C1-C3 and the same family) | C8 deferred: overlaps gallring's runtime/flow_runtime_health.py (quality-program:143) |
| a80280ae1 | The release smoke runs every backend worker role on the published image | none |
| 70a812993 | A tool server whose credential would travel over plain http is refused before the run | eneo-epim.57 |
| 411c98b82, 2cc3e5f50, 267ffa316, 3333ce272, 896706d3b, b049d1b9d | Transcript speaker names: stored with corrections, carried into regeneration, a name a person gives is stored as certain | none |

The quality program document still says A1 and A3 are "Running" (quality-program-20261020.md:146). Both landed (42f51c157, 044ec559c). Update that line when you next edit it.

## 3. Findings, ranked by consequence

Rank is by user consequence: a wrong result shown as success, then data or availability loss, then a wrong error, then maintainability.

| # | Problem | Concrete example | Evidence | Status / bead |
|---|---|---|---|---|
| F1 | **No single owner of "which output part carries a step's content".** Publish decides by output_type plus contract. The run decides by the stored payload shape. They agree today by coincidence | A section step reading a speaker_mapping step is refused at publish and at run, although that step's text is the full named transcript. Authors add a bridge step (långa-möten step 3) | V flow_validators.py:2538-2543 vs runtime/step_input_resolution.py:1444-1455; runtime/step_handlers/speaker_mapping.py:253, :272-275 | Section 4. Builder-side eneo-eaar.1 has a partial patch. No Flows-core bead |
| F2 | **A run says COMPLETED when a model step got the wrong part or no content.** Input resolution writes only an info `flow_underlag_summary`. Nothing raises a typed signal when a model step's resolved input is empty or lacks the real content. `StepDiagnostic.severity` is a bare str | Live case: step 3 read `step_2.output.structured` (333 B of speaker JSON). The 15,872-char named transcript was cited by no edge. The step answered "Transkript saknas" and the run ended COMPLETED | V runtime/step_input_resolution.py:583, :872, :885, :932 (all severity "info"); domain/runtime.py:122-125 (bare str, default "warning"). The runtime does write warnings and errors elsewhere (runtime/rag_retrieval.py:174, runtime/step_execution_runtime.py:1758 and :1788 error, runtime/step_handlers/speaker_mapping.py:298 and :368, runtime/output_formats/base.py:196 and :209), but none of them concern an empty or wrong-part input; S live case in builder-arch APPENDIX-B | Not filed in Flows core. Planned as P7 |
| F3 | **Publish can answer HTTP 500.** The per-source contract rule exists only in the runtime parser. Publish wraps the parser's BadRequest in `FlowPublishedDefinitionInvalidError` (a `FlowInvariantError`), and nothing maps it, so the catch-all answers 500. Any rule later added to the parser also applies to already-published versions | A per_source step whose documents[] lacks `source_label` saves as a draft and answers 500 at publish | V runtime/step_definition_parser.py:696-706; application/flow_service.py:1352-1358; domain/flow_invariant_exceptions.py:7; no FlowInvariantError handler in backend/src/eneo/server/main.py (grep) | eneo-eaar.11 (publish half: split it out as a Flows bead; its Builder plan-hydration half stays Builder) and eneo-be0p.17 (same class). Planned as P2 |
| F4 | **Run history: one unreadable published version aborts the whole page.** `_final_outputs_for_runs` parses every version of the page's COMPLETED runs with no local degradation | One old version that the current parser rejects hides every run on the page. Latent today: nothing has shipped, so only dev and demo databases can hold such a version | V application/flow_run_service.py:1087-1119 | eneo-eaar.9, open P3. How many stored versions are affected: UNKNOWN |
| F5 | **Published snapshots have a current-only parser and no migrator** (DR-1). Tightening the parser makes old versions unreadable (feeds F3 and F4) | Removing a feature breaks the run history and retry of old runs | V published_definition.py:33, :382-395 (the parser accepts only the current version); S APPENDIX-A DR-1 ("has never left 1") | Not filed. Production: no tenant has published flows, because flows is not on develop (V `git ls-tree -r origin/develop` finds no backend/src/eneo/flows file). Pilot or demo databases on the feature branch: UNKNOWN. So this is a cheap pre-release decision (P14) |
| F6 | **A worker shutdown is SIGKILLed mid-drain** (A4a). The execution and maintenance workers have no `stop_grace_period` or `init`. Docker's 10 s grace is below the 60 s `job_completion_wait`. The run stays RUNNING until heartbeat expiry (about 180 s), then FAILED run_worker_stalled | Every deploy can leave running flows to fail late | V docs/deployment/docker-compose.yml:205-245 (no grace), :256-258 (crawler has init and 70 s); backend/src/eneo/worker/worker.py:197 | A4a: no bead (quality-program:47, :124). A4b: eneo-hud2 |
| F7 | **Recovery paging lets healthy reviews take the slots** (A5). Every AWAITING_REVIEW run is in the paged UNION | A stale run waits about (older candidates / 100) minutes | V infrastructure/flow_run_repo.py:215-224, :1066-1070; runtime/tasks.py:694 | A5: no bead (quality-program:48, :126) |
| F8 | **A review edit checks the author's contract in-process on the API loop** | A backtracking `pattern` in a stored contract stalls an API worker when a reviewer edits a value | V application/flow_run_review_checkpoint_service.py:361-365 | eneo-8han, open. Needs a decision: typed_io_contract_not_checked would reach an API route |
| F9 | **A 3xx answer counts as success for webhooks and HTTP input.** `follow_redirects=False`, and only `>= 400` is an error | A moved webhook URL is recorded as delivered. A redirected http_get input becomes an empty successful input | V runtime/egress/http.py:237; runtime/http_orchestration.py:257, :428 | Not filed (found-outside-scope 2026-09-29) |
| F10 | **XLSX extraction invents values.** `df.ffill()` fills empty cells. Number-like text is also reported as coerced to floats (S bead eneo-s61k; no coercion code is visible in the extractor at the tip, so this half is unverified: plausibly pandas int to float promotion when a column has NaN, UNKNOWN without a probe). The shared extractor feeds chat, assistants and knowledge ingestion as well as flow run input, and the `ffill` is a deliberate merged-cell rule | An empty cell takes the value above it | V backend/src/eneo/files/text.py:718-719 (ffill, commented as merged-cell handling); runtime/step_input_resolution.py:22 imports files.text | Outside Flows scope, dropped from the plan (section 5). Already recorded at found-outside-scope.md:86-87. eneo-s61k should be retitled to name all consumers |
| F11 | **No explicit validity phases** (VAL-1). Draft and publish validation are one function and one bool, `require_complete_template_fill_config`, read at many branches. Each authoring path keeps its own ignore list | A flow is a valid draft and is refused at publish (unknown step name, `{{ indata_text }}` on a form flow) | S APPENDIX-A VAL-1; V flow_validators.py:623-637 | Not filed |
| F11b | **Publish and parser assemble the same rules twice** (VAL-3). The parser's errors are uncoded BadRequests. No "validators accept implies parser accepts" test | Root cause of F3 | S flow_validators.py:295-299, :385-398; runtime/step_definition_parser.py:643-694 | R209 (register). Partly in eaar.11 |
| F12 | **A3 follow-ups** | A tool-argument check queued behind a slow contract reports typed_io_tool_invalid_schema (blames the MCP server). copy_input failures say "model output". Deeply nested JSON may raise an untyped RecursionError | V runtime/step_handlers/copy_input.py:80; S bead texts | eneo-v7xp, eneo-nwvw (P4), eneo-sdhq (UNKNOWN, not probed) |
| F13 | **A2 follow-ups** | A row whose preparation always outlasts the send window is released forever without an attempt. Tests drive the test-only `deliver_due`. The mixed-wave hard stop is untested. Idle cadence is about 60 s | S bead texts; quality-program:128; V runtime/flow_webhook_delivery.py:176 (`deliver_due`) | eneo-5gen and eneo-olj8 are kept (P13). eneo-7cet is speculative and eneo-cb1j is a decided trade-off documented in docs/runbooks/flows.md:80-84: both dropped from the plan |
| F14 | **`föregående_steg` on step 1** passes validation and fails at run ("Unknown variable") | The resolver binds it only when step_order > 1 | V variable_resolver.py:288-300. Tool arguments already refuse it (flow_validators.py:1425-1430). Whether prompts and questions still pass: UNKNOWN (not probed) | eneo-epim.50, open |
| F15 | **Structured-path and alias rules differ by reader.** `step_1.output.structured.items.0.name` resolves at run time but is refused for a question template. `step_02` is accepted by one validator and rejected by the resolver | Hand-written flows see different answers from the editor and the run | S APPENDIX-A; V runtime read at runtime/step_input_resolution.py:1245-1275 | Not filed |
| F16 | **DOCX bindings judge a step name opposite to the run.** A step's name binds to its text only at run time | `{{Hämta.output.text}}` is accepted at publish for template_fill | S found-outside-scope 2026-10-13; flow_validators.py:1183, :1241, :1294 (read, not run) | Not filed |
| F17 | **Deleting a user assistant that a step references answers 500** (RESTRICT FK). `create_flow` does not require flow-owned assistants | | V application/flow_service.py:202-205, :801-806 (create path). The 500 itself is by source reading, not executed (S found-outside-scope.md:152) | eneo-f282, open P2 |
| F18 | **Outbound policy gaps**: the transcription service endpoint and the live-transcription relay | | S found-outside-scope.md:151 (2026-10-11, endpoint only); relay: found-outside-scope.md:61 (2026-09-29), api/flow_live_transcription_socket_router.py:190-201 | eneo-plsj (endpoint). Relay not filed |
| F19 | **Graph issue codes sit outside the generated error contract.** 41 `FlowGraphIssueCode` values, 33 not in `FlowApiErrorCode`. The web maps them by hand and most collapse into `flow_step_invalid` ("cannot be saved"), even for a publish refusal | | S probe in the input findings; eneo-7jo5.120 | eneo-7jo5.116, .118, .119, .120 |
| F20 | **Malformed client input answers 500** in eight places, e.g. a 4301-digit step alias | | S bead | eneo-9s1m |
| F21 | **Pattern time limit**: eneo-epim.16 says a `SCHEMA_PATTERN_BUDGET_SECONDS` deadline exists. The symbol is absent at the tip and in every ancestor of 044ec559c. The only commit that contains it is f9324be71 on `backup/r3-before-child-check`, which is not an ancestor of the tip. So the bead's "now evaluates ... under one deadline" describes work that is not on the tip | | V grep in backend/ empty; `git log -S ... HEAD` empty; `git log --all -S` finds only f9324be71; `merge-base --is-ancestor` is false | eneo-epim.16 text disagrees with the tip. Re-verify before acting |
| F22 | **Webhook payload signature** | grep for hmac or signature in the webhook and HTTP transport code finds nothing | V grep | UNKNOWN: a custom header or auth config may carry the trust |
| F23 | **Docs say "{{transkribering}} always resolves the raw transcript".** That is false as an absolute. The handler repoints the run transcript to the named text only when the stored transcript is an `inline_transcript` whose reference identity and text equal the speaker step's own source. Otherwise it stays the raw transcript. The quickstart line is also stated as always | | V docs/flows/external-transcription-service.md:90-91 vs docs/flows/flow-developer-quickstart.md:167-169 (repo-root docs/); V runtime/step_handlers/speaker_mapping.py:302-331 | Not filed. Fix with the section 4 slice |
| F24 | **Docs say MCP-bearing definitions are rejected and package export refuses a flow with a tool step**, while one tool per step now runs through the assistants' chain | | V docs/ARCHITECTURE.md:270-274 | eneo-epim.1 |

Other open Flows beads from the inventory, not re-verified here: eneo-jijy (residual only, section 5a), jijy.2, jijy.3, eneo-2yl (P1, stream step completions), eneo-0t7x (P1, truncated structured results), eneo-7azt (P1, pre-existing test failures), eneo-7jo5.108 (P1, urllib3), eneo-4ok.10.51, eneo-cls, eneo-qyf, eneo-wsv, eneo-gxjl, eneo-1gcl, eneo-dbd1, eneo-0d05, the review v2 epics (eneo-4x2, -078k, -ve7i, -8hn), eneo-f0xg, the MCP epic eneo-epim (S0-S7) and cleanup 7jo5.26-.57.

## 4. The "Del för del" proposal (a section step reads a speaker step directly)

**Verdict: partly legit.** The refusal and the motive are real. The proposal's mechanics are wrong in the ways listed in 4.2. This is an own Flows-core slice. The Builder adopts it later. After validation (section 5, P1): first run the cheap `{{ transkribering }}` probe and fix the docs, then land the owner slice with a fail-closed lookup.

### 4.1 Facts (V at the tip unless marked)
- **Publish rule:** `_validate_section_source_reads`, flow_validators.py:2469-2535.
  - It runs only when strict, which means publish (application/flow_service.py:728-740) and package import (backend/src/eneo/flow_packages/application/flow_package_import_planner.py:240-250). Draft save does not run it (S backend/tests/unittests/flows/test_flow_validators.py:4093).
  - Source selection uses `selected_source_step_order` (step_lineage.py:356-376), the same function the runtime uses (runtime/step_input_resolution.py:1405-1411).
  - **The selector picks a step for any `output.<path>` read, not only for text.** A step reference selects its step for the text forms (bare, `output`, `output.text`). Under section processing it also selects it for every other `output.` path, such as `output.structured` or `output.structured.speakers` (step_lineage.py:371-376). It returns only the step order, not which part was read. Once a step is selected, the whole-step text is the material (runtime/step_input_resolution.py:1405-1411).
  - It refuses when `_yields_structured_output(source)` is true (:2538-2543), which is `resolve_format_spec(output_type).requests_structured_output(contract)`. That is true for json, and for pdf/docx with a contract.
- **Run rule:** a separate check at runtime/step_input_resolution.py:1444-1455. It refuses when `payload["structured"]` is a dict or list. It decides by the stored payload shape and never calls the format owner. Only the selection rule is shared.
- **The speaker step's payload:** `text` is the full named transcript (`apply_speaker_names`, runtime/step_handlers/speaker_mapping.py:253; domain/speaker_labels.py:216-229), and `structured` is the name mapping (:272-275). A review edit rebuilds the same shape (S application/flow_run_review_checkpoint_service.py:586-596).
- **Nothing else is like it.** Only speaker_mapping writes real content in `text` beside a structured value. Every other json producer's text is the JSON rendering of its structured value (S runtime/step_handlers/copy_input.py:98-103, runtime/step_handlers/per_item_map.py:420, runtime/step_handlers/mapped_completion.py:253).
- **The format owner has a second meaning.** `requests_structured_output` also means "ask the provider for JSON" (runtime/step_execution_runtime.py:472). For speaker naming it must stay True.
- **Why both paths refuse today:** commit c0336ec69 made publish refuse what the run already refused. The test pins the speaker case as a refusal on purpose (S backend/tests/unittests/flows/test_flow_validators.py:4017).
- **Read paths into a speaker source (V runtime/step_input_resolution.py:1385-1411, :1493-1545).** Text reads: the implicit previous step, `all_previous_steps`, `{{ step_N.output.text }}` and the bare step form, `föregående_steg`, a `source_ref` binding, and the prompt. `{{ transkribering }}` is different: it reads the run's stored transcript, not a step result, so the per-step payload check never sees it. A raw string transcript is refused under sections (TRANSCRIPT_FORMAT_UNSUPPORTED, :1493-1498). Explicit structured read: `{{ step_N.output.structured... }}`.
- **Långa-möten step 3** (compose_text "Transkript som text", question `{{ step_2.output.text }}`) is needed today. With step 3 removed, publish refuses. With the section question set to `{{ transkribering }}`, publish accepts (probe in the input analysis). Whether the run then accepts `{{ transkribering }}` is by source reading only: after the speaker repoint (F23) the stored transcript is an inline_transcript that names the speaker step, and the code at runtime/step_input_resolution.py:1506-1545 accepts that. It is not executed, and no test combines speaker_mapping and sections. P1 step 1 tests exactly this before any code changes.

### 4.2 Corrected claims
1. "Both refuse" is true, but by two different expressions: output_type plus contract at publish, payload shape at run.
2. "Decide at the format owner" is wrong. `OutputFormatSpec` is keyed by output_type only, and its other consumer needs True for speaker naming.
3. Publish and run must change together, through one owner. A publish-only change brings back the publish-accepts, run-refuses bug that c0336ec69 fixed.
4. The Builder's own validator does not run this rule. A Builder benefits only through publish and import, so the rule must live in Flows core.
5. **The decision is per read path, not per step.** Exempting a speaker step as a whole is wrong: the selector also selects it for `{{ step_2.output.structured }}`, and a per-step exemption would accept that at publish and at run and silently hand the named transcript to the model as the material. That is the wrong-part-silently class of F1 and F2. A speaker source is text for text reads and structured for an explicit structured read.

### 4.3 Recommended single owner
Two small pieces, both in Flows core, one rule.

**(a) The read path becomes typed, in the shared selector.** In `step_lineage.py`, `selected_source_step_order` is replaced by a selector that returns the step order and the part read: `text` for the text forms and for every non-reference text source (implicit previous step, `all_previous_steps`, `föregående_steg`, `source_ref`, the prompt), and `structured` for any other `output.<path>`. Publish and run both call it, so the read path is classified once.

**(b) The format owner, in `flows/output_modes.py`, beside the other per-mode rules:**

```python
def content_output_part(
    *,
    output_mode: FlowOutputMode | str,
    output_type: FlowOutputType | str,
    output_contract: FlowPersistedJsonObject | None,
) -> SourceRefOutput  # "text" | "structured"
```

- An exhaustive `match` over `FlowOutputMode` with `assert_never`. SPEAKER_MAPPING returns "text". The match forces a decision for any new mode.
- Every other mode returns "structured" iff `resolve_format_spec(output_type).requests_structured_output(output_contract)`, so it composes the format owner and does not restate it.
- `output_contract` is required, with no default.
- Layering: output_modes would import runtime.output_formats. flow_validators.py:115 already does this. The input analysis found no import cycle in a shadow-copy prototype.

**The one rule for a section step (applied at publish and at run):** a selected source is refused with `TYPED_IO_INVALID_INPUT_SOURCE_COMBINATION` when

| Read part (from the selector) | Source | Result |
|---|---|---|
| text | `content_output_part(source) == "structured"` | refused |
| text | `content_output_part(source) == "text"` (including a speaker source) | accepted, the named text is the material |
| structured | the source carries a structured value: `content_output_part(source) == "structured"`, or the mode is SPEAKER_MAPPING | refused |
| structured | the source carries no structured value | accepted as today (the whole-step text is the material) |

Non-speaker sources behave as today for every read part. Only a speaker source changes: text reads move from refused to accepted, and an explicit structured read stays refused. If one step is selected by several reads, a single structured read refuses it.

**Call sites (all three, one slice):**
1. Publish and import: `_yields_structured_output` is replaced by the rule above, fed by the selector's read part (flow_validators.py:2496-2504, :2523, :2538-2543). Today `reads` keeps only the order, so it must keep the part too.
2. Run: replace the payload check at runtime/step_input_resolution.py:1444-1455.
   - FlowStepResult carries no step definition. Add a per-order content-part map to `RunExecutionState` (domain/runtime.py:196, the same pattern as tool_step_orders), built in runtime/execution_state_builder.py:15-44 from `steps`. The run collects each selected result together with its read part (today `selected` at :1396-1440 holds results only).
   - **No production caller passes `state=None` (V).** The run builds its state once with `build_run_execution_state(steps=...)` (runtime/executor.py:972-977) and passes it to `_execute_step` (:1165-1170). The webhook path builds one only to resolve variables (runtime/flow_webhook_delivery.py:498-501) and never calls `_resolve_step_materials`. The only bare construction in product code is the `_execute_step` fallback `if state is None: state = RunExecutionState(...)` at runtime/executor.py:1361-1378, with no steps; the other bare constructions are in unit tests (e.g. backend/tests/unittests/flows/test_text_sections.py:159). Production reaches that fallback through no path: the only production call of `_execute_step` is runtime/executor.py:1165, which passes `state`. Tests do use it: 44 calls omit `state=` (41 in backend/tests/unittests/flows/test_typed_io_executor.py, 2 in backend/tests/unittests/flows/test_tool_input_runtime.py, 1 in backend/tests/integration/flows/test_flow_step_file_mapping_contract.py).
   - **Design consequence (revised after validation):** the 13 `state: RunExecutionState | None` parameters in runtime/step_input_resolution.py (e.g. :204, :1374) ignore a missing state, and the existing `step_ref_mapping` read at :1399 falls back to `{}`. A per-order map with an empty default would silently apply no rule on a bare state. The first design made the map a required `RunExecutionState` field, deleted the runtime/executor.py:1371 fallback and moved the 44 fallback test calls. That also breaks 63 `RunExecutionState(` constructions in tests (V grep count) and is too much churn. Use a fail-closed lookup instead: build the map in runtime/execution_state_builder.py next to `tool_step_orders`; a selected order missing from the map raises a typed invariant. Same property (never silently no rule), much less churn.
3. Builder or its replacement, later: its per-site SPEAKER_MAPPING exceptions (listed in the Builder document) call this owner. A replacement tool calls the same owner.

### 4.4 Slice plan: red-first tests, each with the mutant it kills
| # | Test | Mutant it kills |
|---|---|---|
| T1 | Owner table over FlowOutputMode x FlowOutputType x {None, contract}. Expected = the old predicate, except speaker_mapping = "text" | speaker arm answers structured; every json answers text; contract ignored for pdf/docx (the eaar.1 two-argument table); a `case _` default arm |
| T2 | Publish accepts a speaker source for each text read: implicit previous_step, `all_previous_steps`, `{{ step_2.output.text }}`, föregående_steg, a source_ref text and the prompt (move backend/tests/unittests/flows/test_flow_validators.py:4017 from refused to accepted) | `_yields_structured_output` still format-only |
| T3 | Publish still refuses: json pass_through with and without contract, docx with contract, speaker -> json pass_through -> sectioned, and **an explicit `{{ step_2.output.structured }}` and `{{ step_2.output.structured.speakers }}` read of a speaker source** | exempting by the section step's mode; exempting "any speaker upstream"; **a per-step exemption that delivers the transcript for a `.structured` read** |
| T4 | Run: a speaker source payload {text, structured mapping, speaker_mapping} read by a text form gives one material equal to the named text. The same source read by `{{ step_2.output.structured }}` raises TYPED_IO_INVALID_INPUT_SOURCE_COMBINATION. A json pass_through source stays refused (S backend/tests/unittests/flows/test_text_sections.py:702-722 stays) | run keeps the payload-shape check; run exempts anything with text plus structured; run exempts the speaker step for every read path |
| T5 | Agreement table: {speaker, json+contract, text} sources x {text read, structured read} give the same verdict at publish and at run | either path restating the rule; the selector's read part used at one path only |
| T5b | A selected step order that is missing from the content-part map raises a typed invariant at run time (fail closed). A `RunExecutionState` built by the production builder always carries the map | a missing entry silently applies no rule |
| T6 | Full `tests/integration/flows` (runtime touched) | |

**Docs in the same commit:** repo-root docs/flows/flow-developer-quickstart.md:158-169 and docs/flows/external-transcription-service.md:90-91 (fixes F23). New wording for both: `{{transkribering}}` resolves to the named text when the transcript is the speaker step's own inline source, and to the raw transcript otherwise.

**Side gap that becomes reachable:** publish does not check the run's "exactly one material" rule (S runtime/step_input_resolution.py:1580-1587). After the change, speaker -> sectioned(all_previous_steps) passes publish and fails at run with two materials. Today transcribe -> text -> sectioned(all_previous_steps) already does this (S probe). File it as its own small slice (P17), probed first. Never leave a known publish-accepts, run-refuses case unrecorded.

### 4.5 Coordination: one owner, not two
- **Ownership.** This is a Flows slice in the Flows thread. The Builder has no text_processing reference; the editor, publish and package import author it. The Builder's eaar.1 reservation was released and Builder code work is paused (S sessions/log.md 2026-10-20T08:23:06Z).
- **The eaar.1 patch.** `eaar1-partial-uncommitted.patch` adds a two-argument `content_output_part(mode, type)` in the same place of output_modes.py.
  - Its `git apply` fails on that hunk if the Flows owner lands first.
  - A forced apply would define the function twice. The later two-argument definition would silently win and drop the contract handling.
  - Its eaar1-partial/test_output_modes_content_part.py pins the two-argument signature.
- **Resolution.**
  1. The Flows slice lands the three-keyword owner first.
  2. Whoever resumes eaar.1 (or builds the replacement) drops the patch's output_modes.py hunk and its owner test, and calls the Flows owner.
  3. The conversion from `FlowAuthoringOutputMode` happens at the Builder's edge.
  4. All other patch hunks are Builder files and do not overlap.
- **Långa-möten step 3** becomes optional after the slice. Keep the package as it is, since it is a valid positive control. A 1.1.0 without the bridge step makes a good acceptance case.

## 5. Plan for Flows, validated and ranked (independent of the Builder)

Each item lives in Flows core. Gate rules, mutation baseline and landing follow `/home/ccimen/dev/eneo/AGENTS.md`.

This plan was validated on 2026-10-20 by a separate read of the tip (044ec559c). Verdicts in the last column: **validated** (kept as written), **revised** (the item text below already applies the revision), **dropped** (moved to "Dropped after validation"). Table rows keep their P-numbers. The ROI order is in "Order" below the table, not the row order.

| # | Item | Problem -> change | Owner (file) | Before -> after | Proof | Risk | Bead | Validation |
|---|---|---|---|---|---|---|---|---|
| P1 | Content-part owner (F1) | **Step 1 (cheap, red first):** one integration test, speaker_mapping -> sectioned step reading `{{ transkribering }}`, with the expected verdict written down before the run. If it passes, document that path and fix F23 in docs/flows/external-transcription-service.md:90-91 and docs/flows/flow-developer-quickstart.md:167-169 (docs only, gate-exempt, does not wait for step 2). **Step 2 (owner slice):** section 4: typed read path in the shared selector plus the per-mode content-part owner. The run side uses a fail-closed lookup (a selected order missing from the content-part map raises a typed invariant, built in runtime/execution_state_builder.py next to `tool_step_orders`), not a required state field | output_modes.py; step_lineage.py; flow_validators.py; runtime/step_input_resolution.py; domain/runtime.py; runtime/execution_state_builder.py | Bridge step needed; two rules agree by coincidence -> one rule, publish and run agree by construction | Step 1: the probe test. Step 2: T1-T6 (incl. T5b); full flows integration suite | medium (run path touches the material resolver). Exhaustive match, per-read-path rule, agreement test. gallring also edits runtime/executor.py (2 lines) | File a Flows bead. Link eaar.1 as "adopts" | **revised.** Capability and maintainability gain, not a wrong-result bug: both paths refuse the case consistently today (flow_validators.py:2538-2543; runtime/step_input_resolution.py:1444-1455). A zero-code path may exist already (runtime/step_handlers/speaker_mapping.py:300-331 with runtime/step_input_resolution.py:1506-1545; UNKNOWN by execution). A required state field would break 63 `RunExecutionState(` constructions in tests on top of the 44 fallback calls |
| P2 | Publish implies parse, no 500 (F3, F11b) | Put the per_source predicate in one shared rule function, in the pattern of `transcribe_only_violation` and `copy_input_violation` (output_modes.py:16, :69). Call it from flow_validators with a graph code, and from the parser. Keep `FlowPublishedDefinitionInvalidError` as a 500 logged as an invariant (a validator/parser disagreement is a bug, not a user error). Add a parity table: for each validator fixture, accept implies `parse_published_definition` accepts. Fold in be0p.17 (the parser's own copy of the source_refs check) | flow_validators.py; runtime/step_definition_parser.py; application/flow_service.py:1352-1358 | 500 with an error_id -> typed 400 naming the field | Red test: publish of a bad per_source step answers 500 today. Parity table. PostgreSQL integration for publish | low | eaar.11 (publish half; split it out, do not file a duplicate) and eneo-be0p.17 | **validated.** The rule exists only in the parser (runtime/step_definition_parser.py:696-706); the validator checks only max_files (flow_validators.py:2581-2583). Same class: parser copy of the source_refs check (runtime/step_definition_parser.py:409-424 vs flow_validators.py:1108-1125) |
| P3 | Run-history robustness (F4) | Per-version local degradation: a version that does not parse is logged as an invariant and its runs get `final_output=None`. The page loads. Add a typed reason only if a reader must tell "unavailable" from "not completed" (that is a public API and SDK change). Detail, retry and evidence keep their own typed error. Tie to P14: if P14 adopts "old versions stay readable", P3 is defence in depth | application/flow_run_service.py:1087-1119 | One bad version hides all runs -> the page loads | Integration test: mixed old/current versions on one page (red first) | low | eneo-eaar.9 | **revised.** Real (parse and no-executable-steps raise run in a plain loop, application/flow_run_service.py:1087-1118) but latent: flows is not on develop, so no production version exists. Ranked after P10, P5 and P1 |
| P4 | A4a shutdown budget (F6) | One budget = drain + margin, the crawler pattern (`job_completion_wait` plus a margin). The compose `stop_grace_period` and `init: true` for both workers are derived from it. Fix the runbook. One contract test reads compose against the budget; a mutant drops the grace | worker/worker.py; docs/deployment/docker-compose.yml; runbook | SIGKILL mid-drain -> graceful drain | Contract test (one assertion); a mutant drops the grace | low. Check file ownership first: gallring edits backend/src/eneo/worker/arq.py | File (none exists) | **validated.** Workers have no grace (docs/deployment/docker-compose.yml:205-245), the crawler has init and 70 s (:256-258), `job_completion_wait = 60` (backend/src/eneo/worker/worker.py:197). Helps only runs that finish inside the drain; longer runs are P5's job. Land with or just before P5 |
| P5 | A4b typed shutdown outcome (F6) | A run cancelled by shutdown ends with a typed outcome at once, not stalled after 180 s. A requeue is deliberately deferred (it needs idempotency for steps whose outcome is unknown, out of scope by decision): say so in the bead | runtime/tasks.py (execute_flow_run_task) | Late FAILED run_worker_stalled -> immediate typed outcome | Integration test with a cancelled task; Astra's constraints in the bead (quality-program-20261020.md:124-126) | medium (lifecycle: arq on_shutdown order, webhook-waiting runs that are RUNNING, execution-owner fence) | eneo-hud2 | **validated.** The reconciler ends the run FAILED run_worker_stalled "heartbeat expired" (runtime/tasks.py:719-725); CancelledError is re-raised with no typed outcome (runtime/execution_heartbeat.py:108). Same batch as P4 |
| P6 | A5 recovery LIMIT (F7) | Heartbeat-expired candidates get their own LIMITed query each tick. Review pagination stays bounded and is inspected separately. Note: quality-program-20261020.md:48 still describes the older "actionable reviews only in SQL" design; line 125 matches this item. Reconcile the two texts before filing | infrastructure/flow_run_repo.py:215-226 | Stale runs wait behind healthy reviews -> bounded wait | PostgreSQL integration: 150 healthy reviews plus 1 stale, the stale one is recovered in one tick | low | File (none exists) | **validated.** Every AWAITING_REVIEW run sits in the paged UNION (infrastructure/flow_run_repo.py:216-226, :1046-1061; runtime/tasks.py:657-688). A delay, not starvation: minutes at single-tenant scale. Lower than P10, P5, P1 |
| P7 | Honest run outcome (F2) | Split in three. **(a)** `StepDiagnostic.severity` becomes a StrEnum (domain/runtime.py:122-125): a small typed cleanup, anytime. **(b)** An authoring-time typed hint from the P1 owner: a model step whose only material is the structured part of a step whose content part is text. This matches the live case and is typed, not a word list. **(c)** Showing warnings on the run: a UI feature (shadcn, sv+en) behind an owner decision. Dropped: "fail on empty input". If an empty-input fact is kept it is a warning only, because empty input is legal for prompt-only steps | domain/runtime.py:122-125; runtime/step_input_resolution.py; the P1 owner | Silent COMPLETED -> a visible typed signal | Unit test per fact; one integration run for (b) | medium for (c) (product decision) | File (a), (b), (c) separately | **revised.** The live case read 333 B of speaker JSON, so an empty-input fact would not have fired. Empty input is legal (runtime/step_execution_runtime.py:176-183). The web shows no step diagnostics on the run (frontend/apps/web/src/lib/features/flows/components/FlowRunEvidenceToolbar.svelte:26-59 only puts them in a debug export) |
| P8 | Typed validation phase (F11) | Narrowed. Replace the bool `require_complete_template_fill_config` by a `FlowValidationPhase` enum (draft, publish, package_import), so the audio-model exception becomes a named phase rule and not a filter at the import call site (backend/src/eneo/flow_packages/application/flow_package_import_planner.py:238-250). Do it after P2. Defer the read-only "would this publish?" route until the editor or a Builder replacement asks for it (an OpenAPI and SDK change without demand today) | flow_validators.py; application/flow_service.py:738, :754, :761; the import planner | One misleading bool read at 14 branches -> named phases | Phase table test | low (mechanical) | File. Do after P2 | **revised.** The bool governs far more than template fill (14 reads in flow_validators.py:209-656). api/flow_authoring_router.py has 8 routes and none validates; the route has no consumer now |
| P9 | A3 follow-ups (F8, F12) | Split. **(a) now, one small typed-error slice:** sdhq (type the RecursionError from deeply nested model JSON, output_processing.py:149-175 catches only JSONDecodeError and ValueError; probe first), v7xp (an unfinished argument check must not blame the MCP server's schema: runtime/tool_input.py:525-532 answers TYPED_IO_TOOL_INVALID_SCHEMA, while the sibling runtime/contract_check.py:56-68 uses CONTRACT_NOT_CHECKED), nwvw (copy_input failures say "model output"). **(b) deferred:** 8han, until the owner decides on the API code (typed_io_contract_not_checked on an API route) | runtime/tool_input.py; runtime/contract_check.py; output_processing.py; application/flow_run_review_checkpoint_service.py:361-365 | Wrong blame and untyped escape -> typed, correctly blamed | Unit tests; sdhq probe first | low (a); needs a decision (b) | eneo-sdhq, -v7xp, -nwvw now; -8han later | **revised.** All four beads still true. 8han reaches only contracts that authors control |
| P10 | 3xx is a failure (F9) | 3xx becomes a typed failure for webhook and HTTP input, in one owner (the shared status check). Reuse `TYPED_IO_HTTP_NON_SUCCESS` or add a redirect code. Decide whether a webhook 3xx retries or dead-letters (it will not fix itself). Check the gallring diff first | runtime/http_orchestration.py:257, :428; runtime/egress/http.py:237 | Silent "success" -> typed failure and retry or dead letter | Unit test with a 302; webhook integration | low. Expect a forward-merge conflict: origin/feature/gallring differs from the tip in runtime/http_orchestration.py (V `git diff --shortstat`; the validator reports it rewrites the same functions) | File | **validated, moved up to second.** `follow_redirects=False` (runtime/egress/http.py:237); only `>= 400` is an error (runtime/http_orchestration.py:257, :428); no test pins 3xx. Recorded at found-outside-scope.md:61 item (2) |
| P11 | XLSX values (F10) | **Dropped.** See "Dropped after validation" | | | | | | **dropped** |
| P12 | Wave B | Re-verify each row at the tip before filing, then one bead per surviving row. **B9 first:** runtime file integrity reads `diag/constraint_name` from `exc.orig` and falls back to substring matching on the message (flow_runtime_file_integrity.py:19-46), while infrastructure/flow_repo.py:225-229 already reads `exc.orig.__cause__.constraint_name`. Reuse that extractor and delete the substring fallback. Land B4 (webhook error text) after the gallring http_orchestration forward merge. Rows: B1 import graph covers MCP (missing `__init__.py`), B2 MCP failure reason, B3 encoding code, B4 webhook error text, B5 diagnostics projection, B6 web read errors, B7 citations note, B8 false remedy, B9 typed constraint name | per row, quality-program-20261020.md:50-61 | per row | one owner per row; positive control for B1 | low. B5 and B6 are web work, B2 touches MCP | One bead per surviving row (none exist) | **revised.** Only B9 was re-verified. B1-B8 are not re-verified at the tip (UNKNOWN). A bucket, not one item |
| P13 | A2 follow-ups (F13) | Keep two beads. eneo-5gen: move the tests to the production wave path and delete the test-only `deliver_due` (runtime/flow_webhook_delivery.py:176). eneo-olj8 item 1: the mixed-wave hard-stop test (the current test uses `row_count=1`, backend/tests/integration/flows/test_flow_webhook_outbox_delivery.py:2714-2775). Tests-only, gate-exempt, can run in parallel anytime | runtime/tasks.py; runtime/flow_webhook_delivery.py | Gaps -> covered | Integration tests | low | eneo-5gen, eneo-olj8 | **revised.** 5gen and olj8 item 1 validated. 7cet and cb1j are dropped (see below) |
| P14 | Definition compatibility (F5) | An owner decision or ADR before PR A merges, e.g. "a parser change that rejects a previously valid snapshot bumps `schema_version` and keeps a reader for the old version". Add a guard test that pins a fixture of the current published shape. Write no migrator until a bump happens | published_definition.py:33, :382-395 | Silent breakage -> explicit policy | Pinned fixture of the current published shape | low (decision, no code). Pilot or demo databases on the feature branch: UNKNOWN | File (DR-1/DR-6) | **revised.** Production answer is known: flows is not on develop, so no tenant has published flows. A cheap decision now, an expensive one after the first release. Moved up |
| P15 | Smaller correctness (tail) | One bead per item. **f282** after a product decision (refuse a foreign assistant at create, or refuse the delete; application/flow_service.py:801-806). **9s1m:** rewrite to its five remaining items (items 1, 3 and 4 are done). **7jo5.118-.120** (validated P3). **F15 and F16:** probe at the tip before filing (flow_validators.py:1183-1294 shows reference judging but not the opposite verdict). **C8 runbook:** stays blocked until gallring releases runtime/flow_runtime_health.py. **7jo5.116** is outside Flows (settings/setting_service.py:126, files/docx_template_validation.py:57): record it in found-outside-scope.md | per bead | | red-first per bead | low | as listed | **revised.** Bucket split. epim.50 moved out (P15a) |
| P15a | `föregående_steg` on step 1 (F14) | Call the existing `names_step_without_result` owner from the prompt and question paths, as tool arguments already do (flow_validators.py:1425-1430) | flow_validators.py; variable_resolver.py:288-303 | Publishes, then fails every run -> refused at publish | Red test: publish accepts and run fails today | low | eneo-epim.50 | **validated, moved up.** Same class as P2 (a publish-accepts, run-refuses bug) |
| P16 | http_get + template_fill never fetches | Refuse the pair at publish (an http_get input with template_fill is legal and the fetch is never made). The capability check refuses only the TOOL source (flow_capability_manifest.py:924-932) | flow_capability_manifest.py | Silent success without a fetch -> typed refusal | Red test for the pair | low | eneo-be0p.18 | **validated, added from the bead validation.** A cheap silent-success bug the plan had omitted |
| P17 | Section "exactly one material" at publish | Probe first. Publish does not check the run's exactly-one-material rule (runtime/step_input_resolution.py:1580-1587; no material-count check in flow_validators.py). A publish-accepts, run-refuses case that exists today, independent of the speaker case. Either add the count check or record the accepted gap | flow_validators.py; the P1 owner | Publish accepts, run refuses -> one verdict | Probe test, then a parity row | low | File | **validated by source, PLAUSIBLE by execution** (not probed). Split out of section 4.4 |

**Order (ROI, validated).**
1. P2 with be0p.17: a publish 500 and one shared per_source rule plus a parity table.
2. P10: 3xx counted as success (coordinate with gallring's http_orchestration diff).
3. P16 (eneo-be0p.18): http_get with template_fill silently never fetches.
4. P15a (epim.50): `föregående_steg` on step 1 publishes, then fails every run.
5. P17: the section exactly-one-material publish gap, after a probe.
6. P4 and P5 in one batch: deploy SIGKILL, then a 180 s stall.
7. P14: an owner decision or ADR before PR A merges (no code).
8. P1 step 1: the `{{ transkribering }}` speaker -> sections test, plus the F23 docs fix (gate-exempt).
9. P9a: sdhq, v7xp and nwvw typed fixes.
10. P1 step 2: the owner slice with the fail-closed lookup.
11. P6.
12. P3.
13. P12, row by row after re-verification, B9 first.
14. The rest of P15: f282 after its decision, 9s1m's five items, F15 and F16 after probes, 7jo5.118-.120.
15. P7: (a) anytime, (b) after P1, (c) after a decision.
16. P8, narrowed to the typed phase enum.

In parallel at any time: P13 (tests-only, gate-exempt). Check `/home/ccimen/dev/eneo/demo-stack/specs/sessions/gallring-owned-files.sh` before touching files in the same lane.

Order changes from the first draft, with the reason:
- P10 moves up: by the plan's own rule (wrong result shown as success comes first), a tiny change belongs right after P2.
- P3 moves down: it is latent, because nothing has shipped.
- P1 moves down: it removes no silent failure, and its main benefit may be reachable with a test and a docs fix (step 1).
- P14 moves up: the decision costs nothing now and much after the first release.
- P6 stays below P10, P5 and P1: a delay of minutes at single-tenant scale.

### Dropped after validation
| Item | Reason |
|---|---|
| P11 (XLSX `ffill`, F10) | Outside Flows scope. The shared extractor feeds chat, assistants and knowledge ingestion, and the `ffill` is a deliberate merged-cell rule (backend/src/eneo/files/text.py:718-719). The correct fix (fill only real merged ranges) is a design choice for the files owner. Already recorded at found-outside-scope.md:86-87. Retitle eneo-s61k to name all consumers. Do not fix it in a Flows lane |
| eneo-7cet (from P13) | Speculative: the row only loops if preparation always outlasts the deadline, with no known cause |
| eneo-cb1j (from P13) | A decided trade-off, documented in docs/runbooks/flows.md:80-84. Close it as decided |
| "fail on empty input" (from P7) | Empty input is legal for prompt-only steps (runtime/step_execution_runtime.py:176-183) |
| The required `RunExecutionState` field and the 44-call test move (from P1) | Replaced by the fail-closed lookup. Too much churn for the same property |
| The read-only "would this publish?" route (from P8) | Deferred until a consumer asks. Not part of the narrowed P8 |

## 5a. Validated Flows beads
All 102 Flows beads were checked one by one against the tip. The full list, one verdict per bead with its evidence, is in `/home/ccimen/dev/eneo/demo-stack/specs/flows-20261020/FLOWS-BEADS-VALIDATED.md`, with the machine-readable form in `/home/ccimen/dev/eneo/demo-stack/specs/flows-20261020/flows-beads-validated.json` (both are published in this folder with this plan; if a file is absent, that publication has not happened yet).

| Result | Count |
|---|---|
| Flows beads checked | 102 |
| Validated real improvements | 57 |
| Recommended to close | 19 |
| True but not recommended now | 26 |

Beads the plan had omitted and the validation added: eneo-be0p.18 (P16), eneo-epim.50 (P15a), eneo-jijy (residual only: stale http auth config refused at export, backend/src/eneo/flow_packages/application/flow_package_export_service.py:625-633; section 1 no longer calls it the broad P1), eneo-r9jy (retry after a spilled transcript) and eneo-epim.25.

**Not in this plan:** Builder planning, outline and slot classification, Builder word lists (Wave E), and Builder evaluation. See the Builder document. "Outline coverage when declared" has no Flows-core concept: grep finds no outline outside ai_builder except docx rendering. It becomes a Flows rule only if a declared section list is added to the step definition (UNKNOWN whether wanted).

## 6. How to evaluate Flows changes
1. **Deterministic tests first.** Almost every rule above (P1-P6, P9-P17) is decidable without a model. Use unit tables over enums (mode x type x contract), and validator-vs-parser and publish-vs-run agreement tables. Each new test names the mutant it kills (skill test-hygiene).
2. **PostgreSQL integration** for anything that touches locks, recovery, publish transactions, run history or webhooks. Use `lanetest.sh` (2 slots per host, at most 4 xdist workers). Run the full `tests/integration/flows` once per landing batch, when runtime, authoring or classification changed.
3. **E2E on CI** only for editor, run-dialog and template flows. Give a timing budget per spec, and wait on the response rather than polling the wall clock (the eneo-5uce lesson). Run web unit tests with `--project dom --project server`.
4. **No LLM** for Flows rules. A fake completion client covers step runtime.
5. **A live run is warranted** only when a change alters what a model receives, e.g. P1 changes the material a section step reads. Then one run of a fixed package (långa-möten 1.0.0 and a bridge-less variant) on the demo stack. Never use the personal-data run exports as fixtures. Follow validation-ladder-20261020.md: a prediction before the run, and spend only when it can change the landing decision.
6. **Agreement over coverage.** The recurring Flows defect class is "two readers of one rule disagree" (F1, F3, F14, F15, F16). A parity table kills that class. More example tests do not.

## 7. Lessons for Flows
- One owner per rule, called from publish, run, import and the editor. A rule restated per path drifts (F1, F3).
- A publish rule without the matching run rule, or the reverse, recreates c0336ec69's bug class. Change both in one slice.
- Put authoring rules in Flows core, not in an authoring tool. Tools get replaced, and the rule must survive that.
- Typed decisions only. No word lists, and no string matching on codes or messages.
- Persisted data outlives the parser. Readers of old versions must degrade locally (F4, F5).
- A run that "completes" on empty or wrong input is a silent failure. Make it visible with a typed fact (F2).
- Verify bead text against the tip before acting (eneo-epim.16 names a symbol that does not exist).

## 8. How to resume
1. State and beads.
   ```bash
   git -C /home/ccimen/dev/eneo/eneo fetch origin && git -C /home/ccimen/dev/eneo/eneo log --oneline -5 origin/feature/flows-and-builder
   bash ~/.claude/skills/eneo-slice-landing/scripts/brt.sh show eneo-eaar.9
   bash ~/.claude/skills/eneo-slice-landing/scripts/brt.sh show eneo-hud2
   bash ~/.claude/skills/eneo-slice-landing/scripts/brt.sh show eneo-8han
   bash ~/.claude/skills/eneo-slice-landing/scripts/brt.sh show eneo-be0p
   ```
   File the beads that do not exist yet (P1 step 2, the P2 publish half, P4, P6, P7a/b, P10, P17) with `brt.sh create` (check `brt.sh --help` for flags). Section 5 lists the existing ones.
2. Before a slice, check ownership and reserve paths.
   ```bash
   bash /home/ccimen/dev/eneo/demo-stack/specs/sessions/gallring-owned-files.sh
   # then add a reservation line to /home/ccimen/dev/eneo/demo-stack/specs/sessions/log.md
   ```
3. Open a lane on the current tip.
   ```bash
   bash ~/.claude/skills/eneo-slice-landing/scripts/newlane.sh <name> <branch> origin/feature/flows-and-builder
   ```
4. Iterate red-first.
   ```bash
   bash ~/.claude/skills/eneo-slice-landing/scripts/lanecheck.sh --focused
   bash ~/.claude/skills/eneo-slice-landing/scripts/lanetest.sh <test paths>
   ```
   Lane rules: ~/.claude/skills/eneo-slice-landing/lane-implementer-template.md.
5. Squash, rebase, adversary, brief, gate.
   ```bash
   bash ~/.claude/skills/eneo-slice-landing/scripts/squash.sh
   bash ~/.claude/skills/eneo-slice-landing/scripts/rebase_lane.sh
   # pre-gate adversary (skill pre-gate-adversary), then
   bash ~/.claude/skills/eneo-slice-landing/scripts/brief.sh
   ENEO_REVIEWER=claude bash ~/.claude/skills/eneo-slice-landing/scripts/gate_parallel.sh <args>
   bash ~/.claude/skills/eneo-slice-landing/scripts/gate-verdict.sh <session>
   ```
   Tests-only, docs-only and CI slices go through `gate_exempt.sh <session> <sha> --reason "..."`.
6. Land.
   ```bash
   bash /home/ccimen/dev/eneo/demo-stack/builder-guards/land_queue.sh --batch
   ```
   Run the mutation baseline once per batch: `demo-stack/test-audit/rerun_baseline.sh <tag> --flows`.
7. For P1 (step 1 first, then the owner slice), read section 4 and `/home/ccimen/dev/eneo/demo-stack/specs/builder-arch-20261020/eaar1-partial/state.md`. Do NOT apply that patch's output_modes.py hunk. Land the three-keyword owner as a Flows slice.
8. Probes. Use `PYTHONPATH=<lane>/backend/src /home/ccimen/dev/eneo/eneo/backend/.venv/bin/python`. A module that imports ai_builder first needs the table bootstrap, or lanetest.sh.
