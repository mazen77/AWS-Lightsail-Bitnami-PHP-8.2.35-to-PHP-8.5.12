# Emergency rollback: PHP 8.5 to working PHP 8.2.35

**NOT an installation step. Do not run this when the upgrade is working.**

Use only for a customer-facing regression, failed application tests or an explicit decision to abandon the PHP 8.5 cutover. This restores the Apache configuration saved immediately before PHP 8.5 preparation. It does not restore the old Bitnami PHP 8.2.21 configuration.

```bash
sudo bash <<'BASH'
set -euo pipefail
CONF=/opt/bitnami/apache/conf/bitnami/php-fpm.conf
BACKUP=/root/php85-preparation-backup/apache-php82.conf
APACHECTL=/opt/bitnami/apache/bin/apachectl
test -f "$BACKUP"
grep -qF '/run/php/bitnami-php82.sock' "$BACKUP"
/usr/sbin/php-fpm8.2 -t
systemctl start php8.2-fpm
systemctl is-active --quiet php8.2-fpm
test -S /run/php/bitnami-php82.sock
cp -a "$CONF" "${CONF}.before-rollback-$(date +%Y%m%d-%H%M%S)"
cp -a "$BACKUP" "$CONF"
"$APACHECTL" -t
"$APACHECTL" -k graceful
echo 'ROLLED BACK: Confirm PHP 8.2.35 in WordPress Site Health.'
BASH
```

Afterward:

1. Verify PHP **8.2.35** in WordPress Site Health.
2. Verify checkout, login and the previously failing action.
3. Record rollback time, symptoms, affected transactions and redacted log evidence in Jira.
4. Keep PHP 8.5 installed for investigation; do not continue switching versions repeatedly on the live store.

This rollback only changes the web PHP backend. If CLI jobs were migrated later, revert them separately using their recorded original commands. It does not reverse application data changes or third-party transactions. Do not restore the whole-instance snapshot casually on a store receiving new orders.

Never use `/root/bitnami-php82-before-switch/apache-php-fpm.conf` for this rollback: that older backup belongs to the earlier 8.2.21-to-8.2.35 migration.
