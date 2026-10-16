# Zero to Hero: Authentication — Microsoft, Google, Amazon, OTP & Beyond

## Overview

This guide covers the implementation of modern authentication across all major providers — Microsoft Entra ID (Azure AD) with Graph API, Google OAuth2, Amazon Cognito, and OTP/TOTP — with flow diagrams and code examples in both Elixir and Python. The final section provides advanced technical deep-dives into OTP internals, Active Directory architecture, and the Microsoft Graph API model.

---

# Part 1: Microsoft Authentication (Entra ID / Azure AD / Graph API)

## Key Concepts

### The Microsoft Identity Ecosystem

| Term | What It Is |
|---|---|
| **Microsoft Entra ID** | The current name for Azure Active Directory (Azure AD). It is the identity and access management (IAM) service. |
| **Azure AD** | The legacy name for Entra ID. Same service, rebranded in 2023. |
| **Microsoft Identity Platform** | The OAuth 2.0 / OIDC authorization server that issues tokens. Endpoints: `login.microsoftonline.com`. |
| **Microsoft Graph API** | A REST API gateway to Microsoft 365 data (users, emails, calendars, Teams, SharePoint). Protected by Entra ID tokens. |
| **MSAL** | Microsoft Authentication Library — the official SDK for token acquisition. Available for .NET, Python, JS, Java. |
| **App Registration** | Every app that authenticates via Entra ID must be registered in the Entra admin center to get a Client ID and configure permissions. |

### Two Authentication Flows

| Flow | Use Case | Grant Type |
|---|---|---|
| **Delegated (user)** | Web app signs in a user, acts on their behalf | Authorization Code + PKCE |
| **Application (daemon)** | Background service, no user present | Client Credentials |

### Permissions Model

| Permission Type | Set By | Example |
|---|---|---|
| **Delegated** | User consents at login | `User.Read`, `Mail.Read` |
| **Application** | Admin grants in portal | `User.Read.All`, `Mail.Read` |

## How It Works

### Authorization Code Flow (Delegated — User Sign-In)

```
┌──────────┐     1. Redirect to /authorize     ┌───────────────────┐
│  Browser  │──────────────────────────────────▶│  Microsoft Entra   │
│  (User)   │                                   │  Identity Platform │
└──────────┘                                    └────────┬──────────┘
      ▲                                                  │
      │  2. User logs in, consents                       │
      │  3. Redirect back with ?code=AUTH_CODE            │
      │◀─────────────────────────────────────────────────┘
      │
      ▼
┌──────────┐     4. POST /token                 ┌───────────────────┐
│  Your App │       {code, client_secret,        │  Token Endpoint    │
│  Backend  │        redirect_uri, PKCE}         │                   │
│           │──────────────────────────────────▶│                   │
│           │◀──────────────────────────────────│  Returns:          │
│           │     5. {access_token,              │  access_token      │
│           │         id_token,                  │  id_token          │
│           │         refresh_token}             │  refresh_token     │
└─────┬─────┘                                   └───────────────────┘
      │
      │  6. GET /v1.0/me
      │     Authorization: Bearer {access_token}
      ▼
┌───────────────────┐
│  Microsoft Graph   │
│  API               │
│  Returns user data │
└───────────────────┘
```

### Client Credentials Flow (Application — No User)

```
┌──────────┐     1. POST /token                 ┌───────────────────┐
│  Daemon / │       {client_id, client_secret,   │  Token Endpoint    │
│  Service  │        scope=graph/.default,        │                   │
│           │        grant_type=client_credentials}                   │
│           │──────────────────────────────────▶│                   │
│           │◀──────────────────────────────────│  Returns:          │
│           │     2. {access_token}              │  access_token      │
└─────┬─────┘                                   └───────────────────┘
      │
      │  3. GET /v1.0/users
      │     Authorization: Bearer {access_token}
      ▼
┌───────────────────┐
│  Microsoft Graph   │
│  API               │
└───────────────────┘
```

## Code Example

### Elixir — Microsoft Auth with Ueberauth

```elixir
# mix.exs
defp deps do
  [
    {:ueberauth, "~> 0.10"},
    {:ueberauth_microsoft, "~> 0.25"},
    {:req, "~> 0.5"}
  ]
end
```

```elixir
# config/config.exs
config :ueberauth, Ueberauth,
  providers: [
    microsoft: {Ueberauth.Strategy.Microsoft, [prompt: "select_account"]}
  ]

config :ueberauth, Ueberauth.Strategy.Microsoft.OAuth,
  client_id: System.get_env("MICROSOFT_CLIENT_ID"),
  client_secret: System.get_env("MICROSOFT_CLIENT_SECRET"),
  tenant_id: System.get_env("MICROSOFT_TENANT_ID")
```

