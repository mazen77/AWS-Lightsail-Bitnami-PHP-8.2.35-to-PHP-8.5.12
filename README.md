# AWS Lightsail Bitnami: PHP 8.2.35 to PHP 8.5.12

Author: mazen  
Record date: 25 October 2026  
Platform: Debian 12 (bookworm), Bitnami Apache, MariaDB, WordPress/WooCommerce

## Status and scope

This guide records the upgrade performed through an operator's SSH session and documents the remaining cutover and validation. It is not a claim that PHP 8.5 application testing has passed.

| Milestone | Evidence/status |
| --- | --- |
| Original Bitnami PHP | 8.2.21 under `/opt/bitnami/php` |
| Package-managed PHP 8.2.35 | Installed; operator confirmed website working |
| PHP 8.5.12 installation | Confirmed by terminal output |
| Extension comparison | Passed against package-managed PHP 8.2 |
| OPcache disk-cache issue | Resolved with PHP 8.5 configuration override |
| PHP 8.5-FPM configuration test | Successful |
| PHP 8.5 socket | Confirmed, owned by `daemon:daemon` |
| Apache switch to PHP 8.5 | Instructions supplied; execution not yet confirmed |
| WordPress Site Health showing 8.5.12 | Pending evidence |
| Checkout, payment, OTP, shipping, Odoo testing on 8.5 | Pending |
| CLI, WP-CLI and scheduled jobs migration | Pending separate review |

**For the server in this change record, preparation is already complete. Start at section 5 after checking the current Apache socket. Do not reinstall or rerun preparation.**

This is a deployment-specific runbook. Its commands assume the working package-managed PHP 8.2.35 configuration already exists. It is not a universal installer for a fresh Bitnami server, Ubuntu, Nginx, containers, or a differently configured FPM pool.

## 1. Architecture and paths

Apache remains the Bitnami Apache server. PHP 8.5 runs as a separate Debian-managed PHP-FPM service. MariaDB and WordPress files remain in place.

| Component | Working 8.2 baseline | PHP 8.5 target |
| --- | --- | --- |
| Executable | `/usr/bin/php8.2` | `/usr/bin/php8.5` |
| Service | `php8.2-fpm` | `php8.5-fpm` |
| FPM configuration | `/etc/php/8.2/fpm/php.ini` | `/etc/php/8.5/fpm/php.ini` |
| CLI configuration | `/etc/php/8.2/cli/php.ini` | `/etc/php/8.5/cli/php.ini` |
| Pool | `/etc/php/8.2/fpm/pool.d/www.conf` | `/etc/php/8.5/fpm/pool.d/www.conf` |
| Socket | `/run/php/bitnami-php82.sock` | `/run/php/bitnami-php85.sock` |

Apache's PHP configuration is `/opt/bitnami/apache/conf/bitnami/php-fpm.conf`. Its named FastCGI backend remains `www-fpm`; only the Unix socket changes.

Database socket: `/opt/bitnami/mariadb/tmp/mysql.sock`.

The copied FPM pool still includes files under `/opt/bitnami/php/etc`, including environment, common and memory settings. **Do not delete that original tree.** Changes to those shared includes can affect both new PHP services. Detaching the configuration from Bitnami is a separate maintenance task.

## 2. Prerequisites and preserved settings

1. Create a fresh Lightsail snapshot of the working 8.2.35 instance; wait for completion and record its identifier privately in Jira.
2. Prefer testing on an isolated clone. Disable real outbound emails, SMS, live payments and Odoo/shipping writes on the clone; use sandbox credentials where available.
3. Confirm installed WordPress, WooCommerce, theme and plugin versions support the target PHP branch. Installation and extension checks do not establish plugin compatibility.
4. Schedule production cutover during a quiet period and keep SSH access available.
5. Keep PHP 8.2.35 and its service available for rollback. Use the configuration rollback first; restoring an older whole-instance snapshot can lose orders placed after the snapshot.
6. Never paste the entire document into a shell. Run one fenced Bash block at a time and stop on errors. Rollback is in a separate document.

Observed configuration:

| Setting | Value |
| --- | --- |
| Server memory | Approximately 61 GiB |
| PHP memory limit | 1024M |
| Upload / POST limits | 128M / 128M |
| CLI execution time limit | 0; not proof of the effective web limit |
| max_input_vars | 1000 |
| OPcache memory | 512 MB |
| FPM mode | dynamic |
| max_children | 128 |
| start_servers / min_spare_servers | 16 / 16 |
| max_spare_servers | 32 |
| max_requests | 500 |
| Pool user/group | daemon / daemon |

