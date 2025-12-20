# Passkeys - Passwordless Authentication

<div align="center">

![Passkeys](https://img.shields.io/badge/Passkeys-4285F4?style=for-the-badge&logo=google&logoColor=white)
![WebAuthn](https://img.shields.io/badge/WebAuthn-000000?style=for-the-badge)
![Security](https://img.shields.io/badge/Security-Phishing_Resistant-green?style=for-the-badge)

*Replace passwords with biometrics and device security - phishing-resistant by design.*

![Passkeys](https://upload.wikimedia.org/wikipedia/commons/4/45/Security_lock_blue.svg)

</div>

## Why Passkeys?

```
Passwords                      Passkeys
──────────────────────────     ──────────────────────────
Can be phished                 Phishing-resistant
Can be weak                    Strong cryptography
Reused across sites            Unique per site
Stored on servers              Private key never leaves device
Forgotten often                Biometric or PIN
```

## How It Works

```
Registration:
┌────────┐     ┌────────┐     ┌────────┐
│ Server │────▶│Browser │────▶│Device  │
│        │     │        │     │(Touch/ │
│Challenge│     │WebAuthn│     │Face ID)│
└────────┘     └────────┘     └────────┘
     ▲                              │
     └──────── Public Key ──────────┘

Authentication:
┌────────┐     ┌────────┐     ┌────────┐
│ Server │────▶│Browser │────▶│Device  │
│        │     │        │     │Signs   │
│Challenge│     │WebAuthn│     │w/Priv  │
└────────┘     └────────┘     └────────┘
     ▲                              │
     └──────── Signature ───────────┘
```

## SimpleWebAuthn Library

```bash
npm install @simplewebauthn/server @simplewebauthn/browser
```

## Server: Registration

```typescript
// app/api/auth/passkey/register/options/route.ts
import {
  generateRegistrationOptions,
  verifyRegistrationResponse,
} from '@simplewebauthn/server'
import type { AuthenticatorDevice } from '@simplewebauthn/server'

const rpName = 'My App'
const rpID = 'localhost' // Your domain
const origin = 'http://localhost:3000'

export async function POST(req: Request) {
  const { userId, username } = await req.json()

  // Get user's existing passkeys
  const userPasskeys = await getUserPasskeys(userId)

  const options = await generateRegistrationOptions({
    rpName,
    rpID,
    userID: userId,
    userName: username,
    attestationType: 'none',
    excludeCredentials: userPasskeys.map((passkey) => ({
      id: passkey.credentialID,
      type: 'public-key',
      transports: passkey.transports,
    })),
    authenticatorSelection: {
      residentKey: 'preferred',
      userVerification: 'preferred',
      authenticatorAttachment: 'platform', // or 'cross-platform'
    },
  })

  // Store challenge for verification
  await storeChallenge(userId, options.challenge)

  return Response.json(options)
}
```

```typescript
// app/api/auth/passkey/register/verify/route.ts
export async function POST(req: Request) {
  const { userId, response } = await req.json()

  const expectedChallenge = await getStoredChallenge(userId)

  const verification = await verifyRegistrationResponse({
    response,
    expectedChallenge,
    expectedOrigin: origin,
    expectedRPID: rpID,
  })

  if (verification.verified && verification.registrationInfo) {
    const { credentialPublicKey, credentialID, counter } =
      verification.registrationInfo

    // Save passkey to database
    await savePasskey(userId, {
      credentialID,
      credentialPublicKey,
      counter,
      transports: response.response.transports,
    })
  }

  return Response.json({ verified: verification.verified })
}
```

## Server: Authentication

```typescript
// app/api/auth/passkey/login/options/route.ts
import { generateAuthenticationOptions } from '@simplewebauthn/server'

export async function POST(req: Request) {
  const { username } = await req.json()

  const user = await getUserByUsername(username)
  const userPasskeys = await getUserPasskeys(user.id)

  const options = await generateAuthenticationOptions({
    rpID,
    allowCredentials: userPasskeys.map((passkey) => ({
      id: passkey.credentialID,
      type: 'public-key',
      transports: passkey.transports,
    })),
    userVerification: 'preferred',
  })

  await storeChallenge(user.id, options.challenge)

  return Response.json(options)
}
```

```typescript
// app/api/auth/passkey/login/verify/route.ts
import { verifyAuthenticationResponse } from '@simplewebauthn/server'

export async function POST(req: Request) {
  const { username, response } = await req.json()

  const user = await getUserByUsername(username)
  const expectedChallenge = await getStoredChallenge(user.id)
  const passkey = await getPasskeyByCredentialID(response.id)

  const verification = await verifyAuthenticationResponse({
    response,
    expectedChallenge,
    expectedOrigin: origin,
    expectedRPID: rpID,
    authenticator: {
      credentialID: passkey.credentialID,
      credentialPublicKey: passkey.credentialPublicKey,
      counter: passkey.counter,
    },
  })

  if (verification.verified) {
    // Update counter to prevent replay attacks
    await updatePasskeyCounter(passkey.id, verification.authenticationInfo.newCounter)

    // Create session
    const session = await createSession(user.id)

    return Response.json({ verified: true, sessionId: session.id })
  }

  return Response.json({ verified: false }, { status: 401 })
}
```

## Client: Registration

```typescript
'use client'
import { startRegistration } from '@simplewebauthn/browser'

async function registerPasskey() {
  // 1. Get options from server
  const optionsRes = await fetch('/api/auth/passkey/register/options', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: currentUser.id,
      username: currentUser.username,
    }),
  })
  const options = await optionsRes.json()

  // 2. Create passkey (triggers biometric/PIN)
  const registration = await startRegistration(options)

  // 3. Verify with server
  const verifyRes = await fetch('/api/auth/passkey/register/verify', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: currentUser.id,
      response: registration,
    }),
  })
  const { verified } = await verifyRes.json()

  if (verified) {
    alert('Passkey registered!')
  }
}
```

## Client: Authentication

```typescript
'use client'
import { startAuthentication } from '@simplewebauthn/browser'

async function loginWithPasskey(username: string) {
  // 1. Get authentication options
  const optionsRes = await fetch('/api/auth/passkey/login/options', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ username }),
  })
  const options = await optionsRes.json()

  // 2. Authenticate (triggers biometric/PIN)
  const authentication = await startAuthentication(options)

  // 3. Verify with server
  const verifyRes = await fetch('/api/auth/passkey/login/verify', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      username,
      response: authentication,
    }),
  })
  const { verified, sessionId } = await verifyRes.json()

  if (verified) {
    // Store session, redirect, etc.
    router.push('/dashboard')
  }
}
```

## Conditional UI (Autofill)

```typescript
import { browserSupportsWebAuthnAutofill, startAuthentication } from '@simplewebauthn/browser'

async function setupPasskeyAutofill() {
  if (!await browserSupportsWebAuthnAutofill()) return

  // Get options for autofill
  const options = await fetch('/api/auth/passkey/autofill-options').then(r => r.json())

  // Start authentication with autofill
  const auth = await startAuthentication(options, true) // true = use autofill

  // Verify and sign in
  await verifyAndSignIn(auth)
}

// Input with webauthn support
<input
  type="text"
  name="username"
  autoComplete="username webauthn"
/>
```

## Database Schema

```sql
CREATE TABLE passkeys (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id) ON DELETE CASCADE,
  credential_id BYTEA UNIQUE NOT NULL,
  credential_public_key BYTEA NOT NULL,
  counter INTEGER NOT NULL DEFAULT 0,
  transports TEXT[],
  created_at TIMESTAMP DEFAULT NOW(),
  last_used_at TIMESTAMP
);

CREATE INDEX idx_passkeys_user ON passkeys(user_id);
CREATE INDEX idx_passkeys_credential ON passkeys(credential_id);
```

## React Component

```typescript
function PasskeySettings() {
  const [passkeys, setPasskeys] = useState<Passkey[]>([])
  const [isSupported, setIsSupported] = useState(false)

  useEffect(() => {
    // Check browser support
    setIsSupported(
      typeof window !== 'undefined' &&
      window.PublicKeyCredential !== undefined
    )
    fetchPasskeys()
  }, [])

  if (!isSupported) {
    return <p>Passkeys not supported in this browser</p>
  }

  return (
    <div>
      <h2>Your Passkeys</h2>
      {passkeys.map((passkey) => (
        <div key={passkey.id}>
          <span>Added {passkey.createdAt}</span>
          <button onClick={() => deletePasskey(passkey.id)}>Remove</button>
        </div>
      ))}
      <button onClick={registerPasskey}>Add Passkey</button>
    </div>
  )
}
```

---

*Learned: December 20, 2025*
*Tags: Passkeys, WebAuthn, Authentication, Security, Passwordless*
