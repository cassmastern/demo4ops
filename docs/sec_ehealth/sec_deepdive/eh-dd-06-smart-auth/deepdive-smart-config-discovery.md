# SMART Configuration Discovery

## Dynamic Capability Discovery

Different FHIR servers use different OAuth endpoints, support different scopes, and implement different features. Your app can't hardcode these details—you need **dynamic discovery**.

SMART defines a standard endpoint where servers publish their capabilities: `.well-known/smart-configuration`

Think of it as the server's "capabilities business card" that tells your app everything it needs to know to integrate.

---

## The Discovery Endpoint

**Format**: `{FHIR_BASE_URL}/.well-known/smart-configuration`

**Examples**:

```
https://fhir.epic.com/interconnect-fhir-oauth/api/FHIR/R4/.well-known/smart-configuration
https://launch.smarthealthit.org/v/r4/fhir/.well-known/smart-configuration
https://fhir-myrecord.cerner.com/r4/.well-known/smart-configuration
```

**Request**:

```http
GET /.well-known/smart-configuration
Accept: application/json
```

**No authentication required**: Discovery is public information.

---

## Example Configuration

Here is a SMART configuration example:

```json
{
  "authorization_endpoint": "https://launch.smarthealthit.org/v/r4/auth/authorize",
  "token_endpoint": "https://launch.smarthealthit.org/v/r4/auth/token",
  "token_endpoint_auth_methods_supported": [
    "client_secret_basic",
    "client_secret_post"
  ],
  "registration_endpoint": "https://launch.smarthealthit.org/v/r4/auth/register",
  "scopes_supported": [
    "openid",
    "fhirUser",
    "launch",
    "launch/patient",
    "launch/encounter",
    "offline_access",
    "online_access",
    "patient/*.read",
    "patient/*.write",
    "patient/*.*",
    "user/*.read",
    "user/*.write",
    "user/*.*",
    "patient/Patient.read",
    "patient/Observation.read",
    "patient/Observation.write",
    "patient/Condition.read",
    "patient/Condition.write",
    "patient/MedicationRequest.read",
    "patient/AllergyIntolerance.read",
    "patient/Procedure.read",
    "patient/Encounter.read",
    "patient/Immunization.read",
    "patient/DiagnosticReport.read",
    "patient/DocumentReference.read",
    "patient/CarePlan.read",
    "user/Patient.read",
    "user/Practitioner.read",
    "user/Organization.read"
  ],
  "response_types_supported": [
    "code",
    "code id_token",
    "id_token"
  ],
  "capabilities": [
    "launch-ehr",
    "launch-standalone",
    "client-public",
    "client-confidential-symmetric",
    "context-ehr-patient",
    "context-ehr-encounter",
    "context-standalone-patient",
    "context-standalone-encounter",
    "permission-offline",
    "permission-patient",
    "permission-user",
    "sso-openid-connect"
  ],
  "code_challenge_methods_supported": [
    "S256"
  ],
  "grant_types_supported": [
    "authorization_code",
    "refresh_token"
  ],
  "introspection_endpoint": "https://launch.smarthealthit.org/v/r4/auth/introspect",
  "revocation_endpoint": "https://launch.smarthealthit.org/v/r4/auth/revoke",
  "management_endpoint": "https://launch.smarthealthit.org/v/r4/auth/manage"
}
```

!!! note
    `response_types_supported` describes which **OAuth/OIDC response types** the authorization endpoint supports (for SMART, `code` is the most important). A refresh token is **not** a response type; refresh is represented via `grant_types_supported: ["refresh_token"]` and (typically) the SMART scope `offline_access` plus the capability `permission-offline`.

!!! note
    Some discovery documents contain extra, vendor-specific fields (for example `management_endpoint`). Treat unknown fields as optional hints—not requirements—and rely on the standardized ones for interoperability.

Let's break down each section:

### OAuth Endpoints

```json
{
  "authorization_endpoint": "https://launch.smarthealthit.org/v/r4/auth/authorize",
  "token_endpoint": "https://launch.smarthealthit.org/v/r4/auth/token"
}
```

**Critical fields**:
- **`authorization_endpoint`**: Where to redirect user for login/consent
- **`token_endpoint`**: Where to exchange authorization code for token

**Your app uses these** instead of hardcoding URLs.

---

### Supported Scopes

