# Portfolio — Personal Multi-Asset Portfolio Tracker

All project documentation is in the `planning` directory.

The key document is PLAN.md, included in full below. Nothing has been built yet. Start at Section 13 (Build Order) — the endpoint verification spike runs first, because one price source is still unverified and its outcome shapes the provider layer.

Two things that are never negotiable and are easy to get wrong: the accounting currency is VND and money is never floating-point (Section 9), and the ledger is the only writable source of truth — positions, NAV, and P&L are always derived (Section 3).

@planning/PLAN.md
