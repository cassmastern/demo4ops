# OAuth 2.0 Flow

## Understanding OAuth 2.0: The Authorization Framework

OAuth 2.0 is the industry-standard protocol for authorization. It solves a fundamental problem: **How do you give an app access to your data without giving it your password?**

### The Pre-OAuth World

Imagine a blood pressure tracking app that needs to read your EHR data. Without OAuth:

❌ **Option 1**: You give the app your EHR username and password  
→ Problem: The app can do anything you can do, forever. If the app is compromised, your password is exposed.

❌ **Option 2**: EHR creates a special API key just for you  
→ Problem: Managing thousands of API keys is a nightmare. Revoking access is difficult.

### The OAuth Solution

✅ **OAuth flow**: App asks permission, you approve, app gets a temporary token with limited permissions.

→ Benefits:
- No password sharing (app never sees your credentials)
- Limited scope (app can only read vitals, not prescribe medications)
- Revocable (you can revoke access anytime)
- Time-limited (tokens expire, requiring periodic re-authorization)

---

## The OAuth 2.0 Actors

Every OAuth flow involves four parties:

### 1. Resource Owner (You)

The person who owns the data and can grant access to it.

**Examples**:
- Patient accessing their own record
- Clinician with rights to view patient charts
- Administrator with system-wide access

### 2. Client (The App)

The application requesting access to protected resources.

**Examples**:
- Blood pressure tracking mobile app
- Clinical decision support tool
- Research data collection system

**Client types**:
- **Public client**: Can't keep secrets (mobile apps, single-page apps). Uses PKCE.
- **Confidential client**: Can keep secrets (server-side apps). Uses client_secret.

### 3. Authorization Server

Authenticates the resource owner and issues access tokens to the client.

**Responsibilities**:
- Display login screen
- Verify user credentials
- Show consent screen ("Allow app to read your vitals?")
- Issue access tokens and refresh tokens
- Validate tokens on subsequent requests

**Examples**:
- Epic's OAuth server
- Cerner's authorization endpoint
- SMART Health IT sandbox auth server

### 4. Resource Server (FHIR Server)

Hosts the protected resources and validates access tokens.

**Responsibilities**:
- Accept API requests with Bearer tokens
- Validate token signature and expiration
- Check scopes and context
- Return requested FHIR resources
- Enforce access control policies

---

## The Complete Authorization Code Flow 

Here is a diagram showing the flow in detail:

```mermaid
sequenceDiagram
    participant Browser
    participant App as SMART App
    participant Auth as Authorization Server
    participant FHIR as FHIR Server

    Note over Browser,FHIR: EHR Launch Flow
    
    Browser->>App: 1. Launch with iss and launch params
    Note right of Browser: GET /launch?iss=...&launch=...
    
    App->>Auth: 2. Discover endpoints
    Note right of App: GET [iss]/.well-known/smart-configuration
    Auth-->>App: Return configuration
    Note left of Auth: authorization_endpoint, token_endpoint
    
    App->>Auth: 3. Authorization request
    Note right of App: GET /authorize?<br/>response_type=code<br/>client_id=...<br/>redirect_uri=...<br/>scope=...<br/>state=...<br/>aud=...<br/>launch=...
    
    Auth->>Browser: 4. Display login screen
    Browser->>Auth: 5. User credentials
    
    Auth->>Browser: 6. Display consent screen
    Browser->>Auth: 7. User approves
    
    Auth->>App: 8. Redirect with code
    Note right of Auth: GET /callback?<br/>code=...&state=...
    
    App->>Auth: 9. Token request
    Note right of App: POST /token<br/>grant_type=authorization_code<br/>code=...<br/>redirect_uri=...<br/>client_id=...

    Note over Auth,FHIR: Context is created/associated within the EHR ecosystem<br/>Implementation detail: may be internal, or via shared session/state

    Auth-->>App: 11. Access token response
    Note left of Auth: access_token<br/>token_type: Bearer<br/>expires_in<br/>scope<br/>patient (optional)<br/>encounter (optional)<br/>refresh_token (optional)
    
    App->>FHIR: 12. API request with token
    Note right of App: GET /Patient/123<br/>Authorization: Bearer [token]
    
    FHIR->>FHIR: 13. Validate token and scope
    FHIR-->>App: 14. Return resource
    
    Note over Browser,FHIR: Token Refresh Flow
    
    App->>Auth: 15. Refresh token request
    Note right of App: POST /token<br/>grant_type=refresh_token<br/>refresh_token=...
    
    Auth-->>App: 16. New access token
    Note left of Auth: access_token<br/>expires_in<br/>refresh_token
```

