# Maintainerr rule groups

Rules are stored in Maintainerr's own database — these files are the
version-controlled, import-ready source for them (Collections → Rules →
Import on each group's page).

Build order (safe rollout):

1. Create collections with **arrAction = Do nothing**, import the YAML,
   inspect the matched media for a few rule runs (every 8h).
2. Add the global favourites exclusions (Exclusions page, no collection
   selected → applies to every collection, cascades to child seasons).
3. Only then switch actions: show cleanup → "Unmonitor and delete season"
   with delete-after 30 days; movie collections → delete with your chosen
   grace period.

Files:
- `movies-low-rated-watched.yaml` — low rated (TMDB < 5.5) + already watched
- `movies-hogs-keep-good.yaml`  — >20GB, watched, IMDb < 7.5, non-Marvel
- `shows-fully-watched.yaml`    — whole show watched by household + ended (Do nothing)
- `shows-season-cleanup.yaml`   — S2+ of those shows (season granularity)

Replace `USER1` / `USER2` placeholders in any imported YAML with the
household's Plex usernames. (`shows-fully-watched.yaml` needs none —
"any user watched it all" is `EXISTS` on the watched-by list.)
`Sonarr.ended` requires the Sonarr integration to be configured in Maintainerr
settings; the same applies to Radarr props in the movie files.
