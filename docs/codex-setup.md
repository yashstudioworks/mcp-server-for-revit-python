# Connect Codex to pyRevit MCP

Codex can run the same MCP server as Claude over stdio. Configure the installed
extension, so the client and Revit use the same revision. The pyRevit Routes
listener is the downstream Revit bridge; it is not an HTTP MCP endpoint.

## Prepare the installed extension

From the extension directory, install its locked dependencies with
`uv sync --frozen`. Alternatively, use an existing compatible Python environment
containing the pinned dependencies in `pyproject.toml` and `uv.lock`.

In `%APPDATA%\pyRevit\pyRevit_config.ini`, preserve existing settings and set:

```ini
[routes]
enabled = true
host = "127.0.0.1"
port = 48884
```

Reload pyRevit or restart Revit if its listener uses a different address. Multiple
Revit instances can receive different ports; select the intended instance.
These are the keys used by the current typed pyRevit configuration schema;
older tutorials may show `server_host` and `server_port`. Check the installed
schema before reusing those names.
Open a model and choose **MCP > Bridge > Connect** to pair the Windows user.
Pairing remains encrypted in the local Windows profile. The stdio client does
not need a token in its configuration.

## Register the server in Codex

Add the following to `%USERPROFILE%\.codex\config.toml`, replacing each
`C:\path\to\mcp-server-for-revit-python.extension` with the installed extension
directory. Preserve other server entries and settings.

```toml
[mcp_servers.revit]
command = 'C:\path\to\mcp-server-for-revit-python.extension\.venv\Scripts\python.exe'
args = ['C:\path\to\mcp-server-for-revit-python.extension\main.py']
cwd = 'C:\path\to\mcp-server-for-revit-python.extension'
startup_timeout_sec = 60
tool_timeout_sec = 120
enabled = true

[mcp_servers.revit.env]
REVIT_HOST = "127.0.0.1"
REVIT_PORT = "48884"
```

This uses the prepared Python environment directly, avoiding a package download
at MCP startup. If that environment is rebuilt, check the executable again.
The alternative CLI registration is:

```powershell
codex mcp add revit --env REVIT_HOST=127.0.0.1 --env REVIT_PORT=48884 -- uv run --frozen --directory 'C:\path\to\mcp-server-for-revit-python.extension' main.py
```

Use one registration method. Restart the Codex client or start a new session
after registration. Check `codex mcp get revit` and `/mcp`, then ask Codex to call
`get_revit_status`, `get_revit_model_info`, and `get_current_view_info`. Tool
discovery verifies the MCP process; successful read-only calls verify the bridge
to the open Revit document. Check the document title before requesting changes.

If the bridge reports pairing is missing, use **Connect** in Revit. If it reports
no active document, open a model. For a connection error, check the selected
Routes port, enabled extension, and listener address. Preserve Revit's separate
review prompts for model changes and arbitrary Python execution.

### Paired requests still return HTTP 401

Some pyRevit builds construct the route `Request` with an empty header dictionary
and omit forwarding the incoming HTTP headers. This breaks pairing even when the
client and Revit can decrypt the same secret. In an affected installation, inspect
`pyrevitlib/pyrevit/routes/server/server.py`, method
`HttpRequestHandler._prepare_request`.

The parser must preserve the incoming headers on the returned request:

```diff
-        return base.Request(
+        request = base.Request(
             path=path,
             method=method,
             data=data,
             params=router.extract_route_params(route.pattern, path),
             query_params=query_params,
         )
+        for key, value in self.headers.items():
+            request.add_header(key, value)
+        return request
```

Check the exact installed source before applying this change. Back it up first;
an installation under Program Files requires administrator access. Restart
Revit afterward so the framework's cached parser is replaced, and verify a paired
status request succeeds while an unpaired request and a paired request with an
`Origin` header are denied. Retain the
extension's existing pairing and in-Revit review checks. A pyRevit update can
replace this local correction, so recheck the parser after updating.

## Keep references for future sessions

From this checkout, download the pyRevit references (ignored by Git):

```powershell
curl.exe --fail --location https://docs.pyrevitlabs.io/llms.txt --output llms.txt
curl.exe --fail --location https://docs.pyrevitlabs.io/llms-full.txt --output llms-full.txt
```

`AGENTS.md` contains Codex's connection workflow, IronPython conventions, safe
transaction rules, and reference search guidance. Use the ignored
`.revit-codex.local.md` for machine-specific paths and verified Revit details.

Sources: [BIM Pure tutorial](https://www.bimpure.com/blog/pyrevit-mcp-claude-code-tutorial)
and [official Codex MCP configuration](https://learn.chatgpt.com/docs/extend/mcp?surface=cli).
