# Drupal 11 upgrade record

Completed locally on 21 October 2026. This records the implementation of appraisal steps 1–5, the requested clean configuration export, and the checks performed. Production was not accessed or deployed.

## Result

| Component | Before | After |
| --- | --- | --- |
| Drupal core and four core Composer packages | 10.6.10 | 11.4.11 |
| Intermediate Drupal 10 checkpoint | — | 10.6.20 |
| Webform | 6.2.10 | 6.3.1 |
| Responsive Favicons (follow-up below) | 3.0.1 | 4.0.0 |
| Backup and Migrate | 5.1.4 | 5.1.5 |
| Drush | 13.7.3 | 13.8.0 |
| Local PHP | 8.3.30 | 8.4.18 |
| Local MariaDB | 11.8 | 11.8 |
| DDEV project type | drupal10 | drupal11 |

Core requirements now use `^11.4`; Webform uses stable `^6.3`; the project requires PHP `^8.4`. The exact tested dependencies are in `composer.lock`. No custom-theme PHP, Twig, CSS or JavaScript changes were necessary.

## Recovery baseline

Commit `0708cd7` records the Drupal 10 application baseline. Before that commit, Git tracked only `.gitignore` and `README.md`; their existing edits were retained. The baseline adds Composer manifests, the custom theme, exported configuration, safe local settings, DDEV configuration and the appraisal.

Uploads, backups, temporary content, generated dependencies, credentials and incidental Finder files are excluded from Git. The blanket modules ignore was replaced with a contributed-modules ignore, allowing future custom modules to be tracked. Only the intended DDEV configuration and nginx snippet are tracked; generated DDEV files and certificates remain excluded.

Local recovery artifacts are in:

```text
.ddev/backups/drupal11-upgrade-20261021/
  database.sql.gz                  Original database before cache repair
  source-config.tar.gz             Original Composer/settings/config/theme/server files
  files.tar.gz                     Public files, private files and temporary files
  drupal10-patched.sql.gz           Updated Drupal 10 checkpoint, scanner uninstalled
  sync-before-purge.tar.gz          Export directory immediately before its purge
  drupal11-final.sql.gz             Final upgraded local database
  upgrade-status-drupal10.txt       Full compatibility scan output
  upgrade-status-results.json       Structured compatibility results
  drupal11-solver-dry-run.txt        Successful Composer upgrade dry run
  drupal11-database-updates.txt      Database-update log
  final-config-export.txt           Fresh export log
  http-validation.json             HTTP check results
  final-requirements.json           Remaining non-error status warnings
```

The backup directory and its parent have mode 0700. These local backups are ignored by Git, excluded from deployment, and protected by the web server's hidden-directory rule. Keep a separate private copy for off-machine recovery. They contain site data and must not be committed or exposed for download. Working log copies also remain in `/tmp/hyperluminal11-pre-upgrade-20261021/`.

The original gzip database dump was restored into a separate temporary database and verified: five project nodes, one webform node, eight submissions and 31 managed files. The temporary database was subsequently removed. The original source/config and file archives were retained; an end-to-end application restore from those archives has not been rehearsed.

Original artifact SHA-256 checksums:

```text
database.sql.gz       634c5d838231a29f731d8bb5c157cbf8cadfb74d8e1c208b5dfff9ecb4b7784a
files.tar.gz          03d2ec94131dc8ba337d4fc129c8d75ba4d110839766d9fc0380b465095ed0dd
source-config.tar.gz  6548f93a28e14bf1f3556b11d29b5cc1ee9d5967e9556ee03224a8cd691ceb08
```

For recovery, use an isolated copy of baseline commit `0708cd7`, PHP 8.3 and its original lock file. Restore the database, original settings, and matching files together. Importing an old database over a site with newer submissions would discard those newer records; reconcile them before any real rollback. The baseline contains known-vulnerable versions and is a recovery reference, not a deployment target.

## Changes performed

### Access protection and dependency maintenance

The local nginx server previously served files under `private-files`, `tmp` and `sync`. `.ddev/nginx/protected-paths.conf` now denies these directories with exact/prefix locations that take precedence over static-file and PHP handlers. It also protects vendor code, Composer metadata and project tooling/documentation.

