---
name: "dotnet-authentication"
description: "Load before wiring credentials into the Walmart APIs .NET SDK client in C#, or when a call comes back 401/403. Covers HTTP Basic, OAuth 2.0 client credentials and the OAuth 2.0 authorization-code grant: the credential each scheme takes, the WalmartApIsClientOptions property it goes on, and a startup check that fails fast on a missing secret."
---

# Authenticating the Walmart APIs .NET SDK client

This API authenticates with HTTP Basic, OAuth 2.0 client credentials and the OAuth 2.0 authorization-code grant. Each scheme is a **nullable credentials property** on `WalmartApIsClientOptions`; set the ones below, then construct the client (see `dotnet-client-initialization`).

| Scheme | Property on `WalmartApIsClientOptions` | Credential |
| --- | --- | --- |
| HTTP Basic | `BasicClientAuth` | `BasicAuthCredentials` |
| OAuth 2.0 client credentials | `SellerAuthClientCredentials` | `OAuth2ClientCredentials` |
| OAuth 2.0 client credentials | `WalletAuth` | `OAuth2ClientCredentials` |
| OAuth 2.0 authorization code | `SellerAuthAuthorizationCode` | `OAuth2AuthorizationCodeCredentials` |

**That table is the entire authentication surface of this SDK.** `WalmartApIs.Core.Authentication` also holds scheme classes this API does not use, so read what is configurable off the properties above rather than off that folder.

## Basic auth

**API Provider Notes:**

> HTTP Basic with base64(client_id:client_secret). client_secret_post (body) also accepted; public clients send no secret.

```csharp
using WalmartApIs.Core.Authentication.Basic;

options.BasicClientAuth = new BasicAuthCredentials
{
    Username = "...",
    Password = "..."
};
```

Sends `Authorization: Basic base64(username:password)`. Both halves are required — a username with an empty password is misconfigured, not partially configured.

## OAuth 2.0 — client credentials (machine-to-machine) — `SellerAuthClientCredentials`

```csharp
using WalmartApIs.Core.Authentication.OAuth2.ClientCredentials;

options.SellerAuthClientCredentials = new OAuth2ClientCredentials
{
    ClientId = "...",
    ClientSecret = "...",
    Scope = "..."            // optional
};
```

The SDK fetches the token from `https://marketplace.walmart.com/seller/auth/v4/token` and caches it, acquiring a fresh one when it expires; on a `401` it invalidates the cached token and re-acquires. The token request carries the client id and secret as HTTP Basic credentials, so `ClientSecret` is required.

## OAuth 2.0 — client credentials (machine-to-machine) — `WalletAuth`

```csharp
using WalmartApIs.Core.Authentication.OAuth2.ClientCredentials;

options.WalletAuth = new OAuth2ClientCredentials
{
    ClientId = "...",
    ClientSecret = "...",
    Scope = "..."            // optional
};
```

The SDK fetches the token from `https://marketplace.walmart.com/seller/v1/auth/token` and caches it, acquiring a fresh one when it expires; on a `401` it invalidates the cached token and re-acquires. The token request carries the client id and secret as HTTP Basic credentials, so `ClientSecret` is required.

## OAuth 2.0 — authorization code (3-legged, with PKCE)

```csharp
using WalmartApIs.Core.Authentication.OAuth2.AuthorizationCode;

options.SellerAuthAuthorizationCode = new OAuth2AuthorizationCodeCredentials
{
    ClientId = "...",
    ClientSecret = "...",
    RedirectUri = "https://app.example.com/callback",
    Scope = "...",                              // optional
    State = "...",                              // optional CSRF token
    Pkce = PkceMethod.S256,                     // default; RFC 7636
    PromptForAuthorizationCode = async (authorizationUrl, ct) =>
    {
        // Open/redirect the browser to authorizationUrl, then return the
        // authorization code your redirect endpoint received.
        return await GetCodeFromUserAsync(authorizationUrl, ct);
    }
};
```

The SDK sends the user to `https://marketplace.walmart.com/seller/auth/v4/authorize`, exchanges the returned code for a token at `https://marketplace.walmart.com/seller/auth/v4/token`, and refreshes it when it expires; if the refresh fails, it invokes `PromptForAuthorizationCode` again to re-authorize.

`PromptForAuthorizationCode` runs inside the call that needs a new token, so it must not make an SDK call that uses this scheme. It counts against the per-attempt `options.Retry.Timeout`, 100 s by default, and a timed-out attempt on a retryable call is retried and prompts again. A prompt that throws anything but `OperationCanceledException` fails the call with `AuthSchemeException`. `State` is sent as you set it and never checked, so your redirect handler must verify it.

⚠ **`ClientSecret` is required here — PKCE does not make it optional.** The code exchange sends the client id and secret as HTTP Basic credentials. Omit it and the call fails with an `AuthSchemeException` wrapping an `InvalidOperationException`:

- **PKCE disabled** (`Pkce = null`) — *before* `PromptForAuthorizationCode` runs.
- **PKCE enabled** (the default `S256`) — *after* the user has completed the browser round-trip.

## Token caching & refresh

