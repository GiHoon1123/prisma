---
from: "8.0.0-rc.4"
to: "8.0.0-rc.5"
changes:
  - id: distinct-on-requires-postgres-capability
    summary: |
      `Collection#distinctOn(...)` now requires the contract to declare the `postgres.distinctOn`
      capability, mirroring the sql-builder lane's existing gate. A contract without it (e.g.
      SQLite-only) makes the call a compile error — the parameter type narrows to `never` — where
      it used to compile and silently produce undeduped rows at runtime, since the target's
      renderer never rendered `DISTINCT ON` for a target that cannot express it.

      Every `.distinctOn(...)` call your code makes on a `Collection` was already wrong on any
      target lacking `postgres.distinctOn`; the type error is the fix surfacing at compile time
      instead of a silently wrong result set at runtime. Move the call to a contract that
      declares `postgres.distinctOn`, or remove it — there is no runtime opt-out.
    detection:
      glob: "**/*.{ts,mts,cts}"
      regex:
        - '\.distinctOn\('
      anyMatch: true
---

# 8.0.0-rc.4 → 8.0.0-rc.5 — Extension-author upgrade instructions

## `distinct-on-requires-postgres-capability`

`Collection#distinctOn(...)` used to compile and run on any target, but only Postgres ever
rendered its `DISTINCT ON` clause — a call on any other target (SQLite) compiled clean and
silently returned undeduped rows at runtime. The method now carries the same capability gate the
sql-builder lane already enforces: its parameter type narrows to `never` unless the contract
declares `postgres.distinctOn`, so the same call is a compile error on a contract that lacks it,
and a runtime error carrying `ORM.CAPABILITY_MISSING` if reached dynamically (e.g. through a
hand-built `CollectionState`).

Find every `.distinctOn(...)` call your code makes on a `Collection` and check whether the
contract it runs against declares `postgres.distinctOn`. If it does, nothing changes — the call
already worked correctly and keeps compiling. If it does not, the call was already producing the
wrong result set; either move the collection onto a Postgres-capable contract, or remove the
`.distinctOn(...)` call and accept the undeduped rows it was silently returning before.

`Collection#distinct(...)` is unaffected — it lowers to a portable `ROW_NUMBER` dedup and needs
no capability, on any target.