`assets/scaffold/protected-paths.htaccess` is prepended to Drupal's generated root `.htaccess` through Composer's `extra.drupal-scaffold.file-mapping`. It protects the corresponding paths on Apache and survives future Composer scaffolding. The final generated `.htaccess` was compared against the current core file plus this prepend. nginx configuration validation and live local HTTP checks pass. Apache execution itself has not been tested.

Storage locations remain unchanged to preserve existing paths and the requested `/sync` layout. On other nginx hosts, install equivalent server rules; Apache `.htaccess` does not protect nginx. A future document-root restructure could move sensitive storage outside public serving entirely.

Removed the abandoned `oomphinc/composer-installers-extender`, unused Asset Packagist repository and asset installer types; no installed library assets depended on them. Removed `audit.block-insecure: false`, restoring Composer's default vulnerable-package blocking. Added the Symfony Runtime plugin to Composer's allow-list, as required by the Drupal 11 dependency installation.

The cPanel code-copy recipe now builds from the committed lock file and uses rsync exclusions instead of `cp -r *`. It includes the generated root `.htaccess`, and excludes settings, uploads, backups, temporary files, configuration exports and project documentation/tooling. A local rsync dry run verified these exclusions. This recipe copies code only; database/configuration rollout, maintenance mode and rollback still require the production deployment procedure. No cPanel deployment was run.

### Drupal 10 preparation and compatibility checks

`ddev drush cache:rebuild` repaired the initial missing `webform.addons` routing state. Drush then bootstrapped successfully and found no pending Drupal 10 database updates. Webform 6.3 subsequently removes the old addons route from its routing file; its absence on the final site is expected.

Updated Drupal 10 and all compatible locked dependencies, including Webform, before changing core major versions. The patched Drupal 10 audit reported no security advisories.

Temporarily installed Upgrade Status 4.3.10 and scanned all installed projects on Drupal 10. The custom Hyperluminal25 theme, HTTPS/WWW Redirect and Responsive Favicons had no known issues. The overall scan exited 3 because contributed-project findings require manual review; this was not a completely clean scan:

- Numerous test-class findings came from contributed test suites without their test dependencies installed.
- Webform and Honeypot's Drupal 11 hook attributes are unknown under Drupal 10. Optional integrations referenced absent modules/libraries, including HAL, Address, Devel Generate and CAPTCHA; those integrations are not enabled on this site.
- Deprecated optional submodules were reported even though those submodules are not installed.
- Bootstrap5's `spaceless` Twig usage is deprecated but remains available in the installed Twig 3 version. These are upstream maintenance items.
- Backup and Migrate uses readonly service properties that can conflict with Drupal serialization on PHP below 8.4. The local runtime and project PHP requirement were moved to 8.4.

No contributed-code patches were introduced. The temporary scanner was uninstalled, and its development dependencies removed before the final export. Residual upstream warnings warrant normal regression testing; declaration of Drupal 11 support is not a substitute for production testing.

### Drupal 11 database and configuration

Composer's dry run resolved Drupal 11.4.11 without dependency conflicts. The first installation stopped at Symfony Runtime's plugin allow-list check; explicitly allowing that plugin and running `composer install` completed the same resolved lock file. No platform requirements were bypassed.

Applied all Drupal 11 database updates and rebuilt caches. One update exposed an orphaned `block.block.claro_help_search` configuration referencing `search_form_block`, although Search is not enabled. Removed that obsolete help-search block. The optional search block supplied by the custom theme remains optional and inactive.

The database contains 104 pre-existing `dr9d_` legacy tables alongside the active, unprefixed site tables. Drupal's database requirement check flagged `dr9d_honeypot_user` as lacking a primary key. Verified that it was empty, then added an auto-increment unsigned integer `id` primary key. No legacy tables or rows were deleted. The active `honeypot_user` table already had its correct primary key. If deploying against a different database, inspect its actual schema before considering this local repair.

The four pre-existing export differences were resolved by preserving active configuration, rather than importing the stale exports: the backup schedule, Webform text format, global Webform settings and contact form. Existing email routing and form behaviour were retained.

After the upgrade, archived and emptied the entire `sync` directory, ran `ddev drush config:export -y`, regenerated its standard Apache `.htaccess`, and removed ignored Finder metadata. The fresh export includes core's configuration migrations and removal of obsolete RSS/help-search configuration. The final export contains 312 YAML configuration files. A final `config:status` reports no differences.

