# Install and configure AUDITOR core components

### Install specific version for AUDITOR components
```
export AUDITOR_VERSION=0.11.2-1
dnf -y install auditor-${AUDITOR_VERSION} 
```

Now execute the two migration scripts. Note, that the location of these two scripts have changed between
versions, before v0.10, they were located in ```/opt/auditor```. Change below, if needed:

```
psql -h localhost -U postgres -d auditor -f /usr/share/auditor/migrations/20220322080444_create_accounting_table.sql
psql -h localhost -U postgres -d auditor -f /usr/share/auditor/migrations/20240503141800_convert_meta_component_to_jsonb.sql 
```

Afterwards the psql auditor db should look as follows:

```
psql -h localhost -U postgres -d auditor 
psql (17.4)
Type "help" for help.

auditor=# \d
                    List of relations
 Schema |           Name            |   Type   |  Owner   
--------+---------------------------+----------+----------
 public | auditor_accounting        | table    | postgres
 public | auditor_accounting_id_seq | sequence | postgres
(2 rows)

auditor=# 
```

## Configuration


### Configure AUDITOR main component
Adjust the auditor config file as follows, again, the location has changed with v0.10, adjust as necessary:
```
vi /etc/auditor/auditor.yml
```
with the following content:
```
application:
  addr:
    - 0.0.0.0
  port: 8000
database:
  host: "localhost"
  port: 5432
  username: "postgres"
  password: "password"
  database_name: "auditor"
  require_ssl: false
metrics:
  database:
    frequency: 30
    metrics:
      - RecordCount
      - RecordCountPerSite
      - RecordCountPerGroup
      - RecordCountPerUser
log_level: info
tls_config:
  use_tls: false
```
Now you can start the service with the following commands:
```
systemctl enable auditor.service
systemctl start auditor.service
```
You should see that the service is active and running:
```
systemctl status auditor.service
● auditor.service - AUDITOR
     Loaded: loaded (/etc/systemd/system/auditor.service; static)
     Active: active (running) since Tue 2025-03-11 15:09:32 UTC; 33s ago
       Docs: https://alu-schumacher.github.io/AUDITOR/
   Main PID: 91232 (auditor)
      Tasks: 4 (limit: 12148)
     Memory: 3.3M
        CPU: 35ms
     CGroup: /system.slice/auditor.service
             └─91232 /usr/bin/auditor /etc/auditor/config.yml

auditor[<pid>]: {"v":0,"name":"AUDITOR","msg":"starting service: \"actix-web-service-0.0.0.0:8000\", workers: 4, listed
...
```
