# QuantCoder v2.0.0 — Architectural Review & Code Quality Assessment

> **Review Date:** February 2026
> **Scope:** Full codebase review of the QuantCoder CLI, assessed for its ability to interpret academic research papers and generate production-quality QuantConnect LEAN algorithms for backtesting.

---

## Executive Summary

QuantCoder is an ambitious local-first CLI that attempts to bridge the gap between academic quant research and executable QuantConnect algorithms using local LLMs (Ollama). The architecture — featuring a two-stage code generation pipeline, cross-model fidelity assessment, an 11-rule static QC API linter, a multi-agent system, and an AlphaEvolve-inspired evolution engine — is thoughtfully designed. However, several structural issues and missing capabilities significantly limit the system's ability to faithfully translate complex research papers into correct, well-structured QuantConnect code. This review identifies these issues and proposes concrete improvements.

---

## 1. Architecture Overview

### 1.1 High-Level Component Map

```
┌─────────────────────────────────────────────────────────────┐
│                          CLI Layer                           │
│     cli.py (1933 lines) — Click-based entry point           │
│     chat.py — Interactive REPL                              │
│     config.py — TOML-based configuration                    │
└────────────┬───────────────────────────────┬────────────────┘
             │                               │
   ┌─────────▼──────────┐         ┌──────────▼───────────┐
   │    Tools Layer      │         │  Autonomous Modes    │
   │  article_tools.py   │         │  autonomous/pipeline │
   │  code_tools.py      │         │  library/builder     │
   │  deep_search.py     │         │  scheduler/runner    │
   │  file_tools.py      │         │  evolver/engine      │
   └─────────┬───────────┘         └──────────┬───────────┘
             │                                 │
   ┌─────────▼─────────────────────────────────▼──────────┐
   │                    Core Pipeline                       │
   │  processor.py — PDF extraction, NLP, pipeline orch.   │
   │  llm.py — Two-stage codegen, fidelity assessment      │
   │  qc_linter.py — 11-rule static QC API linter          │
   │  summary_store.py — Summary persistence               │
   └─────────┬─────────────────────────────────┬───────────┘
             │                                 │
   ┌─────────▼──────────┐         ┌────────────▼──────────┐
   │   LLM Provider      │         │  Multi-Agent System   │
   │  providers.py        │         │  coordinator_agent    │
   │  OllamaProvider      │         │  alpha_agent          │
   │  LLMFactory          │         │  risk_agent           │
   └──────────────────────┘         │  universe_agent       │
                                    │  strategy_agent       │
                                    └───────────┬───────────┘
                                                │
                                    ┌───────────▼───────────┐
                                    │    MCP Integration     │
                                    │  quantconnect_mcp.py   │
                                    │  Client + Server       │
                                    └────────────────────────┘
```

### 1.2 Core Pipeline Flow

The primary code generation flow (from `generate_code_from_summary` in `processor.py`) proceeds as:

1. **PDF → Sections** — MinerU (preferred, with LaTeX preservation) or pdfplumber + SpaCy  
2. **Pass 1 — Extractive** — LLM verbatim-quotes relevant passages from the paper  
3. **Pass 2 — Interpretive** — LLM converts quotes into structured strategy specification  
4. **Stage 1 — QC Framework** — Coding LLM generates compilable algorithm with stub methods for novel math  
5. **Stage 2 — Mathematical Core** — Coding LLM fills stub methods with implementations  
6. **Syntax Validation Loop** — `ast.parse()` with up to 6 refinement attempts  
7. **QC API Linting** — 11 static rules auto-fix common LLM mistakes  
8. **Fidelity Assessment** — Reasoning LLM cross-checks code against summary (up to 3 rounds)  
9. **Backtest** — Upload, compile, and run on QuantConnect cloud  

This is a well-considered pipeline that addresses the right problems in sequence. The separation of concerns between extraction, interpretation, framework generation, and math implementation is architecturally sound.

---

## 2. Strengths

### 2.1 Two-Stage Code Generation Pipeline
The separation of framework scaffolding (`generate_qc_framework`) from mathematical core implementation (`fill_mathematical_core`) is the most important architectural decision in the codebase. It correctly identifies that LLMs struggle to simultaneously handle both QuantConnect API boilerplate and novel mathematical models. By first generating a compilable scaffold with stubs, Stage 1 guarantees a syntactically valid algorithm, while Stage 2 can focus exclusively on the mathematical logic.