```elixir
# lib/my_app_web/controllers/auth_controller.ex
defmodule MyAppWeb.AuthController do
  use MyAppWeb, :controller
  plug Ueberauth

  def callback(%{assigns: %{ueberauth_auth: auth}} = conn, _params) do
    user_info = %{
      name: auth.info.name,
      email: auth.info.email,
      access_token: auth.credentials.token,
      refresh_token: auth.credentials.refresh_token,
      expires_at: auth.credentials.expires_at
    }

    # Store user session or create account
    conn
    |> put_session(:user, user_info)
    |> redirect(to: "/dashboard")
  end

  def callback(%{assigns: %{ueberauth_failure: failure}} = conn, _params) do
    conn
    |> put_flash(:error, "Authentication failed: #{inspect(failure.errors)}")
    |> redirect(to: "/")
  end
end
```

```elixir
# Calling Microsoft Graph API after authentication
defmodule MyApp.Graph do
  def get_user_profile(access_token) do
    case Req.get("https://graph.microsoft.com/v1.0/me",
           headers: [{"authorization", "Bearer #{access_token}"}]
         ) do
      {:ok, %Req.Response{status: 200, body: body}} ->
        {:ok, body}

      {:ok, %Req.Response{status: 401}} ->
        {:error, :token_expired}

      {:ok, %Req.Response{status: status, body: body}} ->
        {:error, {status, body["error"]["message"]}}
    end
  end

  def list_users(access_token) do
    case Req.get("https://graph.microsoft.com/v1.0/users",
           headers: [{"authorization", "Bearer #{access_token}"}]
         ) do
      {:ok, %Req.Response{status: 200, body: %{"value" => users}}} ->
        {:ok, users}

      {:ok, %Req.Response{status: status, body: body}} ->
        {:error, {status, body}}
    end
  end
end
```

### Python — Microsoft Auth with MSAL

```python
# pip install msal requests

from msal import ConfidentialClientApplication
import requests

# --- Authorization Code Flow (Delegated) ---

app = ConfidentialClientApplication(
    client_id="YOUR_CLIENT_ID",
    client_credential="YOUR_CLIENT_SECRET",
    authority="https://login.microsoftonline.com/YOUR_TENANT_ID"
)

# Step 1: Generate authorization URL
auth_url = app.get_authorization_request_url(
    scopes=["User.Read", "Mail.Read"],
    redirect_uri="http://localhost:8000/callback"
)
print(f"Visit: {auth_url}")

# Step 2: After user logs in and you receive the auth code
result = app.acquire_token_by_authorization_code(
    code="AUTH_CODE_FROM_CALLBACK",
    scopes=["User.Read", "Mail.Read"],
    redirect_uri="http://localhost:8000/callback"
)

if "access_token" in result:
    # Step 3: Call Graph API
    headers = {"Authorization": f"Bearer {result['access_token']}"}
    user = requests.get("https://graph.microsoft.com/v1.0/me", headers=headers)
    print(user.json())
else:
    print(f"Error: {result.get('error_description')}")


# --- Client Credentials Flow (Application / Daemon) ---

app = ConfidentialClientApplication(
    client_id="YOUR_CLIENT_ID",
    client_credential="YOUR_CLIENT_SECRET",
    authority="https://login.microsoftonline.com/YOUR_TENANT_ID"
)

result = app.acquire_token_for_client(
    scopes=["https://graph.microsoft.com/.default"]
)

if "access_token" in result:
    headers = {"Authorization": f"Bearer {result['access_token']}"}
    users = requests.get("https://graph.microsoft.com/v1.0/users", headers=headers)
    print(users.json())
```

---

# Part 2: Google OAuth2

## Key Concepts

| Term | What It Is |
|---|---|
| **Google Cloud Console** | Where you register apps and get OAuth credentials (APIs & Services > Credentials) |
| **Client ID / Client Secret** | Credentials issued per application |
| **Scopes** | Permissions like `openid`, `email`, `profile`, `https://www.googleapis.com/auth/gmail.readonly` |
| **PKCE** | Supported by Google, but client_secret is still required for web apps |

## How It Works

### Google OAuth2 Authorization Code Flow

```
┌──────────┐     1. Redirect to accounts.google.com   ┌──────────────────┐
│  Browser  │        /o/oauth2/v2/auth?                 │  Google Auth      │
│  (User)   │        client_id, redirect_uri,           │  Server           │
│           │        scope, state, code_challenge        │                   │
│           │─────────────────────────────────────────▶│                   │
└──────────┘                                            └────────┬─────────┘
      ▲                                                          │
      │  2. User logs in, consents                               │
      │  3. Redirect back with ?code=AUTH_CODE&state=STATE        │
      │◀─────────────────────────────────────────────────────────┘
      ▼
┌──────────┐     4. POST oauth2.googleapis.com/token    ┌──────────────────┐
│  Your App │       {code, client_id, client_secret,     │  Token Endpoint   │
│  Backend  │        redirect_uri, code_verifier}         │                   │
│           │─────────────────────────────────────────▶│                   │
│           │◀─────────────────────────────────────────│  Returns:          │
│           │     5. {access_token, id_token,           │  access_token      │
│           │         refresh_token, expires_in}        │  refresh_token     │
└─────┬─────┘                                           └──────────────────┘
      │
      │  6. GET googleapis.com/oauth2/v2/userinfo
      │     Authorization: Bearer {access_token}
      ▼
┌──────────────────┐
│  Google API       │
│  Returns profile  │
└──────────────────┘
```