These values are preserved, not newly sized or load-tested. A 1024M per-request ceiling and 128 workers are not a guarantee that all workers can use their maximum memory simultaneously. Monitor actual memory and FPM concurrency; do not multiply these settings across other servers blindly.

## 3. Preflight — read only

```bash
cat /etc/os-release
/usr/bin/php8.2 -v
/usr/bin/php8.5 -v
sudo systemctl status php8.2-fpm php8.5-fpm --no-pager
sudo grep -n 'proxy\|Proxy\|sock' /opt/bitnami/apache/conf/bitnami/php-fpm.conf
free -h
df -h / /opt
```

If Apache already references `bitnami-php85.sock`, skip installation and cutover; continue to verification. If it references the original `/opt/bitnami/php/var/run/www.sock`, this runbook's 8.2.35 baseline is not active: investigate before proceeding.

## 4. Preparation — reference for a matching 8.2.35 baseline

**Already completed on the recorded server. Do not repeat.**

The Sury repository was previously configured in `/etc/apt/sources.list.d/php.list` with a signed-by key at `/usr/share/keyrings/sury-php.gpg`. It is a third-party repository. This procedure reuses it; do not add duplicate repository entries.

The following block is for a matching server that has not yet prepared PHP 8.5. It installs the repository's candidate version and stops unless that version is exactly 8.5.12. It is not a permanent version pin; future package updates can install later patches. Do not downgrade a newer security release merely to match this historical guide.

```bash
sudo bash <<'BASH'
set -euo pipefail
BACKUP=/root/php85-preparation-backup
test ! -e "$BACKUP" || { echo 'STOP: Backup exists; inspect prior progress.'; exit 1; }
grep -qF '/run/php/bitnami-php82.sock' /opt/bitnami/apache/conf/bitnami/php-fpm.conf
systemctl is-active --quiet php8.2-fpm
test -S /run/php/bitnami-php82.sock
install -d -m 700 "$BACKUP"
cp -a /opt/bitnami/apache/conf/bitnami/php-fpm.conf "$BACKUP/apache-php82.conf"
cp -a /etc/php/8.2 "$BACKUP/php82"
OLD_CLI=$(readlink -f /usr/bin/php)
trap 'update-alternatives --set php "$OLD_CLI" >/dev/null' EXIT
apt-get update
apt-get install -y --no-install-recommends \
  php8.5-fpm php8.5-cli php8.5-common php8.5-readline \
  php8.5-mysql php8.5-curl php8.5-gd php8.5-mbstring \
  php8.5-xml php8.5-zip php8.5-intl php8.5-bcmath \
  php8.5-bz2 php8.5-gmp php8.5-ldap php8.5-soap \
  php8.5-sqlite3 php8.5-tidy php8.5-imagick \
  php8.5-memcached php8.5-igbinary php8.5-msgpack
/usr/bin/php8.5 -r 'if (PHP_VERSION !== "8.5.12") { fwrite(STDERR,"STOP: Installed ".PHP_VERSION."; review target version.\n"); exit(1); }'
cp -a /etc/php/8.5 "$BACKUP/php85-original"
for sapi in fpm cli; do
  sed -E '/^[[:space:]]*(zend_extension|extension|extension_dir)[[:space:]]*=/d' \
    "/etc/php/8.2/$sapi/php.ini" > "/etc/php/8.5/$sapi/php.ini"
  cat > "/etc/php/8.5/$sapi/conf.d/zz-opcache-cache-fix.ini" <<'INI'
opcache.enable=1
opcache.file_cache=
opcache.file_cache_only=0
INI
done
cp /etc/php/8.2/fpm/pool.d/www.conf /etc/php/8.5/fpm/pool.d/www.conf
sed -i -E 's|^[[:space:]]*listen[[:space:]]*=.*|listen=/run/php/bitnami-php85.sock|' \
  /etc/php/8.5/fpm/pool.d/www.conf
/usr/bin/php8.2 -r 'echo json_encode(get_loaded_extensions());' > "$BACKUP/php82-extensions.json"
/usr/bin/php8.5 -r '
$old=json_decode(file_get_contents($argv[1]),true);
if (!is_array($old)) { exit(1); }
$missing=array_diff(array_map("strtolower",$old),array_map("strtolower",get_loaded_extensions()));
if ($missing) { fwrite(STDERR,"Missing: ".implode(", ",$missing)."\n"); exit(1); }
echo "PASS: Extension comparison\n";
' "$BACKUP/php82-extensions.json"
/usr/sbin/php-fpm8.5 -t
systemctl enable php8.5-fpm
systemctl restart php8.5-fpm
systemctl is-active --quiet php8.5-fpm
test -S /run/php/bitnami-php85.sock
echo 'READY: PHP 8.5 prepared. Apache has not been switched.'
BASH
```

