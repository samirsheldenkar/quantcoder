# QuantConnect API Knowledge Base — Repository Design Plan

## Goal

Design a standalone, machine-readable QuantConnect API knowledge base repository (`qc-knowledge-base`) that can be consumed by any code generation system (not just QuantCoder) to produce correct QuantConnect LEAN Python algorithms. The knowledge base addresses the four pillars defined in the codebase review §6.6:

1. **Indicator Registry** — Complete catalogue with exact Python signatures
2. **Asset Class Templates** — Per-asset-class code templates
3. **Common Pattern Library** — Verified code snippets for scheduling, risk, sizing, etc.
4. **Error Solution Database** — Common errors with known fixes

> [!IMPORTANT]
> This repository has **zero dependencies** on the QuantCoder codebase. It is a pure data repository with a validation harness, designed to be consumed by any LLM-powered code generation tool via JSON/TOML files.

---

## Repository Structure

```
qc-knowledge-base/
├── README.md
├── LICENSE
├── pyproject.toml                   # Validation tooling only
├── schema/                          # JSON Schema definitions
│   ├── indicator.schema.json
│   ├── asset_class.schema.json
│   ├── pattern.schema.json
│   └── error.schema.json
│
├── indicators/                      # Pillar 1: Indicator Registry
│   ├── _index.json                  # Master list with summary metadata
│   ├── momentum/
│   │   ├── rsi.json
│   │   ├── macd.json
│   │   ├── momp.json
│   │   ├── roc.json
│   │   └── stochastic.json
│   ├── trend/
│   │   ├── sma.json
│   │   ├── ema.json
│   │   ├── adx.json
│   │   ├── ichimoku.json
│   │   └── supertrend.json
│   ├── volatility/
│   │   ├── atr.json
│   │   ├── bb.json
│   │   ├── keltner.json
│   │   └── donchian.json
│   ├── volume/
│   │   ├── obv.json
│   │   ├── vwap.json
│   │   ├── ad.json
│   │   └── mfi.json
│   └── custom/
│       ├── python_indicator.json    # PythonIndicator base class template
│       └── rolling_window.json      # RollingWindow usage patterns
│
├── asset_classes/                   # Pillar 2: Asset Class Templates
│   ├── equity.json
│   ├── forex.json
│   ├── crypto.json
│   ├── futures.json
│   ├── options.json
│   └── cfd.json
│
├── patterns/                        # Pillar 3: Common Pattern Library
│   ├── _index.json
│   ├── algorithm_structure/
│   │   ├── monolithic.py            # Single-file QCAlgorithm
│   │   └── framework.py            # Framework with AlphaModel etc.
│   ├── scheduling/
│   │   ├── daily_rebalance.py
│   │   ├── weekly_rebalance.py
│   │   ├── intraday_close.py
│   │   └── custom_schedule.py
│   ├── risk_management/
│   │   ├── fixed_stop_loss.py
│   │   ├── trailing_stop.py
│   │   ├── atr_stop.py
│   │   ├── max_drawdown.py
│   │   └── volatility_sizing.py
│   ├── position_sizing/
│   │   ├── equal_weight.py
│   │   ├── percent_portfolio.py
│   │   ├── kelly_criterion.py
│   │   └── volatility_parity.py
│   ├── universe_selection/
│   │   ├── coarse_fine.py
│   │   ├── manual_universe.py
│   │   ├── etf_constituents.py
│   │   └── sector_filter.py
│   ├── data_access/
│   │   ├── history_dataframe.py
│   │   ├── on_data_slice.py
│   │   ├── consolidator.py
│   │   └── warm_up.py
│   ├── mathematical_models/
│   │   ├── ou_process.py            # Ornstein-Uhlenbeck
│   │   ├── kalman_filter.py
│   │   ├── hmm_regime.py            # Hidden Markov Model
│   │   ├── pairs_trading.py
│   │   └── mean_reversion.py
│   └── order_management/
│       ├── market_order.py
│       ├── limit_order.py
│       ├── stop_market_order.py
│       └── bracket_order.py
│
├── errors/                          # Pillar 4: Error Solution Database
│   ├── _index.json
│   ├── compilation/
│   │   ├── missing_import.json
│   │   ├── wrong_casing.json
│   │   ├── indicator_arity.json
│   │   ├── undefined_symbol.json
│   │   └── syntax_errors.json
│   ├── runtime/
│   │   ├── indicator_not_ready.json
│   │   ├── trading_during_warmup.json
│   │   ├── key_not_found.json
│   │   ├── division_by_zero.json
│   │   ├── history_empty.json
│   │   └── asset_class_mismatch.json
│   └── logic/
│       ├── wrong_order_direction.json
│       ├── position_never_closes.json
│       └── indicator_substitution.json
│
├── api_reference/                   # Supplementary: Core API methods
│   ├── qc_algorithm.json            # QCAlgorithm base class methods
│   ├── resolution.json              # Resolution enum values
│   ├── order_types.json             # OrderType enum
│   ├── moving_average_types.json    # MovingAverageType enum
│   └── insight.json                 # Insight class (Framework)
│
├── scripts/                         # Validation & build tooling
│   ├── validate_schemas.py          # Validate all JSON against schemas
│   ├── validate_snippets.py         # AST-parse all .py snippets
│   ├── build_prompt_context.py      # Generate prompt text from KB
│   └── build_linter_rules.py        # Generate linter rules from KB
│
└── tests/
    ├── test_schema_validation.py
    ├── test_snippet_syntax.py
    └── test_build_outputs.py
```

