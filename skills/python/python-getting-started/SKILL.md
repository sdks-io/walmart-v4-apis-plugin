---
name: "python-getting-started"
description: "Load before answering any Walmart APIs Python SDK question or writing Python code against the SDK (Python only). The SDK's lookup layer: install, imports, environments and base URL, auth, the SDK map (`sdk-map.md` + `map/operations/`) and how to read it, and the module that owns each fact the map leaves to the source."
---

# Getting started with the Walmart APIs Python SDK

> **Lookup layer** for Walmart APIs Python SDK code: ground every contract fact here, in the SDK map and in the source modules they name — never in recall — and carry those facts onto a contract sheet before you implement. `python-integrate-walmart-apis` holds the workflow around it.

This page owns the SDK's *identity and surface* (install, imports, controllers, which module owns which fact). The API-agnostic companion skills own *usage* — `python-client-initialization`, `python-authentication`, `python-calling-endpoints`, `python-models`, `python-error-handling`, `python-configuration-resilience` and `python-testing`; load each at its step (see **Integration workflow**) and confirm names against the installed package.

## SDK identity

| Fact | Value |
| --- | --- |
| API | Walmart APIs |
| Distribution name (what you install) | `wm-v-4-apis-sdk` |
| Import root (what you import) | `walmart_apis` — not the string you install |
| Source repository | https://github.com/sdks-io/wm-v-4-apis-python-sdk |
| Source branch | `main` |
| Version | `v4` — re-verify this page after a version bump |
| Sync client class | `WalmartApisClient` (alias `Client`) |
| Async client class | `AsyncWalmartApisClient` (alias `AsyncClient`) |
| Client construction | **keyword-only**: `environment` · `timeout` (default `30.0`) · `server_config` · `retry_options` · `logging_options` · `seller_auth_authorization_code` · `seller_auth_authorization_code_token_source` · `seller_auth_client_credentials` · `seller_auth_client_credentials_token_source` · `basic_client_auth` · `wallet_auth` · `wallet_auth_token_source`, and the transport override — `custom_http_client` on the sync client, `custom_async_http_client` on the async one (the names differ) |
| Auth | **OAuth 2.0** authorization code — set `seller_auth_authorization_code` · **OAuth 2.0** client credentials — set `seller_auth_client_credentials` · HTTP **Basic** — set `basic_client_auth` · **OAuth 2.0** client credentials — set `wallet_auth` |
| Environments | 3 environments (default `"production"`) × 2 named servers, through `server_config` |
| Base-URL config | `ServerConfig` (`walmart_apis/server/server_config.py`) |
| Python floor | **`>=3.10`** (classifiers list `3.10–3.14`) |
| Runtime dependencies | `httpx2 (>=2.13.1,<3.0.0)` · `certifi (>=2024.7.4)` · `pydantic[email] (>=2.11.0,<3.0.0)` · `typing-extensions (>=4.13.0,<5.0.0)` |
| Typing | ships `py.typed` — **a type error against this SDK is a real contract violation, not noise** |
| Surface | 174 operations across 30 controllers · 410 models · 0 unions · 119 enums · 174 per-operation error unions |

## Install

Into the same environment your project runs in, install the distribution from its package index:

```bash
pip install wm-v-4-apis-sdk
python -c "import walmart_apis, pathlib; print(pathlib.Path(walmart_apis.__file__).parent)"
```

**Run the second line too.** A `pip install` that reports success is not proof the SDK is importable where your project runs. If the probe raises `ModuleNotFoundError`, the install went into a different interpreter: re-run it through the one your project uses (`python -m pip install …`) and probe again. If it prints a path whose contents are not this SDK, what you installed is not this distribution.

An index name can belong to an unrelated project — install it from its own repository instead, at <https://github.com/sdks-io/wm-v-4-apis-python-sdk>:

```bash
pip install "wm-v-4-apis-sdk @ git+https://github.com/sdks-io/wm-v-4-apis-python-sdk.git@main"
```

