# disrex/magento-security-patches

Adobe's security patches for Magento Open Source, Adobe Commerce and B2B, as one Composer package. `disrex/magento-patch-installer` applies them, checks on every Composer run that they are still applied, and fails the run when one is missing or conflicts.

Every file in `patches/adobe/` comes byte for byte from `https://repo.magento.com/patch/`, and `bin/sync-registry` refuses a download whose sha256 differs from Adobe's patch registry. `registry/patch-registry.json` is the registry as it stood at the last sync, so `git diff` on it shows what Adobe changed.

## What a store gets

| Store | Patches |
|---|---|
| 2.4.7-p10, 2.4.8-p5, 2.4.9 | Every monthly isolated patch from July 2026 on, in order, plus VULN-39341 (APSB26-146) |
| 2.4.6-p15 | July and August. Adobe serves September and later only to Commerce licence holders |
| Older patch levels from 2.4.4 on | VULN-39341 only. The monthly patches need the newest `-p` of the line, so upgrade first |
| Adobe Commerce | The Open Source patches above, plus the EE line for its `magento/product-enterprise-edition` version |
| B2B | Also the B2B line for its `magento/extension-b2b` version (1.4.2-p10, 1.5.2-p5, 1.5.3) |
| Mage-OS | Nothing. Mage-OS has no `magento/magento2-base`; it ships Adobe's fixes in its own releases |

Each area is its own cumulative line, gated on the area's package, so an Open Source store never sees the EE or B2B patches. Adobe's EE and B2B patches only touch EE and B2B modules. Files Adobe serves only to Commerce licence holders are left out, and so is every later month of that line, since a chain may not have a gap. `bin/sync-registry` prints them as `skipped`.

`composer magento-patches:status` on a store shows what it has and what it lacks.

## Adding it to a project

```bash
composer config --no-plugins allow-plugins.disrex/magento-patch-installer true
composer config --no-plugins --json extra.magento-patches.trust '["disrex/*"]'
composer remove --no-update disrex/stylesmuggler-adobe-patches
composer require disrex/magento-security-patches
```

Without the `allow-plugins` line, a CI run skips the plugin without a word and exits 0 on an unpatched store. Without `trust`, the run fails and names this package.

Then add a check after `composer install` in the build job of `.gitlab-ci.yml`, so a store that is not fully patched stops the pipeline before anything reaches the server:

```yaml
    - composer install --no-ansi --no-interaction --optimize-autoloader
    - composer magento-patches:verify --no-interaction
```

Exit code 0 means every patch is applied, 1 means one is missing, 2 is a conflict and 3 a configuration error.

A patch that breaks something in one project can be switched off in that project's root `composer.json`, with the reason:

```json
"extra": {
    "magento-patches": {
        "sources": {
            "disrex/magento-security-patches": {
                "skip": { "2026-10-001@2.4.8-p5": "breaks the PDF customizer, DISC-123" }
            }
        }
    }
}
```

Commit `magento-patches.lock` after the first `composer update`; it lists every patch the project takes, so each merge request that bumps this package shows which patches arrive or change. Renovate runs Composer without plugins by default and then leaves that file as it was; on our self-hosted Renovate, `allowPlugins: true` lets the plugin rewrite it.

Remove other copies of the same patches when a project moves to this package: `disrex/stylesmuggler-adobe-patches`, Adobe patch files in `extra.patches`, and a `vendor/bin/magento-patches apply MCLOUD-...` script for the monthly releases.

## Monthly update

Adobe releases on the second Tuesday of the month.

```bash
bin/sync-registry
git diff --stat registry/ patches/
```

The script downloads what is new, refuses a file whose hash differs from the registry, and regenerates `patches/adobe/patches.json`. Adobe sometimes changes a patch after release day under the same name, or splits part of it into a new one, as it did with the October 2026 patch for 2.4.8-p5. The script then replaces the file and prints `updated`, and the installer swaps the old version for the new one on stores that already have it.

Before tagging, run the installer's end-to-end check on a copy of a store on each line:

```bash
../magento-patch-installer/tests/e2e-local.sh ~/Development/Sites/Magento/isodin-flex2
```

It installs this package in a copy, checks that every patch is in place, that Adobe's `vendor/bin/patch-status` lists nothing missing, and that a reinstalled core package is patched again. Then tag the next version: a minor version (`0.2.0`) for a new month of patches, a patch version (`0.1.1`) when Adobe re-issues or fixes a file. Renovate opens a merge request in every project for each tag, with the release notes listing the patches.

## License

The patch files are Adobe's changes to Magento Open Source and Adobe Commerce, under the same OSL-3.0 licence as Magento. How the installer applies them is documented at https://disrex-group.github.io/magento-patch-installer/.