## Code Example

### Elixir — Google Auth with Ueberauth

```elixir
# mix.exs
defp deps do
  [
    {:ueberauth, "~> 0.10"},
    {:ueberauth_google, "~> 0.12"},
    {:req, "~> 0.5"}
  ]
end
```

```elixir
# config/config.exs
config :ueberauth, Ueberauth,
  providers: [
    google: {Ueberauth.Strategy.Google, [default_scope: "email profile"]}
  ]

config :ueberauth, Ueberauth.Strategy.Google.OAuth,
  client_id: System.get_env("GOOGLE_CLIENT_ID"),
  client_secret: System.get_env("GOOGLE_CLIENT_SECRET")
```

```elixir
# Controller callback (same pattern as Microsoft)
def callback(%{assigns: %{ueberauth_auth: auth}} = conn, _params) do
  user_info = %{
    name: auth.info.name,
    email: auth.info.email,
    image: auth.info.image,
    access_token: auth.credentials.token
  }

  conn
  |> put_session(:user, user_info)
  |> redirect(to: "/dashboard")
end
```

### Python — Google Auth with google-auth

```python
# pip install google-auth google-auth-oauthlib requests

from google_auth_oauthlib.flow import Flow

flow = Flow.from_client_secrets_file(
    "client_secrets.json",
    scopes=["openid", "https://www.googleapis.com/auth/userinfo.email",
            "https://www.googleapis.com/auth/userinfo.profile"],
    redirect_uri="http://localhost:8000/callback"
)

# Step 1: Generate authorization URL
auth_url, state = flow.authorization_url(
    access_type="offline",
    include_granted_scopes="true",
    prompt="consent"
)
print(f"Visit: {auth_url}")

# Step 2: Exchange code for tokens
flow.fetch_token(code="AUTH_CODE_FROM_CALLBACK")
credentials = flow.credentials

# Step 3: Call Google API
import requests
headers = {"Authorization": f"Bearer {credentials.token}"}
userinfo = requests.get("https://www.googleapis.com/oauth2/v2/userinfo", headers=headers)
print(userinfo.json())
```

---

# Part 3: Amazon Cognito

## Key Concepts

| Term | What It Is |
|---|---|
| **User Pool** | A user directory with sign-up/sign-in, MFA, and OAuth2/OIDC token issuance |
| **Identity Pool** | Maps authenticated users to temporary AWS credentials (IAM roles) |
| **App Client** | Configuration within a User Pool defining allowed flows, scopes, and callback URLs |
| **Managed Login** | Hosted UI pages provided by Cognito for sign-in (customizable branding) |
| **PKCE** | Fully supported; recommended for all clients |

## How It Works

### Cognito Authorization Code Flow with PKCE

```
┌──────────┐     1. Redirect to {domain}/oauth2/authorize    ┌───────────────┐
│  Browser  │        ?response_type=code                       │  Cognito       │
│  (User)   │        &client_id=APP_CLIENT_ID                  │  Managed Login │
│           │        &redirect_uri=CALLBACK_URL                │                │
│           │        &code_challenge=CHALLENGE                 │                │
│           │        &code_challenge_method=S256               │                │
│           │───────────────────────────────────────────────▶│                │
└──────────┘                                                   └──────┬────────┘
      ▲                                                               │
      │  2. User authenticates (+ optional MFA)                       │
      │  3. Redirect back with ?code=AUTH_CODE                        │
      │◀──────────────────────────────────────────────────────────────┘
      ▼
┌──────────┐     4. POST {domain}/oauth2/token                ┌───────────────┐
│  Your App │        {code, redirect_uri,                      │  Token         │
│  Backend  │         code_verifier, client_id}                │  Endpoint      │
│           │───────────────────────────────────────────────▶│                │
│           │◀──────────────────────────────────────────────│  Returns:       │
│           │     5. {access_token, id_token,                │  access_token   │
│           │         refresh_token}                          │  id_token       │
└─────┬─────┘                                                 └───────────────┘
      │
      │  6. Use access_token for your API
      │     OR exchange id_token with Identity Pool for AWS credentials
      ▼
┌───────────────┐
│  Your API  /   │
│  AWS Services  │
└───────────────┘
```

## Code Example

### Python — Amazon Cognito with Warrant / Boto3