### 2.2 Cross-Model Fidelity Assessment
Using the reasoning model (mistral) to evaluate the coding model's (qwen2.5-coder) output addresses the fundamental threat of LLM substitution — where the model silently replaces a complex model (e.g., OU process) with a simple indicator (e.g., SMA). The structured response format (`FAITHFUL/SCORE/ISSUES/CORRECTION_PLAN`) and the regeneration loop with critique feedback is well-designed.

### 2.3 Two-Pass Summarization
The `extract_key_passages` → `interpret_strategy` pipeline is a significant improvement over single-pass summarization. By first forcing verbatim quotation, the system preserves exact formulas, parameter values, and novel terminology before interpretation. This is a textbook approach for reducing hallucination in extractive-to-generative pipelines.

### 2.4 QC API Linter
The 11-rule linter addresses the most common QuantConnect API mistakes that local LLMs produce:
- **QC001**: PascalCase → snake_case API method conversion (200+ mappings)
- **QC007**: Resolution casing normalization
- **QC004**: C# `Action()` wrapper removal
- **QC002/QC003**: RollingWindow `len()` / `.Values` fixes
- **QC008**: Indicator name shadowing detection
- **QC009**: Asset class detection (forex/crypto ticker → correct `add_*` method)
- **QC010**: Data access normalization
- **QC011**: Missing `from AlgorithmImports import *`

This is pragmatic, well-targeted engineering. The auto-fix capability (vs. warning-only) is correctly calibrated by rule severity.

### 2.5 Graceful Degradation
The codebase consistently implements fallback paths: MinerU → pdfplumber, two-pass summary → legacy keyword-filtered summary, Stage 1+2 → single-shot codegen, fidelity loop → last valid code. This resilience is critical for a system powered by non-deterministic LLMs.

### 2.6 MinerU Integration for LaTeX Preservation
Using MinerU for PDF extraction preserves mathematical equations as LaTeX (`$...$` / `$$...$$`), which is critical for papers with novel mathematical models. The intelligent GPU management (unloading Ollama models before MinerU runs) shows awareness of resource constraints.

---

## 3. Critical Issues

### 3.1 `max_tokens` Configuration Bottleneck

**Severity: Critical**

The `max_tokens` is configured globally at **3000 tokens** in `config.py` and used as the default for code generation:

```python
# config.py line 33
max_tokens: int = 3000

# llm.py line 363-364
code = _run_async(
    self._code_llm.chat(messages=messages, max_tokens=self.max_tokens, ...)
)
```

3000 tokens is severely insufficient for generating non-trivial QuantConnect algorithms. A strategy with custom indicators, risk management, scheduled rebalancing, and universe selection can easily exceed 200 lines / 6000+ tokens. The `max_tokens` cap silently truncates generation, producing incomplete algorithms with missing closing brackets, half-written methods, or truncated `on_data` handlers. This is particularly devastating for Stage 2 (`fill_mathematical_core`), which must reproduce the entire algorithm with stubs filled — effectively doubling the token requirement.

**Impact:** Silently truncated code that fails syntax validation, wasting all refinement attempts on an unrecoverable error.

**Recommendation:** Set per-task token limits. Code generation should use at minimum 8192 tokens (already used by `fix_runtime_error`). Summary and fidelity tasks can remain at lower limits. Make this configurable per-task in `config.toml`.

### 3.2 Multi-Agent System Does Not Integrate with Core Pipeline

**Severity: Critical**

The multi-agent system (`agents/`) and the core pipeline (`core/`) operate as **two entirely separate code generation paths** that are never unified:

- The **core pipeline** (`processor.py` → `llm.py`) is used by `cli.py`'s `generate_code` command and produces single-file algorithms
- The **multi-agent system** (`CoordinatorAgent`) is used only by the autonomous pipeline (`autonomous/pipeline.py`) and produces multi-file Framework algorithms (Main.py + Alpha.py + Risk.py + Universe.py)