---

## Pillar 1: Indicator Registry

### JSON Schema (`schema/indicator.schema.json`)

Each indicator file follows this schema:

```json
{
  "id": "rsi",
  "name": "Relative Strength Index",
  "category": "momentum",
  "qc_class": "RelativeStrengthIndex",
  "helper_method": "self.rsi",
  "signature": {
    "parameters": [
      {"name": "symbol", "type": "Symbol", "required": true},
      {"name": "period", "type": "int", "required": true},
      {"name": "moving_average_type", "type": "MovingAverageType", "required": false, "default": "MovingAverageType.WILDERS"},
      {"name": "resolution", "type": "Resolution", "required": false, "default": null}
    ],
    "returns": "RelativeStrengthIndex",
    "min_args": 2,
    "max_args": 4
  },
  "properties": [
    {"name": "current.value", "type": "float", "description": "Current RSI value (0-100)"},
    {"name": "is_ready", "type": "bool", "description": "True when indicator has enough data"}
  ],
  "warmup_period": "period + 1",
  "value_range": {"min": 0, "max": 100},
  "common_thresholds": {
    "oversold": 30,
    "overbought": 70
  },
  "common_mistakes": [
    {
      "id": "rsi_missing_ma_type",
      "description": "Calling self.rsi(symbol, 14) with only 2 args when MovingAverageType is intended",
      "wrong": "self.rsi(symbol, 14)",
      "correct": "self.rsi(symbol, 14, MovingAverageType.WILDERS, Resolution.DAILY)",
      "severity": "warning"
    }
  ],
  "usage_example": "self._rsi = self.rsi(symbol, 14, MovingAverageType.WILDERS, Resolution.DAILY)\n# In on_data:\nif self._rsi.is_ready and self._rsi.current.value < 30:\n    self.set_holdings(symbol, 1.0)",
  "csharp_aliases": ["RSI", "Rsi"],
  "related_indicators": ["stochastic", "williams_percent_r"]
}
```

### Indicator Coverage

The registry should cover **all 100+ QC indicators**, organized by category:

| Category | Indicators (examples) | Count |
|----------|----------------------|-------|
| Momentum | RSI, MACD, Stochastic, MOM, ROC, Williams %R, CCI | ~15 |
| Trend | SMA, EMA, ADX, Ichimoku, DEMA, TEMA, KAMA, SuperTrend, Aroon | ~15 |
| Volatility | ATR, BB, Keltner, Donchian, Standard Deviation | ~8 |
| Volume | OBV, VWAP, AD, MFI, CMF | ~8 |
| Oscillators | PPO, APO, Ultimate, Trix | ~6 |
| Candlestick | Doji, Hammer, Engulfing, etc. | ~40+ |
| Custom | PythonIndicator, IndicatorBase, RollingWindow | 3 |

### Key Design Decisions

1. **`min_args` / `max_args`** — Enables linters to validate argument counts without parsing the full signature
2. **`common_mistakes`** — Each indicator carries its own LLM-specific error patterns, enabling targeted prompting
3. **`csharp_aliases`** — Maps PascalCase C# names to snake_case Python names for linter auto-correction
4. **`warmup_period`** — Expressed as a formula so consumers can compute the required warm-up dynamically

---

## Pillar 2: Asset Class Templates

