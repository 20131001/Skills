# Eval Runner Notes

Use this directory for skill evaluation prompts and outputs.

Recommended flow:

1. Create or select benchmark cases under `evals/benchmarks/`.
2. Run the current skill against each benchmark codebase.
3. Save the resulting report and timing notes next to the benchmark.
4. Compare results against the expected findings in `compare.md`.

Keep benchmark prompts realistic: ask for the same audit modes users actually run, such as default repo scans, focused file scans, mixed FunC/Tolk/Tact scopes, and `deep` reviews of message-heavy contracts.
