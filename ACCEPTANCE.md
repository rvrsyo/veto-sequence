# Acceptance and verification

Validation date: 13 October 2026. “Verified” means the indicated environment only. No live Supabase project was configured, and OBS was not installed or controlled for this delivery.

| Prompt test | Status and evidence |
|---|---|
| 1. Create match and configure teams | Verified in local UI; demo room created with PHANTOM/NOVA. Editable configuration is implemented. |
| 2. Open separate control/display tabs | Verified in local browser. |
| 3. Ban immediately updates display | Verified across the two local tabs. |
| 4. Pick updates map, acting team, progress | Verified in a complete BO3 browser workflow. |
| 5. Undo synchronizes correct state | Undo and redo exercised locally; engine tests verify map availability and branch behavior. |
| 6. Refresh preserves match | Verified by reloading the completed broadcast. |
| 7. Different computer sees shared state | Requires configured Supabase; not verified. |
| 8. Completed correct lineup | Verified: Haven, Icebox, Sunset decider, with starting sides. |
| 9. Match isolation | Engine and database ownership tests pass; per-match subscriptions and URL IDs implemented. |
| 10. OBS 1920×1080, no controls/scrollbars | Browser DOM measured 1920×1080, equal scroll dimensions, zero buttons/inputs/selects. Actual OBS capture not verified. |
| 11. Temporary network interruption recovery | Revision reconciliation tested; retry/reconnect/focus implementation included. Real Supabase outage recovery not verified. |
| 12. Reject invalid and duplicate actions | Automated engine tests pass. |
| 13. Broadcast URL cannot administer | PostgreSQL tests reject anon reads and authenticated direct writes/RPC; Edge function checks user ownership for all writes. Live endpoint verification required. Local demo is intentionally unauthenticated. |
| 14. Customize pool and veto sequence | Editable UI and custom five-map BO1 engine test included. |
| 15. Final display stays visible | Verified locally; final summary restored after refresh. |

## Required cloud rehearsal

After completing README setup:

1. Sign in as owner A, create two matches, and open one match’s display on a second computer/OBS machine.
2. Exercise BAN, PICK, SELECT, SIDE, undo/redo, pause/resume, and all presentation modes. Verify the other room stays unchanged.
3. Open the display without a token and with the wrong token. It must not show a match. Use a separate browser signed into unrelated owner B; neither direct reads nor commands may access owner A’s match. Try using a viewer token to call `command`; expect 401 or 404.
4. Send two different commands from the same saved revision concurrently. Exactly one must commit; the other must report a revision conflict. Retry an already committed command with the original request ID; no second action should be added.
5. Disconnect the broadcast machine. It should retain its last visuals and show reconnection status. Change the match on the operator machine, reconnect, and verify the latest saved state arrives without replaying old actions/audio.
6. Test 1920×1080 and a smaller Browser Source, transparency, lower third, HIDE, reduce motion, scene changes, and refresh in OBS. Confirm no audio is produced with sound disabled. Autoplay restrictions may suppress optional enabled audio.
7. Reset/correct/delete only the rehearsal room; confirm destructive dialogs and audit behavior. Use new rooms for actual matches.

## Automated checks

`npm test` runs the pure state machine tests and schema tests on local PGlite PostgreSQL. The latter exercise RLS owner filtering, table/function grants, compare-and-swap conflicts, idempotent requests, and owner isolation. They do not use or mutate a remote database.

`npm run build` type-checks the frontend/shared engine and builds the static frontend. The Edge Function entrypoint also passed local TypeScript checking with `npx tsc -p tsconfig.edge.json`; its actual Deno runtime deployment remains unverified. See README before enabling cloud mode.

Final automated result: **32 tests passed** across two files. The production bundle built successfully. The installed dependency audit returned **zero known vulnerabilities**. Browser testing also confirmed HIDE clears all graphics; the lower-third composition and final lineup render in the shared preview/display.