```python
# pip install boto3 requests

import boto3
import hashlib
import base64
import secrets
import urllib.parse

COGNITO_DOMAIN = "https://your-domain.auth.eu-central-1.amazoncognito.com"
CLIENT_ID = "your_app_client_id"
REDIRECT_URI = "http://localhost:8000/callback"

# Step 1: Generate PKCE pair
code_verifier = secrets.token_urlsafe(64)
code_challenge = base64.urlsafe_b64encode(
    hashlib.sha256(code_verifier.encode()).digest()
).rstrip(b"=").decode()

# Step 2: Build authorization URL
params = urllib.parse.urlencode({
    "response_type": "code",
    "client_id": CLIENT_ID,
    "redirect_uri": REDIRECT_URI,
    "scope": "openid email profile",
    "code_challenge": code_challenge,
    "code_challenge_method": "S256"
})
auth_url = f"{COGNITO_DOMAIN}/oauth2/authorize?{params}"
print(f"Visit: {auth_url}")

# Step 3: Exchange code for tokens
import requests
token_response = requests.post(
    f"{COGNITO_DOMAIN}/oauth2/token",
    data={
        "grant_type": "authorization_code",
        "code": "AUTH_CODE_FROM_CALLBACK",
        "redirect_uri": REDIRECT_URI,
        "client_id": CLIENT_ID,
        "code_verifier": code_verifier
    },
    headers={"Content-Type": "application/x-www-form-urlencoded"}
)
tokens = token_response.json()
print(f"Access Token: {tokens['access_token'][:50]}...")
```

### Elixir — Cognito via Ueberauth (or raw Req)

```elixir
# Using raw Req for Cognito token exchange
defmodule MyApp.Cognito do
  @cognito_domain "https://your-domain.auth.eu-central-1.amazoncognito.com"
  @client_id System.get_env("COGNITO_CLIENT_ID")
  @redirect_uri "http://localhost:4000/auth/callback"

  def generate_pkce do
    verifier = :crypto.strong_rand_bytes(64) |> Base.url_encode64(padding: false)

    challenge =
      :crypto.hash(:sha256, verifier)
      |> Base.url_encode64(padding: false)

    {verifier, challenge}
  end

  def authorization_url(code_challenge) do
    params =
      URI.encode_query(%{
        "response_type" => "code",
        "client_id" => @client_id,
        "redirect_uri" => @redirect_uri,
        "scope" => "openid email profile",
        "code_challenge" => code_challenge,
        "code_challenge_method" => "S256"
      })

    "#{@cognito_domain}/oauth2/authorize?#{params}"
  end

  def exchange_code(auth_code, code_verifier) do
    case Req.post("#{@cognito_domain}/oauth2/token",
           form: [
             grant_type: "authorization_code",
             code: auth_code,
             redirect_uri: @redirect_uri,
             client_id: @client_id,
             code_verifier: code_verifier
           ]
         ) do
      {:ok, %Req.Response{status: 200, body: tokens}} ->
        {:ok, tokens}

      {:ok, %Req.Response{status: status, body: body}} ->
        {:error, {status, body}}
    end
  end
end
```

---

# Part 4: OTP / TOTP Authentication

## Key Concepts

| Term | Standard | What It Is |
|---|---|---|
| **OTP** | — | One-Time Password — generic term for any single-use code |
| **HOTP** | RFC 4226 | HMAC-based OTP — counter-based, code valid until used |
| **TOTP** | RFC 6238 | Time-based OTP — extends HOTP, code changes every 30 seconds |
| **Shared Secret** | — | Base32-encoded key shared between server and authenticator app |
| **otpauth:// URI** | — | Standard format for QR codes scanned by authenticator apps |

## How It Works

### TOTP Registration and Verification Flow

```
┌──────────┐                                    ┌──────────────┐
│  Server   │  1. Generate shared secret         │  Database     │
│           │     (Base32, 160+ bits)             │               │
│           │────────────────────────────────────▶│  Store secret │
│           │                                    │  per user     │
│           │  2. Build otpauth:// URI            └──────────────┘
│           │     otpauth://totp/MyApp:user@email
│           │     ?secret=JBSWY3DPEHPK3PXP
│           │     &issuer=MyApp&digits=6&period=30
│           │
│           │  3. Render QR code from URI
│           │─────────────────────────────────────▶ Browser shows QR
└──────────┘
                                                 ┌──────────────┐
    User scans QR with Authenticator app ────────▶│  Auth App     │
                                                 │  (Google/MS)  │
                                                 │  Stores secret│
                                                 └──────┬───────┘
                                                        │
   ── Login Attempt ──                                   │
                                                        │
┌──────────┐     4. User enters 6-digit code    ┌──────┴───────┐
│  Server   │◀───────────────────────────────────│  Auth App     │
│           │                                    │  Computes:    │
│           │  5. Server independently computes  │  TOTP(secret, │
│           │     TOTP(secret, floor(time/30))   │  floor(t/30)) │
│           │                                    └──────────────┘
│           │  6. Compare codes (allow ±1 step
│           │     for clock drift)
│           │
│           │  7. Match → authenticated
│           │     No match → reject
└──────────┘
```