These two paths have:
- **Different prompting strategies** — the core pipeline has detailed QC API rules and indicator signatures; the agents have generic "return ONLY Python code" instructions
- **Different validation** — the core pipeline runs the QC linter and fidelity assessment; the agents do not
- **Different code styles** — the core pipeline enforces `snake_case`; the agents request `PascalCase` Framework API (`Initialize()`, `SetAlpha()`)
- **No shared knowledge** — the agents' system prompts are sparse compared to the core pipeline's detailed instructions

The agent system prompts in `alpha_agent.py`, `risk_agent.py`, `universe_agent.py`, and `strategy_agent.py` are significantly weaker than the core pipeline prompts. For example, `AlphaAgent` asks for `Update()` and `InsightDirection` (PascalCase), while the core pipeline enforces `snake_case`. The `StrategyAgent` requests `Initialize()` and `SetAlpha()` (also PascalCase).

**Impact:** The multi-agent path produces code that would fail the core pipeline's own linter. The most sophisticated pipeline (two-stage generation, fidelity assessment, linting) is not available for Framework-style algorithms.

**Recommendation:** Either (a) unify the two paths by having the coordinator use the core pipeline's `LLMHandler` methods with agent-specific context, or (b) port the core pipeline's prompt quality and validation stack into the multi-agent system.

### 3.3 No Semantic Validation of Generated Code

**Severity: High**

Validation is limited to `ast.parse()` (syntax only) and the QC linter (API pattern matching). There is **no semantic validation** that checks whether the generated code actually implements the strategy specification. Specifically:

- **No indicator correctness check** — the code may use `self.rsi(symbol, 14)` (wrong arity, needs 4 args) and this won't be caught until QuantConnect runtime
- **No strategy logic verification** — if the summary says "go long when RSI < 30" but the code says `if rsi.current.value > 70: self.set_holdings(...)`, nothing catches this
- **No parameter value verification** — if the summary specifies a 20-day lookback but the code uses 50, nothing catches this

The fidelity assessment (reasoning LLM cross-check) is the intended semantic check, but it's entirely prompt-based and runs against the *summary* — not against the *paper*. If the summary itself lost fidelity (e.g., misinterpreted "OU process" as "mean reversion"), the fidelity check will approve the wrong implementation.

**Recommendation:** 
1. Add AST-based semantic checks: verify indicator argument counts, detect unused variables, check that `on_data` actually references registered symbols
2. Add a "chain of verification" where the fidelity check receives both the original paper extractions AND the code (not just the summary)
3. Consider using the LEAN compiler (via QC API compile endpoint) as an early validation step before backtesting

### 3.4 `_run_async` Thread-Safety and Event Loop Handling

**Severity: Medium-High**

The `_run_async()` helper in `llm.py` uses a fragile pattern:

```python
def _run_async(coro):
    try:
        loop = asyncio.get_running_loop()
    except RuntimeError:
        loop = None
    if loop and loop.is_running():
        import concurrent.futures
        with concurrent.futures.ThreadPoolExecutor() as pool:
            return pool.submit(asyncio.run, coro).result()
    else:
        return asyncio.run(coro)
```

This spawns a new thread with a new event loop every time it detects an existing running loop. When called from the autonomous pipeline (which is already async), this creates nested event loops with new `aiohttp.ClientSession` instances per call, leading to:
- Thread proliferation during long runs
- Connection pool exhaustion (new `aiohttp.ClientSession` per request, never reused)
- Potential deadlocks if the `ThreadPoolExecutor` is exhausted

**Recommendation:** Refactor `LLMHandler` methods to be `async` natively. The `_run_async` bridge should only exist at the CLI boundary, not at the core layer. Reuse a single `aiohttp.ClientSession` per provider instance.

### 3.5 No Rate Limiting or Backoff for Ollama

**Severity: Medium**

The `OllamaProvider.chat()` method has no rate limiting, retry logic, or exponential backoff. If Ollama is under load (e.g., during evolution with multiple variants), requests can time out after 600s with no retry. The QuantConnect MCP client has proper retry logic (3 retries with exponential backoff) but the Ollama provider does not.

**Recommendation:** Add retry with exponential backoff to `OllamaProvider.chat()`, similar to `_call_api()` in `quantconnect_mcp.py`.

---

## 4. QuantConnect-Specific Concerns

