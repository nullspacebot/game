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
vendor/bin/pint                       # formatting
vendor/bin/phpstan analyse            # static analysis
php artisan nullspace:play            # play a turn by hand in the local harness
php artisan nullspace:simulate        # run scripted players for many ticks
```

Run tests, Pint and PHPStan before you finish. All three must pass.

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
