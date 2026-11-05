# nextcloud-app-phpunit

Shared PHPUnit utilities, used only by [rotdrop/cafevdb](https://github.com/rotdrop/cafevdb)
and [rotdrop/cafevdb-members](https://github.com/rotdrop/cafevdb-members).

## Composer installation

Install `rotdrop/nextcloud-app-phpunit` as a development dependency in the app's
`vendor-bin/phpunit` Composer project, not in the namespace-scoped
`composer-wrapped.json`. Until a tagged release is available, add this VCS
repository to `vendor-bin/phpunit/composer.json`:

```json
{
  "repositories": [
    {
      "type": "vcs",
      "url": "https://github.com/rotdrop/nextcloud-app-phpunit"
    }
  ]
}
```

After the Composer refactor is merged to `main`, run from the app root:

```sh
composer require --dev --working-dir=vendor-bin/phpunit \
  rotdrop/nextcloud-app-phpunit:dev-main
```

Composer maps `OCA\RotDrop\Tests\` to `src/`; consumers must remove their old
PSR-4 mapping of that namespace to `tests/phpunit/lib/` and remove the shared
git subrepo. Keep app-specific test namespaces and subclasses unchanged.
Nextcloud's `OC`/`OCP` APIs and the active app's scoped toolkit remain supplied
by the consuming environment; Composer installs the other declared dependencies.

## PHPUnit bootstrap integration

Before loading `DatabaseProvider` (including app-specific subclasses) or
`DatabaseConfig`, the consuming app must:

1. Initialize Nextcloud, including its server container.
2. Load its PHPUnit bin-project, app Composer, and scoped (`vendor-wrapped`)
   autoloaders.
3. Define `PHPUNIT_NC_APP_NAMESPACE` as its app namespace, without a trailing
   backslash, and `PHPUNIT_NC_APP_NAME` as its app ID.
4. Load the app so its services are registered before resolving the provider.

Both helpers automatically load `bootstrap.php`. Alternatively, the app's
`tests/phpunit/bootstrap.php` can explicitly load it after the steps above and
before resolving the provider:

```php
require_once __DIR__ . '/../../vendor-bin/phpunit/vendor/autoload.php';
// Initialize the app as described above before loading the shared bootstrap.
require_once __DIR__ . '/../../vendor-bin/phpunit/vendor/rotdrop/nextcloud-app-phpunit/bootstrap.php';
```

`bootstrap.php` is a procedural entry point, not a PSR-4 class. It deliberately
is not listed in Composer's `autoload.files`: those files run immediately when
the autoloader is loaded, before the app constants and scoped toolkit are ready.
The lazy `require_once` in `DatabaseProvider` and `DatabaseConfig` preserves
automatic setup, while explicit inclusion remains available. Merely loading
Composer is safe for standalone tests that do not initialize Nextcloud.

The shared bootstrap aliases the active app's
`PHPUNIT_NC_APP_NAMESPACE . '\\Wrapped\\OCA\\RotDrop\\Toolkit'` DTO base class,
logger trait, and executable finder to `OCA\RotDrop\Tests\Toolkit`. It also
registers a Nextcloud container alias for the finder, preserving the active
app's existing service instance and constructor type safety.

Members' server bootstrap already loads the PHPUnit bin-project autoloader;
CafeVDB's bootstrap should explicitly load it too. Members'
`bootstrap-standalone.php` must not explicitly include the shared bootstrap.
Each app must run in its own PHP process, since
the app namespace constant and toolkit aliases are process-wide. The other
app does not need to be installed.

## Regression checks

After `composer install`, run `composer test` to check real Composer autoloading,
lazy and explicit bootstrap loading, and standalone autoloader safety.

Run each app and helper load order in a separate process:

```sh
for app in 'OCA\CAFEVDB' 'OCA\CAFeVDBMembers'; do
  php tests/verify-bootstrap.php "$app" DatabaseProvider
  php tests/verify-bootstrap.php "$app" DatabaseConfig
done
php tests/verify-bootstrap.php missing-namespace
php tests/verify-bootstrap.php missing-toolkit
```

These dependency-free checks use isolated toolkit and container fixtures. They
cover alias selection, DTO inheritance and serialization, logger access, finder
type safety and service identity, repeated bootstrap loading, and invalid setup
diagnostics. They do not start a database or replace the consuming apps' PHPUnit
suites.
