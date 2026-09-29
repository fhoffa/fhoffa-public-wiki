# Research notes

## Agenda restructure (observed Sep 29, 2026)

Descartes restructured the event agenda mid-extraction:

- **Tuesday** flipped from product roadmap/training sessions to a keynote/strategy/panel day (47 sessions: keynotes, executive panels, customer panels).
- **Product roadmap/training** moved to **Wednesday** with new titles, times, rooms, and speakers (~87 sessions, mostly "Training:" / "Roadmap:" prefixed).
- **Thursday** became focus groups and workshops (14 sessions).
- **Room names changed:** Grand Ballroom C/D/E/F/G and United A/B/C are no longer Tuesday rooms; Tuesday now uses Grand Ballroom BC, Grand Ballroom DEFGH, Rosemont Ballroom AB/CD, DFW AB, Grand Ballroom Foyer.
- **Session counts moved 108 → 149** between passes on the same day.

## Methodology lessons

- **API-first, then fall back deliberately.** The Cvent data calls were tied to page/session state — no clean replayable bulk endpoint. Fell back to opening session dialogs one by one. Lesson: report the API failure early instead of silently grinding through dialogs.
- **Re-verify currency before tasking.** A stale session list sent a whole extraction task hunting for 16 sessions that no longer existed. Check the working catalog against the live site before dispatching.
- **The agenda widget is flaky.** It intermittently fails with "We couldn't load AgendaV2 widget" — retry across multiple loads and verify each day tab independently.
- **Never invent or paraphrase descriptions.** Mark "no description published" explicitly; flag placeholder text.
- **The site changes actively.** Record capture dates on everything; treat counts and details as provisional until the event.

## Related threads

- Geotab MCP connection remains blocked: the connect flow demands a manual Client ID despite Geotab supporting dynamic client registration (server side verified working). Broken-behavior report filed with the Muse team.
- Tech Fair (Tue 6–9 PM, Grand Ballroom DEFGH) is the week's main networking event — open bar, heavy hors d'oeuvres, carving and pasta stations.
