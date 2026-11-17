# ZeroCode CLI — Upstream Sync Inventory

**Scope:** Part A / Phase 0 only. No product source or test was changed for this inventory.

## Known defect candidate: named-user tilde expansion

- **Package:** `packages/core`
- **Test:** `packages/core/test/shell-parse.test.ts`
- **Case:** `ShellParse.scan("cd ~root/secret && cd '~root'", "/bin/bash", "/workspace")`
- **Expected directory:** `/root/secret`
- **Received directory:** a temporary test-home path, observed as `/tmp/oc-3I2GqA/secret`
- **Classification:** Known upstream behavior defect / bug candidate in the shell parser.
- **Evidence:** [`reports/baseline/core-failing-rerun-no-color.txt`](../reports/baseline/core-failing-rerun-no-color.txt), lines 41–70.

The controlled rerun produced this assertion difference:

```text
expect(received).toEqual(expected)

[
-   "/root/secret",
+   "/tmp/oc-3I2GqA/secret",
    "~root",
]

at packages/core/test/shell-parse.test.ts:164:36
```

The failure remains after Git identity, default branch, and forced-color environment variables were controlled. The 13 unrelated Git/session/snapshot/VCS failures disappear when Git color is disabled; this shell-parser failure does not. It was **not fixed in Part A**. Before M1, decide whether to carry it as an explicit upstream patch/exception or require an upstream fix; do not silently alter the baseline test or implementation.
