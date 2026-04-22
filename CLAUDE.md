# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

TradingAgents-CN is a Chinese-language multi-agent A-share stock analysis skill for the OpenClaw Agent platform. It implements a 12-step serial workflow that produces professional PDF reports with multi-perspective analysis, bull/bear debates, and risk management consensus.

## Key Commands

```bash
# Initialize report.json for a stock
python3 scripts/validate_step.py --init '{"stock_code":"000729","stock_name":"燕京啤酒","current_price":5.50,"news_data":{"news_list":[]}}' --stock-code 000729

# Validate + save LLM output for a step (pipe LLM output via stdin)
echo '<json>' | python3 scripts/validate_step.py --step tech --stock-code 000729 --save

# Debate round 2 requires --round flag
echo '<json>' | python3 scripts/validate_step.py --step bull_debate --stock-code 000729 --round 2 --save

# Get default value for a step (useful for testing)
python3 scripts/validate_step.py --step tech --default

# Generate PDF from completed report.json
python3 scripts/generate_report.py --input scripts/results/000729_report.json
python3 scripts/generate_report.py --input scripts/results/000729_report.json --output-dir ./custom_reports
```

Set log file path before running:
```bash
export TRADINGAGENTS_LOG_FILE="scripts/logs/000729_20260422_120000.log"
```

## Architecture

### Workflow (12 steps, strictly serial)

```
User Input → Data Extraction → News Search → Tech/Fundamentals/News Analysts
→ Bull/Bear Debate (2 rounds) → Research Manager → Trader Plan
→ Risk Debate (Aggressive/Conservative/Neutral) → Portfolio Manager → PDF
```

All steps run in the main agent process. No subagents. Each step depends on the previous step's output written to `report.json`.

### Data Pipeline

Every LLM output is piped through `validate_step.py --save`, which:
1. Extracts JSON from LLM output (handles markdown code blocks, escaped newlines)
2. Validates required fields for that step
3. On success (`exit 0`): writes to `scripts/results/{stock_code}_report.json` and prints JSON to stdout
4. On failure (`exit 1`): prints error JSON with retry hint to stderr — agent retries up to 2-3 times, then falls back to default

### report.json Structure

```json
{
  "stock_code": "000729", "stock_name": "燕京啤酒", "current_price": 5.50,
  "news_data": { "news_list": [...] },
  "parallel_analysis": {
    "tech_analyst": { "report": "...", "key_points": [...] },
    "fundamentals_analyst": { "report": "...", "key_points": [...] },
    "news_analyst": { "report": "...", "key_points": [...] }
  },
  "investment_debate": {
    "bull_r1": { "debate_text": "...", "core_logic": "...", "bull_case": [...], "confidence": 0.8 },
    "bear_r1": { "debate_text": "...", "core_logic": "...", "bear_case": [...], "confidence": 0.7 },
    "bull_r2": { ... }, "bear_r2": { ... }
  },
  "manager_decision": { "recommendation": "买入", "rationale": "...", "investment_plan": "..." },
  "trading_plan": { "decision": "买入", "buy_price": 5.2, "target_price": 6.0, "stop_loss": 4.9 },
  "risk_debate": {
    "aggressive": { "debate_text": "...", "stance": "...", "position_size": "30%-40%", "key_points": [...] },
    "conservative": { "debate_text": "...", "stance": "...", "position_size": "5%-10%", "key_points": [...] },
    "neutral": { "debate_text": "...", "stance": "...", "position_size": "15%-20%", "key_points": [...] }
  },
  "final_decision": { "rating": "买入", "executive_summary": "...", "investment_thesis": "...", "risk_level": "中" }
}
```

### Key Design Decisions

- **English JSON keys, Chinese display**: All keys in `report.json` are English for LLM output stability. `pdf_generator.py` maps them to Chinese via `KEY_CN_MAP` (84 entries) for rendering.
- **Retry on validation failure**: `validate_step.py` exits 1 with a structured hint; the agent retries with the hint injected. After max retries, `--default` provides a safe fallback.
- **Trader price validation**: If `decision` is `买入`, `buy_price`/`target_price`/`stop_loss` must be numeric — special-cased in `validate_trader()`.
- **Valid final ratings**: 买入 / 增持 / 持有 / 减持 / 卖出

### Key Files

| File | Purpose |
|------|---------|
| `SKILL.md` | Complete 12-step workflow definition — the agent's operating instructions |
| `scripts/validate_step.py` | JSON validation + report.json writer (625 lines) |
| `scripts/pdf_generator.py` | HTML→PDF rendering engine with `KEY_CN_MAP` |
| `scripts/generate_report.py` | PDF generation entry point |
| `references/*_prompt.md` | Role-specific system prompts for each analyst/debater/manager |

### Output Artifacts

- `scripts/results/{stock_code}_report.json` — intermediate analysis data
- `scripts/reports/{stock_code}_{name}_{timestamp}.pdf` — final PDF report
- `scripts/logs/{stock_code}_{timestamp}.log` — execution log (JSON lines)
