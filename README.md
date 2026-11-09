# 1994/coding-standard

Shared code style, static analysis and CI for 1994 and Porro PHP packages.

- **Pint**: `pint.json` (Laravel preset). Repos extend it.
- **PHPStan**: `phpstan/base.neon`, plus `phpstan/laravel.neon` with Larastan.
- **CI**: reusable workflow `.github/workflows/php.yml`.
- **Local checks**: a `pre-push` hook that runs the same `composer ci` as CI.

## The rule: CI only calls composer scripts

CI never runs `vendor/bin/pint` or `phpstan` directly. It runs the repo's own
scripts:

| Where | Command | What it runs |
|---|---|---|
| CI quality job (once) | `composer ci:quality` | `pint --test` and `phpstan analyse` |
| CI test matrix (each leg) | `composer ci:test` | the test suite |
| Local / pre-push | `composer ci` | both of the above |
| Local, optional | `composer ci:lowest` | tests on lowest deps, then restores |

Because both sides call the same scripts, what passes `composer ci` on your
machine passes the quality job and the test leg that matches your PHP
version. The other legs (older PHP, other framework versions) only run in
CI. To try one locally, use `composer ci:lowest`, or change PHP with Herd
and run `composer ci` again.

## Adding it to a repo

1. Install:

   ```bash
   composer require --dev 1994/coding-standard
   ```

2. Replace `pint.json` with:

   ```json
   { "extend": "vendor/1994/coding-standard/pint.json" }
   ```

   Keep any repo-only `exclude` or `notPath` entries next to `extend`.

3. Replace `phpstan.neon` with:

   ```neon
   includes:
       - vendor/1994/coding-standard/phpstan/base.neon   # or laravel.neon
   parameters:
       level: 6        # 8 for *-core
       paths:
           - src
   ```

   If it fails, record the current errors and fix them later. Don't lower
   the level:

   ```bash
   vendor/bin/phpstan analyse --generate-baseline
   ```

   Then add `- phpstan-baseline.neon` to `includes`. Add `.phpstan-cache/` to
   `.gitignore`.

4. Merge the scripts from `stubs/composer-scripts.json` into `composer.json`.
   Change `test` if the repo uses PHPUnit (`phpunit`) or `php artisan test`.
   The `post-install-cmd` and `post-update-cmd` entries install the pre-push
   hook. The installer does nothing in CI, and it leaves alone any pre-push
   hook it didn't write.

5. Copy `stubs/ci.yml` to `.github/workflows/ci.yml`, edit the matrix, and
   delete the old workflow.

## Matrix entries

```json
{"php": "8.4", "require": "laravel/framework:13.* orchestra/testbench:11.*"}
{"php": "8.2", "deps": "lowest"}
{"php": "8.2", "require": "laravel/framework:11.*", "insecure": true}
```

`insecure` turns off Composer's advisory block for that leg. Use it for
end-of-life framework versions we still support, such as Laravel 11.

`extensions` replaces the workflow's `extensions` input for one leg, e.g.
`{"php": "8.4", "extensions": ":sodium"}` to test without sodium.

## Repos that need their own test job

If tests need a service container or a scaffolded app (Craft with
Postgres, for example), the reusable workflow can't build it. Keep your own
workflow, but follow the same rule: one job runs `composer ci:quality`, and
the test job's last step runs `composer ci:test`.

## Exceptions

- **porro-wordpress-connector** must keep running on PHP 5.6, and Pint would
  rewrite it in newer syntax. Keep PHPCS with PHPCompatibilityWP there, and
  set its `ci:quality` script to run those instead. It can still use the
  shared workflow.
- **Porro app** already runs `composer ci:check` (Pint, Larastan, Vite+,
  vue-tsc and sharded tests). Switch it to the shared Pint and PHPStan base,
  but keep its own workflow for the sharding.

## Releasing

Tag `v1.x.y` and move the `v1` tag. Callers pin `@v1`. A rule change that
could fail existing repos goes out as a new major version.