```json
{
  "scopes_supported": [
    "openid",
    "fhirUser",
    "launch",
    "launch/patient",
    "launch/encounter",
    "offline_access",
    "patient/*.read",
    "patient/*.write",
    "patient/Patient.read",
    "patient/Observation.read",
    "user/*.read",
    "user/Patient.read"
  ]
}
```

**What this tells you**:
- Server supports EHR launch (`launch`, `launch/patient`)
- Supports wildcards (`patient/*.read`)
- Supports granular scopes (`patient/Observation.read`)
- Offers offline access (`offline_access`)

**Your app checks**: Before requesting a scope, verify it's in `scopes_supported`.

```javascript
function isScopeSupported(config, scope) {
  return config.scopes_supported?.includes(scope) || false;
}

// Usage
if (isScopeSupported(config, 'offline_access')) {
  scopes.push('offline_access');
}
```

---

### Server Capabilities

```json
{
  "capabilities": [
    "launch-ehr",
    "launch-standalone",
    "client-public",
    "client-confidential-symmetric",
    "context-ehr-patient",
    "context-ehr-encounter",
    "context-standalone-patient",
    "permission-offline",
    "permission-patient",
    "permission-user",
    "sso-openid-connect"
  ]
}
```

**Key capabilities**:

- **`launch-ehr`**: Supports EHR-initiated launch
- **`launch-standalone`**: Supports standalone (patient-initiated) launch
- **`client-public`**: Supports public clients (mobile apps, SPAs) with PKCE
- **`client-confidential-symmetric`**: Supports confidential clients with client_secret
- **`context-ehr-patient`**: Provides patient context in EHR launch
- **`context-ehr-encounter`**: Provides encounter context in EHR launch
- **`permission-offline`**: Supports refresh tokens
- **`permission-patient`**: Supports patient-scoped access
- **`permission-user`**: Supports user-scoped access
- **`sso-openid-connect`**: Supports OpenID Connect SSO

**Your app adapts**:

```javascript
if (config.capabilities.includes('launch-ehr')) {
  showEHRLaunchInstructions();
}

if (!config.capabilities.includes('permission-offline')) {
  warnUserAboutPeriodicReauth();
}
```

---

### Authentication Methods

```json
{
  "token_endpoint_auth_methods_supported": [
    "client_secret_basic",
    "client_secret_post"
  ]
}
```

**Methods**:
- **`client_secret_basic`**: HTTP Basic authentication (client_id:client_secret in Authorization header)
- **`client_secret_post`**: Credentials in POST body
- **`private_key_jwt`**: Asymmetric JWT authentication (for backend services)

**Your app uses**: Appropriate method based on server support and client type.

---

### PKCE Support

```json
{
  "code_challenge_methods_supported": [
    "S256"
  ]
}
```

**`S256`**: SHA-256 hash for PKCE (most secure).

**Your public client must use PKCE** if server supports it.

---

### Grant Types

```json
{
  "grant_types_supported": [
    "authorization_code",
    "refresh_token"
  ]
}
```

**Common types**:
- **`authorization_code`**: Standard OAuth flow
- **`refresh_token`**: Token refresh capability
- **`client_credentials`**: Backend service authentication

---

### Optional Endpoints

```json
{
  "registration_endpoint": "https://launch.smarthealthit.org/v/r4/auth/register",
  "introspection_endpoint": "https://launch.smarthealthit.org/v/r4/auth/introspect",
  "revocation_endpoint": "https://launch.smarthealthit.org/v/r4/auth/revoke",
  "management_endpoint": "https://launch.smarthealthit.org/v/r4/auth/manage"
}
```

**Endpoints**:
- **`registration_endpoint`**: Dynamic client registration
- **`introspection_endpoint`**: Token validation
- **`revocation_endpoint`**: Revoke tokens
- **`management_endpoint`**: Manage authorizations

---

## Using Configuration in Your App

### Step 1: Fetch Configuration

```javascript
async function discoverSMARTConfig(issuer) {
  const configUrl = `${issuer}/.well-known/smart-configuration`;
  
  try {
    const response = await fetch(configUrl);
    if (!response.ok) {
      throw new Error(`Configuration discovery failed: ${response.status}`);
    }
    return await response.json();
  } catch (error) {
    console.error('Discovery failed:', error);
    throw error;
  }
}

// Usage
const config = await discoverSMARTConfig('https://fhir.example.org');
```

---

### Step 2: Cache Configuration

