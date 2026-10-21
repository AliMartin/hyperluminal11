# Hyperluminal Drupal 11 appraisal

Historical assessment: steps 1–5 were subsequently implemented locally. See [the Drupal 11 upgrade record](docs/DRUPAL-11-UPGRADE.md) for the final state and verification.

Assessed 21 October 2026. Scope: local working tree, Composer metadata, running DDEV environment, local database, and HTTP responses. No application code, dependencies, settings, or database updates were changed. Existing working-tree changes were preserved. Read-only Drupal requests can populate normal caches.

## Assessment

This is a good candidate for an in-place Drupal 11 upgrade. The application is small and uses standard Drupal functionality, with little custom code. The main work is stabilising the current installation, updating Webform, and improving deployment and file protection. It is not yet ready for a production upgrade.

Production infrastructure and production content have not been inspected. Local compatibility declarations do not substitute for a completed Drupal 11 upgrade rehearsal and functional testing.

## Current site

| Area | Observed state |
| --- | --- |
| Drupal | 10.6.10 in core, composer.lock, and installed Composer metadata |
| Project layout | drupal/legacy-project; repository root is the document root |
| Local runtime | DDEV 1.25.1, nginx-fpm, PHP 8.3.30, MariaDB 11.8 |
| Default theme | Custom `hyperluminal25`, based on Bootstrap5; Claro administration theme |
| Custom code | One subtheme, an effectively empty `.theme` file, SCSS and compiled CSS; no custom modules found |
| Content | Five project nodes, one webform node; one image media entity; 31 managed files |
| Users/submissions | Three user records and eight webform submissions; no personal submission data inspected |
| Files | Approximately 21 MB public files, 2.9 MB private files, 1.6 MB custom theme |
| Configuration | 53 enabled module entries including the installation profile; four installed themes |
| Checks | Composer validation passes; installed platform requirements pass; custom theme PHP syntax passes |
| HTTP | `/`, `/home`, and `/contact` return HTTP 200 through the local DDEV router |

The homepage uses a Views listing of projects ordered by `field_order`, with image styles and external project links. It also renders a looping background video and a contact link. Media Library, Layout Builder, CKEditor 5, caching, Honeypot, and Webform are enabled. Article and basic-page content types exist in configuration but have no current nodes in the queried local database.

## Priority findings

### 1. Security updates are overdue

`composer audit --locked --format=json` reports 46 advisory entries across six packages:

| Package | Advisory entries |
| --- | ---: |
| drupal/core | 11 |
| drupal/webform | 20 |
| guzzlehttp/guzzle | 9 |
| guzzlehttp/psr7 | 4 |
| symfony/http-foundation | 1 |
| symfony/yaml | 1 |

These are package matches, not proof that every issue is exploitable on this site. They include critical core and Webform advisories. Core's Field UI module is enabled, which is relevant to the current core remote-code-execution advisory. Drupal's advisory identifies 10.6.19, 11.3.19 and 11.4.10 as fixes on their respective branches. Webform 6.3.1 contains security fixes and supports Drupal 10.3 and 11. Select current supported patched releases when implementing, and rerun the audit.

Composer also has `audit.block-insecure: false`, so dependency resolution is explicitly allowed to select vulnerable packages. It reports `oomphinc/composer-installers-extender` as abandoned. The `libraries/` directory contains no installed library assets; review whether the extender, Asset Packagist repository, and related installer configuration are still needed before removing them.