- Tokens are cached in-memory, **per client instance**, and reused until **30s** before expiry.
- **Only the authorization-code grant refreshes.** The other grants above re-run in full when the token expires, and a `refresh_token` in their token response is ignored — do not infer refresh behaviour from what the provider returns.
- On `401`, the cached token is invalidated and re-acquired on the next call — the failing request is **not** retried. For the authorization-code grant, invalidation drops the refresh token along with the access token, so "re-acquired" means the *full* grant runs again and `PromptForAuthorizationCode` fires: a `401` there is an interactive re-authorization, not a silent refresh. **Plan for that in a non-interactive host** — a background worker will block on a prompt nobody can answer.
- After a `401` clears the cache, a token fetch already in flight can still re-populate it, so the next call is not guaranteed a newly fetched token.

⚠ **If the token response omits `expires_in`, the SDK treats the token as never expiring**: it is cached for the client's lifetime and replaced only after a `401`, so a token revoked server-side keeps being sent until a request fails with it. For proactive rotation, supply your own token strategy (below) rather than trying to bound the cache.

### Supplying your own token source

The token strategy is a public extension point: `WalmartApIsClientOptions` has a strategy property beside each OAuth credentials property, and the built-in strategy applies only while you leave it null. The strategy is used only when the credentials property beside it is also set, and `GetToken` receives that object.

| Grant | Strategy property on `WalmartApIsClientOptions` | Type |
| --- | --- | --- |
| Client credentials | `SellerAuthClientCredentialsTokenStrategy` | `IOAuth2TokenStrategy<OAuth2ClientCredentials>` |
| Client credentials | `WalletAuthTokenStrategy` | `IOAuth2TokenStrategy<OAuth2ClientCredentials>` |
| Authorization code | `SellerAuthAuthorizationCodeTokenStrategy` | `IOAuth2RefreshableTokenStrategy<OAuth2AuthorizationCodeCredentials>` |

```csharp
options.SellerAuthClientCredentials = new OAuth2ClientCredentials { /* its required members */ };
options.SellerAuthClientCredentialsTokenStrategy = new MyTokenStrategy();   // Task<OAuthToken> GetToken(creds, ct)
```

Reach for it when the token must come from somewhere other than the SDK's own call to the token endpoint — a shared cache across processes, a secrets broker, a sidecar that already holds a valid token, or a test double. The per-client in-memory cache above still wraps whatever you return.

## Combining the schemes

This SDK has more than one credentials property, and its operations compose them; you choose by which properties on `WalmartApIsClientOptions` you set.

## Operations that need no credentials

Some operations on this API are public: they send no credential, whatever you set on `WalmartApIsClientOptions`. You still construct the same client — there is no separate unauthenticated client — and the credentials you configured are simply not applied to those calls.

## ⚠ Missing credentials must stop the app from starting

⚠ **Configure nothing and the request goes out unauthenticated — no exception.** An unset credentials property on `WalmartApIsClientOptions` sends the call with no credential, and it comes back `401`; the SDK never reports the missing credential. A required credential that is not configured is a deployment fault, not a request fault.

**Validate at startup and refuse to boot.** Bind the credentials into your own options type and make the host check it before the app serves anything:

```csharp
builder.Services
    .AddOptions<WalmartApIsClientSettings>()
    .Bind(builder.Configuration.GetSection("WalmartApIsClient"))
    .ValidateDataAnnotations()      // [Required] on each credential property
    .ValidateOnStart();             // throws during startup, not on first request
```

`ValidateOnStart()` is the load-bearing call — without it, `IOptions<T>` validation is lazy and fires on first resolution, which is a request, which is exactly the late failure you are trying to avoid. For a console app, an explicit guard is equally acceptable as long as it runs **before** the app is ready:

```csharp
if (string.IsNullOrWhiteSpace(settings.Username))
    throw new InvalidOperationException(
        "WalmartApIsClient:Username is not configured. Set it via environment variable, " +
        "user-secrets, or your secret store before starting the app.");
```

Three rules for the message it fails with:

- **Name the missing config key**, so the operator knows what to set — `"WalmartApIsClient:Username is not configured"`, not `"authentication failed"`.
- **Never echo the value**, present or absent — no length, no prefix, no masked form. A "configured: AC1234…" line is a secret in a log.
- **Do not fall back to a default, a placeholder, or an unauthenticated client.** Booting degraded hides the fault and pushes it to the first caller.

Check every value the schemes actually need. For this SDK that is:

| Value to validate | Feeds `WalmartApIsClientOptions` property |
| --- | --- |
| `Username` | `BasicClientAuth` |
| `Password` | `BasicClientAuth` |
| `ClientId` | `SellerAuthClientCredentials` |
| `ClientSecret` | `SellerAuthClientCredentials` |
| `ClientId` | `WalletAuth` |
| `ClientSecret` | `WalletAuth` |
| `ClientId` | `SellerAuthAuthorizationCode` |
| `ClientSecret` | `SellerAuthAuthorizationCode` |
| `RedirectUri` | `SellerAuthAuthorizationCode` |

## Notes

- Set credentials **before** constructing the client, or inside the `AddWalmartApIsClient(options => ...)` callback when registering via DI. `WalmartApIsClientOptions` is read at construction; mutating it afterwards does not re-wire an existing client.
- Keep secrets out of source — load them from configuration (environment variables, a secret store, or any other `IConfiguration` source) instead of hardcoding, either inside the `AddWalmartApIsClient(options => ...)` callback for a host or via a `ConfigurationBuilder()...Build()` chain for a console app.