## Verification

| Check | Result |
| --- | --- |
| Drupal bootstrap | Successful, Drupal 11.4.11 / PHP 8.4.18 |
| Database updates | None pending |
| Entity-definition changes | Empty change summary |
| Drupal error-level requirements | None |
| Composer strict validation | Pass |
| Composer platform requirements | Pass |
| Composer locked-package audit | No security advisories |
| Production `composer install --no-dev --dry-run` | Installable, no package changes required |
| nginx configuration | Valid |
| `/`, `/home`, `/contact` | HTTP 200 |
| Contact form markup | Form rendered successfully; no submission made |
| Local images, video, favicons, CSS and JavaScript referenced by the pages | 19 asset URLs return HTTP 200 |
| Backup archive, temporary archive, configuration export, Composer files, vendor PHP and deployment metadata | HTTP 403 |
| Active vs exported configuration | No differences |
| Existing content | Five project nodes, one webform node, eight submissions, 31 managed files, three users retained |

Warnings that remain are documented rather than hidden: DDEV intentionally permits writable settings; History is deprecated; Bootstrap5 currently depends on deprecated Stable9; Drupal 11.4 flags the future HTML5-validation default; the Update module may lack fetched release data. These are distinct from Composer's successful security audit and do not appear as error-level requirements. History and Stable9 were retained to preserve behaviour and theme dependencies.

## Future use and production handover

Use `ddev composer install` to reproduce the lock file; do not run a general dependency update simply to deploy this upgrade. After database changes, run `ddev drush updatedb -y`, `ddev drush cache:rebuild`, and verify `ddev drush config:status` before exporting or importing configuration. Never purge `/sync` during routine deployment; the purge in this task was specifically requested and backed up first.

Before production rollout, verify PHP 8.4 and required extensions, database compatibility, web-server protection, settings permissions, backups and restore procedures. The production config-sync path must remain available to administrative tooling even though the code-copy recipe intentionally excludes `/sync`; use a protected release checkout or a separate non-public config directory.

Complete staging/production regression work from appraisal step 6: visual/mobile review, administrator/editor workflows, actual contact submissions with captured mail, Honeypot, redirects/error pages, cron and backup/restore. No production access, browser visual QA, form submission, email send or deployment was performed in this task.

References: [official Drupal 10-to-11 upgrade guide](https://www.drupal.org/docs/upgrading-drupal/upgrading-drupal/how-to-upgrade-from-drupal-10-to-drupal-11), [Webform 6.3.1](https://www.drupal.org/project/webform/releases/6.3.1), [DDEV customisation](https://docs.ddev.com/en/stable/users/extend/customization-extendibility/).

## Follow-up: Responsive Favicons 4.0.0

Updated on 21 October 2026 at the user's request. Changed the Composer requirement from `^3.0` to stable `^4.0` and updated the lock file from 3.0.1 to 4.0.0. No other package versions changed. Version 3 was Drupal 11 compatible; version 4 is the newer stable release and refactors favicon handling into a service. The available 4.1 beta was not selected.

Saved a database checkpoint as `.ddev/backups/drupal11-upgrade-20261021/before-favicons4.sql.gz`. Ran database updates (none pending), rebuilt caches, and exported configuration. The module's existing path and five configured tags were preserved; the export remains identical to active configuration.

The new `responsive_favicons.manager` service reports five links and no missing files. Both `/` and `/contact` render exactly the expected PNG, SVG, ICO, Apple touch icon and web-manifest links; all linked files return HTTP 200. Composer's audit and strict manifest/lock validation pass.

Composer's scaffold operation initially replaced the project README with Drupal's generic README. Restored the project README and added `[web-root]/README.md: false` to the scaffold file mapping to preserve it on subsequent installations. Refreshed lock metadata without further dependency changes.

Drupal initially showed a cached unsupported-contributed-release status. Refreshed its project and release metadata; every installed contributed project now reports the current recommended stable release, including Responsive Favicons 4.0.0. This supersedes the earlier note about missing release data. No form submissions or production changes were made.

Release reference: [Responsive Favicons 4.0.0](https://www.drupal.org/project/responsive_favicons/releases/4.0.0).
