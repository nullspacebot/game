# AGENTS.md

NULL SPACE is a turn-based trading game played by posting commits to a bot account on dev-null.sh. This repo is the Laravel app that runs the bot. It is hosted on Laravel Cloud.

Read these before changing behaviour:

- [Game design](docs/gdd.md): the rules. It is the source of truth for what the game does.
- [Technical design](docs/technical.md): the architecture, tick resolution order, storage and platform limits.

If the code and the docs disagree, don't guess. Ask, or update the doc in the same PR and say so in the description.

## Commands

The commands below are the planned names. Update this list when they exist.

```bash
composer install && cp .env.example .env && php artisan key:generate
php artisan migrate
php artisan test                      # all tests (Pest)
vendor/bin/mago fmt                   # format
vendor/bin/mago fmt --check           # check formatting without changing files
vendor/bin/mago lint                  # lint
vendor/bin/mago analyze               # static analysis
php artisan nullspace:play            # play a turn by hand in the local harness
php artisan nullspace:simulate        # run scripted players for many ticks
```

Run the tests and all three Mago commands (`fmt --check`, `lint`, `analyze`) before you finish. All of them must pass.

## Code quality: Mago

[Mago](https://mago.carthage.software) is the only formatter, linter and static analyzer. Don't add Pint, PHP-CS-Fixer, PHPStan or Psalm.

- **Strictest settings.** `mago.toml` turns on every analyzer check in Mago's maximum strictness preset, enables every linter rule, and sets `minimum-fail-level = "note"` for both the linter and the analyzer, so any issue fails the run.
- **No baselines and no ignores.** Don't add a baseline file, an `ignore` entry, a rule exclusion or an inline suppression to get code through. Fix the code. If a rule really is wrong for this project, change `mago.toml` in its own PR and explain why.
- **Nothing is committed without passing.** Every commit must pass `mago fmt --check`, `mago lint` and `mago analyze`. CI runs all three and blocks merging when any fails.

## Architecture rules

- **`app/Game` is pure PHP.** It must not import anything from `Illuminate\`, touch the database, make HTTP calls, read config or env, or log. It takes state and orders and returns new state and reports.
- **No hidden randomness or time in `app/Game`.** Never call `rand`, `mt_rand`, `random_int`, `shuffle`, `array_rand`, `uniqid`, `now()`, `time()` or `date()` there. Use the seeded RNG and the tick number that are passed in.
- **Keep ordering stable.** Sort before iterating over anything whose order affects the result. Never rely on database or hash-map order.
- **Everything that talks to dev-null.sh lives in `app/Platform`.** The engine never knows the platform exists.
- **Every bot post goes through the outbox.** Never call the dev-null.sh API to post from anywhere except the poster job. That job is what keeps the bot under 10 posts a minute and stops double posts.
- **Ticks resolve in one database transaction:** state, the tick record and outbox rows together, or nothing.

## Testing rules

- Every rule change needs a unit test in `tests/Unit/Game` and, if it changes tick results, a replay fixture.
- Replay fixtures record a starting state, the orders and the expected result. Don't edit an expected result just to make a test pass. If a result really should change, regenerate the fixture and explain why in the PR.
- **Never contact dev-null.sh from tests or local runs.** Call `Http::preventStrayRequests()` in the base test case and fake every platform call. Real posting only happens when `NULLSPACE_POSTING_ENABLED=true`, which is set only in production.
- Test that every bot post template stays within 4,000 characters for the worst case, such as a long scan list or maximum-length names.

## Bot output

- All bot text comes from templates. Never generate post text with a language model at runtime, because dev-null.sh's rules ban AI-generated posts.
- Keep the voice short and dry. When a template has variants, choose with the seeded RNG.
- Every post that mentions the next tick shows a countdown and a UTC time.

## Database

- Never edit a migration that has been merged. Add a new one.
- Store only what changes. Hex contents come from the season seed, and only changes such as field stock and station prices are stored.

## Secrets

- Never commit the bot's SSH private key, the season seed or anything from `.env`. Use environment variables, set in Laravel Cloud for production.

## Pull requests

- Keep each PR to one change: one rule, one fix or one feature.
- If a PR changes game rules, link the section of the game design it implements or changes.
- Say which tests you added, and whether any replay fixtures changed.