The comparison checks loaded CLI extension names, not binary equivalence, extension configuration or end-to-end functionality. Verify required extensions in the web runtime as well. The new installation loads its own extension files; never copy PHP 8.2 `.so` binaries into PHP 8.5.

## 5. Production cutover — run once after preparation

This block changes the live website runtime. It preserves the `www-fpm` backend name and gracefully reloads Apache. A server reboot is not required. Graceful reload allows existing requests to finish; it does not guarantee that application code is compatible with the new runtime.

```bash
sudo bash <<'BASH'
set -euo pipefail
CONF=/opt/bitnami/apache/conf/bitnami/php-fpm.conf
BACKUP=/root/php85-preparation-backup/apache-php82.conf
APACHECTL=/opt/bitnami/apache/bin/apachectl
test -f "$BACKUP"
grep -qF '/run/php/bitnami-php82.sock' "$BACKUP"
grep -qF '/run/php/bitnami-php82.sock' "$CONF"
systemctl is-active --quiet php8.2-fpm
systemctl is-active --quiet php8.5-fpm
test -S /run/php/bitnami-php82.sock
test -S /run/php/bitnami-php85.sock
/usr/sbin/php-fpm8.5 -t
sed -i 's|/run/php/bitnami-php82.sock|/run/php/bitnami-php85.sock|g' "$CONF"
if "$APACHECTL" -t && "$APACHECTL" -k graceful; then
  echo 'SWITCHED: Verify PHP 8.5.12 in WordPress Site Health.'
else
  cp -a "$BACKUP" "$CONF"
  "$APACHECTL" -t && "$APACHECTL" -k graceful
  echo 'Switch failed; restored the PHP 8.2.35 configuration.'
  exit 1
fi
BASH
```

Automatic restoration above covers a failed Apache configuration check or reload command. It does not automatically detect broken plugins, HTTP errors, payment failures or subsequent PHP service failures. See the separate `ROLLBACK.md` only if rollback is needed.

## 6. Verification and acceptance

### Confirm the web runtime

Open WordPress **Tools → Site Health → Info → Server** and record PHP **8.5.12**. This is stronger evidence of the website runtime than `php -v`, which checks CLI only. Record web memory limits and required extensions as well, using Site Health / WooCommerce Status.

Do not leave a public `phpinfo()` file on the website.

### Functional checks

| Test | Expected result | Result/evidence |
| --- | --- | --- |
| WordPress admin | Loads; settings and order screens work | Pending |
| Product and variation selection | Correct power, quantity, price and stock | Pending |
| Cart changes | Correct totals and two-lens rules | Pending |
| Checkout | Address, shipping, taxes and fees correct | Pending |
| Payments | Authorized test order; callback/status correct | Pending |
| Email and mobile OTP | Delivery, validation, login/registration work | Pending |
| Shipping | Shipment and label flow works | Pending |
| Odoo | Order, tax, stock and status synchronization verified | Pending |
| Uploads / image processing | Thumbnail creation and Imagick/GD operation work | Pending |
| Scheduled actions | No new failed WooCommerce actions | Pending |

Use controlled test data. Check callbacks and stored order results, not only whether a page loads. A successful homepage response does not verify checkout.

### Service and log checks

```bash
sudo systemctl status php8.5-fpm --no-pager
sudo journalctl -u php8.5-fpm --since '15 minutes ago' --no-pager -n 100
sudo tail -n 100 /opt/bitnami/apache/logs/error_log
free -h
```

PHP application errors may be written to a pool-specific error log rather than the system journal. Inspect configured `error_log` values if necessary. Redact customer data, credentials and tokens before attaching logs to GitHub/Jira. Keep production error display disabled; do not suppress all logging merely to hide compatibility problems.

Compare error rates, checkout results, CPU/memory use and response times with the baseline through a representative traffic period. No performance improvement has been measured in this record.

## 7. CLI, WP-CLI and background jobs — separate follow-up

