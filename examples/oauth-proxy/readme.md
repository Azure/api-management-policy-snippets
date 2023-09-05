# OAuth Proxy Azure API Management policies

The policies in this folder provide support for an OAuth Proxy that works in a similar way to App Service Authentication.

## Policies
| Policy Name | Purpose | How to use |
| -- | -- | -- |
| [oauth-proxy-token-endpoint.xml](oauth-proxy-token-endpoint.xml) | Identifies the token endpoint to obtains tokens from | Deploy as a fragment named ```oauth-proxy-token-endpoint```. You **must** include this fragment above the ```<session-check-fragment>``` as it sets a required variable  (you cannot nest fragments in Azure API Management)  |
| [oauth-proxy-session-fragment.xml](oauth-proxy-session-fragment.xml) | A fragment that checks for a Session cookie, and either initiate a sign-in flow, or attaches valid tokens to the ongoing request. This will refresh tokens if necessary | Deploy as a fragment named ```oauth-proxy-session-fragment```, and wrap it around the ```<inbound>``` policy of any Web Apps you want to protect with a session cookie |
| [oauth-proxy-sign-in.xml](oauth-proxy-sign-in.xml) | Initiates a front-channel code / pkce flow with an IdP  | Configure this as the 'signin' operation within an API called 'OAuth' |
| [oauth-proxy-construct-authorization-redirect.xml](oauth-proxy-construct-authorization-redirect.xml) | A fragment that constructs an OIDC Authorize request to your endpoint | Deploy as a fragment named ```oauth-proxy-construct-authorization-redirect```. If using a different IdP, use ```oauth-proxy-construct-authorization-redirect.xml``` as a guide to configuring for your IdP |
| [oauth-proxy-callback.xml](oauth-proxy-callback.xml) | Handles an IdP's callback in response to a sign-in request  | Configure this as the 'callback' operation within an API called 'OAuth' |
| [oauth-proxy-slide-session-fragment.xml](oauth-proxy-slide-session-fragment.xml) | A fragment that slides any issued session cookie | Deploy as a fragment named ```oauth-proxy-slide-session-fragment```, and wrap it around the ```<outbound>``` policy of any Web Apps you want to protect with a session cookie |
| [oauth-proxy-sign-out.xml](oauth-proxy-sign-out.xml) | Clears a user's session cookie, and removes all token data from the cache.  | Configure this as the 'signout' operation within an API called 'OAuth' |

### Required Named Values

| Named Value | Purpose |
| -- | -- |
| ClientId | AAD ClientId Id representing the application you are signing in against |
| ClientSecret | AAD Client Secret used to exchange codes for tokens |
| SessionCookieKey | A Base 64 Encoded string representing a 128 random byte array. Used to sign cookies to check their validity |
| TokenEncryptionKey | A Base 64 Encoded string of 32 random bytes, used as the Key for an AES 256 encryption algorithm for encrypting tokens at rest |
| SessionCookieExpirationInSeconds | How long to allow session cookies to stay active for |
| RefreshTokenExpirationInSeconds | How long to cache refresh tokens for (a good guide would be how long your average user's session lasts for) |

> You can generate the Base 64 random bytes in dotnet using ``` Convert.ToBase64String(RandomNumberGenerator.GetBytes(<size>)) ```

### Required Named Values for the ```oauth-proxy-construct-authorization-redirect```

| Named Value | Purpose |
| -- | -- | 
| TenantId | AAD Tenant Id that owns the ClientId you want users to sign-in to |
| AdditionalScopes | Space separated string of other scopes to request delegated consent for |

# Policy Details

## Sample wrapper policy around Web Apps

```xml
<policies>
    <inbound>
        <include-fragment fragment-id="oauth-proxy-token-endpoint" />
        <include-fragment fragment-id="oauth-proxy-session-fragment" />
        <!-- Adds the following headers: 
            
            Authorization: Bearer {access-token}
            x-proxy-id-token: {id-token} 
            x-proxy-id-token-name: {id-token name claim}
            x-proxy-id-token-preferred-username: {id-token preferred-username claim}

            to the downstream request -->
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

## Session Check fragment
> Implemented by [oauth-proxy-session-fragment.xml](./oauth-proxy-session-fragment.xml)

### Purpose
This fragment is intended to sit at a high-level around any calls you need to protect.

It checks for an incoming session-cookie and either
 - Redirects to oauth/signin if invalid / expired
 - Fetches tokens from Redis, and decrypts them using the Session-Id (IV), and the TokenEncryptionKey (Key) 
 - Renews access tokens using a refresh-token if they are nearing expiration
 - Appends the Bearer token to the request for use by the downstream API.


## /oauth/signin
> Implemented by [oauth-proxy-sign-in.xml](./oauth-proxy-sign-in.xml)

### Purpose
This policy initiates an OIDC 3-legged sign-in flow.

### Steps

### Required QueryString Values

| Query String key | Purpose |
| -- | -- |
| redirect | Fully qualified URL to redirect to after a successful sign-in flow |

## Authorization Request Fragment (sample for AAD)
> Implemented by [oauth-proxy-construct-authorization-redirect.xml](./oauth-proxy-construct-authorization-redirect.xml)

### Purpose
This fragment must be called ```oauth-proxy-construct-authorization-redirect``` and must assign a valid URI to the ```oauth-proxy-redirect``` variable that will redirect a User to an IdP to initiate a sign-in.

The following variables are set by the ```sign-in.xml``` policy to use here:
- state
- nonce
- codeChallengeSha256

This fragment must set a variable called ```oauth-proxy-redirect``` which initiates the sign-in flow.


## /oauth/callback
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
- Encrypts the tokens using the above IV, and the TokenEncryptionKey (Key) 
- Store the encrypted tokens in Redis
- Set a session-cookie which comprises of our cache-key, the IV, the cookies expiry timestamp. Signs it using a HMAC-SHA-512 signature creating using the SessionCookieKey named value.

## Authorization Token Endpoint fragment (sample for AAD)
> Implemented by [oauth-proxy-token-endpoint-fragment.xml](./oauth-proxy-token-endpoint-fragment.xml)

### Purpose
This fragment must be created as ```oauth-proxy-token-endpoint``` and creates a valid URI that where we can exchange a code for a token.

This fragment must set a variable called ```idpTokenEndpoint``` where we can POST to.

## Sliding Session Cookie fragment
> Implemented by [oauth-proxy-slide-session-fragment.xml](oauth-proxy-slide-session-fragment.xml)

### Purpose
An Outbound processing fragment that slides the session cookie associated with the session. Use it at the same API scope as the above Session check fragment.

### Steps
- Issues a new session cookie on all requests, which slides forward to ```UtcNow + SessionCookieExpirationInSeconds```


## /oauth/signout
> Implemented by [oauth-proxy-signout.xml](./oauth-proxy-signout.xml)

### Purpose
This policy performs a User 'sign-out'. It removes the session cookies, and also removes all tokens from cache.

### Steps
- Clears all cached tokens
- Clears the session cookie
- Redirects the user to the provided redirect parameter (returns a 200 OK if no, or invalid redirect).

NB: This does not invalidate the access tokens. If any API cached the token it will still be valid until its expiry date.
