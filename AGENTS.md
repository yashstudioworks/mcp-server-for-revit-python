# Revit work with Codex

Read `.revit-codex.local.md`, when present, for the installed extension path,
Python executable, Revit version, and selected Routes port on this machine.

## Connect and verify

- Use stdio for Codex. The CPython MCP server runs with locked dependencies:
  `uv run --frozen main.py`. See `docs/codex-setup.md` for client configuration.
- Connect to `127.0.0.1:48884` by default through `REVIT_HOST` and `REVIT_PORT`.
  Select the port belonging to the intended Revit instance; keep it on loopback.
- Enable pyRevit Routes and pair with **MCP > Bridge > Connect** in Revit.
  Keep the pairing secret encrypted in the Windows profile, out of logs and Git.
- Call `get_revit_status`, then inspect the model and active view before changing
  elements. A healthy status requires an open document; MCP tool discovery alone
  does not verify the Revit connection. Confirm the intended document.
- Perform only the requested model operations. Respect Revit's payload review
  prompts for model changes and Python execution.

## Write reliable Revit code

- `main.py` and `tools/` run in CPython outside Revit. `revit_mcp/`, `startup.py`,
  ribbon commands, and `execute_revit_code` run inside Revit under IronPython.
  `revit_mcp/pairing.py` is shared by both engines; keep it compatible with both.
- Match the active IronPython engine. For IronPython 2.7, use `.format()` instead
  of f-strings and avoid Python 3 syntax and CPython-only packages.
- Use the active document and existing repository patterns. Check that each
  element exists, is valid, and is editable; check parameter storage types and
  `IsReadOnly` before setting values. Account for worksharing ownership.
- Resolve .NET overloads with explicit types and typed collections. Convert
  lengths using the installed Revit API's unit conversion methods.
- Filter `FilteredElementCollector` by class, category, view, or IDs before
  iterating. Avoid collecting the whole model for a narrow request.
- Wrap edits in an appropriate named Revit transaction. Commit on success and
  roll back on failure; account for any transaction already owned by a wrapper.
  Read-only queries need no transaction.
- Use MCP for one-off tasks. When asked to make a toolbar button, follow the
  existing `.tab/<name>.panel/<name>.pushbutton/` conventions and put shared
  behavior in a module.

## Search references

`LLM.txt` is this repository's Revit/pyRevit integration guide. Downloaded
`llms.txt` and `llms-full.txt` are pyRevit's documentation index and full reference.
Search these files with `rg -n` for the classes and methods needed, then read
relevant excerpts. Keep large references out of the conversation's initial load.

## Repository changes

Inspect status and unstaged/staged diffs first; preserve unrelated work. Create a
descriptive `feature/`, `fix/`, `docs/`, or `chore/` branch before editing. Validate
the change, review the staged diff, and use a focused Conventional Commit. Push a
review branch and open a PR describing the problem, behavior, scope, validation,
and material limitations. Merge or deploy only when explicitly requested.