### TOTP Algorithm (RFC 6238)

```
C = floor((CurrentUnixTime - T0) / TimeStep)     # T0=0, TimeStep=30s
                    │
                    ▼
    HMAC = HMAC-SHA1(SharedSecret, C)             # 20-byte hash
                    │
                    ▼
    Offset = HMAC[19] & 0x0F                      # last nibble
                    │
                    ▼
    TruncatedHash = HMAC[Offset..Offset+3] & 0x7FFFFFFF
                    │
                    ▼
    OTP = TruncatedHash mod 10^Digits             # 6 digits → mod 1000000
```

## Code Example

### Elixir — TOTP with NimbleTOTP

```elixir
# mix.exs
defp deps do
  [
    {:nimble_totp, "~> 1.0"},
    {:eqrcode, "~> 0.1"}  # QR code generation
  ]
end
```

```elixir
defmodule MyApp.TwoFactor do
  @issuer "MyApp"

  def setup_totp(user_email) do
    secret = NimbleTOTP.secret()

    uri = NimbleTOTP.otpauth_uri("#{@issuer}:#{user_email}", secret,
      issuer: @issuer
    )

    qr_svg = uri |> EQRCode.encode() |> EQRCode.svg(width: 200)

    %{secret: secret, uri: uri, qr_svg: qr_svg}
  end

  def verify_totp(secret, user_code) do
    NimbleTOTP.valid?(secret, user_code)
  end
end

# Usage
setup = MyApp.TwoFactor.setup_totp("user@example.com")
# Store setup.secret in database (encrypted!)
# Render setup.qr_svg in your LiveView

# During login
MyApp.TwoFactor.verify_totp(stored_secret, "123456")
# => true or false
```

### Python — TOTP with pyotp

```python
# pip install pyotp qrcode

import pyotp
import qrcode

# Setup: Generate secret and QR code
secret = pyotp.random_base32()
totp = pyotp.TOTP(secret)

# Generate otpauth:// URI for QR code
uri = totp.provisioning_uri(name="user@example.com", issuer_name="MyApp")
# => "otpauth://totp/MyApp:user%40example.com?secret=JBSWY3DPEHPK3PXP&issuer=MyApp"

# Generate QR code image
qrcode.make(uri).save("totp_qr.png")

# Store `secret` in your database (encrypted!)

# Verification: During login
user_code = "123456"
is_valid = totp.verify(user_code, valid_window=1)  # ±1 time step tolerance
print(f"Valid: {is_valid}")

# Generate current code (for testing)
print(f"Current code: {totp.now()}")
```

---

# Part 5: Other Authentication Providers

## Auth0

```
┌──────────┐     1. Redirect to {tenant}.auth0.com/authorize  ┌────────────┐
│  Browser  │───────────────────────────────────────────────▶│  Auth0      │
│           │◀──────────── 2. ?code=AUTH_CODE ──────────────│  Universal  │
│           │                                                │  Login      │
└─────┬─────┘                                                └────────────┘
      ▼
┌──────────┐     3. POST {tenant}.auth0.com/oauth/token       ┌────────────┐
│  Backend  │───────────────────────────────────────────────▶│  Returns    │
│           │◀──────────── 4. {access, id, refresh} ────────│  tokens     │
└──────────┘                                                 └────────────┘
```

### Elixir — Auth0 with Ueberauth

```elixir
defp deps do
  [{:ueberauth_auth0, "~> 2.1"}]
end

# config.exs
config :ueberauth, Ueberauth,
  providers: [
    auth0: {Ueberauth.Strategy.Auth0, [
      domain: System.get_env("AUTH0_DOMAIN"),
      audience: System.get_env("AUTH0_AUDIENCE")
    ]}
  ]

config :ueberauth, Ueberauth.Strategy.Auth0.OAuth,
  domain: System.get_env("AUTH0_DOMAIN"),
  client_id: System.get_env("AUTH0_CLIENT_ID"),
  client_secret: System.get_env("AUTH0_CLIENT_SECRET")
```

## GitHub OAuth

```
┌──────────┐     1. github.com/login/oauth/authorize   ┌───────────┐
│  Browser  │─────────────────────────────────────────▶│  GitHub    │
│           │◀──────── 2. ?code=AUTH_CODE ─────────────│           │
└─────┬─────┘                                          └───────────┘
      ▼
┌──────────┐     3. POST github.com/login/oauth/       ┌───────────┐
│  Backend  │         access_token                      │  Returns   │
│           │─────────────────────────────────────────▶│  token     │
│           │◀──────── 4. {access_token} ──────────────│           │
│           │                                          └───────────┘
│           │     5. GET api.github.com/user
└──────────┘
```

### Elixir — GitHub

