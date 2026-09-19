# Backup Handler

A modular Python tool for backing up databases (PostgreSQL, MongoDB, MariaDB, SQL Server, Elasticsearch) and files to local storage with optional S3 upload, compression, encryption, and Prometheus monitoring.

## ✨ Features

* Modular backup for multiple database types (PostgreSQL, MongoDB, MariaDB, SQL Server, Elasticsearch)
* Multiple named instances per database type, each with its own credentials and settings
* File/directory backup support
* Compression and encryption using 7z (AES-256)
* Upload to S3-compatible storage (e.g., MinIO, AWS)
* Per-instance compression toggle (`enabled_compression`)
* Retention policy for local file backups and Elasticsearch snapshots
* Prometheus Pushgateway monitoring
* Full logging to file and console
* YAML-based configuration
* CLI with per-instance control
* Cron-friendly

## ⚙️ Requirements

* Python 3.8+
* pip packages (see `requirements.txt`)
* Database clients: `pg_dump` (postgresql-client), `mongodump` (mongodb-tools), `mysqldump` (mariadb-client), `sqlpackage` (see [SQL Server](#sql-server))
* Compression/encryption: `7z` (p7zip-full)

### System Dependencies Installation (Ubuntu/Debian)

```bash
# Update package list
sudo apt update

# Install database clients
sudo apt install postgresql-client mongodb-tools mariadb-client

# Install compression tools
sudo apt install p7zip-full
```

### SQL Server

SQL Server backups use `sqlpackage` which must be installed on the machine running this tool:

```bash
sudo mkdir -p /opt/sqlpackage && cd /opt/sqlpackage
sudo curl -sL https://go.microsoft.com/fwlink/?linkid=2316314 -o sqlpackage.zip
sudo unzip sqlpackage.zip && sudo chmod +x sqlpackage
sudo ln -sf /opt/sqlpackage/sqlpackage /usr/local/bin/sqlpackage
```

Notes:
* `sqlpackage` cannot export all databases at once — every instance must list its databases explicitly.
* `sqlpackage` requires .NET runtime; the standalone zip for Linux x64 bundles what it needs.
* Orphaned database users (users without a matching server login, e.g. leftovers from a dev restore) make the export fail with `Error SQL71564`. Fix them on the server (`DROP USER`, or `ALTER USER ... WITH LOGIN = ...`) before backing up.

## 📦 Installation

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

## 📝 Configuration

Copy `config.sample.yaml` to `config.yaml` and edit the settings as needed. The script reads settings from `config.yaml` next to itself.

Top-level sections:

| Section | Purpose |
|---------|---------|
| `backup` | File/directory backup (source, destination, name) |
| `retention` | Keep only the N newest local file backups |
| `s3` | S3-compatible upload settings, also used as the default bucket for Elasticsearch s3 repositories |
| `logging` | Log file and handlers |
| `compression` | 7z compression level and encryption key |
| `metrics` | Prometheus Pushgateway settings |
| `postgresql` / `mongodb` / `mariadb` / `sqlserver` / `elasticsearch` | Named instance lists per database type |

Key points:

* `source_dir` is the directory you back **up** (input); `backup_dir` is where the local `.tar.gz` file backups are **written** (output).
* `output_dir` inside a database instance is a **temporary** location on this machine where dumps are written before compression/upload. Processed files are removed after upload, so it only needs roughly the size of the largest database.
* `databases: []` means "all databases" for PostgreSQL/MongoDB/MariaDB. SQL Server always requires an explicit list.
* `enabled_compression` per instance controls whether its dumps get the 7z compress+encrypt treatment.
* For Elasticsearch s3 repositories, omitting `bucket` in `repository_settings` falls back to the bucket of the global `s3` section.

See `config.sample.yaml` for the full reference of every option.

## 🛠 Usage

```bash
source .venv/bin/activate

# Run all enabled backups (files + all enabled DB instances)
python3 backup.py --all

# Only back up files (source_dir)
python3 backup.py --files

# Back up one database type (all enabled instances)
python3 backup.py --postgresql
python3 backup.py --mongodb
python3 backup.py --mariadb
python3 backup.py --sqlserver
python3 backup.py --es

# Back up specific instances (comma-separated names)
python3 backup.py --postgresql keycloak,camunda
```

Running without arguments behaves like `--all`.

## 🚀 How It Works

1. Reads `config.yaml` and runs the selected targets (enabled instances only)
2. Database dumps are written per database to the instance `output_dir`
   * PostgreSQL/MariaDB: one `.sql` file per database
   * MongoDB: one dump directory per database (`mongodb_<db>_<timestamp>`)
   * SQL Server: one `.bacpac` file per database
3. Each dump is compressed and encrypted into a `.7z` archive (when `enabled_compression` is set)
4. Archives are uploaded to S3 (when `s3.enabled`) and removed locally after a successful upload
5. Elasticsearch takes snapshots through its API
6. Old local file backups are cleaned based on the retention policy
7. Statuses are pushed to Prometheus Pushgateway

## ⏰ Cron Example

```cron
# Backup every day at 03:00 AM
0 3 * * * root /opt/backup-handler/.venv/bin/python /opt/backup-handler/backup.py --all
```

No venv activation is needed in cron — call the venv's python binary directly.

## 🔐 Restore Instructions

Backup archives are `.7z` files (compressed and encrypted with the key from `compression.encryption.key`).

1. **Extract backup**:

   ```bash
   7z x -p"your_key_from_config" backup_file.7z
   ```

2. **Restore database**:
   * PostgreSQL: `psql -h host -U user -d db < backup.sql`
   * MongoDB: `mongorestore --host host --db db backup_dir/<db>`
   * MariaDB: `mysql -h host -u user -p db < backup.sql`
   * SQL Server: `sqlpackage /Action:Import /SourceFile:backup.bacpac /TargetConnectionString:"Server=host,port;Database=db;User Id=user;Password=pass;TrustServerCertificate=True"`
   * Elasticsearch: restore the snapshot through the ES API

## 🔧 Troubleshooting

### Common Issues

* **7z not found**: Install `p7zip-full` (`apt install p7zip-full` on Ubuntu)
* **Database connection failed**: Check host, port, credentials in config.yaml
* **S3 upload failed**: Verify endpoint, access keys, bucket permissions
* **Encryption key missing**: Ensure `key` is set in `compression.encryption` section
* **Permission denied**: Run with appropriate user permissions for backup directories
* **MongoDB `not authorized on config`**: remove internal databases (`config`, `local`) from the instance `databases` list
* **SQL Server `SQL71564` orphaned user**: see [SQL Server](#sql-server) above
* **Elasticsearch `path.repo` / `repository_missing_exception`**: make sure the ES server has `path.repo` set to a writable directory matching the repository `location`

### Logs

Check `./backup.log` for detailed error messages. Logs include timestamps and error details.

## TODO

* only use 7z, drop remaining zstd/gzip references in code and config
* postgresql instances are created regardless of the `enabled` flag when selected explicitly
* elasticsearch s3 repository vs fs decision

## Contribute

https://github.com/m-nik/backup-handler
