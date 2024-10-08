# OAuth Proxy Azure API Management policies

The policies in this folder provide support for an OAuth Proxy that works in a similar way to App Service Authentication.

## Setup

### Required Named Values

| Named Value | Purpose |
| -- | -- |
| AdditionalScopes | Space separated string of other scopes to request delegated consent for |
| ClientId | AAD ClientId Id representing the application you are signing in against |
| ClientSecret | AAD Client Secret used to exchange codes for tokens |
| CookiePrefix | The name we use for the cookie used to control the oauth-proxy |
| CookieEncryptionKey | 1 or 2. Selects the key (CookieEncryptionKey**1** or CookieEncryptionKey**2**) used to protect newly issued cookies. This allows you to periodically rotate keys |
| CookieEncryptionKey1 | A Base 64 Encoded string of a 32 random bytes array. Used by an AES 256 encryption algorithm to encrypt cookies |
| CookieEncryptionKey2 | A Base 64 Encoded string of a 32 random bytes array. Used by an AES 256 encryption algorithm to encrypt cookies |
| TokenEncryptionKey | 1 or 2. Selects the key (TokenEncryptionKey**1** or TokenEncryptionKey**2**)  used to protect newly issued tokens. This allows you to periodically rotate keys  |
| TokenEncryptionKey1 | A Base 64 Encoded string of 32 random bytes. Used as the Key for an AES 256 encryption algorithm for encrypting tokens at rest |
| TokenEncryptionKey2 | A Base 64 Encoded string of 32 random bytes. Used as the Key for an AES 256 encryption algorithm for encrypting tokens at rest |
| SessionCookieExpirationInSeconds | How long to allow session cookies to stay active for |
| RefreshTokenExpirationInSeconds | How long to cache refresh tokens for (a good guide would be how long your average user's session lasts for) |

> You can generate the Base 64 random bytes in dotnet using ``` Convert.ToBase64String(RandomNumberGenerator.GetBytes(<size>)) ```, or in bash using ```openssl rand -base64 32```

### Required Named Values for Azure Active Directory

| Named Value | Purpose |
| -- | -- | 
| TenantId | AAD Tenant Id that owns the ClientId you want users to sign-in to |


## Fragments
| Fragment File Name | Fragment Name | Purpose | How to use |
| -- | -- | -- | -- |
| [oauth-proxy-token-endpoint-fragment.xml](oauth-proxy-token-endpoint-fragment.xml) | oauth-proxy-token-endpoint-fragment | Identifies the token endpoint to obtains tokens from | You **must** place this fragment above the ```oauth-proxy-session-fragment``` as it sets a required variable used by other policies.  |
| [oauth-proxy-validate-token-fragment.xml](oauth-proxy-validate-token-fragment.xml) | oauth-proxy-validate-token-fragment | Custom token validation policy | Place it after the ```oauth-proxy-session-fragment``` to provide an additional JWT validation step on the access-token returned by your IdP. The reference implementation uses the  [validate-azure-ad-token](https://learn.microsoft.com/en-us/azure/api-management/validate-azure-ad-token-policy) policy. For other IdPs use the [validate-jwt](https://learn.microsoft.com/en-us/azure/api-management/validate-jwt-policy) policy. |
| [oauth-proxy-session-fragment.xml](oauth-proxy-session-fragment.xml) | oauth-proxy-session-fragment | A fragment that checks for a Session cookie, and either initiate a sign-in flow, or attaches valid tokens to the ongoing request. This will refresh tokens if necessary | Place inside the ```<inbound>``` policy of any Web Apps you want to protect with a session cookie |
| [oauth-proxy-construct-authorization-redirect-fragment.xml](oauth-proxy-construct-authorization-redirect-fragment.xml) | oauth-proxy-construct-authorization-redirect-fragment | A fragment that constructs an OIDC Authorize request to your endpoint | If using a different IdP, use ```oauth-proxy-construct-authorization-redirect.xml``` as a guide to configuring for your IdP |
| [oauth-proxy-slide-session-fragment.xml](oauth-proxy-slide-session-fragment.xml) | oauth-proxy-slide-session-fragment | A fragment that slides any issued session cookie | Place inside the ```<outbound>``` policy of any Web Apps you want to protect with a session cookie |

## Policies for Oidc Endpoints 
| Policy Name | API path | Method | Purpose | How to use |
| -- | -- | -- | -- | -- |
| [oauth-proxy-sign-in.xml](oauth-proxy-sign-in.xml) | ```/oauth/signin``` | GET | Initiates a front-channel code / pkce flow with an IdP  | Configure this as the 'signin' operation within an API called 'OAuth' |
| [oauth-proxy-callback.xml](oauth-proxy-callback.xml) | ```/oauth/callback``` | POST | Handles an IdP's callback in response to a sign-in request  | Configure this as the 'callback' operation within an API called 'OAuth' |
| [oauth-proxy-sign-out.xml](oauth-proxy-sign-out.xml) | ```/oauth/signout``` | GET | Clears a user's session cookie, and removes all token data from the cache.  | Configure this as the 'signout' operation within an API called 'OAuth' |

## Simple policy to protect Web Applications

```xml
<policies>
    <inbound>
        <include-fragment fragment-id="oauth-proxy-token-endpoint-fragment" />
        <include-fragment fragment-id="oauth-proxy-session-fragment" />
        <include-fragment fragment-id="oauth-proxy-validate-token-fragment" />
        
        <!-- Adds the following headers to the downstream request: 
            
            Authorization: Bearer {access-token}
            x-proxy-id-token: {id-token} 
            x-proxy-id-token-name: {id-token name claim}
            x-proxy-id-token-preferred-username: {id-token preferred-username claim}
            -->
        <base />
    </inbound>
    <backend>
        <base />
    </backend>
    <outbound>
        <include-fragment fragment-id="oauth-proxy-slide-session-fragment" />
        <base />
    </outbound>
    <on-error>
        <include-fragment fragment-id="oauth-proxy-slide-session-fragment" />
        <base />
    </on-error>
</policies>

```


# Policy Details

## oauth-proxy-session-fragment

### Purpose
This fragment is intended to sit at a high-level around any calls you need to protect.

It checks for an incoming session-cookie and either
 - Redirects to oauth/signin if invalid / expired
 - Fetches tokens from Redis, and decrypts them using the Session-Id (IV), and the TokenEncryptionKey(1 or 2) key
 - Renews access tokens using a refresh-token if they are nearing expiration
 - Appends the Bearer token to the request for use by the downstream API.

## oauth-proxy-construct-authorization-redirect-fragment

### Purpose
Assigns a valid URI to the ```oauth-proxy-redirect``` variable that will redirect a User to an IdP to initiate a sign-in.

The following variables are set by the ```oauth-proxy-sign-in``` policy to use here:
- state
- nonce
- codeChallengeSha256

This fragment must set a variable called ```oauth-proxy-redirect``` which initiates the sign-in flow.

## oauth-proxy-slide-session-fragment

### Purpose
An Outbound processing fragment that slides the current session cookie. Use it at the same API scope as the above Session check fragment.

### Steps
- Issues a new session cookie on all requests, which slides forward to ```UtcNow + SessionCookieExpirationInSeconds```

## oauth-proxy-token-endpoint-fragment

### Purpose
This fragment creates a valid URI that where we can exchange a code for a token.

This fragment must set a variable called ```idpTokenEndpoint``` where we can POST to.


## oauth-proxy-sign-in

### Purpose
This policy initiates an OIDC 3-legged sign-in flow.

### Steps
- Checks for a valid redirect on the incoming URL
- Creates state, nonce, and code-challenges which are stored in Redis
- Uses the ```oauth-proxy-construct-authorization-redirect-fragment``` to construct the authorisation request
- Returns a cookie with a lookup the the above state, nonce and code-challenge
- 302 Redirects the browser to initiate the front-channel sign-in.

### Required QueryString Values

| Query String key | Purpose |
| -- | -- |
| redirect | Url to redirect to after a successful sign-in flow. This must be a root path beginning with '/'.  |

## oauth-proxy-callback
> Implemented by [oauth-proxy-callback.xml](./oauth-proxy-callback.xml)

### Purpose
This policy handles a callback from an IdP to complete an OIDC flow.

### Steps
- Get the ```code``` and ```state``` parameter from the incoming querystring
- Check for an incoming ```oidc``` cookie suffixed with the ```state``` parameter 
- Lookup the state and nonce properties from cache which were previously stored in the ```signin``` policy
- Return 401 if we cannot find them
- If the state parameter in the querystring from the IdP matches the cookie, and was stored in our cache, then switch the code for a token using a PKCE code-
- Check the nonce in the returned token matches the nonce stored in session
- Return 401 if we cannot match the nonce
- Creates an IV which is round-tripped in the session cookie (not stored server-side)
- Encrypts the tokens using the above IV, and the TokenEncryptionKey(1 or 2)
- Store the encrypted tokens in Redis
- Set a session-cookie which comprises of our cache-key, the IV, the cookies expiry timestamp. Signs it using a HMAC-SHA-512 signature creating using the SessionCookieKey(1 or 2) named value.

## oauth-proxy-sign-out
> Implemented by [oauth-proxy-signout.xml](./oauth-proxy-signout.xml)

### Purpose
This policy performs a User 'sign-out'. It removes the session cookies, and also removes all tokens from cache.

### Steps
- Clears all cached tokens
- Clears the session cookie
- Redirects the user to the provided redirect parameter (returns a 200 OK if no, or invalid redirect).

NB: This does not invalidate the access tokens. If any API cached the token it will still be valid until its expiry date.
