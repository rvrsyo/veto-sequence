# Veto Studio

A working React + TypeScript VALORANT map veto application, with a production control panel and a separate OBS broadcast renderer. The interface uses Vite, Tailwind CSS, Framer Motion, and Lucide icons. Supabase Auth, Postgres, Edge Functions, and Realtime provide the optional cloud backend.

**Delivery status:** the local application is built and tested. Cloud integration is implemented but has not been deployed to a Supabase project or verified across computers. An unrelated inactive project was deliberately left untouched. Do not use the local demo as an OBS or multi-computer production backend. Actual OBS capture and real network interruption testing remain deployment acceptance checks.

## Quick start: local demo

Requires Node.js 22.12+ (Node 24 recommended).

```sh
npm ci
npm run dev
```

Open the printed localhost URL. Click **Create match** (or **Create demo match**) to get PHANTOM vs NOVA, a seven-map example pool, and BO3 rules. Open the display from the control panel, start the veto, select a map, and confirm the action. The demo uses browser storage, BroadcastChannel, and Web Locks for persistent, serialized updates between same-origin tabs in the **same browser profile**. It is intentionally unauthenticated and is not a security boundary. It does not sync separate computers, browsers, or OBS’s embedded browser profile.

```sh
npm test          # engine + real local PostgreSQL (PGlite) tests
npm run build    # type check and production bundle
npm run preview  # serve the production bundle
```

All dependencies are pinned and the lockfile is included. `npm ci` reproduces the tested dependency tree. Original SVG placeholder illustrations are included; regenerate them with `node scripts/generate-art.mjs` if needed. They are stylized environment illustrations, not official game screenshots or accurate depictions of the named maps.

## Supabase cloud setup

1. Create or choose a dedicated Supabase project. The app does not require a paid tier specifically; select capacity suitable for your production needs.
2. Install dependencies, then authenticate and link the CLI:

   ```sh
   npx supabase login
   npx supabase link --project-ref YOUR_PROJECT_REF
   npx supabase db push
   npx supabase functions deploy veto-api
   ```

   The checked-in `supabase/config.toml` disables the platform JWT precheck **only for this function** because broadcast viewers use a separate read-only token. The function itself verifies an operator JWT with `auth.getUser` before any create, list, command, or delete operation. Anonymous Auth accounts are rejected. Viewer tokens authorize only `read` on their one match. Do not remove these checks.

3. In Supabase Auth, create the operator account(s) with email and password. Each match belongs to its creator. Two browser sessions signed into that same owner can operate that match concurrently; unrelated accounts cannot read or modify it. There is no public account signup form in the app.
4. Enable Supabase Realtime and permit public broadcast channels in the project’s Realtime settings. A broadcast channel carries **only a change notification**, never trusted match state. Its topic includes the match’s random viewer token. Clients always fetch and validate access to the latest database snapshot. A forged channel event cannot change the match. Ten-second reconciliation and reconnect/focus events recover missed notifications.
5. Copy `.env.example` to `.env.local` and set:

   ```dotenv
   VITE_SUPABASE_URL=https://YOUR_PROJECT_REF.supabase.co
   VITE_SUPABASE_PUBLISHABLE_KEY=YOUR_PUBLISHABLE_KEY
   ```

   These are public frontend configuration values. **Never** put the service-role/secret key in a `VITE_` variable, source file, or broadcast URL. The Edge Function uses the project-provided `SUPABASE_SERVICE_ROLE_KEY` exclusively on the server.
6. Restart Vite or rebuild the frontend. Sign in as an operator and create a **new cloud match**. Demo matches are not silently migrated to the cloud.
7. Run the cloud and OBS checks in `ACCEPTANCE.md` before using the app on air. If deployment uses restrictive function origins or network policies, allow the frontend origin. The function’s CORS default allows browser clients; authorization, not CORS, is the access control.

### Database schema and write path

The migration is `supabase/migrations/20261013100342_veto_rooms.sql`.

- `veto_matches`: UUID match ID, owner UUID, JSON state, numeric revision, random read-only broadcast token, SHA-256 token hash, timestamps. RLS restricts direct reads to the owner. Browser roles have no direct writes.
- `veto_commands`: unique `(match_id, request_id)` and committed revision. Browser roles have no access. This table makes duplicate command requests idempotent.
- `commit_veto_state`: service-role-only, **security invoker** function. It locks the match row, verifies owner and expected revision, saves state, and records the command in one transaction. Concurrent commands with stale revisions fail with a conflict instead of overwriting a completed action.

The Edge Function authenticates, validates the payload using Zod, runs the shared pure veto engine, calls this atomic function, and sends a Realtime invalidation after commit. The operator UI never treats an unacknowledged command as committed. On failure it loads the latest saved state and shows the error. Repeat requests must retain their request ID when retrying programmatically; the UI deliberately asks operators to review the current state before retrying a failed command.

The broadcast token is in the URL fragment (`#token=...`), avoiding normal web server URL logs and referrers. The page uses a no-referrer policy for remote assets. A token holder can view that one match and send harmless invalidation hints, but cannot access owner data or write commands. Invalidate a leaked token by replacing the match with a newly created room and deleting the old room. There is no token rotation UI in this version.

## Operator workflow