### 4.1 Indicator Signature Knowledge Is Prompt-Only

The system's knowledge of QuantConnect indicator signatures exists exclusively in LLM prompts. The exact signatures are duplicated across `generate_qc_code`, `generate_qc_framework`, and `fix_runtime_error`:

```python
# Repeated in 3 different prompts
"- self.rsi(symbol, period, moving_average_type, resolution) -> 4 args"
"- self.atr(symbol, period, moving_average_type, resolution) -> 4 args"
```

This approach has two problems:
1. **No enforcement** — the LLM may ignore the signature guidance, producing 3-arg `rsi()` calls that the linter can't detect
2. **Maintenance burden** — if QuantConnect updates its API, three separate prompts must be manually updated

**Recommendation:** Create a machine-readable indicator registry (JSON/TOML) that:
- Is used to generate the prompt text dynamically
- Is used by the linter to validate argument counts in generated code
- Can be updated independently of the prompts

### 4.2 Limited Asset Class Support

The linter's asset class detection (QC009) handles equities, forex, and crypto, but the code generation prompts are overwhelmingly equity-focused:

```python
# From generate_qc_code system prompt
"- Add securities with self.add_equity()"
```

Papers covering futures, options, CFDs, or bonds will produce incorrect code that calls `add_equity()` for non-equity instruments.

**Recommendation:** 
1. Add asset class detection to the summarization pipeline
2. Use asset-class-specific prompt templates
3. Extend the linter to detect and fix more asset class mismatches

### 4.3 No Use of QuantConnect's Algorithm Framework

The core pipeline generates monolithic single-file algorithms (all logic in `initialize` + `on_data`). This misses QuantConnect's Algorithm Framework — `AlphaModel`, `RiskManagementModel`, `PortfolioConstructionModel`, `ExecutionModel`, and `UniverseSelectionModel` — which provides:

- **Better separation of concerns** — alpha generation, risk management, portfolio construction, and execution are independent
- **Built-in optimization** — Framework models can be swapped without code changes
- **Better backtesting** — Framework algorithms receive proper Insight-based event flow

The multi-agent system (`agents/`) generates Framework-style code, but as noted in §3.2, this path lacks the core pipeline's quality controls.

**Recommendation:** Add a configurable option to generate Framework-style vs. monolithic algorithms, with the full pipeline quality controls applied to both.

### 4.4 Hardcoded Backtest Date Overrides

The `BacktestTool._override_dates()` method uses regex to replace `set_start_date` / `set_end_date` calls with CLI-provided dates. This works for the common case but fails if:
- The algorithm uses `datetime()` objects instead of integer args
- The date is set via a variable (`start = datetime(2020, 1, 1)`)
- Multiple date-setting patterns exist in the same file

**Recommendation:** Use AST transformation instead of regex for date overrides.

---

## 5. Code Quality Assessment

### 5.1 Positive Patterns

| Pattern | Assessment |
|---------|-----------|
| **Dataclasses for config** | Clean, type-hinted, with sensible defaults |
| **Structured logging** | Consistent `logging.getLogger("quantcoder.ClassName")` pattern throughout |
| **Abstract base classes** | `BaseAgent` and `LLMProvider` ABCs enforce interface contracts |
| **Factory pattern** | `LLMFactory.create()` cleanly maps tasks to models |
| **Fallback chains** | MinerU → pdfplumber, two-pass → legacy summary, Stage 1+2 → single-shot |
| **Static analysis tooling** | ruff, mypy, black, pytest all configured in `pyproject.toml` |

### 5.2 Negative Patterns

| Pattern | Location | Issue |
|---------|----------|-------|
| **God file** | `cli.py` (1933 lines) | Mixes UI, business logic, tool orchestration, formatting, and Notion publishing. Should be decomposed |
| **Duplicated code extraction** | `_extract_code()` in `BaseAgent`, `_strip_markdown()` in `LLMHandler`, `_extract_code()` in `VariationGenerator` | Three separate implementations of the same markdown-stripping logic |
| **Duplicated metric parsing** | `_parse_pct()` / `_parse_float()` in `BacktestTool`, `QCEvaluator`, and `cli.py` | Same parsing logic implemented three times |
| **Mixed sync/async** | `LLMHandler` methods are sync wrappers around async calls | Creates thread proliferation and connection pool issues (§3.4) |
| **No dependency injection** | `LLMHandler(config)` creates its own providers | Prevents testing and makes the code harder to compose |
| **Broad exception catching** | `except Exception as e` throughout | Catches everything including `KeyboardInterrupt`, making debugging difficult |
| **No type annotations on dicts** | Return types like `-> Dict` without value types | `Dict[str, Any]` used pervasively — no TypedDict or Pydantic models |

