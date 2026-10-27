# SkillBridge

AI-powered academia–industry collaboration, implemented as a React + TypeScript + Vite application with Firebase deployment scaffolding.

## Current MVP scope

The frontend runs in **local demo mode** without cloud credentials. Firebase mode adds a signed-in network directory with college/company offerings, student connection requests, organization approvals, an AI-generated resume assessment, role-specific mock tests, scored results, and in-platform offer letters. The AI assessment accepts pasted resume text and saves scores and strengths to the student's private profile; an approved organization can view that profile and send a mock test with a 70%, 75%, or 80% pass threshold. Offer letters can only be sent after a student passes. Local demo accounts cannot call these secured Firebase workflows.

Browser-local demo accounts are not authentication and do not provide production security. The resume assessment requires the student to paste resume text; it does not upload or parse PDF/DOCX files. AI scores are practice feedback and should not be treated as an objective hiring decision. Organization accounts do not require platform-admin approval; students request connections and the organization decides.

## Requirements

- Node.js 20 or newer
- npm

## Run locally

```sh
npm install
npm run dev
```

For the frontend alone, open the Vite URL printed in the terminal. To use the server actions locally, configure the Supabase URL/public key in `.env.local` and deploy the Edge Function with the required Supabase secrets. Firebase-backed accounts and profile data persist in Firestore; when Firebase is not configured, the app falls back to browser-local demo data under `skillbridge-demo-v1`.

Keep the root `.env.local` for public frontend settings only (`VITE_FIREBASE_*`, `VITE_SUPABASE_URL`, and `VITE_SUPABASE_ANON_KEY`). Supabase Edge Functions handle privileged AI and assessment requests while Firebase Authentication and Firestore remain the identity and data services. The function verifies Firebase ID tokens and uses the Firebase Admin SDK server-side.

## Firebase web app setup

The web client reads Firebase's public web configuration from `.env.local` (already populated for the `skillbridge-sh` web app in this workspace). Before using cloud accounts:

1. In Firebase Console, enable **Authentication → Email/Password** and **Google**, and create a **Cloud Firestore** database.
2. Run `firebase login`, then deploy the Firestore rules with `firebase deploy --only firestore:rules --project skillbridge-sh`.
3. Restart the Vite dev server after changing `.env.local`.

Registration and sign-in use Firebase Authentication. Google and email/password methods can be linked under **Settings** so the same Firebase UID and profile are used for both. User profile fields are stored in `users/{uid}`; skills, resume filename, and application IDs are stored in the account's private `userData/{uid}` document. Assessment summaries are stored in `studentProfiles/{uid}` and visible to an organization only after an approved connection. Colleges and companies publish their offerings in the network directory. The Admin demo button remains a local preview and does not create a privileged Firebase admin account.

## Deploy the frontend to Vercel

`vercel.json` configures the Vite build, `dist` output, and SPA route fallback. In Vercel Project Settings → Environment Variables, add the public frontend variables listed in `.env.example` (`VITE_FIREBASE_*`, `VITE_SUPABASE_URL`, and `VITE_SUPABASE_ANON_KEY`). The server function runs in Supabase; add its private secrets in Supabase → Edge Functions → Secrets: `FIREBASE_SERVICE_ACCOUNT_JSON`, `FIREBASE_API_KEY`, and optional `OPENAI_API_KEY` or `GEMINI_API_KEY`. Turn off Supabase gateway JWT verification for the `skillbridge` function; the function verifies the Firebase ID token itself. Never put the service-account JSON in source control or a `VITE_` variable. Use a dedicated Firebase service account with the minimum Firestore access needed (Cloud Datastore User). Keys entered in Admin → AI Configuration are stored in protected Firestore.

The Admin **API Keys** page stores provider keys in the server-only `privateSettings/apiKeys` Firestore document. Supabase Edge Functions perform provider checks and AI requests. No Firebase Functions deployment is required.

