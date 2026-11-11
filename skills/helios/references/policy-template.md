# Helios approval policy

<!-- Written by `/helios setup`. Edit by hand or re-run setup. One `key: value` per line. -->

## Tenant and run

host: <HELIOS_HOST>
approve_mode: <auto | confirm_batch | review_only>
report_language: <中文 | English>
carryover: <reuse | re-review>
minor_issues: <flag | approve_and_note>

## Company thresholds (USD)

link_any_usd: <number>
link_contract_usd: <number>
ceo_approval_usd: <number>
kol_flag_usd: <number>

## Strictness

cpv_outlier_multiple: <number | off>
views_above_median: <flag | ignore>
unsigned_docusign: <flag | allow>
fx_tolerance_pct: <number | off>

## Cross-document

split_window_days: <number | off>
duplicate_lookback_months: <number>
