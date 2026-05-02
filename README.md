# WAL-G Ansible Role

Installs WAL-G for PostgreSQL, writes a JSON config, configures WAL archiving, and creates a systemd timer for full backups.

## Installation

```bash
ansible-galaxy install envy16.wal_g_postgresql
```

## Example

Install from the default public GitHub release URL:

```yaml
- hosts: postgres
  become: true
  roles:
    - role: envy16.wal_g_postgresql
      vars:
        wal_g_config:
          WALG_S3_PREFIX: s3://my-backups/prod/postgres
          WALG_UPLOAD_CONCURRENCY: 4
          WALG_DOWNLOAD_CONCURRENCY: 4
          AWS_ACCESS_KEY_ID: "{{ vault_aws_access_key_id }}"
          AWS_SECRET_ACCESS_KEY: "{{ vault_aws_secret_access_key }}"
          AWS_ENDPOINT: "https://minio.example.com"
          AWS_REGION: eu-central-1
          AWS_S3_FORCE_PATH_STYLE: "true"
```

Install from another URL, including a private source with Basic Auth:

```yaml
- hosts: postgres
  become: true
  roles:
    - role: envy16.wal_g_postgresql
      vars:
        wal_g_download_url: "https://downloads.example.com/wal-g/v3.0.8/wal-g"
        wal_g_download_username: "{{ vault_download_user }}"
        wal_g_download_password: "{{ vault_download_password }}"
        wal_g_download_force_basic_auth: true
        wal_g_config:
          WALG_S3_PREFIX: s3://my-backups/prod/postgres
          WALG_UPLOAD_CONCURRENCY: 4
          WALG_DOWNLOAD_CONCURRENCY: 4
          AWS_ACCESS_KEY_ID: "{{ vault_aws_access_key_id }}"
          AWS_SECRET_ACCESS_KEY: "{{ vault_aws_secret_access_key }}"
          AWS_ENDPOINT: "https://minio.example.com"
          AWS_REGION: eu-central-1
          AWS_S3_FORCE_PATH_STYLE: "true"
```

## Main Variables

`wal_g_config` is the only variable that must be set for a real backup target. Other variables have role defaults and can be overridden when needed. Override PostgreSQL paths and service name when they differ from the defaults used by your distribution.

### Installation

| Variable | Default | Description |
| --- | --- | --- |
| `wal_g_version` | `v3.0.8` | WAL-G release tag used by the default GitHub URL. |
| `wal_g_arch` | `amd64` | Architecture suffix used in `wal_g_binary_name`. |
| `wal_g_binary_name` | `wal-g-pg-20.04-{{ wal_g_arch }}` | WAL-G binary asset name. Override for another release asset or custom build. |
| `wal_g_download_url` | GitHub release URL | HTTP(S) URL to download WAL-G from. Override for private mirrors or custom builds. |
| `wal_g_download_username` | unset | Optional username for authenticated download. |
| `wal_g_download_password` | unset | Optional password for authenticated download. |
| `wal_g_download_force_basic_auth` | unset | Optional `get_url` Basic Auth toggle. |
| `wal_g_install_dir` | `/usr/local/bin` | Directory where WAL-G is installed. |
| `wal_g_binary_path` | `/usr/local/bin/wal-g` | Final WAL-G binary path used by commands. |
| `wal_g_owner` | `root` | Owner for the WAL-G binary and systemd unit files. |
| `wal_g_group` | `root` | Group for the WAL-G binary and systemd unit files. |
| `wal_g_packages` | `[]` | Optional OS packages to install before downloading WAL-G. |

### WAL-G Config And Logs

| Variable | Default | Description |
| --- | --- | --- |
| `wal_g_config` | `{}` | Required for a real backup target. JSON config written to `wal_g_config_path`; set storage prefix and credentials here. |
| `wal_g_config_dir` | `/etc/wal-g` | Directory for the WAL-G JSON config. |
| `wal_g_config_path` | `{{ wal_g_config_dir }}/config.json` | WAL-G JSON config path. |
| `wal_g_log_dir` | `/var/log/postgresql` | Directory for WAL-G archive, restore, and backup logs. |
| `wal_g_archive_log_path` | `{{ wal_g_log_dir }}/wal-g-archive.log` | Log file for `wal-push`. |
| `wal_g_restore_log_path` | `{{ wal_g_log_dir }}/wal-g-restore.log` | Log file for `wal-fetch`. |
| `wal_g_backup_log_path` | `{{ wal_g_log_dir }}/wal-g-backup.log` | Log file for full backup service output. |

