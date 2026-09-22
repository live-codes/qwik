# Taste

- Prefers to de-risk new feature work with a minimal, self-contained proof-of-concept (a single standalone HTML file that runs with no build/install/server) before wiring it into the full codebase. Confidence: 0.6
- When a PoC has proved the happy path but has known gaps, prefers to extend the throwaway PoC to cover them (e.g. multi-file support) rather than move on to integrating into the real codebase. Confidence: 0.55
- Develops features on dedicated GitHub branches with a PR, and shares the branch/PR links as the reference implementation, expecting the agent to read the real code (checked out locally) and align with it rather than analyze from the outside or reimplement in parallel. Confidence: 0.5
- Keeps experimental work strictly isolated: explicitly says not to write to the real project yet and to leave it as a PoC, so integrations/branch edits should wait for an explicit go-ahead. Confidence: 0.7
- When understanding changes, wants the PoC's README rewritten wholesale against the new findings (including correcting earlier wrong claims) rather than left stale or patched piecemeal. Confidence: 0.5
