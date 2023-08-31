# OAuth Proxy APIm policies

The policies in this folder provide support for an OAuth Proxy similar that works in a similar way to App Service Authentication.

The 4 policies fulfill different parts of the flow. I recommend setting up an API called 'oauth' in your APIm, and having 2 operations off it, both unauthenticated.

### Required Named Values

| Named Value | Purpose |
| -- | -- |
| TenantId | AAD Tenant Id to sign-in against |
| ClientId | AAD ClientId Id representing the application you are signing in against |
| ClientSecret | AAD Client Secret used to exchange codes for tokens |
| SessionCookieKey | A Base 64 Encoded string representing a 128 random byte array. Used to sign cookies to check their validity |
| SessionCookieExpirationInSeconds | How long to allow session cookies to stay active for |


## Session Check fragment
> Implemented by [session-check.xml](./session-check.xml)

### Purpose
This fragment is intended to sit at a high-level around any calls you need to protect.

It checks for an incoming session-cookie and either
 - Redirects to oauth/signin if invalid / expired
 - Renews access tokens using a refresh-token if they are nearing expiration
 - Appends the Bearer token to the request for use by the downstream API.


## /oauth/signin
> Implemented by [sign-in.xml](./sign-in.xml)

### Purpose
This policy initiates an OIDC 3-legged sign-in flow.

### Steps

### Required QueryString Values

| Query String key | Purpose |
| -- | -- |
| redirect | Fully qualified URL to redirect to after a successful sign-in flow |


## /oauth/callback
> Implemented by [sign-in.xml](./sign-in.xml)

### Purpose
This policy handles a callback from an IdP to complete an OIDC flow.

### Steps
- Get the ```code``` and ```state``` parameter from the incoming querystring
- Check for an incoming ```oidc``` cookie suffixed with the ```state``` parameter 
- Lookup the state and nonce properties from cache which were previously stored in the ```signin``` policy
- Return 401 if we cannot find them
- If the state parameter in the querystring from the IdP matches the cookie, and was stored in our cache, then switch the code for a token
- Check the nonce in the returned token matches the nonce stored in session
- Return 401 if we cannot match the nonce
- Store the id_token, access_token, and refresh_tokens in cache.
- Set a session-cookie which comprises of our cache-key, the cookies expiry timestamp, and sign it using a HMAC-SHA-512 signature creating using the SessionCookieKey named value.

## Sliding Session Cookie fragment
> Implemented by [sliding-session-cookie-fragment.xml](sliding-session-cookie-fragment.xml)

### Purpose
An Outbound processing fragment that slides the session cookie associated with the session. Use it at the same level as the above Session check fragment

### Steps
- Issues a new session cookie on all requests, which slides forward to ```UtcNow + SessionCookieExpirationInSeconds```