### PostgreSQL

| Variable | Default | Description |
| --- | --- | --- |
| `wal_g_postgres_user` | `postgres` | PostgreSQL OS user that owns WAL-G config and log files. |
| `wal_g_postgres_group` | `postgres` | PostgreSQL OS group for WAL-G config and log files. |
| `wal_g_pgdata` | `/var/lib/postgresql/data` | PostgreSQL data directory passed to `backup-push`. |
| `wal_g_configure_postgresql_archive` | `true` | Manage WAL archive settings in `postgresql.conf`. |
| `wal_g_postgresql_conf_path` | `{{ wal_g_pgdata }}/postgresql.conf` | Path to `postgresql.conf`. |
| `wal_g_archive_mode` | `on` | Value written to `archive_mode`. |
| `wal_g_archive_command` | `wal-g --config ... wal-push "%p"` | PostgreSQL `archive_command`. |
| `wal_g_restore_command` | `wal-g --config ... wal-fetch "%f" "%p"` | PostgreSQL `restore_command`. |
| `wal_g_postgresql_service_name` | `postgresql` | PostgreSQL systemd service name. |
| `wal_g_postgresql_parameters` | `{}` | Optional extra `postgresql.conf` parameters managed by the role. |

For PGDG packages on RedHat-like systems, the PostgreSQL service and data directory are usually versioned:

```yaml
postgresql_version: "14"
wal_g_pgdata: "/var/lib/pgsql/{{ postgresql_version }}/data"
wal_g_postgresql_service_name: "postgresql-{{ postgresql_version }}"
```

### Backup Timer And Retention

| Variable | Default | Description |
| --- | --- | --- |
| `wal_g_enable_full_backup_timer` | `true` | Enable and start the full backup systemd timer. |
| `wal_g_full_backup_schedule` | `*-*-* 02:00:00` | systemd `OnCalendar` schedule for full backups. |
| `wal_g_full_backup_randomized_delay_sec` | `900` | Random delay before scheduled backup start. |
| `wal_g_backup_service_name` | `wal-g-backup` | Base name for systemd service and timer units. |
| `wal_g_backup_command` | `wal-g --config ... backup-push {{ wal_g_pgdata }}` | Command executed by the backup service. |
| `wal_g_backup_user` | `{{ wal_g_postgres_user }}` | User that runs full backups. |
| `wal_g_backup_group` | `{{ wal_g_postgres_group }}` | Group that runs full backups. |
| `wal_g_enable_retention` | `false` | Run retention command after a successful full backup. |
| `wal_g_retention_full_backups` | `7` | Number of latest full backups retained by the default retention command. |
| `wal_g_retention_command` | `wal-g --config ... delete retain FULL {{ wal_g_retention_full_backups }} --confirm` | Retention command executed when retention is enabled. |

## Notes

- `wal_g_config` must be defined by the caller. For S3, set `WALG_S3_PREFIX`; for MinIO also set endpoint/path-style options.
- Changes in `wal_g_config_path` do not require a PostgreSQL restart; WAL-G reads this file every time `archive_command` or the backup service starts it.
- WAL-G logs are written to `/var/log/postgresql/wal-g-archive.log`, `/var/log/postgresql/wal-g-restore.log`, and `/var/log/postgresql/wal-g-backup.log` by default.
- Config and log files are owned by `wal_g_postgres_user` with restrictive permissions.
- PostgreSQL is restarted when the role changes managed PostgreSQL settings.
- WAL archiving starts after PostgreSQL applies `archive_mode` and `archive_command`; enabling `archive_mode` requires a PostgreSQL restart.
- The full backup can be started manually with:

```bash
sudo systemctl start wal-g-backup.service
```

Useful checks after applying the role:

```bash
sudo -u postgres psql -Atc "SHOW archive_mode;"
sudo -u postgres psql -Atc "SHOW archive_command;"
sudo -u postgres psql -Atc "SELECT archived_count, failed_count, last_archived_wal, last_failed_wal FROM pg_stat_archiver;"
```
