# Hyperluminal

Drupal 11.4 website using the custom Hyperluminal25 Bootstrap subtheme. Local development uses DDEV, PHP 8.4 and MariaDB 11.8.

See [the upgrade record](docs/DRUPAL-11-UPGRADE.md) for changes, validation, backups, recovery notes and production handover requirements. The [original appraisal](DRUPAL-11-APPRAISAL.md) records the state before the upgrade.

For an existing local copy:

```sh
ddev start
ddev composer install
ddev drush status
```

A fresh checkout also needs the site's database, public/private files and environment-specific settings; these are deliberately excluded from Git. After Composer creates core, restart DDEV so it can generate local settings, then restore the site's database/files. Do not install a new empty Drupal site as a substitute for restoring Hyperluminal's content.

Configuration is exported into `/sync`; web access to it is denied. Use `ddev drush config:status` to review differences and `ddev drush config:export -y` to export intended active changes. Database credentials and hash salts belong in ignored settings/environment configuration, not in tracked `settings.local.php`.

The cPanel recipe copies code only. Follow the production handover notes for database/configuration updates and staging checks before any deployment.

---

<img alt="Drupal Logo" src="https://www.drupal.org/files/Wordmark_blue_RGB.png" height="60px">

Drupal is an open source content management platform supporting a variety of
websites ranging from personal weblogs to large community-driven websites. For
more information, visit the Drupal website, [Drupal.org][Drupal.org], and join
the [Drupal community][Drupal community].

## Contributing

Drupal is developed on [Drupal.org][Drupal.org], the home of the international
Drupal community since 2001!

[Drupal.org][Drupal.org] hosts Drupal's [GitLab repository][GitLab repository],
its [issue queue][issue queue], and its [documentation][documentation]. Before
you start working on code, be sure to search the [issue queue][issue queue] and
create an issue if your aren't able to find an existing issue.

Every issue on Drupal.org automatically creates a new community-accessible fork
that you can contribute to. Learn more about the code contribution process on
the [Issue forks & merge requests page][issue forks].

## Usage

For a brief introduction, see [USAGE.txt](/core/USAGE.txt). You can also find
guides, API references, and more by visiting Drupal's [documentation
page][documentation].

You can quickly extend Drupal's core feature set by installing any of its
[thousands of free and open source modules][modules]. With Drupal and its
module ecosystem, you can often build most or all of what your project needs
before writing a single line of code.

## Changelog

Drupal keeps detailed [change records][changelog]. You can search Drupal's
changes for a record of every notable breaking change and new feature since
2011.

## Security

For a list of security announcements, see the [Security advisories
page][Security advisories] (available as [an RSS feed][security RSS]). This
page also describes how to subscribe to these announcements via email.

For information about the Drupal security process, or to find out how to report
a potential security issue to the Drupal security team, see the [Security team
page][security team].

## Need a helping hand?

Visit the [Support page][support] or browse [over a thousand Drupal
providers][service providers] offering design, strategy, development, and
hosting services.

## Legal matters

Know your rights when using Drupal by reading Drupal core's
[license](/core/LICENSE.txt).

Learn about the [Drupal trademark and logo policy here][trademark].

[Drupal.org]: https://www.drupal.org
[Drupal community]: https://www.drupal.org/community
[GitLab repository]: https://git.drupalcode.org/project/drupal
[issue queue]: https://www.drupal.org/project/issues/drupal
[issue forks]: https://www.drupal.org/drupalorg/docs/gitlab-integration/issue-forks-merge-requests
[documentation]: https://www.drupal.org/documentation
[changelog]: https://www.drupal.org/list-changes/drupal
[modules]: https://www.drupal.org/project/project_module
[security advisories]: https://www.drupal.org/security
[security RSS]: https://www.drupal.org/security/rss.xml
[security team]: https://www.drupal.org/drupal-security-team
[service providers]: https://www.drupal.org/drupal-services
[support]: https://www.drupal.org/support
[trademark]: https://www.drupal.com/trademark
