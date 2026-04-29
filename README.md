![React Hosted Login Banner](/public/assets/react-banner.png)

# React Hosted Login — Device Authorization Sample

This sample demonstrates how to integrate Frontegg's hosted login into a React app, with a complete implementation of the **OAuth 2.0 Device Authorization Grant** ([RFC 8628](https://datatracker.ietf.org/doc/html/rfc8628)). Use this as a reference for building verification pages that let users approve or deny device access from a browser.

## What this app showcases

- Redirect users to Frontegg's hosted login
- Enable a fully integrated self-service portal
- Manage and track user authentication state
- Access and display user profile details
- Handle account state and data with ease
- Implement seamless account switching functionality
- **Authorize devices via the OAuth 2.0 Device Authorization Grant flow**

---

## Device Authorization Grant

The Device Authorization Grant allows input-constrained devices (smart TVs, CLIs, IoT devices) to obtain tokens by having the user sign in and approve access on a secondary device (phone or laptop).

### How the flow works

```
  Device App                     Frontegg APIs              This App (Verification Page)
  (e.g. CLI, TV)                                              /activate?user_code=XXXX-XXXX
       |                               |                                |
       | 1. POST /device/authorize     |                                |
       |------------------------------>|                                |
       |    { device_code, user_code } |                                |
       |<------------------------------|                                |
       |                               |                                |
       |  Display user_code + URL      |                                |
       |  pointing to this app         |                                |
       |                               |   2. User opens /activate      |
       |                               |<-------------------------------|
       |                               |   3. Auto-redirect to login    |
       |                               |      (if not authenticated)    |
       |                               |<-------------------------------|
       |                               |   4. GET /device?user_code=    |
       |                               |<-------------------------------|
       |                               |      { appName, scopes, ... }  |
       |                               |------------------------------>|
       |                               |   5. User clicks Approve/Deny  |
       |                               |   POST /device/verify          |
       |                               |<-------------------------------|
       |                               |                                |
       | 6. POST /token (polling)      |                                |
       |------------------------------>|                                |
       |  { access_token, id_token,    |                                |
       |      refresh_token }          |                                |
       |<------------------------------|                                |
```

### Key components

#### `DeviceModal` — [src/components/DeviceModal.tsx](src/components/DeviceModal.tsx)

A modal dialog rendered on the main account page. The authenticated user enters the `XXXX-XXXX` code shown on their device. The modal validates the code format and navigates the user to the verification page via React Router:

```
/activate?user_code=XXXX-XXXX
```

The modal is triggered by the **Verify Device** button in [AccountInfo](src/components/AccountInfo.tsx).

#### `DeviceVerifyPage` — [src/components/DeviceVerifyPage.tsx](src/components/DeviceVerifyPage.tsx)

The verification page mounted at `/activate`. It handles the complete user-side flow:

| State | What happens |
|-------|-------------|
| `loading` | Reads `user_code` from the URL. If the user is not authenticated, calls `loginWithRedirect` — Frontegg hosted login redirects back after sign-in. |
| `confirm` | Calls `GET /frontegg/oauth/device?user_code=…` with the user's bearer token. Displays the app name, user code, and requested scopes. Shows **Approve** and **Deny** buttons. |
| `done` | Calls `POST /frontegg/oauth/device/verify` with `{ user_code, approved }`. Shows a success or denial confirmation and a **Back to Account** button. |
| `error` | Shown when no code is provided or the code is expired/invalid. |

The component uses `ContextHolder.getAccessToken()` from `@frontegg/react` to obtain the bearer token — no manual token handling is needed.

### Routing

```
/          → Main (AccountInfo + Verify Device button)
/activate  → DeviceVerifyPage (approval/denial UI)
*          → redirect to /
```

`BrowserRouter` wraps `FronteggProvider` in [App.tsx](src/App.tsx) so that React Router navigation works correctly without triggering Frontegg re-initialization. The `*` catch-all route handles Frontegg's OAuth callback redirects gracefully.

### Frontegg API endpoints used

| Method | Endpoint | Auth | Purpose |
|--------|----------|------|---------|
| `GET` | `/frontegg/oauth/device?user_code=` | Bearer token | Fetch device info (app name, scopes, status) |
| `POST` | `/frontegg/oauth/device/verify` | Bearer token | Approve or deny the device request |

> The device-side endpoints (`POST /device/authorize` and `POST /token`) are called by the device app, not this verification page.

---

## What you'll need

- [Node.js](https://nodejs.org)
- npm (comes with Node.js)

Don't have an account yet? No worries. This project includes **sandbox credentials** so you can test it right away!

---

## Get started in 3 simple steps

If you don't have a Frontegg account or prefer to use the sandbox credentials, feel free to skip to step below.

If you're using your own credentials, follow the guidelines below.

### 1. Configure your Frontegg application (if using your own account)

1. Go to [Frontegg Portal](https://portal.frontegg.com/)
2. Get your application ID from [ENVIRONMENT] → Applications
3. Get your Frontegg domain from the Frontegg Portal → [ENVIRONMENT] → Keys & domains
4. This sample runs on `http://localhost:3000`. Make sure to add it under → [ENVIRONMENT] → Authentication → Login method → Redirect URLs
5. Add `http://localhost:3000` under → [ENVIRONMENT] → Keys & domains → Allowed origins
6. Update your application's credentials in `src/config/sanboxContextOptions.ts`

### 2. Clone the repository

```bash
git clone <repo>
```

### 3. Install dependencies

```bash
npm install
```

### 4. Run the application

```bash
npm start
```

The app will be available at [http://localhost:3000](http://localhost:3000).

![React sample](/public/assets/sample-react-device-login.png)

---

## Testing the device flow end-to-end

> **Note:** `client_id` in OAuth 2.0 / RFC 8628 terminology refers to the registered OAuth application — in Frontegg terms, this is the **applicationId** found under [ENVIRONMENT] → Applications in the Frontegg Portal.

1. Start the app and sign in
2. Click **Verify Device** on the account page
3. Request a device code from the Frontegg API (simulating a device):
   ```bash
   curl -X POST https://{your-domain}/frontegg/oauth/device/authorize \
     -H "Content-Type: application/json" \
     -d '{"client_id": "your-client-id"}'
   ```
4. Copy the `user_code` from the response (e.g. `BCKF-DHLM`)
5. Enter it in the modal and click **Continue** — you will be taken to `/activate?user_code=BCKF-DHLM`
6. Review the app name and scopes, then click **Approve** or **Deny**
7. Poll for tokens on the device side to confirm the flow completed:
   ```bash
   curl -X POST https://{your-domain}/frontegg/oauth/token \
     -H "Content-Type: application/json" \
     -d '{
       "grant_type": "urn:ietf:params:oauth:grant-type:device_code",
       "device_code": "<device_code from step 3>",
       "client_id": "your-client-id"
     }'
   ```