```elixir
defp deps do
  [{:ueberauth_github, "~> 0.8"}]
end

config :ueberauth, Ueberauth,
  providers: [github: {Ueberauth.Strategy.Github, [default_scope: "user:email"]}]

config :ueberauth, Ueberauth.Strategy.Github.OAuth,
  client_id: System.get_env("GITHUB_CLIENT_ID"),
  client_secret: System.get_env("GITHUB_CLIENT_SECRET")
```

## Apple Sign In

```
┌──────────┐     1. appleid.apple.com/auth/authorize   ┌───────────┐
│  Browser  │─────────────────────────────────────────▶│  Apple ID  │
│           │◀──────── 2. POST callback with           │  Server    │
│           │             id_token + code              │           │
└─────┬─────┘           (form_post response_mode)      └───────────┘
      ▼
┌──────────┐     3. POST appleid.apple.com/auth/token  ┌───────────┐
│  Backend  │          (use client_secret JWT)          │  Returns   │
│           │─────────────────────────────────────────▶│  tokens    │
└──────────┘                                           └───────────┘
```

Apple requires generating a `client_secret` as a signed JWT using your private key from the Apple Developer portal.

### Python — Apple

```python
# pip install pyjwt requests

import jwt
import time
import requests

# Generate client_secret JWT
private_key = open("AuthKey_XXXXXXXXXX.p8").read()
client_secret = jwt.encode(
    {
        "iss": "YOUR_TEAM_ID",
        "iat": int(time.time()),
        "exp": int(time.time()) + 86400 * 180,
        "aud": "https://appleid.apple.com",
        "sub": "YOUR_CLIENT_ID"
    },
    private_key,
    algorithm="ES256",
    headers={"kid": "YOUR_KEY_ID"}
)

# Exchange authorization code for tokens
response = requests.post("https://appleid.apple.com/auth/token", data={
    "grant_type": "authorization_code",
    "code": "AUTH_CODE",
    "redirect_uri": "https://your-app.com/callback",
    "client_id": "YOUR_CLIENT_ID",
    "client_secret": client_secret
})
tokens = response.json()
```

---

# Part 6: Provider Comparison Matrix

| Provider | Protocol | PKCE | MFA Built-in | User Directory | Free Tier |
|---|---|---|---|---|---|
| **Microsoft Entra ID** | OAuth 2.0 / OIDC | Yes | Yes (Authenticator, SMS, FIDO2) | Full AD/LDAP | 50,000 MAU |
| **Google** | OAuth 2.0 / OIDC | Yes (but requires client_secret for web) | Via Google account | Google accounts only | Unlimited (own accounts) |
| **Amazon Cognito** | OAuth 2.0 / OIDC | Yes | Yes (TOTP, SMS) | Built-in User Pool | 50,000 MAU |
| **Auth0** | OAuth 2.0 / OIDC | Yes | Yes (push, TOTP, SMS) | Built-in + social | 25,000 MAU |
| **GitHub** | OAuth 2.0 | No | Via GitHub account | GitHub accounts only | Unlimited |
| **Apple** | OAuth 2.0 / OIDC | No | Via Apple ID (biometric) | Apple accounts only | Unlimited |
| **TOTP/OTP** | RFC 6238/4226 | N/A | IS the MFA | N/A (add-on) | Free (self-hosted) |

---

# Part 7: Advanced Technical Deep-Dive

## OTP Internals (RFC 4226 / RFC 6238)

### The HOTP Algorithm (Foundation)

HOTP is the base algorithm. TOTP replaces the counter `C` with `floor(time / 30)`.

```
Input:
  K = Shared secret (≥128 bits, typically 160 bits / 20 bytes)
  C = Counter (8-byte big-endian integer)

Step 1: HMAC
  HS = HMAC-SHA1(K, C)
  # HS is a 20-byte (160-bit) string

Step 2: Dynamic Truncation
  Offset = HS[19] & 0x0F                        # value 0..15
  P = HS[Offset]   & 0x7F  << 24                # mask sign bit
    | HS[Offset+1] & 0xFF  << 16
    | HS[Offset+2] & 0xFF  << 8
    | HS[Offset+3] & 0xFF
  # P is a 31-bit unsigned integer

Step 3: Compute OTP
  OTP = P mod 10^d                               # d = 6 (typically)
  # Zero-pad to d digits
```

### TOTP Time-Step Calculation

```python
import hmac
import hashlib
import struct
import time

def generate_totp(secret_bytes, digits=6, period=30, algorithm=hashlib.sha1):
    """Raw TOTP implementation per RFC 6238."""
    # Time counter: integer number of time steps since epoch
    counter = int(time.time()) // period

    # Pack counter as 8-byte big-endian
    counter_bytes = struct.pack(">Q", counter)

    # HMAC
    hmac_digest = hmac.new(secret_bytes, counter_bytes, algorithm).digest()

    # Dynamic truncation
    offset = hmac_digest[-1] & 0x0F
    truncated = (
        (hmac_digest[offset] & 0x7F) << 24
        | (hmac_digest[offset + 1] & 0xFF) << 16
        | (hmac_digest[offset + 2] & 0xFF) << 8
        | (hmac_digest[offset + 3] & 0xFF)
    )

    # Modulo to get desired digits
    otp = truncated % (10 ** digits)
    return str(otp).zfill(digits)

# Example
import base64
secret = base64.b32decode("JBSWY3DPEHPK3PXP")
print(generate_totp(secret))  # Current 6-digit TOTP
```