## Validate / build

```sh
npm run typecheck
npm run build
```

## Configure Firebase (separate project)

1. Create a new Firebase project specifically for SkillBridge; do not reuse a production project.
2. Register a web app and copy its public web configuration into a local `.env` based on `.env.example`. These values are public identifiers, not service-account credentials.
3. Enable Email/Password in Firebase Authentication; create Firestore and Storage.
4. Install Firebase CLI and run `firebase login`. The repository's Firebase project alias points to `skillbridge-sh`.
5. Review and deploy rules/indexes with `firebase deploy --only firestore:rules,firestore:indexes,storage --project skillbridge-sh`.
6. Create a Firebase service account with the minimum Firestore permissions needed, then add its JSON credential as the private Supabase function secret `FIREBASE_SERVICE_ACCOUNT_JSON`.
7. Never expose provider keys or Admin SDK credentials in the browser. Organization approvals are limited to their own student connection requests by Firestore rules and server-side API checks.
8. Run local services using `firebase emulators:start` after Firebase CLI setup. Configure the frontend Auth, Firestore, and Storage SDK clients and emulator connections before expecting persistence outside demo mode.
9. Deploy the frontend to Vercel and the Edge Function to Supabase. Firebase Hosting is not used by this app deployment.

## AI configuration

The server-only `generateCareerAssessment` and `generateOrganizationMockTest` actions use the OpenAI Chat Completions API with strict JSON Schema output. They read the administrator-managed key from protected Firestore, with optional `OPENAI_API_KEY` and `GEMINI_API_KEY` Supabase function secrets as fallbacks. These keys stay server-side. Resume text is limited to 20,000 characters and is sent to the selected AI provider for assessment. The prompt treats resume content as untrusted input, and answer keys stay server-side. The handlers use `gpt-4o-mini` for structured results supported by the [official Structured Outputs guide](https://developers.openai.com/api/docs/guides/structured-outputs).

## Data and security notes

`firestore.rules`, `storage.rules`, and `firestore.indexes.json` are a starting policy set. Firebase Admin SDK bypasses these rules, so each privileged Edge Function action checks identity, ownership, and request validity itself. Students request connections; the college or company named in each request approves or declines it. Organizations do not need platform-admin approval. Server actions keep answer keys private, score assessments and mock tests, and allow an organization to issue an offer only after a passing result. Review the policies and run Firebase Emulator security checks before production use. This MVP processes pasted resume text; it does not upload or parse resume files, verify credentials, email offer letters, or provide legally binding contract signatures.

## Project structure

- `src/` React app and responsive design system
- `supabase/functions/skillbridge/` Supabase Edge Function and Firebase Admin handlers
- `functions/` legacy Cloud Functions source
- `firestore.rules`, `storage.rules`, `firestore.indexes.json` Firebase policy scaffolding
- `.env.example` public frontend config placeholders
# Shared AI API key management

The dashboard's **API Keys** page is available to signed-in accounts on the server allowlist (`shfoundationsltd@gmail.com` and `johnseyd52@gmail.com`) when Firebase reports the email as verified. The Admin demo identity is browser-only and cannot write shared credentials. The website checks this permission through Supabase, which independently enforces the same allowlist. Other users and the Admin demo are denied by the server.

Deploy Firestore rules, the Supabase Edge Function, and the web app to activate this access on the hosted site. A key entered in **API Keys** is saved in the server-only `privateSettings/apiKeys` Firestore document; Firestore rules deny all client access. Supabase reads the managed key on the server and can fall back to its `OPENAI_API_KEY` or `GEMINI_API_KEY` function secrets when no managed key has been saved. The UI can read connection status but never receives stored provider keys.

Firestore encrypts stored data at rest using Google-managed encryption. For environments requiring customer-managed encryption keys or provider-level secret rotation, use Secret Manager with an appropriately restricted service account instead.
