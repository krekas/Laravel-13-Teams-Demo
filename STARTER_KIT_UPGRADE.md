# Starter Kit Upgrade Report

- Date: 2026-05-20
- Kit: laravel/livewire-starter-kit
- Branch tracked: teams
- Upgrade branch: starter-kit-upgrade/20260520-2008-passkeys
- Commits: `084ffc9` (passkeys), `931bfbc` (lockfiles)

## Features applied

### Passkeys (laravel/livewire-starter-kit@51fe9ad + scoped bits of eed0e1a) — 19 files

- Applied (new):
  - app/Http/Responses/Concerns/RedirectsToCurrentTeam.php
  - app/Http/Responses/PasskeyLoginResponse.php
  - database/migrations/2024_01_01_000000_create_passkeys_table.php
  - resources/js/passkeys.js
  - resources/views/components/passkey-registration.blade.php
  - resources/views/components/passkey-verify.blade.php

- Applied (upstream scoped wholesale, took 51fe9ad):
  - app/Http/Responses/LoginResponse.php
  - app/Http/Responses/RegisterResponse.php
  - app/Http/Responses/TwoFactorLoginResponse.php

- Applied (manual merge, passkey-only scope):
  - app/Models/User.php — added `PasskeyUser` interface + `PasskeyAuthenticatable` trait
  - app/Providers/FortifyServiceProvider.php — `PasskeyLoginResponse` singleton binding + `passkeys` rate limiter
  - composer.json — `laravel/fortify ^1.34 → ^1.37.2`; added `laravel/passkeys ^0.2.0`
  - config/fortify.php — `passkeys` config section, `Features::passkeys(['confirmPassword' => true])`, `passkeys` limiter
  - package.json — added `@laravel/passkeys ^0.2.0`
  - resources/views/pages/auth/login.blade.php — `<x-passkey-verify />`
  - resources/views/pages/auth/confirm-password.blade.php — `<x-passkey-verify ...>` with confirm routes/labels
  - resources/views/pages/settings/⚡security.blade.php — hand-ported passkey list + delete modal into the user's renamed file
  - tests/Feature/Auth/AuthenticationTest.php — Pest-style passkey redirect test
  - vite.config.js — added `resources/js/passkeys.js` to inputs

- Skipped (chisel scaffolding, per user request):
  - app/Console/Commands/InstallFeaturesCommand.php
  - chisel.php
  - chisel-paths.php

- Skipped (no passkey content / kept user's version):
  - routes/web.php — user's team-prefixed routing, no passkey changes upstream
  - routes/settings.php — not modified by 51fe9ad
  - resources/views/welcome.blade.php — only `@fonts` swap (depends on bunny plugin we skipped) + registration markers
  - resources/views/pages/settings/⚡profile.blade.php — no passkey content upstream
  - resources/views/pages/settings/profile.blade.php — would clash with user's `⚡profile.blade.php` (rename)
  - resources/views/pages/settings/security.blade.php — content hand-ported into `⚡security.blade.php` instead
  - tests/Feature/Settings/SecurityTest.php — upstream uses PHPUnit class-style (user has Pest function-style); assertions depend on hand-ported UI

- Later-edit drift scoped manually:
  - app/Providers/FortifyServiceProvider.php — used `51fe9ad:` content (HEAD adds unrelated email-verification handling, commit `8f23797`)
  - composer.json — manual scoped edit (HEAD drift swaps Pest→PHPUnit, adds `laravel/chisel`, drops `laravel/boost`, adds `laravel/pao` — all unrelated)
  - package.json — manual scoped edit (HEAD's `4f869d2` removes axios, unrelated)
  - vite.config.js — used `51fe9ad`-only content (eed0e1a bundles bunny fonts plugin, unrelated)

- Discovered upstream gap:
  - Upstream `composer.json` does **not** declare `laravel/passkeys` as a dependency, even though `PasskeyLoginResponse.php` and `FortifyServiceProvider.php` import `Laravel\Passkeys\*`. Added explicitly here (`^0.2.0`).

## Lockfile updates

Committed separately as `931bfbc`:
- `composer.lock` — regenerated via `composer update laravel/fortify laravel/passkeys --with-all-dependencies`
- `package-lock.json` — regenerated via `npm install`
- `AGENTS.md`, `CLAUDE.md`, `GEMINI.md` + `.{agents,claude,cursor}/skills/fortify-development/SKILL.md` — refreshed by composer's `post-update-cmd → boost:update` hook; adds "passkeys"/"WebAuthn" to the fortify-development skill description

## Verification

- Baseline: `/var/folders/z5/.../skup-baseline.XXXXXX.json.wQzpwrCcJk`
  - `php_tests`: pre-existing failure (Pint `--test` lint detected formatting fixes needed before tests run)
  - `js_typecheck`: not discovered
  - `js_build`: pass
- Result after upgrade (`--compare`): **PASS** — exit 0, no regressions.
- Direct Pest run (bypassing the `lint:check` precheck): **125 passed, 271 assertions, 2.74s**.
- New test `passkey login response redirects to the current team dashboard` passes.

## Outstanding TODOs (your call)

- **Migration**: `database/migrations/2024_01_01_000000_create_passkeys_table.php` is unrun. Recommend `php artisan migrate:status` first; on a populated DB the timestamp prefix `2024_01_01_000000` will sort it to the start of the migration history, which is fine for fresh installs but can look odd in production.
- **Auth views (welcome)**: registration link still under `@if (Route::has('register'))` — fine. The `@fonts` directive from upstream wasn't applied (would require the bunny fonts plugin we skipped).
- **Settings/Security tests**: the assertions about "Passkeys" / "No passkeys yet" text — your `⚡security.blade.php` now has the text, so you can re-add those Pest-style asserts if you want test coverage. Sample:
  ```php
  $response->assertSee('Passkeys');
  $response->assertSee('No passkeys yet');
  ```
- **Pre-existing Pint lint**: `composer test` is gated by `pint --test` which currently fails on a bunch of `single_blank_line_at_eof` fixes. Run `vendor/bin/pint` once to clear it.
- **`passkeys-with-boost` branch**: you have an unrelated branch from a prior passkeys attempt. Decide whether to keep or drop (`git branch -D passkeys-with-boost`).

## How to revert

- Drop both commits: `git revert 931bfbc 084ffc9` (revert lockfiles first, then code)
- Discard the whole branch:
  ```
  git checkout main
  git branch -D starter-kit-upgrade/20260520-2008-passkeys
  composer install   # restore lock state
  npm install
  ```
- Revert only the lockfile commit (keep code, redo deps yourself): `git revert 931bfbc`