!!! note
    In SMART, the `patient` and `encounter` fields in the token response are **launch-context hints** (when available). Don’t assume they will always be present—apps must still function when context is absent (e.g., some standalone flows, or when the server chooses not to return it).

Let's examine the flow step by step.

### Step 1: Launch with ISS and Launch Parameters

**EHR launch scenario**:

```http
GET https://your-app.com/launch?iss=https://fhir.example.org&launch=eyJhbGciOiJIUz...
```

**Parameters**:
- **`iss`** (issuer): FHIR server base URL
- **`launch`**: Opaque token containing context (patient ID, encounter ID)

**Your app stores**: `iss` and `launch` for use in step 3.

**Standalone launch scenario**:

Your app knows the FHIR server URL already (user selected EHR). No `launch` token—context will come from user authentication.

---

### Step 2: Discover Authorization Endpoints

**Request**:

```http
GET https://fhir.example.org/.well-known/smart-configuration
Accept: application/json
```

**Response**:

```json
{
  "authorization_endpoint": "https://auth.example.org/authorize",
  "token_endpoint": "https://auth.example.org/token",
  "scopes_supported": [
    "openid",
    "fhirUser",
    "launch",
    "launch/patient",
    "patient/*.read",
    "user/*.read"
  ],
  "capabilities": [
    "launch-ehr",
    "client-public",
    "context-ehr-patient",
    "permission-offline"
  ]
}
```

**Why discovery?**: Different EHR vendors use different OAuth endpoints. Discovery lets your app work universally.

**Your app caches**: Authorization and token endpoints for the rest of the flow.

---

### Step 3: Authorization Request

**Your app redirects the browser**:

```http
GET https://auth.example.org/authorize?
  response_type=code&
  client_id=your_app_id&
  redirect_uri=https://your-app.com/callback&
  scope=launch%20launch/patient%20patient/*.read%20offline_access%20openid%20fhirUser&
  state=random_string_abc123&
  aud=https://fhir.example.org&
  launch=eyJhbGciOiJIUz...
```

**Parameters**:

- **`response_type=code`**: Using authorization code flow (most secure)
- **`client_id`**: Your app's identifier (registered with EHR)
- **`redirect_uri`**: Where to send user after authorization (must match registration)
- **`scope`**: Permissions requested (space-separated list)
  - `launch`: Required for EHR-initiated launches
  - `launch/patient`: Request patient context
  - `patient/*.read`: Read all resources for patient
  - `offline_access`: Request refresh token
  - `openid`: OIDC authentication
  - `fhirUser`: Get FHIR User resource URL
- **`state`**: Random string to prevent CSRF attacks
- **`aud`** (audience): FHIR server URL (proves intent)
- **`launch`**: Context token from step 1 (EHR launch only)

**Security notes**:
- `state` parameter prevents cross-site request forgery
- `redirect_uri` must exactly match registered value
- `aud` parameter prevents token substitution attacks

---

### Step 4: Display Login Screen

**Authorization server shows login form**:

```
┌─────────────────────────────────────┐
│  Example Hospital Login             │
├─────────────────────────────────────┤
│                                     │
│  Username: [__________________]     │
│  Password: [__________________]     │
│                                     │
│  [ Login ]                          │
│                                     │
│  Forgot password?                   │
└─────────────────────────────────────┘
```

**What happens**:
- User enters EHR credentials (NOT given to your app)
- Authorization server validates credentials
- Server may require MFA (multi-factor authentication)
- Server establishes session

**Security**: Your app NEVER sees the password. Authorization server handles all authentication.

---

### Step 5: User Provides Credentials

**Behind the scenes**:
- Authorization server validates username/password
- May check against Active Directory, LDAP, or database
- Creates authenticated session
- Associates session with pending authorization request

---

### Step 6: Display Consent Screen

**Authorization server shows permissions request**:

```
┌────────────────────────────────────────┐
│  Blood Pressure Tracker                │
│  is requesting access to:              │
├────────────────────────────────────────┤
│                                        │
│  ☐ Your basic profile information      │
│  ☐ Your vital signs                    │
│  ☐ Your observation history            │
│                                        │
│  This app will be able to:             │
│  • View your blood pressure readings   │
│  • View your heart rate                │
│  • View measurement history            │
│                                        │
│  [ Deny ]         [ Allow ]            │
│                                        │
└────────────────────────────────────────┘
```

**Consent details**:
- Human-readable description of requested scopes
- Duration of access (one-time vs. ongoing if `offline_access`)
- Link to app's privacy policy
- Options to reduce scope (some servers allow partial approval)

**User actions**:
- **Allow**: Proceeds to step 7
- **Deny**: Returns error to app's redirect_uri

---

### Step 7: User Approves Access

