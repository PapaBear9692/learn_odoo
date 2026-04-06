# Setup and Run Odoo 19

## 1. Clone Repository

```bash
git clone https://github.com/odoo/odoo.git odoo-19
cd odoo-19
git checkout 19.0
```

## 2. Create Virtual Environment

```bash
python3 -m venv .odoo
source .odoo/bin/activate
```

## 3. Install Python Dependencies
1. open requirements.txt
2. find ```lxml==5.2.1```
3. change ```lxml==5.3.0```
4. save
5. execute :
```bash
pip install -r requirements.txt
```

## 4. PostgreSQL Setup

### Install PostgreSQL
```bash
# Debian/Ubuntu/Kali
sudo apt update
sudo apt install postgresql postgresql-contrib

# Start service
sudo systemctl start postgresql
sudo systemctl enable postgresql
```

### Create Odoo Database User
```bash
sudo -u postgres psql -c "CREATE USER odoo WITH PASSWORD 'odoo' CREATEDB SUPERUSER;"
```

### Verify Connection
```bash
psql -U odoo -h localhost -l
```

## 5. Create odoo.conf

Create `odoo.conf` in the project root:

```ini
[options]
# Paths to addon directories (comma-separated)
addons_path = /home/username/Work/odoo-19/odoo/addons,/home/nishat/Work/odoo-19/addons,/home/username/Work/odoo-19/custom_addons

# Database settings
db_host = localhost
db_port = 5432
db_user = odoo
db_password = odoo

# Master password for database operations
admin_passwd = admin

# Show productivity apps by default
default_productivity_apps = True
```

### odoo.conf Fields Explained

| Field | Description |
|-------|-------------|
| `addons_path` | Directories where Odoo looks for modules. Must include `custom_addons` for your modules to be found |
| `db_host` | PostgreSQL server host. Use `localhost` for local install |
| `db_port` | PostgreSQL port (default: 5432) |
| `db_user` | PostgreSQL username created earlier |
| `db_password` | PostgreSQL password for the user |
| `admin_passwd` | Master password to create/delete databases via web UI |

## 6. Start Odoo

```bash
python odoo-bin -c odoo.conf
```

Access at: `http://localhost:8069`

## 7. Create Database

1. Open `http://localhost:8069` in browser
2. Enter Master Password: `admin`
3. Fill in:
   - Database name (e.g., `my_odoo_db`)
   - Email
   - Password
4. Click **Create database**
5. Wait for initialization to complete

## 8. Update Apps List

Before installing custom modules:

0. Go to **Settings**, scroll down, **activate developer mode**
1. Go to **Apps** menu
2. Click **Update Apps List** (top left, 2nd option)
3. Click **Update**

## Common Commands

```bash
# Start Odoo normally
python odoo-bin -c odoo.conf  

./odoo-bin -c odoo.conf

# Start with specific database
python odoo-bin -c odoo.conf -d my_odoo_db_name

# Start with debug logging
python odoo-bin -c odoo.conf --log-level=debug

# Generate/save config file
python odoo-bin -c odoo.conf --save --stop-after-init
```

## Troubleshooting

### PostgreSQL Connection Failed
```bash
# Check PostgreSQL is running
sudo systemctl status postgresql

# Check pg_hba.conf allows password auth
sudo cat /etc/postgresql/*/main/pg_hba.conf

# Should have this line:
# host    all    all    127.0.0.1/32    scram-sha-256

# Recreate user if needed
sudo -u postgres psql -c "DROP USER odoo;"
sudo -u postgres psql -c "CREATE USER odoo WITH PASSWORD 'odoo' CREATEDB SUPERUSER;"
```

### Module Not Found in Apps
1. Verify `custom_addons` path is correct in `odoo.conf`
2. Restart Odoo
3. Go to **Settings**, scroll down, **activate developer mode**
4. Go to **Apps** → **Update Apps List**
