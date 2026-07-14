## Issues to fix

### 1) Logos can't be changed
The admin uploads to the `ads` storage bucket, which requires the moderator/admin role via storage RLS. All screenshots show the account is Super Admin, so upload itself should work. Need concrete failure info to fix without guessing.

Ask: does the upload button spin forever, does a toast error appear (validation? "Only admins…"? bucket error?), or does the image preview appear but Save doesn't persist? Please try uploading a small square PNG and paste the exact toast text.

### 2) "Canceling statement due to statement timeout" on bulk delete
`delete_teams_bulk` runs 6 large cascading DELETEs in a single statement with sub-selects across matches → markets → odds → bet_selections. On 55 rows this trips the 60s cap.

Fix (single migration):
- Rewrite `delete_teams_bulk` to first collect target match/market IDs into temp arrays, then delete in the correct order using those arrays (no repeated JOINs).
- Add supporting indexes: `matches(home_team_id)`, `matches(away_team_id)`, `markets(match_id)`, `odds(market_id)`, `bet_selections(match_id)`, `players(team_id)`, `tournament_matches(participant_a_id, participant_b_id)`.
- Same treatment for `delete_players_bulk`.

### 3) No voucher for Instant E-Football, Championship, Instant Championship Football
These modes bypass the standard `bets` table:
- Instant E-Football writes to `user_virtual_rounds` (via `start_user_virtual_round`) — no `bets` row → no `/ticket/:id` voucher.
- Championship writes to `championship_bets` (via `place_championship_bet`) — same problem.

Fix:
- Modify `start_user_virtual_round` to ALSO insert a `bets` row (status auto-settled won/lost, potential_payout=payout, single-selection bet_selection with the picked side). Return the bet id.
- Modify `place_championship_bet` (and `cancel_championship_bet`) to insert/delete a paired `bets` row with a "future" championship selection so the user gets a real ticket voucher visible in Bet History and admin Bet Tracker.
- Both bets flagged `is_virtual = true` so they show up under the new admin filter.

### 4) Admin Bet Tracker — add "Virtual" filter
- Extend the `filter` select in `BetTrackerPanel` with `virtual` (all is_virtual=true) and `real` options, and change the query to filter by `is_virtual` when those are picked.

## Files touched
- `supabase/migrations/<new>.sql` — bulk delete rewrite + indexes; RPC updates for virtual+championship writing into `bets`.
- `src/routes/admin.tsx` — Bet Tracker filter additions.
- `src/routes/virtual.football-instant.tsx` — after `start_user_virtual_round` returns a bet_id, link to `/ticket/:id` from the result card.
- `src/components/ChampionshipBetPanel.tsx` — show "View voucher" link after bet placed.

## Open question
Please provide the logo-upload failure symptom (toast text or blank behavior) so #1 can be fixed in the same batch. Everything else I'll ship as described.