**What happens**:
- Authorization server records consent
- Generates authorization code (short-lived, typically 5 minutes)
- Associates code with:
  - Client ID
  - Redirect URI
  - Approved scopes
  - User identity
  - Patient context (for EHR launches)

---

### Step 8: Redirect with Authorization Code

**Authorization server redirects back to your app**:

```http
HTTP/1.1 302 Found
Location: https://your-app.com/callback?
  code=auth_code_xyz789&
  state=random_string_abc123
```

**Your app validates**:
1. Check `state` matches value from step 3 (prevents CSRF)
2. Extract `code` parameter
3. Prepare for token exchange

**Security**: Authorization code is single-use and expires quickly. Even if intercepted, it's useless without client credentials.

---

### Step 9: Token Request

**Your app makes a server-side request** (NOT in browser):

```http
POST https://auth.example.org/token
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code&
code=auth_code_xyz789&
redirect_uri=https://your-app.com/callback&
client_id=your_app_id&
client_secret=your_app_secret
```

**For public clients** (can't keep secrets), use PKCE instead of client_secret:

```http
POST https://auth.example.org/token
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code&
code=auth_code_xyz789&
redirect_uri=https://your-app.com/callback&
client_id=your_app_id&
code_verifier=random_verifier_from_pkce
```

**Parameters**:
- **`grant_type`**: `authorization_code` (trading code for token)
- **`code`**: Authorization code from step 8
- **`redirect_uri`**: Must match exactly from step 3
- **`client_id`**: Your app identifier
- **`client_secret`**: Your app secret (confidential clients only)
- **`code_verifier`**: PKCE verifier (public clients only)

---

### Step 10: Validate and Create Context

**Authorization server**:
1. Validates authorization code (not expired, not already used)
2. Verifies client credentials (client_secret or PKCE)
3. Confirms redirect_uri matches
4. Retrieves associated patient context
5. Generates access token (JWT)
6. Generates refresh token (if `offline_access` was approved)

**Access token contains**:
- User identity (sub claim)
- Approved scopes (scope claim)
- Token expiration (exp claim)
- Patient context (patient claim) - SMART extension
- Encounter context (encounter claim) - SMART extension
- FHIR user URL (fhirUser claim) - SMART extension

---

### Step 11: Access Token Response

**Authorization server returns tokens**:

```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "scope": "launch/patient patient/*.read offline_access openid fhirUser",
  "patient": "smart-1288992",
  "encounter": "smart-enc-456",
  "fhirUser": "https://fhir.example.org/Practitioner/dr-johnson",
  "refresh_token": "refresh_token_abc123",
  "id_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

**Token details**:

- **`access_token`**: JWT for API requests (typically expires in 1 hour)
- **`token_type`**: Always "Bearer" for OAuth 2.0
- **`expires_in`**: Seconds until token expires (3600 = 1 hour)
- **`scope`**: Actually granted scopes (may be subset of requested)
- **`patient`**: Patient ID in context (SMART extension)
- **`encounter`**: Encounter ID if available (SMART extension)
- **`fhirUser`**: FHIR resource URL for authenticated user (SMART extension)
- **`refresh_token`**: Long-lived token to get new access tokens
- **`id_token`**: OpenID Connect identity token (if `openid` scope granted)

**Your app stores**: Access token (in memory), refresh token (securely), patient ID, expiration time.

---

### Step 12: API Request with Token

**Your app makes FHIR API calls**:

```http
GET https://fhir.example.org/Patient/smart-1288992
Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...
Accept: application/fhir+json
```

**Header format**: `Authorization: Bearer {access_token}`

---

### Step 13: Validate Token and Scope

**FHIR server**:
1. Extracts Bearer token from Authorization header
2. Validates JWT signature (using public key from authorization server)
3. Checks expiration (exp claim)
4. Verifies audience (aud claim matches FHIR server URL)
5. Checks scope includes `patient/Patient.read` or `patient/*.read`
6. Verifies patient ID in URL matches patient claim in token

**If any validation fails**: Returns 401 Unauthorized or 403 Forbidden with OperationOutcome.

---

### Step 14: Return Resource

**FHIR server returns requested data**:

```json
{
  "resourceType": "Patient",
  "id": "smart-1288992",
  "name": [{
    "family": "Shaw",
    "given": ["Amy", "Victoria"]
  }],
  "gender": "female",
  "birthDate": "1987-02-20"
}
```

---

## Token Refresh Flow (Step 15-16)

### Step 15: Refresh Token Request

**When access token expires**, your app requests a new one:

```http
POST https://auth.example.org/token
Content-Type: application/x-www-form-urlencoded

grant_type=refresh_token&
refresh_token=refresh_token_abc123&
client_id=your_app_id&
client_secret=your_app_secret
```

**Parameters**:
- **`grant_type`**: `refresh_token`
- **`refresh_token`**: From step 11 response
- **`client_id`**: Your app identifier
- **`client_secret`**: Your app secret (if confidential client)

---

### Step 16: New Access Token

**Authorization server returns new tokens**:

```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "scope": "launch/patient patient/*.read offline_access openid fhirUser",
  "patient": "smart-1288992",
  "refresh_token": "new_refresh_token_def456"
}
```

**Important notes**:
- New access token with fresh expiration
- May include new refresh token (some servers rotate them)
- Patient context remains the same
- No user interaction required

---

## OAuth 2.0 Flow Diagram

**Keep your existing Mermaid diagram** (`deepdive-oauth-flow.md`). It provides excellent visual reference. The narrative above explains WHY each step exists and HOW to implement it.

---

## Security Considerations

### PKCE (Proof Key for Code Exchange)

**Problem**: Public clients (mobile apps, SPAs) can't keep client_secret.

**Solution**: PKCE adds cryptographic proof to prevent code interception.

**How it works**:

**Step 3 (Authorization Request)** - Generate random values:

```javascript
const codeVerifier = generateRandomString(128); // Store securely
const codeChallenge = base64url(sha256(codeVerifier));

// Add to authorization URL
const authUrl = `${authEndpoint}?` +
  `response_type=code&` +
  `client_id=${clientId}&` +
  `redirect_uri=${redirectUri}&` +
  `code_challenge=${codeChallenge}&` +
  `code_challenge_method=S256&` +
  `state=${state}`;
```

**Step 9 (Token Request)** - Send verifier:

```http
POST /token
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code&
code=auth_code_xyz&
redirect_uri=https://your-app.com/callback&
client_id=your_app_id&
code_verifier=random_verifier_from_pkce
```

**Authorization server validates**: `sha256(code_verifier) == code_challenge`

**Why it works**: Even if attacker intercepts authorization code, they can't use it without the verifier.

---

### State Parameter (CSRF Protection)

**Attack scenario**: Malicious site tricks user into authorizing attacker's session.

**Prevention**: `state` parameter ties authorization request to your app's session.

**Implementation**:

```javascript
// Step 3: Generate and store state
const state = generateRandomString(32);
sessionStorage.setItem('oauth_state', state);

// Step 8: Validate state in callback
const receivedState = new URLSearchParams(window.location.search).get('state');
const storedState = sessionStorage.getItem('oauth_state');

if (receivedState !== storedState) {
  throw new Error('CSRF attack detected!');
}
```

---

### Token Storage

**Never store tokens in**:
- LocalStorage (vulnerable to XSS attacks)
- URL parameters (visible in browser history)
- Cookies without httpOnly flag (XSS risk)

**Best practices**:
- **Access tokens**: Store in memory (JavaScript variable)
- **Refresh tokens**: Store in httpOnly, secure cookies (server-side) or encrypted storage (mobile)
- **Clear tokens on logout**: Invalidate session completely

---

### Token Expiration

**Access tokens should expire quickly** (1 hour typical).

**Why**:
- Limits damage if token is stolen
- Forces periodic re-validation
- Allows policy changes to take effect

**Refresh tokens can be long-lived** (days to months).

**Why**:
- Avoids frequent user re-authentication
- Enables offline access
- Can be revoked when needed

---

## Common OAuth Errors

### invalid_request

**Cause**: Malformed request (missing parameter, wrong format)

**Example**:

```json
{
  "error": "invalid_request",
  "error_description": "Missing required parameter: client_id"
}
```

**Fix**: Double-check all required parameters.

---

### invalid_client

**Cause**: Invalid client credentials

**Example**:

```json
{
  "error": "invalid_client",
  "error_description": "Client authentication failed"
}
```

**Fix**: Verify client_id and client_secret are correct.

---

### invalid_grant

**Cause**: Authorization code expired or already used

**Example**:

```json
{
  "error": "invalid_grant",
  "error_description": "Authorization code has expired"
}
```

**Fix**: Authorization codes expire quickly (5 minutes). Don't delay token exchange.

---

### unauthorized_client

**Cause**: Client not authorized for this grant type

**Example**:

```json
{
  "error": "unauthorized_client",
  "error_description": "Client not authorized for refresh_token grant"
}
```

**Fix**: Request `offline_access` scope for refresh tokens.

---

### access_denied

**Cause**: User denied consent

**Example**:

```json
{
  "error": "access_denied",
  "error_description": "User denied authorization"
}
```

**Fix**: This is normal—user chose not to authorize. Handle gracefully in UI.

---

## Next Steps

Next, let's explore SMART's healthcare-specific launch patterns.