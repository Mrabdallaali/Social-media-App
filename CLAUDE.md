# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run dev      # Start dev server (http://localhost:3000)
npm run build    # Production build
npm run start    # Serve the production build
npm run lint     # next lint
```

There is no test suite configured in this repo.

`.npmrc` sets `legacy-peer-deps=true` — install with `npm install`, expect peer-dep mismatches otherwise (React 19 vs. `@types/react` pinned to ^18).

### Environment

Firebase is configured via `NEXT_PUBLIC_*` env vars read in [firebase.ts](firebase.ts): `NEXT_PUBLIC_API_KEY`, `NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN`, `NEXT_PUBLIC_FIREBASE_PROJECT_ID`, `NEXT_PUBLIC_FIREBASE_STORAGE_BUCKET`, `NEXT_PUBLIC_FIREBASE_MESSAGING_SENDER_ID`, `NEXT_PUBLIC_FIREBASE_APP_ID`. No `.env.example` exists in the repo — a `.env.local` with these keys against a real Firebase project is required for the app to function (auth and Firestore calls will fail silently/throw otherwise).

## Architecture

Next.js App Router app (`app/`) with no backend layer of its own — client components talk to Firebase (Auth + Firestore) directly. There are no API routes.

- **Data flow for posts**: [components/PostFeed.tsx](components/PostFeed.tsx) opens a live Firestore `onSnapshot` listener on the `posts` collection (ordered by `timestamp desc`) and holds results in local `useState`; there is no Redux slice for post data. The first snapshot also dispatches `closeLoadingScreen()`, which is what dismisses the global [LoadingScreen](components/LoadingScreen.tsx) — the loading screen isn't tied to route/auth state, only to the first posts snapshot arriving.
- **Redux (`redux/`) is for UI/session state only**, not server data: `user` (the signed-in profile mirrored from Firebase Auth), `modals` (open/closed flags for sign-up/log-in/comment modals, plus `commentPostDetails` used to pass context into the comment modal), and `loading` (the boot loading screen flag). Store is wired in [redux/store.ts](redux/store.ts) and provided via [redux/StoreProvider.tsx](redux/StoreProvider.tsx) (a client component wrapping `RootLayout`).
- **Auth state sync**: the `onAuthStateChanged` listener that populates the Redux `user` slice lives inside [components/modals/SignUpModal.tsx](components/modals/SignUpModal.tsx) (mounted globally via `SignUpPrompt` on every page), not in a dedicated auth provider — look there first when touching sign-in/sign-out behavior. `username` is derived client-side as the local part of the account email (`email.split("@")[0]`), not a stored field.
- **Gating on auth**: components don't redirect for unauthenticated users; they dispatch `openLogInModal()` on the attempted action (posting, liking, commenting — see [components/PostInput.tsx](components/PostInput.tsx) and [components/Post.tsx](components/Post.tsx)) and rely on `!user.username` as the "is logged in" check.
- **Modals** are MUI `Modal` components that read their `open` flag from the `modals` Redux slice and toggle it via dispatched actions — there's no shared modal wrapper, each modal (`LogInModal`, `SignUpModal`, `CommentModal`) duplicates the MUI `Modal` shell.
- **Post detail route**: `app/[id]/page.tsx` is an async Server Component that fetches a single post directly from Firestore (`getDoc`) at request time — this is the one place data isn't loaded via a client-side listener.
- **Styling**: Tailwind CSS utility classes are the default; MUI (`@mui/material` + Emotion) is used only for `Modal` and `LinearProgress`. Brand accent color is `#F4AF01`, hardcoded inline throughout rather than as a Tailwind theme token (`tailwind.config.ts` doesn't define it).
- Path alias `@/*` maps to the repo root (see `tsconfig.json`), used for all internal imports (`@/firebase`, `@/redux/...`, `@/components/...`).