### JSON Schema (`schema/asset_class.schema.json`)

```json
{
  "id": "forex",
  "name": "Forex",
  "add_method": "self.add_forex",
  "symbol_format": "EURUSD",
  "symbol_examples": ["EURUSD", "GBPJPY", "USDJPY", "AUDUSD"],
  "detection_patterns": [
    {"pattern": "^[A-Z]{3}/[A-Z]{3}$", "description": "Slash-separated pair"},
    {"pattern": "^[A-Z]{6}$", "description": "Concatenated 6-char pair", "condition": "both halves are ISO 4217 currency codes"}
  ],
  "default_resolution": "Resolution.MINUTE",
  "available_resolutions": ["Resolution.TICK", "Resolution.SECOND", "Resolution.MINUTE", "Resolution.HOUR", "Resolution.DAILY"],
  "market": "Market.OANDA",
  "add_method_signature": {
    "parameters": [
      {"name": "ticker", "type": "str", "required": true},
      {"name": "resolution", "type": "Resolution", "required": false, "default": "Resolution.MINUTE"},
      {"name": "market", "type": "str", "required": false, "default": "Market.OANDA"},
      {"name": "leverage", "type": "float", "required": false, "default": 50.0}
    ]
  },
  "order_methods": ["self.market_order", "self.limit_order", "self.set_holdings"],
  "data_properties": {
    "bid": "data[symbol].bid.close",
    "ask": "data[symbol].ask.close",
    "price": "data[symbol].price"
  },
  "common_mistakes": [
    {
      "id": "forex_add_equity",
      "description": "Using add_equity() for forex pairs",
      "wrong": "self.add_equity(\"EURUSD\")",
      "correct": "self.add_forex(\"EURUSD\")",
      "severity": "error"
    }
  ],
  "template_snippet": "# --- Forex Example ---\nfrom AlgorithmImports import *\n\nclass ForexAlgorithm(QCAlgorithm):\n    def initialize(self):\n        self.set_start_date(2022, 1, 1)\n        self.set_cash(100000)\n        self.pair = self.add_forex(\"EURUSD\", Resolution.HOUR, Market.OANDA)\n        self.symbol = self.pair.symbol\n        self._sma = self.sma(self.symbol, 20, Resolution.HOUR)\n\n    def on_data(self, data):\n        if not self._sma.is_ready:\n            return\n        price = data[self.symbol].price\n        if price > self._sma.current.value:\n            self.set_holdings(self.symbol, 1.0)\n        else:\n            self.liquidate(self.symbol)\n"
}
```

### Design Decision: Ticker Detection