Sources: [core security advisory](https://www.drupal.org/sa-core-2026-014), [Webform 6.3.1](https://www.drupal.org/project/webform/releases/6.3.1).

### 2. Private and configuration files are accessible locally

`settings.local.php` places private files, temporary files, and configuration exports under the document root: `/private-files`, `/tmp`, and `/sync`. Header-only HTTP requests to a sample file in each directory returned 200 through the DDEV router. Private files include Backup and Migrate backups; temporary files include a configuration archive.

The folders contain Apache `.htaccess` restrictions, but the local server is nginx. Its inspected configuration has no explicit denial for these directories and permits static gzip files. This finding is confirmed locally only; production exposure remains unknown.

Move sensitive storage outside the document root, or add and verify explicit server-level denials as appropriate. Exclude backups and temporary content from Git and deployment artifacts. Check production protections separately. Backup existence is not evidence of a tested restore.

### 3. Drush commands fail while loading Webform

`drush status`, `drush updatedb:status`, and `drush config:status` all fail with:

```text
Route "webform.addons" does not exist.
```

Drush connects to the database and boots Drupal's kernel, then fails when constructing Webform's help service for its command registration. The route exists in the installed Webform routing YAML but is absent from the database router table. This suggests inconsistent routing/cache state; the cause has not been established. No cache rebuild or repair was performed during this appraisal.

Resolve this on Drupal 10 before attempting dependency or schema changes. Pending database updates cannot yet be certified as clear.

### 4. Git does not yet contain the application baseline

`git ls-files` currently lists only `.gitignore` and `README.md`. Composer manifests, the custom theme, configuration exports, and deployment configuration are untracked. The two tracked files already have local changes.

The ignore rules blanket-ignore `/modules/*`, which would also hide future custom modules. Private backups and temporary archives appear within untracked directories. Create an intentional baseline containing Composer manifests, custom code, approved configuration and deployment instructions, while excluding secrets, generated dependencies, uploads and backups. Do not indiscriminately stage all untracked files.

### 5. Active configuration differs from the exports

A direct comparison using Drupal's configuration storage API found four active items absent or different on disk; all four exported names exist:

- `backup_migrate.backup_migrate_schedule.daily_schedule`
- `filter.format.webform_default`
- `webform.settings`
- `webform.webform.contact`

No exported configuration names were missing from active storage. Review and reconcile these differences before importing configuration. In particular, importing the contact form could overwrite current form behaviour or mail settings. No values were changed or imported.

## Dependency readiness

| Project | Locked version | Drupal 11 assessment |
| --- | --- | --- |
| Admin Toolbar | 3.6.3 | Declares `^11` support |
| Backup and Migrate | 5.1.4 | Declares `^11` support; verify backup and restore behaviour |
| Bootstrap5 | 4.0.8 | Declares `^11.0` support |
| Honeypot | 2.2.2 | Declares `^11` support |
| HTTPS and WWW Redirect | 2.1.3 | Declares `^11` support; test redirects behind production proxy |
| Responsive Favicons | 3.0.1 | Declares `^11` support |
| Webform | 6.2.10 | Blocks Drupal 11: requires core `^10.2`; move to a patched 6.3 release |
| Drush | 13.7.3 | PHP requirement is satisfied locally; supports Symfony 6/7 dependencies |
| Hyperluminal25 | Custom | Declares `^10.1 || ^11.0`; minimal PHP migration surface |

The root Webform constraint `^6.2@beta` permits 6.3 releases; the lock file is the immediate incompatibility. Replace the beta allowance with an intentional stable constraint during the upgrade. All four core-related root requirements currently constrain core to `^10` and must be updated together for Drupal 11.

The common core extensions removed in Drupal 11, such as Book, Forum, Statistics, Tour, Tracker and Actions UI, are not listed as enabled. No obvious dependency on these was found. This does not replace an Upgrade Status scan of the active site.

Local PHP and MariaDB satisfy Drupal 11's published baseline requirements; production PHP, extensions, database, Composer and web-server settings still need verification against the exact target minor release. Sources: [PHP requirements](https://www.drupal.org/docs/getting-started/system-requirements/php-requirements), [database requirements](https://www.drupal.org/docs/getting-started/system-requirements/database-server-requirements), [removed extensions](https://www.drupal.org/docs/core-modules-and-themes/deprecated-and-obsolete).

## Theme and deployment observations

- The subtheme disables Bootstrap5's global styling library and supplies compiled CSS. Its SCSS imports the parent theme's SCSS directly, making parent-theme changes a visual regression concern.
- Sass compilation is documented, but there is no pinned frontend package manifest or build command in the project. There are no custom Twig or JavaScript files in the inspected subtheme.
- The background video, project logos, favicon assets, contact form and editor styling should be explicit regression checks. Asset byte sizes, responsive layout, accessibility and browser rendering have not been audited.
- The exported content-editor role has permissions for articles and pages, but no explicit project editing permissions. This is an existing editorial configuration observation, not a Drupal 11 blocker.
- Both configured 403 and 404 pages point to `/home`. Check error responses and visitor behaviour during regression testing.
- `.cpanel.yml` uses recursive wildcard copying into `public_html`. It has no explicit Composer install, database updates, cache rebuild, backup or rollback steps, and wildcard copying omits dotfiles such as `.htaccess`. It may copy inappropriate root directories depending on the deployment checkout contents. Review the actual production deployment process before use.
- No project-specific automated tests, CI configuration, Upgrade Status installation, PHPStan or Rector configuration were found. Generated/contributed test suites were excluded from this inventory.

## Recommended upgrade sequence

1. Establish a clean, intentional Git baseline and tested database/files restore procedure. Verify production runtime and file-access protection.
2. Repair the Drupal 10 route/Drush issue. Obtain successful status, pending-update and configuration-difference reports.
3. Apply current Drupal 10 security updates and a patched stable Webform 6.3 release on the local clone. Apply pending updates and verify contact submissions and mail capture in Mailpit.
4. Reconcile configuration drift and run Upgrade Status/deprecation checks. Record a working functional and visual baseline.
5. Update the coordinated Composer core constraints to a supported patched Drupal 11 release and resolve all dependencies. Change DDEV's project type to `drupal11` as part of this work. Run database updates and rebuild caches.
6. Verify anonymous homepage/contact behaviour, form validation, Honeypot, stored submissions, captured email, administrator login, project editing/order, CKEditor, media, image styles, video, favicons, redirects, errors, cron and backup/restore. Confirm unchanged existing content and submission counts.
7. Rehearse deployment and rollback against a staging copy. Deploy the tested Composer lock file and intended configuration, then verify production behaviour and a clean security audit.

Drupal 10 reaches end of life on 9 December 2026, so this work should be scheduled promptly: [official release schedule](https://www.drupal.org/about/core/policies/core-release-cycles/schedule?showAll=true).

## Limits of this appraisal

No upgrade, configuration import, cache repair, form submission, email send, production access or deployment was performed. A browser preview was unavailable, so HTTP and rendered markup checks do not constitute visual QA. A complete solver dry run, Upgrade Status scan, database-update validation and production runtime check remain prerequisites to claiming upgrade readiness.