### 5.3 Test Coverage Observations

The test suite (18 test files) covers a broad range of functionality, but notable gaps include:
- **No end-to-end pipeline tests** — no test exercises the full PDF → code path with a real PDF
- **No linter regression tests with real LLM output** — the linter tests use synthetic patterns
- **No multi-agent integration tests** — the coordinator is tested in isolation from the actual agents
- **Heavy mocking** — most tests mock the LLM layer, which means the prompts themselves are never validated

---

## 6. Areas of Improvement for Future Implementation Plans

### 6.1 Research Paper Interpretation Fidelity

**Priority: Critical**

This is the system's core value proposition and its weakest point. The README itself acknowledges: *"local LLMs (14-32B) consistently fail to implement non-trivial math faithfully, substituting simple indicator proxies instead."*

**Proposed improvements:**

1. **Equation-Aware Extraction** — When MinerU extracts LaTeX equations, tag them separately and include them as structured inputs to Stage 2. Currently, LaTeX is embedded in free text and may be ignored by the coding LLM

2. **Reference Implementation Database** — Build a curated database of verified QuantConnect implementations of common mathematical models (OU process, HMM, Kalman filter, regime-switching). Use these as few-shot examples in Stage 2 prompts, selected by model type detected in Pass 2

3. **Model Classification Step** — Before code generation, classify the paper's mathematical model into categories (standard indicators, statistical models, ML models, stochastic processes). Use this classification to select the appropriate generation template and examples

4. **AST-Level Verification** — After generation, parse the code AST and verify that the expected mathematical constructs are present (e.g., if the summary mentions "OU process", check for mean-reversion parameter estimation in the AST)

### 6.2 Unified Code Generation Pipeline

**Priority: High**

Merge the core pipeline and multi-agent paths into a single pipeline that:

1. Uses the core pipeline's prompt quality and validation stack
2. Supports both monolithic and Framework-style output
3. Routes through the same linting and fidelity assessment regardless of generation mode
4. Shares a single indicator registry and QC API knowledge base

### 6.3 Dynamic Context Window Management

**Priority: High**

The `OllamaProvider` already queries the model's context window (`_query_context_length`), but this information is never used to manage prompt sizes. The `_format_sections_for_prompt` method uses a hardcoded `max_chars=60000`, which may overflow 32K contexts or underutilize 128K contexts.

**Proposed improvements:**

1. Estimate token count from character count (rough: chars ÷ 4)
2. Reserve a proportion of the context window for the response (`max_tokens`)
3. Dynamically set `max_chars` based on: `(context_window - max_tokens - system_prompt_tokens) * 4`

### 6.4 Structured Output Contracts

**Priority: High**

Replace free-text LLM responses with structured output contracts:

1. **Summary output** — Define a JSON schema for the strategy specification (indicators, entry/exit rules, risk parameters). Parse and validate against the schema after generation
2. **Fidelity output** — The current regex parsing of `FAITHFUL/SCORE/ISSUES/CORRECTION_PLAN` is fragile. Use JSON-mode generation (supported by Ollama)
3. **Code output** — Use Ollama's structured output feature to ensure the response is always valid Python (no markdown fences, no explanatory text)

### 6.5 Backtest Feedback Loop

**Priority: Medium-High**

Currently, backtest results are displayed to the user but never fed back into the generation pipeline. A strategy that generates a Sharpe ratio of -2.0 receives no automatic improvement.

**Proposed improvements:**