1. Create or duplicate a match from `/control`.
2. Before starting, open **Match setup**. Edit tournament/stage/title/date/subtitle; team names, seeds, logos, and colors; enabled maps and artwork; branding; and the veto order. Team logos support HTTPS URLs or PNG/JPEG/WebP uploads up to 450 KB. Larger files should use hosted asset URLs.
3. Choose BO1, BO3, or BO5. Format and first-team changes regenerate a seven-map default sequence. Adapt it in **Veto rules** for another pool size or tournament rules. BAN/PICK/DECIDER actions must consume every enabled map exactly once, and selected maps must total the format. The sole decider is the last consuming action. SIDE steps refer to a previously selected map and may occur at most once per map. CUSTOM steps require a note.
4. Defaults are the prompt’s sequences. For BO5, pick order is A, B, B, A (after the two bans), swapped if B acts first. By default the non-picking team chooses sides, and the first-acting team chooses the decider side. These are configurable examples, not claims about current official rules. Delete SIDE steps if the event does not assign sides in this tool.
5. Start the veto. Select an available map to highlight it on the broadcast, then confirm. The decider is determined automatically; **Reveal decider** commits its reveal. Choose sides using ATTACK/DEFENSE; the other team receives the opposite side.
6. Use pause/resume, undo, or redo when necessary. `Ctrl/Cmd+Z` and `Ctrl/Cmd+Shift+Z` work outside inputs and dialogs. New actions after undo discard the redo branch. Reset, skip, correction, archive, and deletion have confirmation dialogs.
7. **Edit** a completed history entry to correct it. This replaces that action and clears dependent later actions for safe replay; the previous action and correction reason remain in the audit trail. Skip is an audited exceptional override, not a bypass for duplicate maps or malformed final lineups. A skipped consuming action may need correction before the decider can be resolved.
8. Final map order and sides remain visible in MAP SUMMARY. Presentation modes can be changed at any time. Archive keeps the record; restore re-enables editing. **Clear match data** permanently removes the room and requires typing DELETE.

Names and pool settings are locked after starting. Reset retains audit history and re-enables setup. Defaults intentionally use a sample map pool and do not assert the official competitive pool on any date.

## Broadcast and OBS

- `/control`: match manager.
- `/control/:matchId`: operator console.
- `/display/:matchId#token=...`: read-only broadcast URL, copied from that room.
- `/display`: passive instruction to use a match URL.

Add an OBS **Browser Source**, paste the room’s broadcast URL, and use **1920 × 1080**. The canvas scales to other sizes without scrollbars. Transparent mode clears the page background; HIDE clears all graphics. LOWER THIRD uses a compact composition near the bottom. No operator controls appear in the display.

Modes: INTRO, VETO, MAP SUMMARY, LOWER THIRD, BREAK, HIDE. Animation presets are FAST, NORMAL, BROADCAST, with a reduced-motion switch and respect for system motion preferences. Unchanged elements retain stable keys. The control preview renders the same broadcast component from the saved state.

Keep **Shutdown source when not visible** off for a continuous connection. Avoid automatic scene-activation refreshes if you want continuous animations. Temporary disconnection retains the last known graphics and displays a small reconnecting label. Rejoining/focus and periodic reads reconcile the saved database revision; old and duplicate revisions are ignored. Deleted or invalid-access matches clear instead of retaining an obsolete picture.

Sound effects are synthesized, disabled by default, and only attempted for a newly committed pick, ban, or decider within three seconds of its action timestamp. Initial load, old snapshots, undo, reset, and duplicate snapshots do not replay them. Browser autoplay rules can prevent sound entirely; the graphics do not require interaction or audio permission. Do not rely on optional sound as a production cue.

**A private Sites preview, if published without Supabase environment values, runs local demo mode.** Publication is pending explicit user approval: automatic approval review blocked passing the source repository credential to the external terminal publishing helper. No source was pushed. A private Sites URL also has an owner sign-in boundary and should not be assumed usable in OBS. For operational OBS use, deploy the static frontend to a publicly reachable host with Supabase configured; access to data is still controlled by operator authentication and match viewer tokens.

## Vercel deployment

Import this directory as a Vite project. Use `npm ci`, build `npm run build`, output `dist`, and add the two `VITE_SUPABASE_*` values in the Vercel environment. `vercel.json` rewrites application routes to `index.html` while keeping `/assets/` static. Deploy Supabase first. Environment changes require a new frontend build.

Other static hosts need an SPA fallback to `index.html`. Set HTTPS, serve assets with their correct content types, and keep fragment tokens intact. Do not deploy `.env.local`, `node_modules`, or server secrets.

## Architecture

```text
src/App.tsx                         Routes, operator UI, dialogs, match management
src/Broadcast.tsx                   Shared preview and OBS graphics
src/store.ts                       Local/cloud adapters and synchronization
src/styles.css                     Responsive dashboard and broadcast styling
supabase/functions/_shared/engine.ts Validated state machine, presets, types
supabase/functions/veto-api/        Authorized cloud command/read endpoint
supabase/migrations/                Tables, RLS, atomic commit function
tests/engine.test.ts                Rules and state recovery tests
tests/database.test.ts              PostgreSQL transaction/permission tests
public/art/                        Original placeholder vector artwork
```

## Operational limits

Cloud setup and cross-computer synchronization need real project credentials and must be exercised before production. This delivery does not claim that unconfigured cloud features are live. The local PostgreSQL tests validate schema/RLS/transactions, not the hosted Supabase Realtime service or Edge deployment. Audit/state JSON grows with operation count; use one room per match, and archive finished rooms. The Edge request body is limited to 4 MB, so prefer image URLs over multiple embedded uploads. Team/account invitations, role administration, storage buckets, external tournament feeds, and token rotation are outside this version.

Original artwork can be used with this app. Supplied team/map/tournament assets remain their owners’ property; use assets you are permitted to broadcast. This independent application is not affiliated with Riot Games or VCT.

## Reference documentation

- [Supabase Realtime Broadcast](https://supabase.com/docs/guides/realtime/broadcast)
- [Securing Supabase Edge Functions](https://supabase.com/docs/guides/functions/auth)
- [Supabase Row Level Security](https://supabase.com/docs/guides/database/postgres/row-level-security)