Install it as a package, into site-packages. Do not vendor its source into your project, add its directory to `sys.path`, or install it editable (`-e`) from a throwaway clone — an editable install points at that directory, so deleting the clone breaks every import.

## Imports — the package splits its surface across four modules

Python does not re-export child modules transitively, so `from walmart_apis import models` alone does **not** make enums, error unions, or runtime types reachable. Import each kind of type from the module that owns it.

`walmart_apis/__init__.py` exports exactly 8 names beside the `walmart_apis.models` subpackage it re-exports:

```python
from walmart_apis import (
    AsyncClient,
    AsyncWalmartApisClient,
    Client,
    Environment,
    ServerConfig,
    ServerConfigDict,
    ServerConfigOrDict,
    WalmartApisClient,
)
```

Everything else comes from its own subpackage:

| What you need | Where it lives |
| --- | --- |
| Domain models, their `…Dict` companions | `walmart_apis.models` |
| Enums (and their open `…OrStr`/`…OrInt` aliases) | `walmart_apis.models.enums` |
| `ApiError` · `RawError` · `ApiResult` · `RequestOptions` · `AuthorizationCodeCredentials` · `ClientCredentials` · `BasicAuthCredentials` · `HttpClient` · `SdkBaseModel` · `UNSET` · `Optional` · `FileInput` · `AsyncFileInput` · `NamedFile` · `BinaryContent` · `AsyncBinaryContent` · `BinaryReader` · `AsyncBinaryReader` · `FileResponse` · `AsyncFileResponse` | `walmart_apis.core` |
| Per-operation error *unions* | `walmart_apis.errors` (`CreateWarrantyErrorBody`, …) |

`walmart_apis.core` re-exports its whole public surface, so import from `…core` rather than from the private modules beneath it (`…core.results`, `…core.exceptions`, `…core.auth.schemes`).

## Environments and servers

This SDK has 2 named servers across 3 environments, so the constructor takes `environment` first, then `timeout`, then `server_config: ServerConfigOrDict | None = None`. The environment selects which set of base URLs the config resolves against:

| `environment=` | Hosting |
| --- | --- |
| `"production"` *(default)* | Production |
| `"environment2"` | Sandbox |
| `"environment3"` | mock |

Each of the 2 servers resolves its own base URL per environment, so a call's host is the pair (server, environment) — `sdk-map.md` **Servers & auth** has the full server × environment URL matrix and each server's override point.

Consequences to state on every contract sheet that touches configuration:

- Omitting `environment` gives you **`"production"`**, silently.
- `server_config` overrides individual server URLs within the selected environment.
- The token endpoint (`/auth/v4/token`) resolves against the `default` server rather than a client-wide base URL, so `server_config` is what moves token traffic, and it follows `environment` with that server.
- The token endpoint (`/auth/token`) resolves against the `default1` server rather than a client-wide base URL, so `server_config` is what moves token traffic, and it follows `environment` with that server.
- `ServerConfig` is a frozen pydantic model with `extra="forbid"`: a misspelled keyword raises `ValidationError` at construction rather than being ignored.
- `timeout` is validated too — the base client raises `ValueError` for any non-positive value.

## Auth pattern (four schemes)

OAuth 2.0 authorization code, exposed as the client's `seller_auth_authorization_code=` keyword taking `AuthorizationCodeCredentials` **or a plain dict**. The client fetches and caches the bearer token itself, lazily, from `/auth/v4/token` on the `default` server.

```python
from walmart_apis import Client
from walmart_apis.core import AuthorizationCodeCredentials

def prompt(url: str) -> str:
    return input(f"Open {url}, then paste the code: ")

client = Client(seller_auth_authorization_code=AuthorizationCodeCredentials(client_id="…", client_secret="…", redirect_uri="…", prompt_for_authorization_code=prompt))
client = Client(seller_auth_authorization_code={"client_id": "…", "client_secret": "…", "redirect_uri": "…", "prompt_for_authorization_code": prompt})   # equivalent
```