### Elixir — Raw TOTP Implementation

```elixir
defmodule MyApp.TOTP do
  @period 30
  @digits 6

  def generate(secret_bytes) do
    counter = div(System.os_time(:second), @period)
    counter_bytes = <<counter::unsigned-big-integer-size(64)>>

    hmac = :crypto.mac(:hmac, :sha, secret_bytes, counter_bytes)
    hmac_bytes = :binary.bin_to_list(hmac)

    offset = Enum.at(hmac_bytes, 19) |> Bitwise.band(0x0F)

    truncated =
      Bitwise.bsl(Bitwise.band(Enum.at(hmac_bytes, offset), 0x7F), 24)
      |> Bitwise.bor(Bitwise.bsl(Enum.at(hmac_bytes, offset + 1), 16))
      |> Bitwise.bor(Bitwise.bsl(Enum.at(hmac_bytes, offset + 2), 8))
      |> Bitwise.bor(Enum.at(hmac_bytes, offset + 3))

    otp = rem(truncated, trunc(:math.pow(10, @digits)))
    String.pad_leading(Integer.to_string(otp), @digits, "0")
  end
end
```

### Security Considerations for TOTP

- **Secret storage**: Always encrypt the shared secret at rest (AES-256-GCM). If the secret database leaks, all TOTP tokens are compromised.
- **Clock drift**: Accept ±1 time step (±30s). Wider windows increase vulnerability to replay attacks.
- **Rate limiting**: Lock accounts after 3-5 failed TOTP attempts within a time window.
- **Replay prevention**: Track the last successfully used counter value. Reject any code with a counter ≤ the last used.
- **Backup codes**: Generate 8-10 single-use recovery codes at setup time. Store them hashed (bcrypt/argon2).

## Active Directory Architecture

### The AD Object Model

```
┌──────────────────────────────────────────────────┐
│                    Forest                        │
│  (Trust boundary — one or more domains)          │
│                                                  │
│  ┌─────────────────────┐  ┌───────────────────┐  │
│  │   Domain: corp.com   │  │  Domain: dev.com  │  │
│  │                     │  │                   │  │
│  │  ┌─────────────┐   │  │  ┌─────────────┐  │  │
│  │  │ OU: Users   │   │  │  │ OU: DevTeam │  │  │
│  │  │  ├─ User    │   │  │  │  ├─ User    │  │  │
│  │  │  ├─ User    │   │  │  │  └─ Group   │  │  │
│  │  │  └─ Group   │   │  │  └─────────────┘  │  │
│  │  └─────────────┘   │  │                   │  │
│  │  ┌─────────────┐   │  └───────────────────┘  │
│  │  │ OU: Servers │   │                          │
│  │  │  └─ Computer│   │                          │
│  │  └─────────────┘   │                          │
│  └─────────────────────┘                          │
└──────────────────────────────────────────────────┘
```

| Concept | Description |
|---|---|
| **Forest** | Top-level container. Defines the trust boundary and schema. |
| **Domain** | A logical partition with its own security policies, user accounts, and domain controllers. |
| **OU (Organizational Unit)** | A container within a domain for organizing objects. Group Policy is applied at the OU level. |
| **Domain Controller (DC)** | Server running AD DS (Directory Services). Handles authentication (Kerberos/NTLM) and replication. |
| **LDAP** | Lightweight Directory Access Protocol — the query protocol for reading/writing AD objects. |
| **Kerberos** | The default authentication protocol. Issues Ticket Granting Tickets (TGT) and service tickets. |
| **Group Policy (GPO)** | Configuration rules applied to users/computers within OUs, domains, or sites. |

### Kerberos Authentication Flow (AD Internal)

```
┌──────────┐    1. AS-REQ (username + timestamp    ┌──────────────┐
│  Client   │       encrypted with password hash)   │  KDC (Domain  │
│           │─────────────────────────────────────▶│  Controller)  │
│           │◀─────────────────────────────────────│               │
│           │    2. AS-REP: TGT (encrypted with    │  Key          │
│           │       krbtgt hash, 8-10 hr lifetime)  │  Distribution │
│           │                                      │  Center       │
│           │    3. TGS-REQ: TGT + target SPN      │               │
│           │─────────────────────────────────────▶│               │
│           │◀─────────────────────────────────────│               │
│           │    4. TGS-REP: Service ticket        │               │
│           │       (encrypted with service key)    └──────────────┘
│           │
│           │    5. AP-REQ: Service ticket
│           │─────────────────────────────────────▶ Target Service
└──────────┘
```