Each asset class definition includes `detection_patterns` — regex patterns with optional conditions that allow consuming tools (like QuantCoder's linter) to automatically classify tickers and select the correct `add_*` method. This directly addresses the linter rule QC009's hardcoded logic.

---

## Pillar 3: Common Pattern Library

### Structure

Patterns are **verified Python files** (not JSON) that can be directly used as few-shot examples in LLM prompts. Each pattern file includes:

1. A structured docstring header with metadata
2. Complete, compilable QuantConnect code
3. Inline comments explaining every design decision

### Pattern File Format

```python
"""
Pattern: ATR-Based Trailing Stop
Category: risk_management
Tags: [stop_loss, trailing, atr, volatility]
Asset Classes: [equity, forex, crypto]
Complexity: intermediate
QC Features Used: [self.atr, self.schedule.on, self.liquidate, RollingWindow]
Description: |
    Implements a trailing stop-loss that adapts to market volatility
    using the Average True Range (ATR) indicator. The stop trails
    behind the highest price since entry by a configurable ATR multiple.
"""
from AlgorithmImports import *

class AtrTrailingStopAlgorithm(QCAlgorithm):
    def initialize(self):
        self.set_start_date(2022, 1, 1)
        self.set_end_date(2024, 1, 1)
        self.set_cash(100000)

        equity = self.add_equity("SPY", Resolution.DAILY)
        self.symbol = equity.symbol

        # ATR with 14-period lookback, Wilders smoothing
        self._atr = self.atr(self.symbol, 14, MovingAverageType.WILDERS, Resolution.DAILY)
        self._sma = self.sma(self.symbol, 50, Resolution.DAILY)

        # Trailing stop state
        self._highest_since_entry = 0.0
        self._stop_price = 0.0
        self._atr_multiple = 2.0  # Stop distance = 2x ATR

        self.set_warm_up(50)

    def on_data(self, data):
        if self.is_warming_up:
            return
        if not (self._atr.is_ready and self._sma.is_ready):
            return
        if not data.contains_key(self.symbol):
            return

        price = data[self.symbol].close

        if self.portfolio[self.symbol].invested:
            # Update trailing stop
            self._highest_since_entry = max(self._highest_since_entry, price)
            self._stop_price = self._highest_since_entry - (self._atr.current.value * self._atr_multiple)

            if price <= self._stop_price:
                self.liquidate(self.symbol, "Trailing stop hit")
                self._highest_since_entry = 0.0
        else:
            # Entry: go long when price above SMA
            if price > self._sma.current.value:
                self.set_holdings(self.symbol, 1.0)
                self._highest_since_entry = price
                self._stop_price = price - (self._atr.current.value * self._atr_multiple)
```

### Pattern Index (`patterns/_index.json`)

```json
{
  "patterns": [
    {
      "id": "atr_trailing_stop",
      "file": "risk_management/atr_stop.py",
      "category": "risk_management",
      "tags": ["stop_loss", "trailing", "atr", "volatility"],
      "complexity": "intermediate",
      "asset_classes": ["equity", "forex", "crypto"],
      "qc_features": ["self.atr", "self.schedule.on", "self.liquidate"],
      "description": "ATR-based trailing stop that adapts to volatility"
    }
  ]
}
```

### Mathematical Model Patterns

The `patterns/mathematical_models/` directory is critical for addressing QuantCoder's core weakness — LLM substitution of complex models with simple indicators. Each file is a **verified, working QuantConnect implementation** of a mathematical model commonly found in research papers:

| Model | File | Purpose |
|-------|------|---------|
| Ornstein-Uhlenbeck | `ou_process.py` | Mean-reversion with calibrated OU parameters |
| Kalman Filter | `kalman_filter.py` | Kalman-filtered spread for pairs trading |
| HMM Regime Detection | `hmm_regime.py` | Hidden Markov regime switching |
| Pairs Trading | `pairs_trading.py` | Cointegration-based pairs with z-score entry |
| Mean Reversion | `mean_reversion.py` | Half-life estimation + Hurst exponent |

These serve as few-shot examples for Stage 2 of QuantCoder's pipeline — when the coding LLM needs to implement a novel mathematical model, the system can retrieve the most relevant pattern as context.

---

## Pillar 4: Error Solution Database

### JSON Schema (`schema/error.schema.json`)

```json
{
  "id": "indicator_not_ready",
  "category": "runtime",
  "error_pattern": "indicator.*not.*ready|is_ready.*False|NoneType.*current",
  "error_message_examples": [
    "RuntimeError: Indicator has not been warmed up",
    "'NoneType' object has no attribute 'value'"
  ],
  "root_cause": "Accessing indicator.current.value before the indicator has received enough data points to compute a valid value.",
  "detection_rules": [
    {
      "type": "ast_pattern",
      "description": "Accessing .current.value without checking .is_ready",
      "pattern": "indicator_access_without_ready_check"
    }
  ],
  "solutions": [
    {
      "id": "guard_is_ready",
      "description": "Add is_ready guard before accessing value",
      "wrong_code": "value = self._rsi.current.value\nif value < 30:\n    self.set_holdings(symbol, 1.0)",
      "correct_code": "if not self._rsi.is_ready:\n    return\nvalue = self._rsi.current.value\nif value < 30:\n    self.set_holdings(symbol, 1.0)",
      "auto_fixable": true,
      "fix_strategy": "Insert is_ready check before first .current.value access in on_data"
    },
    {
      "id": "add_warmup",
      "description": "Add warm-up period to initialize()",
      "correct_code": "self.set_warm_up(50)  # Set warm-up to cover longest indicator period",
      "auto_fixable": true,
      "fix_strategy": "Add set_warm_up() call in initialize() with max(indicator_periods) days"
    }
  ],
  "related_errors": ["trading_during_warmup", "history_empty"],
  "severity": "error",
  "frequency": "very_common",
  "tags": ["indicator", "warmup", "initialization"]
}
```

### Error Coverage

| Category | Error ID | Frequency | Auto-Fixable |
|----------|----------|-----------|-------------|
| **Compilation** | `missing_import` | Very Common | ✅ |
| | `wrong_casing` | Very Common | ✅ |
| | `indicator_arity` | Common | ✅ |
| | `undefined_symbol` | Common | ❌ |
| | `syntax_errors` | Common | ❌ |
| **Runtime** | `indicator_not_ready` | Very Common | ✅ |
| | `trading_during_warmup` | Common | ✅ |
| | `key_not_found` | Common | ✅ |
| | `division_by_zero` | Occasional | ❌ |
| | `history_empty` | Common | ✅ |
| | `asset_class_mismatch` | Common | ✅ |
| **Logic** | `wrong_order_direction` | Common | ❌ |
| | `position_never_closes` | Common | ❌ |
| | `indicator_substitution` | Common | ❌ |

---

## Build Scripts

### `scripts/build_prompt_context.py`

Generates human-readable prompt text from the knowledge base for use by any LLM code generator:

```
Usage: python scripts/build_prompt_context.py --indicators rsi,macd,atr --asset-class equity --output prompt.txt
```

Output is a formatted string suitable for injection into an LLM system prompt, containing:
- Exact indicator signatures with argument types
- Asset class setup boilerplate
- Relevant common mistakes to avoid
- Selected pattern examples as few-shot context

### `scripts/build_linter_rules.py`

Generates linter rule definitions from the knowledge base:

```
Usage: python scripts/build_linter_rules.py --output rules.json
```

Output is a JSON file containing:
- Indicator name → snake_case method name mapping (for QC001-style rules)
- Indicator arity constraints (for arg-count validation)
- Ticker → asset class classification patterns (for QC009-style rules)
- C# → Python API method mappings

### `scripts/validate_schemas.py`

Validates all JSON files against their respective JSON schemas.

### `scripts/validate_snippets.py`

AST-parses all `.py` files in `patterns/` to verify they are syntactically valid Python. Optionally checks for QC-specific patterns (imports, class inheritance).

---

## Verification Plan

### Automated Tests

All tests are run via `pytest`:

```bash
# Install dev dependencies
pip install -e ".[dev]"

# Run all tests
pytest tests/ -v

# Run specific test suites
pytest tests/test_schema_validation.py -v     # All JSON validates against schemas
pytest tests/test_snippet_syntax.py -v        # All .py files parse without SyntaxError
pytest tests/test_build_outputs.py -v         # Build scripts produce valid output
```

**Test coverage:**

1. **`test_schema_validation.py`** — Loads every JSON file in `indicators/`, `asset_classes/`, `patterns/`, `errors/`, and `api_reference/`, validates it against the corresponding JSON schema in `schema/`. Fails if any file doesn't conform.

2. **`test_snippet_syntax.py`** — Uses `ast.parse()` on every `.py` file in `patterns/` to verify syntax validity. Also checks that each pattern file contains `from AlgorithmImports import *` and a class inheriting from `QCAlgorithm`.

3. **`test_build_outputs.py`** — Runs `build_prompt_context.py` and `build_linter_rules.py` and validates their output format: prompt text is non-empty and contains indicator signatures; linter rules JSON is valid and covers all indicators.

### Manual Verification

1. **Indicator accuracy** — Cross-reference 10 randomly selected indicator signatures against the official QuantConnect documentation at `https://www.quantconnect.com/docs/v2/writing-algorithms/indicators/supported-indicators/` to confirm arg counts and types are correct.

2. **Pattern compilation** — Upload 3 pattern files (`atr_stop.py`, `pairs_trading.py`, `framework.py`) to QuantConnect cloud and verify they compile without errors.

---

## Implementation Phases

| Phase | Scope | Est. Time |
|-------|-------|-----------|
| **Phase 1** | Repository scaffold, JSON schemas, 20 core indicators (RSI, SMA, EMA, MACD, ATR, BB, ADX, MOMP, ROC, Stochastic, OBV, VWAP, MFI, CCI, Williams%R, Aroon, DEMA, TEMA, KAMA, Ichimoku) | Small |
| **Phase 2** | All 6 asset class templates, 10 core patterns (monolithic, framework, daily rebalance, fixed stop, trailing stop, ATR stop, equal weight, percent portfolio, history dataframe, on_data slice) | Small |
| **Phase 3** | Error database (all 13 errors above), API reference enums, build scripts | Small |
| **Phase 4** | Mathematical model patterns (OU, Kalman, HMM, pairs, mean-reversion), remaining 80+ indicators, remaining patterns | Medium |
| **Phase 5** | CI/CD with automated schema + syntax validation on every PR | Small |
