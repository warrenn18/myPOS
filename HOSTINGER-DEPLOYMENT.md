# Hostinger deployment review — 2026-09-21

**Status: do not deploy the current build unchanged.** Local functional checks pass, but framework/PDF security updates and production hosting configuration remain outstanding. This review did not access Hostinger or change any database records or schema.

## Required before release

1. **Update the bundled CodeIgniter framework.** `system/CodeIgniter.php` reports 4.7.0. Update to a supported patched release, at least 4.7.4, following the intervening upgrade notes. The framework lives in `system/` and is the Composer root project; running `composer update` alone will not update it. Preserve application configuration while reviewing required config changes. Re-run tests after updating.

   The official [HTTPS-header advisory](https://github.com/codeigniter4/CodeIgniter4/security/advisories/GHSA-7wmf-pw8j-mc78) affects versions below 4.7.4. Also review the [batch-delete advisory](https://github.com/codeigniter4/CodeIgniter4/security/advisories/GHSA-c9w5-rwh3-7pm9) and [upload advisory](https://github.com/codeigniter4/CodeIgniter4/security/advisories/GHSA-hhmc-q9hp-r662). No `deleteBatch()` calls were found in application code, and both located upload flows use generated filenames; these findings do not establish that those particular exploits are reachable in this app.

2. **Update Dompdf.** `composer.lock` contains `dompdf/dompdf` 3.1.5. `composer audit --locked --no-dev` reported six advisories. Update to at least 3.1.6, then re-run the audit and exercise report/stock-log PDF exports. The existing `^3.1` constraint permits that patch. Remote fetching is disabled in current report controllers, but this does not fix all local-resource issues. See the [maintainer's advisories](https://github.com/dompdf/dompdf/security/advisories) and [patched-version details](https://github.com/dompdf/dompdf/security/advisories/GHSA-cx96-42px-69fm).

3. **Create a production `.env` on the host.** The local environment is `development`, uses a localhost URL and a root database user. Do not upload it unchanged. Use the template below with the actual Hostinger account values.

4. **Keep private application files outside `public_html`.** Do not upload the whole workspace into the web root. Follow the layout below and enable SSL/HTTPS redirection in hPanel.

## Local checks completed

| Check | Result |
| --- | --- |
| Application/public PHP syntax | 167 files passed |
| JavaScript syntax | 5 files passed |
| Literal view and asset references | 164 view references and 15 asset references checked |
| Application class filename case | 48 classes checked |
| Route controller targets | 103 targets checked; no missing/non-public methods or filename-case problems found |
| PHPUnit | 28 tests, 20,411 assertions passed |
| Discount JavaScript tests | Passed |
| Composer manifest validation | Passed |
| Production platform requirements | Passed on local PHP 8.3.0; still must check the hosting runtime |
| Local database compatibility fields | All six fields listed below exist |
| Production dependency advisory audit | Failed: six Dompdf advisories |

The local database was started to run read-only metadata checks. Hostinger's PHP configuration, database, permissions, URL routing, cron and live user flows remain unverified. Static checks cannot resolve every dynamically generated path or prove every business workflow correct.

## Suggested shared-hosting layout

Use the actual domain directory shown in hPanel; the path below is illustrative. Hostinger documents `public_html` as the web root and restricts changing it on shared hosting: [root directory guidance](https://www.hostinger.com/support/1583494-what-is-the-path-to-your-website-s-root-home-directory-and-how-to-change-it-in-hostinger/).

```text
/home/uXXXX/domains/example.com/
    pharxmaco/                  # private project directory
        app/
        system/
        vendor/
        writable/
        public/                 # keep this for the existing spark bootstrap
        composer.json
        composer.lock
        spark
        .env                    # hosting values, not the local development file
    public_html/                # copy the CONTENTS of public/ here
        index.php
        .htaccess
        assets/
        favicon.ico
        robots.txt
```

In the **hosted `public_html/index.php` only**, replace its Paths require with:

```php
require FCPATH . '../pharxmaco/app/Config/Paths.php';
```

The current `Paths.php` resolves `system`, `writable`, views and `.env` relative to the private application directory. Keep that arrangement. Retain the private `public/` directory because the existing `spark` entry point changes into it. Publish asset updates to `public_html/assets/` too. If the host restricts access outside `public_html`, resolve that hosting restriction before using this layout.

Include the hidden `public/.htaccess` file when copying public files. Test a routed URL such as `/login`, not just `/index.php`.

Do not publish `tests/`, `build/`, database SQL files, deployment notes, `composer.phar`, local logs, sessions, backups, or anything from `writable/receipt-review/`. Preserve existing production uploads and backups when updating an existing installation.

## Production environment template

Create this as the private project's `.env`, replacing every placeholder. Keep it out of the public directory and out of shared archives.

```ini
CI_ENVIRONMENT = production
app.baseURL = 'https://example.com/'
app.forceGlobalSecureRequests = true
cookie.secure = true

database.default.hostname = 'HOSTINGER_DATABASE_HOST'
database.default.database = 'HOSTINGER_DATABASE_NAME'
database.default.username = 'HOSTINGER_DATABASE_USER'
database.default.password = 'HOSTINGER_DATABASE_PASSWORD'
database.default.DBDriver = MySQLi
database.default.DBDebug = false
```

Use the exact hostname in hPanel (it may be `localhost`) and the assigned database username, not the development root account. Keep the trailing slash on the base URL. This layout assumes the app is at the domain root. Enable HTTPS redirection at the hosting layer as well as in the app.

## Runtime, dependencies and writable folders

- Select PHP 8.3 or a newer version that has been tested with this application; the project requires PHP 8.2 or above. Ensure the web runtime and cron/SSH PHP versions agree. [Hostinger PHP configuration](https://www.hostinger.com/support/1575755-how-to-change-the-php-version-of-your-hostinger-hosting-plan/).
- Ensure `intl`, `mbstring`, `mysqli`, `dom`, `iconv`, `ctype` and `fileinfo` are available. Enable `gd` for PDF image support. Run `composer check-platform-reqs --no-dev` on the host when SSH/Composer is available; Composer does not cover every optional extension used by application features.
- After patching and testing dependencies, build a deployment copy with `composer install --no-dev --prefer-dist --optimize-autoloader` using the tested lockfile. Do not run an unrestricted dependency update on production. If there is no SSH, upload that deployment copy's `vendor/` together with `system/` and the app.
- Ensure PHP can write to `writable/cache`, `writable/logs`, `writable/session`, `writable/uploads` and `writable/backups`. Use host-appropriate ownership/permissions, not blanket `777`. Keep the protective `writable/.htaccess` file. Cache and sessions must work for login and throttling.
- Set upload/post limits for the actual CSV/backup sizes you need. CSV import permits up to 5 MB; application backup restore permits up to 256 MB, but PHP and hosting limits may be lower. Check memory/time limits for larger report exports and restores.

## Database: no new changes for the discount release

The latest discount fixes use the existing schema. Do not run a September 20 snapshot migration or SQL import. If an unused `discount_snapshot` column already exists, it can remain; do not remove data or migration history for this upload.

For an existing Hostinger database, verify the previously required fields with these **read-only** queries after selecting the website database in phpMyAdmin:

```sql
SELECT DATABASE();
SHOW COLUMNS FROM users WHERE Field = 'session_version';
SHOW COLUMNS FROM sales WHERE Field = 'checkout_token_hash';
SHOW COLUMNS FROM refund_items
WHERE Field IN ('return_condition', 'refund_method', 'refund_event_id');
SHOW COLUMNS FROM suppliers WHERE Field = 'email';
```

Expected row counts: 1, 1, 3 and 1 for the four `SHOW` queries. All six fields exist locally. The September 17 verification file also checks its migration record. Missing fields on the host would mean its earlier baseline update is incomplete, not that the latest discount fix needs a new schema.

For a completely empty database, the five included migrations are incremental and cannot create the entire original schema. A complete SQL export of the source database would be needed for initial setup. The application's `.pmbak` restore requires a matching schema and is not an empty-database installer. Never replace an existing live database with a local export merely to deploy file changes.

## Forecast schedule

If daily saved forecast history is required, configure a daily Hostinger cron task to run this command with the account's PHP CLI binary:

```sh
/path/to/php /home/uXXXX/domains/example.com/pharxmaco/spark forecast:reorder
```

Replace both paths with actual host paths. The command writes forecast snapshots; it was not run during this audit. Verify the scheduler's timezone separately from the application's `Asia/Manila` timezone. [Hostinger cron setup](https://support.hostinger.com/en/articles/1583465-how-to-set-up-a-cron-job-at-hostinger).

## Final checks on a staging copy

1. Back up the existing hosted files and database before replacing files. Keep production `.env` and writable data.
2. Verify HTTP redirects to HTTPS, `/login` works, CSS/JS load, and admin/cashier permissions behave correctly. Confirm there is no debug toolbar or detailed exception page.
3. Confirm requests for `/.env`, `/composer.lock`, `/database/` and `/writable/` cannot download private files.
4. On a staging database, exercise checkout with and without discounts, minimum/cap boundaries, insufficient payment, refunds and sale corrections. Check stock movements and totals.
5. Exercise receipt printing, dark mode and mobile layouts, CSV import preview, report PDF export and backup creation. Test backup restore only on a disposable database copy.
6. Confirm login sessions persist, logs are writable and the forecast schedule runs successfully if configured.

Security updates, a clean repeat of the relevant tests/audit, and these host checks are still required before marking the release ready.
