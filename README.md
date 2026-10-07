# pyLoad: code execution through Js2Py in the Click'n'Load endpoint (CVE-2023-0297)

Case study of a code-injection advisory (CWE-94) in [pyload/pyload](https://github.com/pyload/pyload): a
Click'n'Load form field is evaluated as JavaScript by the Js2Py library; the upstream fix calls
`js2py.disable_pyimport()` when the module is imported.

> **Demo only, not for use.** This repository is a case study built from unmodified snapshots of
> [pyload/pyload](https://github.com/pyload/pyload) at two historical commits. It is not a fork and is not maintained. Do not install, deploy, or
> depend on it: the `baseline` branch contains code affected by the advisories below.

**This branch: `fix`** (tag `pone`): snapshot of `pyload/pyload` @ [`7d73ba7919`](https://github.com/pyload/pyload/commit/7d73ba7919e594d783b3411d7ddb87885aea782d) (2023-01-03), the upstream fix commit. The vulnerable snapshot is on [`baseline`](https://github.com/ferralon-demo/pyload-js2py-rce/tree/baseline).

## Advisory

| | |
|---|---|
| **CVE** | [CVE-2023-0297](https://nvd.nist.gov/vuln/detail/CVE-2023-0297) |
| **GHSA** | [GHSA-pf38-5p22-x6h6](https://github.com/advisories/GHSA-pf38-5p22-x6h6), "Code Injection in pyload-ng", CWE-94 |
| **Affected** | `pyload-ng` < 0.5.0b3.dev31, per the GHSA record |
| **Upstream fix** | commit [`7d73ba7919`](https://github.com/pyload/pyload/commit/7d73ba7919e594d783b3411d7ddb87885aea782d), "fix arbitrary python code execution by abusing js2py functionality", 2023-01-03 (pushed directly; no pull request) |

## Ingress and sink at `baseline`

All line numbers are at upstream commit `6aa71cd99a`.

| | Location |
|---|---|
| **Ingress** | Flask blueprint route `@bp.route("/flash/addcrypted2", methods=["POST"])` on handler `addcrypted2`, [`src/pyload/webui/app/blueprints/cnl_blueprint.py:113-115`](https://github.com/pyload/pyload/blob/6aa71cd99a6f510de7137d17ec99f930916704c0/src/pyload/webui/app/blueprints/cnl_blueprint.py#L113-L115); form field `jk` read at [line 120](https://github.com/pyload/pyload/blob/6aa71cd99a6f510de7137d17ec99f930916704c0/src/pyload/webui/app/blueprints/cnl_blueprint.py#L120) and passed to `eval_js(f"{jk} f()")` at [line 124](https://github.com/pyload/pyload/blob/6aa71cd99a6f510de7137d17ec99f930916704c0/src/pyload/webui/app/blueprints/cnl_blueprint.py#L124) |
| **Guard on the route** | decorator `local_check` ([lines 22-36](https://github.com/pyload/pyload/blob/6aa71cd99a6f510de7137d17ec99f930916704c0/src/pyload/webui/app/blueprints/cnl_blueprint.py#L22-L36)) admits the request when `REMOTE_ADDR` is a loopback address or the `Host` header is `127.0.0.1:9666` or `[::1]:9666` |
| **Wrapper** | first-party `eval_js(script, es6=False)`, [`src/pyload/core/utils/misc.py:25-27`](https://github.com/pyload/pyload/blob/6aa71cd99a6f510de7137d17ec99f930916704c0/src/pyload/core/utils/misc.py#L25-L27) |
| **Sink** | `(js2py.eval_js6 if es6 else js2py.eval_js)(script)` at [`misc.py:27`](https://github.com/pyload/pyload/blob/6aa71cd99a6f510de7137d17ec99f930916704c0/src/pyload/core/utils/misc.py#L27): a call into the Js2Py dependency |

The first-party wrapper and the dependency function are both named `eval_js`.

## What the fix changes

At `7d73ba7919`, `src/pyload/core/utils/misc.py` calls `js2py.disable_pyimport()` at module level
([line 8](https://github.com/pyload/pyload/blob/7d73ba7919e594d783b3411d7ddb87885aea782d/src/pyload/core/utils/misc.py#L8))
and drops two unused imports. The route, the wrapper and the call into Js2Py are all unchanged; the fix changes
Js2Py's configuration, not the call path.

## Provenance

This is not a fork: a fresh repository whose code branches hold upstream source snapshots, with no upstream
history and no upstream remote.

| Branch | Tag | Upstream commit | Date | Snapshot commit | Snapshot tree |
|---|---|---|---|---|---|
| `baseline` | `poe` | [`6aa71cd99a`](https://github.com/pyload/pyload/commit/6aa71cd99a6f510de7137d17ec99f930916704c0) | 2022-12-11 | `f6f518a5be9b5f0f19c694bb2aed5c15eeaa102c` | `210c38eeeb0a3bef006841b6338d76cade8dcef6` |
| `fix` | `pone` | [`7d73ba7919`](https://github.com/pyload/pyload/commit/7d73ba7919e594d783b3411d7ddb87885aea782d) | 2023-01-03 | `bfee6cd3da32a30d3ba2847a3aaf5cf51c687d15` | `196f4cb288021a36f2d48b0b02e488f51a345e19` |

Each snapshot commit is the upstream tree at that commit with one change: `.github/workflows/` (4 files) is
removed so that upstream CI does not run here. No other file is added, removed, or edited. The `fix` snapshot's
parent is the `baseline` snapshot, so the diff between the two snapshot commits
([compare](https://github.com/ferralon-demo/pyload-js2py-rce/compare/f6f518a5be9b5f0f19c694bb2aed5c15eeaa102c...bfee6cd3da32a30d3ba2847a3aaf5cf51c687d15)) is exactly the upstream fix commit's diff. On each code branch, a second
commit replaces upstream's `README.md` with a branch-specific version of this file; upstream's README is in the snapshot commit.

To check a snapshot against upstream from a clone of this repository:

```sh
git fetch https://github.com/pyload/pyload 6aa71cd99a6f510de7137d17ec99f930916704c0
git diff --stat FETCH_HEAD f6f518a5be9b5f0f19c694bb2aed5c15eeaa102c   # lists only .github/workflows/*
```

## Licence

pyLoad is licensed under the GNU Affero General Public License v3 (`LICENSE.md`, kept verbatim at the root of the
`baseline` and `fix` branches; upstream ships no `NOTICE` file). This repository is a modified copy within the
meaning of AGPL-3.0 section 5: on 2026-10-07, `.github/workflows/` was removed and this `README.md` replaced
upstream's. Nothing else was changed. The complete corresponding source is this repository; the original is at
https://github.com/pyload/pyload.