### AD ↔ Entra ID Relationship

| On-Premises AD | Entra ID (Cloud) | Bridge |
|---|---|---|
| Kerberos / NTLM | OAuth 2.0 / OIDC | **Entra Connect** (formerly Azure AD Connect) |
| LDAP queries | Graph API / REST | Sync agent replicates objects |
| Group Policy | Conditional Access | Different policy engines |
| Domain join | Entra join / registration | Hybrid join for both |
| OU-based structure | Flat directory + groups | Groups map to security groups |

## Microsoft Graph API Architecture

### The API Model

```
┌─────────────────────────────────────────────────┐
│              Microsoft Graph API                 │
│              graph.microsoft.com                 │
├─────────────────────────────────────────────────┤
│  /v1.0/  (stable)     │  /beta/  (preview)      │
├───────────────────────┼─────────────────────────┤
│  /me                  │  User profile            │
│  /me/messages         │  Outlook mail            │
│  /me/calendar/events  │  Calendar                │
│  /me/drive/root       │  OneDrive                │
│  /users               │  All users (admin)       │
│  /groups              │  Microsoft 365 Groups    │
│  /teams               │  Teams channels/chats    │
│  /sites               │  SharePoint sites        │
│  /applications        │  App registrations       │
└───────────────────────┴─────────────────────────┘
```

### Key API Patterns

| Pattern | Example | Purpose |
|---|---|---|
| **Single entity** | `GET /v1.0/me` | Fetch one object |
| **Collection** | `GET /v1.0/users` | List objects |
| **$filter** | `GET /v1.0/users?$filter=department eq 'Engineering'` | Server-side filtering |
| **$select** | `GET /v1.0/me?$select=displayName,mail` | Return only specific fields |
| **$expand** | `GET /v1.0/me?$expand=memberOf` | Include related entities inline |
| **$top / $skip** | `GET /v1.0/users?$top=10&$skip=20` | Pagination |
| **@odata.nextLink** | Follow the link in response | Cursor-based pagination for large sets |
| **$batch** | `POST /v1.0/$batch` | Bundle up to 20 requests in one HTTP call |
| **Delta query** | `GET /v1.0/users/delta` | Get only changes since last sync |
| **Change notifications** | `POST /v1.0/subscriptions` | Webhook notifications for changes |

### Graph API Rate Limits

| Resource | Limit |
|---|---|
| Per app per tenant | 10,000 requests / 10 minutes |
| Per app across all tenants | 150,000 requests / 5 minutes |
| Mailbox operations | 10,000 requests / 10 minutes per mailbox |
| Teams operations | 30-60 requests / second (varies) |

Handle `429 Too Many Requests` by reading the `Retry-After` header and backing off.

### Elixir — Graph API Pagination and Delta Query

```elixir
defmodule MyApp.Graph do
  def list_all_users(access_token) do
    fetch_pages("https://graph.microsoft.com/v1.0/users?$top=100", access_token, [])
  end

  defp fetch_pages(url, token, acc) do
    case Req.get(url, headers: [{"authorization", "Bearer #{token}"}]) do
      {:ok, %Req.Response{status: 200, body: body}} ->
        users = Map.get(body, "value", [])
        next_link = Map.get(body, "@odata.nextLink")

        case next_link do
          nil -> {:ok, acc ++ users}
          url -> fetch_pages(url, token, acc ++ users)
        end

      {:ok, %Req.Response{status: 429, headers: headers}} ->
        retry_after =
          headers
          |> Enum.find_value(fn {k, v} -> if k == "retry-after", do: String.to_integer(v) end)
          |> Kernel.||(60)

        Process.sleep(retry_after * 1000)
        fetch_pages(url, token, acc)

      {:ok, %Req.Response{status: status, body: body}} ->
        {:error, {status, body}}
    end
  end
end
```

## Related Topics

- [What is an LLM?](../llm_fundamentals/what_is_an_llm.md) — foundational concepts (this guide lives in the AI Engineering knowledge base)
- [Elixir AI Zero to Hero](../api_integration/elixir_ai_zero_to_hero.md) — Elixir AI stack overview
- [Prompt Engineering](../prompt_engineering/) — for LLM-based auth decision systems

### References

- [Microsoft Entra ID OAuth 2.0 Auth Code Flow](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-auth-code-flow)
- [Microsoft Graph API Overview](https://learn.microsoft.com/en-us/graph/overview)
- [Google OAuth 2.0 Guide](https://developers.google.com/identity/protocols/oauth2)
- [Amazon Cognito PKCE](https://docs.aws.amazon.com/cognito/latest/developerguide/using-pkce-in-authorization-code.html)
- [RFC 6238 — TOTP](https://datatracker.ietf.org/doc/html/rfc6238)
- [RFC 4226 — HOTP](https://datatracker.ietf.org/doc/html/rfc4226)
- [ueberauth_microsoft v0.25](https://hex.pm/packages/ueberauth_microsoft)