```javascript
class SMARTClient {
  constructor(issuer) {
    this.issuer = issuer;
    this.config = null;
  }
  
  async getConfig() {
    if (!this.config) {
      this.config = await discoverSMARTConfig(this.issuer);
    }
    return this.config;
  }
  
  async authorize(options) {
    const config = await this.getConfig();
    
    // Build authorization URL using discovered endpoints
    const authUrl = `${config.authorization_endpoint}?` +
      `response_type=code&` +
      `client_id=${options.clientId}&` +
      `redirect_uri=${encodeURIComponent(options.redirectUri)}&` +
      `scope=${encodeURIComponent(options.scope)}&` +
      `state=${options.state}&` +
      `aud=${encodeURIComponent(this.issuer)}`;
    
    window.location.href = authUrl;
  }
}

// Usage
const client = new SMARTClient('https://fhir.example.org');
await client.authorize({
  clientId: 'your_app_id',
  redirectUri: 'https://your-app.com/callback',
  scope: 'launch launch/patient patient/*.read',
  state: 'random_string'
});
```

---

### Step 3: Validate Capabilities

```javascript
async function validateServerCapabilities(issuer, requiredCapabilities) {
  const config = await discoverSMARTConfig(issuer);
  const missing = [];
  
  for (const cap of requiredCapabilities) {
    if (!config.capabilities?.includes(cap)) {
      missing.push(cap);
    }
  }
  
  if (missing.length > 0) {
    console.warn(`Server missing capabilities: ${missing.join(', ')}`);
    return false;
  }
  
  return true;
}

// Usage
const required = ['launch-ehr', 'context-ehr-patient', 'permission-patient'];
const supported = await validateServerCapabilities('https://fhir.example.org', required);

if (!supported) {
  alert('This FHIR server does not support required features');
}
```

---

## Fallback to CapabilityStatement

**If `.well-known/smart-configuration` fails**, try FHIR CapabilityStatement:

```javascript
async function discoverOAuthEndpoints(issuer) {
  // Try SMART configuration first
  try {
    return await discoverSMARTConfig(issuer);
  } catch (error) {
    console.warn('SMART configuration not found, trying CapabilityStatement');
  }
  
  // Fallback to CapabilityStatement
  try {
    const response = await fetch(`${issuer}/metadata`);
    const capability = await response.json();
    
    // Extract OAuth URLs from security extension
    const security = capability.rest[0]?.security;
    const oauthExt = security?.extension?.find(
      ext => ext.url === 'http://fhir-registry.smarthealthit.org/StructureDefinition/oauth-uris'
    );
    
    if (oauthExt) {
      const authEndpoint = oauthExt.extension.find(e => e.url === 'authorize')?.valueUri;
      const tokenEndpoint = oauthExt.extension.find(e => e.url === 'token')?.valueUri;
      
      return {
        authorization_endpoint: authEndpoint,
        token_endpoint: tokenEndpoint
      };
    }
  } catch (error) {
    console.error('CapabilityStatement fallback failed:', error);
  }
  
  throw new Error('Could not discover OAuth endpoints');
}
```

---

## Best Practices

### 1. Always Discover, Never Hardcode

```javascript
// BAD
const AUTH_ENDPOINT = 'https://fhir.example.org/authorize';

// GOOD
const config = await discoverSMARTConfig(issuer);
const authEndpoint = config.authorization_endpoint;
```

---

### 2. Cache Configuration Per Session

```javascript
// Store in sessionStorage to avoid repeated requests
const cacheKey = `smart-config-${issuer}`;
let config = JSON.parse(sessionStorage.getItem(cacheKey));

if (!config) {
  config = await discoverSMARTConfig(issuer);
  sessionStorage.setItem(cacheKey, JSON.stringify(config));
}
```

---

### 3. Handle Discovery Failures Gracefully

```javascript
try {
  const config = await discoverSMARTConfig(issuer);
} catch (error) {
  showErrorMessage(
    'Unable to connect to FHIR server. Please check the URL and try again.'
  );
  return;
}
```

---

### 4. Validate Before Requesting Unsupported Features

```javascript
const scopes = ['launch', 'launch/patient', 'patient/*.read'];

// Only request offline_access if supported
if (config.scopes_supported.includes('offline_access')) {
  scopes.push('offline_access');
}
```

---

## Next Steps

Next, let's explore the multiple layers of security protecting patient data.