The installation preserved the system `/usr/bin/php` selection. Bitnami's PATH may still select `/opt/bitnami/php/bin/php` (8.2.21). Web requests can use PHP 8.5 while command-line jobs use a different version.

Read-only inspection:

```bash
command -v php
php -v
readlink -f /usr/bin/php
/usr/bin/php8.5 -v
command -v wp
crontab -l
sudo crontab -l
sudo systemctl list-timers --all --no-pager
sudo grep -RnsE 'php|wp-cron|wp-cli' /etc/cron.d /etc/crontab 2>/dev/null
```

No crontab can produce a nonzero exit code; it does not imply an upgrade failure. These outputs can contain secrets: review locally and redact before sharing.

Inventory job commands, service units, WP-CLI wrappers and absolute PHP paths. Test each job under `/usr/bin/php8.5` with its normal user and environment before changing that specific invocation. Changing `update-alternatives` alone does not override Bitnami PATH precedence or hardcoded executable paths. Do not replace the original Bitnami executable with a symlink.

After a successful observation period, review whether unused FPM services should be stopped to save idle resources. Do not remove original configuration directories while they are still included by the active pool. Keep the rollback service available until the change is accepted.

## 8. Troubleshooting and lessons learned

### OPcache startup failure

Observed message:

```text
Fatal Error opcache.file_cache must be a full path of an accessible directory
```

The inherited disk-cache setting caused PHP 8.5's FPM test to fail. The runbook now creates `zz-opcache-cache-fix.ini` for both CLI and FPM with an empty `opcache.file_cache`, `opcache.file_cache_only=0`, and `opcache.enable=1`. This disables the optional disk cache while retaining RAM OPcache. The subsequent PHP 8.5 configuration test and socket creation passed. Re-enabling a disk cache would require a dedicated accessible directory and separate validation.

### Salt repository warning

APT could not resolve `repo.saltproject.io`, but Debian and Sury indexes were fetched and the PHP installation succeeded. The warning is a separate repository-maintenance issue. Investigate whether Salt is used before disabling or replacing its repository; do not treat this as proof of a general DNS outage or disable signature checking.

### Preparation backup already exists

The guard prevents overwriting recovery evidence. Do not delete the backup to make the script run again. Identify the failed step and resume only that step. In this session, installation and configuration were already complete when the OPcache test failed.

### Wrong PHP version in terminal

Use WordPress Site Health to check web PHP. Inspect `command -v php` and absolute executables for CLI. A terminal showing 8.2 does not alone prove that Apache is still on 8.2.

### 502/503 or failed checkout after cutover

Check the PHP 8.5 service, socket ownership, Apache backend configuration and recent logs. If customer-facing functionality is affected, use the separate rollback procedure to restore 8.2.35 before investigating plugin compatibility on a clone.

### Why no reboot?

The new PHP-FPM service is restarted to load its configuration, and Apache is gracefully reloaded to use the new socket. No reboot is needed solely for this PHP switch. Unrelated OS/kernel maintenance may have its own requirements.

## 9. Publishing to GitHub and Jira

For GitHub, upload `README.md`, `ROLLBACK.md` and `JIRA-CHANGE-RECORD.md` as individual files in a repository or documentation folder. The README renders as the guide. The ZIP is a convenience download, not a requirement for a repository.

For Jira, use the title and content in `JIRA-CHANGE-RECORD.md`, or attach the Markdown files/ZIP. Jira editors vary in Markdown support; paste sections into the rich-text editor and use code blocks for commands if automatic formatting does not apply.

This package intentionally omits server IP addresses, instance identifiers, customer data and credentials. Keep snapshot identifiers and internal evidence in a private issue. Do not publish server configuration backups: included environment files can contain secrets.

## References

- [AWS: Update PHP on Lightsail, including Bitnami in-place migration](https://docs.aws.amazon.com/lightsail/latest/userguide/amazon-lightsail-update-php-security-vulnerabilities.html)
- [PHP: OPcache built into PHP 8.5](https://wiki.php.net/rfc/make_opcache_required)
- [WordPress: PHP compatibility reference](https://make.wordpress.org/core/handbook/references/php-compatibility-and-wordpress-versions/)
- [WooCommerce: server requirements](https://woocommerce.com/document/server-requirements/)

References were consulted during the upgrade discussion. Review current package availability and compatibility before reusing this dated procedure. Commands were documented from the session and adapted for clarity; they have not been independently executed against another production server.