OAuth 2.0 client credentials, exposed as the client's `seller_auth_client_credentials=` keyword taking `ClientCredentials` **or a plain dict**. The client fetches and caches the bearer token itself, lazily, from `/auth/v4/token` on the `default` server.

```python
from walmart_apis import Client
from walmart_apis.core import ClientCredentials

client = Client(seller_auth_client_credentials=ClientCredentials(client_id="…", client_secret="…"))
client = Client(seller_auth_client_credentials={"client_id": "…", "client_secret": "…"})   # equivalent
```

HTTP Basic authentication, exposed as the client's `basic_client_auth=` keyword taking `BasicAuthCredentials` **or a plain dict**.

```python
from walmart_apis import Client
from walmart_apis.core import BasicAuthCredentials

client = Client(basic_client_auth=BasicAuthCredentials(username="…", password="…"))
client = Client(basic_client_auth={"username": "…", "password": "…"})   # equivalent
```

OAuth 2.0 client credentials, exposed as the client's `wallet_auth=` keyword taking `ClientCredentials` **or a plain dict**. The client fetches and caches the bearer token itself, lazily, from `/auth/token` on the `default1` server.

```python
from walmart_apis import Client
from walmart_apis.core import ClientCredentials

client = Client(wallet_auth=ClientCredentials(client_id="…", client_secret="…"))
client = Client(wallet_auth={"client_id": "…", "client_secret": "…"})   # equivalent
```

**Every credentials keyword is optional** — omit it and requests go out unauthenticated, with nothing failing at construction; `python-authentication` covers how to catch that, and what a *failed token fetch* raises.

## Controllers

30 controllers, reached as `client.<attr>`. Every controller has an `Async…` peer whose operations are identical in name and parameters and differ solely by being awaited — state that once on the sheet rather than a row per async operation. `sdk-map.md`'s **Operations — by controller** table lists each controller's attribute, classes and operation count, and links its page (`map/operations/<controller>.md`), one block per operation.

## SDK map — look up first, open the module second

The SDK ships a generated map at its **root** — the directory holding `pyproject.toml`, the `walmart_apis/` source directory, and these two entries:

- **`sdk-map.md`** — the index: constructor keywords, the error model (`ApiError` / `ApiResult` / `RawError`, Case A vs Case B), where models, enums and error aliases live, servers and auth, and links to the operations pages.
- **`map/operations/<controller>.md`** — one `###` block per operation: verb and route, signature, parameter roles and wire names, both return types, error arms with the status each maps from, and a **Type sources** table naming the module of every *generated* type it mentions.

**Installing the distribution does not give you the map.** `pyproject.toml` ships the `walmart_apis/` package only, so `sdk-map.md` and `map/operations/` are absent from the installed package — they live in the SDK's own source tree, at the root of its repository, <https://github.com/sdks-io/wm-v-4-apis-python-sdk>, in lockstep with the code the map describes — clone it to a temporary directory, outside the project repo:

```bash
git clone --depth 1 --branch main https://github.com/sdks-io/wm-v-4-apis-python-sdk
```

Confirm the map comes from the release you installed before you read anything out of it: the clone above is pinned to `main`, the branch this SDK is released from — the repository's default branch may carry a different version. A map read from any other release describes code you do not have.

Every `Source` path on the map is relative to that SDK root, so `walmart_apis/models/access_point.py` opens as written from there — and the same path resolves inside the installed package, which is where you read a module's body once the map has named it.

**The map is the locator; the source modules are the shapes.** Read the map first — an operation's signature, route, parameter roles and return types, the client's constructor keywords, the `timeout` default, the members of `ApiError` / `Success` / `Failure` / `RawError`, each operation's error union with the status each arm maps from, base-URL and auth wiring, and which module declares a type are all answered there; don't open a module for them. Open the one module the map names only for what it leaves out: a model's members, an enum's values, a field's wire alias. The map says so itself — *"Shapes live only in the source … Never grep for a type."*

