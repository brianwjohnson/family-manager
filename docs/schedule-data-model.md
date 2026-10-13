# Schedule Data Model

`data/family-planner.json` contains lightweight household seed data for the console. The UI can change freely, but this file should remain easy to transform into dashboard state and future Supabase rows.

The dashboard must not treat this file as the source of truth for the current date. Today is computed from the real date in the household timezone. If today does not exist inside this prototype data, the app should show the real date with empty events and a missing-baseline state.

## Current Assumptions

- Personal and sports calendar data is configured through `/admin` and stored in the household calendar feed.
- Connected feeds import the previous 31 days and the following year, so current sports seasons are not tied to seed data dates.
- Routine templates are configured through the admin experience rather than generated from a fixed seasonal plan.

## Calendar Setup Path

1. Use `/admin` to add Apple/shared family, Crossbar, school, and other ICS or `webcal://` sources.
2. Preview each source and apply it to the dashboard feed.
3. Resolve conflicts by date and time, preferring applied calendar events over optional routine templates.
4. Promote calendar sources and applied events into Supabase so setup syncs across devices.

## Likely Supabase Tables

- `planner_seasons`
- `household_members`
- `day_templates`
- `schedule_blocks`
- `fixed_events`
- `calendar_sources`
- `chores`
- `chore_assignment_templates`
- `chore_completions`
- `allowance_entries`

The broader household console will also need durable records for households, routines, routine items, profile preferences, reminders, and device enrollment. Sensitive identity artifacts such as face images or recognition embeddings should stay local unless a later privacy review creates a specific reason to sync them.

## Chores

Chores live under `chores` in `data/family-planner.json`.

- `routineChores` are daily checklists, currently the kids' weekday morning routine.
- `weeklyChores` are the reusable chore bank and can optionally carry an `allowanceAmount`.
- `weeklyAssignmentTemplates` assigns weekly chore slots by child and weekday.
- `completions` is intentionally empty for now; each completion should eventually record who completed the task and when.
- `allowance.entries` is the earned allowance ledger. Paid chore completions should create one entry tied to that completion so undoing the chore also removes the credit.

The current target is five weekly assigned chores per child. Morning routine chores do not count toward that weekly target.

The old chore manager stored chore edits and completions in browser `localStorage` under `family-manager:chores:v1`. The profile dashboard stores checklist completions under `family-manager:dashboard:v1`.

The dashboard now also stores user-created same-day tasks, same-day reminders, locally added recurring routine steps, and locally added recurring responsibilities in `family-manager:dashboard:v1`.

These browser storage keys are temporary. Supabase durable tables now exist for households, members, calendar records, action items, assignments, completions, chores, and chore templates. The next step is moving each UI surface from browser-local JSON into those normalized tables.

## Durable Action Item Model

The first Supabase migration in `supabase/migrations` promotes the dashboard's three editable surfaces into one durable action-item model.

- `routine`: recurring checklist step with `days_of_week`, plus either explicit `start_time` and `end_time` or duration/offset metadata anchored to each member's wake-up defaults.
- `task`: dated responsibility with `occurrence_date`, or recurring responsibility with `days_of_week`, `start_time`, and `end_time`.
- `reminder`: dated remember item with `occurrence_date`.
- `household_action_item_completions`: occurrence-level checkoff records for routines and tasks.

The durable action-item contract remains the target shape for routines, tasks, reminders, and completions. All durable household tables carry `household_id`, and RLS checks `household_users` membership to separate one family's data from another household.

Count-based habits should extend this contract rather than becoming a separate checklist system. Examples include push-ups, sit-ups, cups of water, pages read, or minutes practiced. The action item should define a numeric unit and optional daily target, while the occurrence record stores the value entered for that person and date.

This contract also works for a future client-side voice AI. The voice layer should turn speech into structured action-item requests and call the same authenticated API routes as the iPad UI, rather than writing directly to browser storage or bypassing household authorization.