1. **Backtest-Driven Refinement** — If the backtest Sharpe is below a threshold, feed the backtest statistics (drawdown, trade frequency, return distribution) back to the LLM with instructions to diagnose and fix the strategy logic
2. **Error Pattern Learning** — The autonomous pipeline has `AutonomousPipeline._apply_learned_fixes`, but this only tracks error strings, not root causes. Build a structured error taxonomy with verified fixes
3. **Evolution Integration** — The evolver already uses backtest results for fitness, but it operates independently of the core pipeline. Integrate evolution as a post-generation optimization step in the main pipeline

### 6.6 QuantConnect API Knowledge Base

**Priority: Medium**

Replace the hardcoded prompt knowledge with a queryable QuantConnect API knowledge base:

1. **Indicator Registry** — Complete catalogue of indicators with exact Python signatures, argument types, and usage examples
2. **Asset Class Templates** — Per-asset-class code templates (equities, forex, crypto, futures, options)
3. **Common Pattern Library** — Verified code snippets for scheduling, risk management, position sizing, universe selection
4. **Error Solution Database** — Common QuantConnect compilation and runtime errors with known fixes

This knowledge base would be used for:
- Dynamic prompt construction (only include relevant indicators/patterns)
- Post-generation linting (verify indicator usage against registry)
- Error fixing (look up known solutions before asking the LLM)

### 6.7 Paper-to-Code Traceability

**Priority: Medium**

Add traceability from the original paper to the generated code:

1. **Annotated Code** — Add comments linking each code section to the specific paper passage it implements
2. **Parameter Provenance** — For each numeric parameter in the code, record whether it came from the paper, was a default, or was LLM-generated
3. **Assumption Log** — Record where the LLM made implementation choices not specified by the paper

### 6.8 CLI Decomposition

**Priority: Medium**

Break `cli.py` (1933 lines) into focused modules:

```
cli/
├── __init__.py        # Click group registration
├── search.py          # search, download commands
├── summarize.py       # summarize, list-summaries
├── generate.py        # generate, validate, backtest
├── autonomous.py      # auto start/status/report
├── library.py         # library build/status/export
├── evolve.py          # evolve start/list/export
├── scheduler.py       # scheduler start/stop/status
└── config_cmd.py      # config set/show
```

---

## 7. Summary of Recommendations by Priority

| Priority | Issue | Section |
|----------|-------|---------|
| **Critical** | Increase `max_tokens` for code generation (3000 → 8192+) | §3.1 |
| **Critical** | Unify multi-agent and core pipeline paths | §3.2, §6.2 |
| **Critical** | Improve research paper interpretation fidelity | §6.1 |
| **High** | Add semantic validation of generated code | §3.3 |
| **High** | Fix async/sync architecture in LLMHandler | §3.4 |
| **High** | Implement dynamic context window management | §6.3 |
| **High** | Use structured output contracts | §6.4 |
| **Medium-High** | Add backtest feedback loop to generation pipeline | §6.5 |
| **Medium-High** | Add Ollama retry/backoff | §3.5 |
| **Medium** | Build QuantConnect API knowledge base | §6.6 |
| **Medium** | Add paper-to-code traceability | §6.7 |
| **Medium** | Decompose `cli.py` | §6.8 |
| **Medium** | Create machine-readable indicator registry | §4.1 |
| **Medium** | Support more asset classes in prompts | §4.2 |
| **Lower** | Deduplicate code extraction / metric parsing helpers | §5.2 |
| **Lower** | Replace broad `except Exception` with specific catches | §5.2 |
| **Lower** | Add TypedDict / Pydantic models for return types | §5.2 |

---

## 8. Conclusion

QuantCoder v2.0.0's architecture makes the right bets: two-stage generation, cross-model fidelity assessment, and static QC API linting are exactly the techniques needed for this problem domain. The primary limitation is not architectural but parametric — the `max_tokens` cap, the lack of a structured QuantConnect knowledge base, and the disconnection between the multi-agent and core pipeline paths reduce the system's effective capability below its architectural potential.

The most impactful improvements would be:
1. **Raising the token ceiling** for code generation tasks (immediate fix, high impact)
2. **Building a reference implementation database** for common mathematical models (reduces LLM substitution errors)
3. **Unifying the generation paths** to ensure all generated code benefits from linting and fidelity assessment
4. **Adding structured output contracts** to eliminate parsing fragility

These changes would bring the system significantly closer to its goal of faithfully translating academic research into production-quality QuantConnect algorithms.