**Meaning is not on the map.** When what to pass depends on semantics — values a field accepts beyond its type, a rule that couples two fields, what a defaulted parameter selects — read the operation's docstring in `walmart_apis/apis/<controller>.py` (or `client.py` where operations sit on the client) and its entry in `api-reference.md` at the SDK root before writing the sheet row, and record what you found — even for a value you think you know.

`sdk-map.md` carries the invariants every operation block assumes, so load it before any `map/operations/` page; the pages are written to be read beside it.

## Contract facts — the map first, then the module

For everything the map leaves out, and the full body behind a map row, read the one module that owns the fact **inside the installed package** — locate it with the probe from *Install*. Failing that, it is under the project's environment (`.venv/Lib/site-packages/walmart_apis` on Windows, `.venv/lib/python3.*/site-packages/walmart_apis` elsewhere). **If the package is not installed, install it first** — see *Install* above; until then mark the fact `UNVERIFIED` and say what would settle it rather than answering from memory. Paths below are relative to that package root:

| Question | Module |
| --- | --- |
| An operation's real signature, parameters and return type | `apis/<controller>.py` |
| Client construction, keywords, controller wiring | `client.py`, `async_client.py`, `base_client.py` |
| Timeout default and validation | `base_client.py` (`DEFAULT_TIMEOUT = 30.0`) |
| The request/response pipeline, 401 handling, 2xx-vs-error split | `core/raw_client.py` |
| Exception shape (`ApiError.error`, `.status_code`, `.headers`) | `core/exceptions.py` |
| `Success`/`Failure`/`RawError` | `core/results.py` |
| Per-call overrides | `core/request_options.py` |
| Logging options, levels and masking | `core/logging.py` |
| `UNSET`, `Optional`, `OptionalNullable` | `core/optionality.py` |
| Model base config, `to_dict`/`to_json` | `core/models.py` |
| A model's members, required vs `UNSET`, wire aliases | `models/<model_name>.py` |
| An enum's members and wire values | `models/enums/` |
| Open-enum coercion | `core/converters/open_enum.py` |
| Date/time wire formats — `Date`, `RFC3339DateTime`, `RFC1123DateTime`, `UnixSecondsDateTime` (`Annotated` aliases over `datetime.date` / `datetime.datetime`; not in the map's Type sources) | `core/converters/date_time.py` |
| What a file or binary parameter accepts, and the reader protocols behind it (the aliases are in the import table above; not in the map's Type sources) | `core/files.py` |
| Consuming a downloaded body — `FileResponse` / `AsyncFileResponse`, and what closes it (not in the map's Type sources) | `core/file_responses.py` |
| Transport protocols (the test seam) | `core/transport.py` |
| httpx2 adapter, proxy/TLS knobs | `core/httpx2_transport.py` |
| Token fetch, credential placement | `core/auth/`, `core/auth/models.py` |
| Base-URL resolution | `server/server_config.py`, `server/server.py` |
| An operation's error mapper (status → schema) | `errors/<operation>_error.py` |

**Read scoped.** These modules carry long design docstrings; `grep -n` for the symbol and read the surrounding lines rather than whole files. Never quote a docstring's design rationale onto a contract sheet — the sheet carries facts an implementer must obey, not the reasoning behind them.

Keep lookups cheap:

- Collect the contracts for **every** in-scope operation in **one** pass — signature, required members with wire aliases, the error union, enum values — onto the contract sheet, then implement from it; never re-open a module for a fact the sheet carries.
- Expand a model's members only where the task sets them.
- If a name here fails to type-check or import, re-read the module the table names and report the drift; never patch around it from memory.

## Integration workflow — load the companion skill at each step

Before you write the code for each step, load the named companion skill — even if you have already read the relevant module; it owns the traps the signature hides. A typical integration reaches them in this order:

1. **Client construction & lifetime** — load **python-client-initialization** before you write `Client(...)` or `AsyncClient(...)`.
2. **Authentication** — load **python-authentication** before you set credentials.
3. **Calling an endpoint** — load **python-calling-endpoints** before the first `client.<controller>.<operation>(...)` call. A multipart body is **one flat parameter list** — every field, JSON part and file is its own parameter, and an optional part is omitted with `None` rather than by a second call shape. Load **python-file-handling** before you pass a file position anything but a `Path`.
4. **Models** — load **python-models** the moment a request/response member is not a plain string or number.
5. **Error handling** — load **python-error-handling** before you write any `try/except`.
6. **Configuration & resilience** — load **python-configuration-resilience** when you set the base URL, timeouts, retries, proxies, TLS, or logging.
7. **Testing** — load **python-testing** before you stub the SDK.

## What a contract sheet must carry for this SDK

Beyond the usual signatures and model members, a Python sheet is incomplete without these, because each one is a decision the implementer cannot make correctly from the signature alone:

1. **Sync or async** — the client class, where it is held, and its `close()`/`aclose()`.
2. **The keyword-only boundary** for each operation: what is positional (each required parameter without a fixed value, path → query → header → body) and what sits after `*`. Every keyword-only parameter has a real default, so write no defensive `None`s.
3. **The file positions** — `feeds.create_feed`, each annotated `FileInput` on the sync client and `AsyncFileInput` on the async one, with `list[…]` for an array part. What those admit is wider than the annotation shows and **never a `str`**, and the arm you pick decides what stays resident, who closes your handle and what the file is called on the wire — so the sheet cites **python-file-handling** for the row rather than restating it.
4. **The 4 operations that return `None`** — the map's **Returns (parsed)** bullet says which. Their raw peers are `ApiResult[None, …]`, so `with_raw_response` is the only way to observe the status code.
5. **The 4 operations that return a file** — `feeds.get_feed_document` · `reconciliation.download_reconciliation_report` · `reports.download_new_report` · `reports.download_old_report`. The payload is `FileResponse` (`AsyncFileResponse` on the async client) with its body unread and its connection still held, so the sheet names the consuming call and where it is closed:
   1. `save_to(...)` · `read()` · a drained `iter_bytes()` — each consumes and closes. Getting it wrong leaks a connection rather than raising, so the row cites **python-file-handling** for the lifecycle: the async spellings, the single-use rule, and the warning an abandoned response emits.
   2. The raw peer observes the status and headers only — a `Success` carries no body of its own, because on a download the body *is* the payload.
6. **Required vs `UNSET`** for every model member the task sets.
7. **The `ApiError.error` union** for each operation in scope, copied from the **Error arms** bullet of its block in `map/operations/` — this SDK has **7 typed error bodies** across 174 operations, so never assume an arm is shared. Every union ends in `RawError`; an operation with no typed arm is `RawError` alone. An arm that is not a class (`Any`, `list[...]`, `dict[...]`) is not an `isinstance` target: narrow the class arms first, then branch on `e.status_code` for what remains.
8. **That a decode failure raises `ValidationError`/`ValueError`, not `ApiError`, in both response modes. And that the 2xx path declares at least one required member on 64 return types, so a truncated body fails to decode there and passes silently everywhere else.** Any sheet row for a call whose result is used must name the members the implementer has to assert on.
9. **Retries are on by default** — the sheet says whether this integration keeps, tunes (`retry_options`) or disables them (`max_retries` set to `0`), and for which methods; a write is retried only when the policy names its method.
10. **Whether logging is on, and how loud** — the sheet says whether this integration raises the `walmart_apis` logger, which `logging_options` dumps it turns on — a JSON body is logged as sent — and any per-call `log_level`.
11. **What limits a whole call.** `timeout` limits only each wait (`python-configuration-resilience`). In async code, state the budget and the one place `asyncio.wait_for` enforces it for every call, and what the boundary returns for its `asyncio.TimeoutError`; in sync code, say there is none.
12. **Which environment the `environment` keyword selects**, because omitting it is silently `"production"`.
13. A **REQUIRED READING** block naming the `python-*` companions that govern the steps, with `MUST load` pointers.

