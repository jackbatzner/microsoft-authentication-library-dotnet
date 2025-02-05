---
title: Extract authentication parameters from WWW-Authenticate headers
description: "This article is both a conceptual article of why you'd want to get information from WWW-authenticate headers, and how to do it."
---

# Extract authentication parameters from WWW-Authenticate headers

This article is both a conceptual article of why you'd want to get information from WWW-authenticate headers, and how to do it.

Jump to [How to - use the WwwAuthenticateParameters class](#how-to---use-the-wwwauthenticateparameters-class) for implementation details.

## Concepts - Why use WwwAuthenticateParameters

Diagram:

![Flow between web API and a client](../media/auth-parameters-diagram.png)

Sometimes, you call a web API without being authenticated with the intension to get the parameters that you'll need to use to authenticate the user (for instance the authority and the scopes). This is done, for APIs that support it, by replying with a WWW-Authenticate header.

There are also scenarios where a web API needs more claims from the user. But given web APIs don't have UI interaction, it needs to propagate back the request to the client, again with information in a WWW-Authenticate header.

### Unauthenticated call to an API

Some web APIs, when called unauthenticated, send back an `HTTP 401 (Unauthorized)` with a `wwwAuthenticate` header. Note that this is not a response from Microsoft Entra ID, but really from a web API that the client app would call without authentication or with the wrong authentication.

For instance:

- If you use your browser to havigate to `https://graph.microsoft.com/v1.0/me`, you'll get an HTTP 401 (Unauthorized) error, but the wwwAuthenticate header will tell you: get a token using this IdP (defined by its authorize endpoint URI), for this resource (Graph):

  ```text
  HTTP 401; Unauthorized
  WWW-Authenticate: Bearer realm="", authorization_uri="https://login.microsoftonline.com/common/oauth2/authorize", client_id="00000003-0000-0000-c000-000000000000"
  ```

- if you navigate to `https://yourVault.vault.azure.net/secrets/CertName/CertVersion`, you'll get a wwwAuthenticate header like the following:

  ```text
  HTTP 401; Unauthorized
  WWW-Authenticate: Bearer authorization="https://login.windows.net/yourTenantId", resource="https://vault.azure.net"
  ```

### Continuous Access Evaluation (CAE)

[Continuous access evaluation](/azure/active-directory/conditional-access/concept-continuous-access-evaluation) enabled web APIs also send a wwwAuthenticate header when more claims are needed. For details see [How to use Continuous Access Evaluation enabled APIs in your applications](/azure/active-directory/develop/app-resilience-continuous-access-evaluation). The wwwAuthenticate header will have the following form:

```text
HTTP 401; Unauthorized
WWW-Authenticate=Bearer
  authorization_uri="https://login.windows.net/common/oauth2/authorize",
  error="insufficient_claims",
  claims="eyJhY2Nlc3NfdG9rZW4iOnsibmJmIjp7ImVzc2VudGlhbCI6dHJ1ZSwgInZhbHVlIjoiMTYwNDEwNjY1MSJ9fX0="
```

The way the client application processes it is, when the error is `insufficient_claims`, it extract the `claims`, property and if it's not already a JSON blob, convert it from Base64 (if it's already a JSON blob, it uses it as is). Then this claims is used with `.WithClaims()` in MSAL.NET.

### CA auth context

CA auth context relies on web APIs sending back a wwwAuthenticate header. For details about CA auth context see:

- [Developers’ guide to Conditional Access authentication context](/azure/active-directory/develop/developer-guide-conditional-access-authentication-context)
- [Code sample to use CA Auth context in a web API](https://github.com/Azure-Samples/ms-identity-ca-auth-context/blob/main/README.md)
- [Recorded session: Use Conditional Access Auth Context in your app for step-up authentication – May 2021](https://www.youtube.com/watch?v=_iO7CfoktTY)

The `wwwAuthenticate` header returned by CA auth context is similar to the one returned by CAE enabled web APIs:

```text
HTTP 401; Unauthorized
WWW-Authenticate=Bearer
  client_id="guid of the resource"
  authorization_uri="https://login.windows.net/common/oauth2/authorize",
  error="insufficient_claims",
  claims="eyJhY2Nlc3NfdG9rZW4iOnsibmJmIjp7ImVzc2VudGlhbCI6dHJ1ZSwgInZhbHVlIjoiMTYwNDEwNjY1MSJ9fX0="
```

The processing will be the same.

### Web APIs using Microsoft identity web and reacting to claims challenges

Microsoft.Identity.web can send, a wwwAuthenticate header through one of the overrides of the <xref:Microsoft.Identity.Web.ITokenAcquisition.ReplyForbiddenWithWwwAuthenticateHeader(System.Collections.Generic.IEnumerable{System.String},Microsoft.Identity.Client.MsalUiRequiredException,Microsoft.AspNetCore.Http.HttpResponse)> method.

It sends non-standard parameters:

- Consent Url, in order to help multi-tenant web API developers to provide a link so that the tenant users or admins can consent for the web API to be installed in their tenant.
- Claims (in the case of claims challenge)
- Scopes for the required scopes. This might not be great.
- `ProposedAction` ("consent")

## How to - use the WwwAuthenticateParameters class

### Calling a protected API, unauthenticated

For the first scenario, you just call `WwwAuthenticateParameters.CreateFromResourceResponse` passing the URI of the web API.

```csharp
 WwwAuthenticateParameters parameters = 
   await WwwAuthenticateParameters.CreateFromResourceResponseAsync("https://yourVault.vault.azure.net/secrets/secret/version");

   IConfidentialClientApplication app = ConfidentialClientApplicationBuilder.Create(clientId)
     .WithAuthority(parameters.Authority)     
     .Build();

   // Token Caching explained at: /azure/active-directory/develop/msal-net-token-cache-serialization
   app.AppTokenCache.SetCacheOptions(CacheOptions.EnableSharedCacheOptions);

   AuthenticationResult result = await app.AcquireTokenForClient(new[] {"you_should_know_the_scope_in_advance")
     .WithClaims(parameters.Claims)
     .ExecuteAsync();
```

### Claim challenge, CAE, CA auth context

```csharp
HttpResponseMessage response;
using (HttpRequestMessage httpRequestMessage = new HttpRequestMessage(
                effectiveOptions.HttpMethod,
                apiUrl))
            {
                if (content != null)
                {
                    httpRequestMessage.Content = content;
                }

                httpRequestMessage.Headers.Add(
                    Constants.Authorization,
                    authResult.CreateAuthorizationHeader());
                reponse = await _httpClient.SendAsync(httpRequestMessage).ConfigureAwait(false);
            }

            WwwAuthenticateParameters parameters = await WwwAuthenticateParameters.CreateFromResourceResponse(response );
```

## See also

- The spec is available in a [related GitHub issue](https://github.com/AzureAD/microsoft-authentication-library-for-dotnet/issues/2679).
- This [pull request](https://github.com/Azure-Samples/active-directory-aspnetcore-webapp-openidconnect-v2/pull/512) shows how the API was used for CAE.
