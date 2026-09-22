<div align="center">

<img src="https://capsule-render.vercel.app/api?type=waving&color=0:000000,100:0D0208&height=200&section=header&text=BUCHAREST&fontColor=00FF41&fontSize=40&fontAlignY=32&animation=fadeIn&desc=Connecting%20to%20Postgres%20%E2%80%94%20SadServers%20Troubleshooting%20Lab&descAlignY=55&descSize=15&descColor=39FF14" width="100%"/>

<img src="https://img.shields.io/badge/PLATFORM-SADSERVERS-000000?style=for-the-badge&logoColor=00FF41" />
<img src="https://img.shields.io/badge/DIFFICULTY-EASY-000000?style=for-the-badge&logoColor=00FF41" />
<img src="https://img.shields.io/badge/TAG-POSTGRESQL-000000?style=for-the-badge&logo=postgresql&logoColor=00FF41" />
<img src="https://img.shields.io/badge/OS-DEBIAN%2013-000000?style=for-the-badge&logo=debian&logoColor=00FF41" />
<img src="https://img.shields.io/badge/STATUS-SOLVED-000000?style=for-the-badge&logo=checkmarx&logoColor=00FF41" />

</div>

<br/>

## `> cat scenario.log`

```bash
krikox@matrix:~$ cat scenario.log
Scenario:      "Bucharest": Connecting to Postgres
Level:          Easy
Type:           Fix
Access:         Email
Root (sudo):    True
Time to Solve:  15 minutes

Description:
  A web application relies on the PostgreSQL database on this server.
  After a client-authentication change, that connection fails.

  The application connects over TCP to 127.0.0.1, database app1,
  user app1user, password app1user. The role and password are
  already set. Do not replace the database or the account —
  restore this connection. It must still work after a reboot.

Test (must exit 0, no error):
  PGPASSWORD=app1user psql -h 127.0.0.1 -d app1 -U app1user -c '\q'
```

<br/>

## `> ./reproduce_issue.sh`

```bash
krikox@matrix:~$ PGPASSWORD=app1user psql -h 127.0.0.1 -d app1 -U app1user -c '\q'

psql: error: connection to server at "127.0.0.1", port 5432 failed:
FATAL:  pg_hba.conf rejects connection for host "127.0.0.1",
        user "app1user", database "app1", SSL encryption
connection to server at "127.0.0.1", port 5432 failed:
FATAL:  pg_hba.conf rejects connection for host "127.0.0.1",
        user "app1user", database "app1", SSL encryption
```

> Both the SSL and non-SSL attempts are explicitly **rejected** — not "no entry found." That distinction matters: it means a rule *does* match, and that rule's method is `reject`.

<br/>

## `> ./diagnose.sh`

Command → symptom → what it told us:

```bash
krikox@matrix:~$ sudo -u postgres psql -c "show hba_file;"
                hba_file
-----------------------------------------
 /var/lib/postgresql/17/main/pg_hba.conf
```
**Finding:** the active `hba_file` is **not** the Debian-standard `/etc/postgresql/17/main/pg_hba.conf` — it points into the data directory under `/var/lib/postgresql/`. This is the actual "client-authentication change": `postgresql.conf`'s `hba_file` directive (or the file itself) was moved off the standard path, so edits made in `/etc/...` were silently having zero effect.

```bash
krikox@matrix:~$ sudo -u postgres psql -c \
  "SELECT line_number, type, database, user_name, address, auth_method \
   FROM pg_hba_file_rules();"

 line_number | type  | database | user_name  |  address  | auth_method
-------------+-------+----------+------------+-----------+-------------
           6 | local | {all}    | {postgres} |           | peer
           7 | local | {all}    | {all}      |           | peer
           9 | host  | {all}    | {all}      | 0.0.0.0   | reject
          10 | host  | {all}    | {all}      | ::        | reject
          12 | host  | {app1}   | {app1user} | 127.0.0.1 | trust
          13 | host  | {app1}   | {app1user} | ::1       | trust
```
**Finding:** `pg_hba.conf` is evaluated **top-down** and stops at the **first match**. The generic `host all all 0.0.0.0/0 reject` rule on line 9 matches `127.0.0.1` before the parser ever reaches the specific `app1user` rule on line 12 — so the correct rule existed, but could never be reached.

```bash
krikox@matrix:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
17  main    5432 online postgres /var/lib/postgresql/17/main ...
```
**Finding:** ruled out the "wrong cluster / wrong port" theory — only one cluster, listening exactly where expected.

<br/>

## `> cat root_cause.log`

```text
1. postgresql.conf's `hba_file` was pointed at
   /var/lib/postgresql/17/main/pg_hba.conf instead of the
   Debian-packaged /etc/postgresql/17/main/pg_hba.conf.

2. In that active file, a hardened "drop everything, then allow"
   ruleset was written in the WRONG order:

     host  all   all  0.0.0.0/0  reject   <- evaluated first, matches
     host  all   all  ::/0       reject
     host  app1  app1user  127.0.0.1/32  trust   <- never reached
     host  app1  app1user  ::1/128       trust   <- never reached

   pg_hba.conf rules are first-match-wins. A generic reject placed
   above a specific allow permanently shadows it.
```

<br/>

## `> ./apply_fix.sh`

```bash
# 1. Back up the file actually in use
sudo cp /var/lib/postgresql/17/main/pg_hba.conf \
        /var/lib/postgresql/17/main/pg_hba.conf.bak

# 2. Rewrite it with the specific allow rule BEFORE the generic reject
sudo tee /var/lib/postgresql/17/main/pg_hba.conf > /dev/null <<'EOF'
local   all             postgres                                peer
local   all             all                                     peer

host    app1            app1user        127.0.0.1/32            trust
host    app1            app1user        ::1/128                 trust

host    all             all             0.0.0.0/0               reject
host    all             all             ::/0                    reject
EOF

# 3. Postgres runs as the "postgres" user — it must own the file it reads
sudo chown postgres:postgres /var/lib/postgresql/17/main/pg_hba.conf

# 4. Force a full reread (a reload alone did not pick it up here)
sudo systemctl restart postgresql
```

**Why this is safe:** the role and password (`app1user` / `app1user`) were never touched — only the *authentication rule order* changed, exactly as the challenge requires ("do not replace the database or the account").

**Why it survives a reboot:** the fix lives on disk, in the file `hba_file` actually points to, and `postgresql.service` is `enabled` — it comes back up and re-reads that same file automatically. No `ALTER SYSTEM`, no in-memory workaround.

<br/>

## `> ./verify.sh`

```bash
krikox@matrix:~$ sudo -u postgres psql -c "show hba_file;"
                hba_file
-----------------------------------------
 /var/lib/postgresql/17/main/pg_hba.conf

krikox@matrix:~$ sudo -u postgres psql -c \
  "SELECT line_number, type, database, user_name, address, auth_method \
   FROM pg_hba_file_rules();"

 line_number | type  | database | user_name  |  address  | auth_method
-------------+-------+----------+------------+-----------+-------------
           1 | local | {all}    | {postgres} |           | peer
           2 | local | {all}    | {all}      |           | peer
           4 | host  | {app1}   | {app1user} | 127.0.0.1 | trust
           5 | host  | {app1}   | {app1user} | ::1       | trust
           7 | host  | {all}    | {all}      | 0.0.0.0   | reject
           8 | host  | {all}    | {all}      | ::        | reject

krikox@matrix:~$ PGPASSWORD=app1user psql -h 127.0.0.1 -d app1 -U app1user -c '\q'
krikox@matrix:~$ echo $?
0
```

`app1user`'s rule now has a lower `line_number` than the `reject` rules — first match wins in the right order — and the test command exits clean.

<br/>

## `> cat lessons_learned.log`

```text
- pg_hba.conf is first-match-wins, top-down. A "hardened" reject
  placed above a legitimate allow is a self-inflicted outage.

- `SHOW hba_file;` is the source of truth for which file Postgres
  is actually reading — never assume it's the distro-default path.

- pg_hba_file_rules() beats reading the file by eye: it shows the
  parsed rules in evaluation order, plus any syntax errors, exactly
  as the server sees them.

- `systemctl reload` is usually enough for pg_hba.conf changes, but
  when behavior doesn't match the file on disk, `restart` removes
  all doubt.

- A file rewritten as root (`tee`) needs its ownership restored to
  `postgres:postgres`, or the server may fail to read it correctly.
```

<br/>

## `> ./references.sh`

```bash
krikox@matrix:~$ ls ./references/
```

- 📘 [PostgreSQL Docs — The pg_hba.conf File](https://www.postgresql.org/docs/current/auth-pg-hba-conf.html)
- 📘 [PostgreSQL Docs — pg_hba_file_rules view](https://www.postgresql.org/docs/17/view-pg-hba-file-rules.html)
- 💬 [PostgreSQL mailing list — precedence/ordering of pg_hba.conf rules](https://www.postgresql.org/message-id/16668.1141254061%40sss.pgh.pa.us)
- 💬 [PostgreSQL mailing list — pg_hba.conf changes without restarting postmaster](https://www.postgresql.org/message-id/014601c4788e%24844db1f0%240a00a8c0%40lrp43008)
- 📄 [Chat2DB — Fix: no pg_hba.conf entry for host](https://chat2db.ai/resources/blog/postgres-no-pg-hba-conf-entry-fix)
- 🧪 [SadServers — "Bucharest": Connecting to Postgres](https://sadservers.com/scenario/bucharest)

<br/>

<div align="center">

**Stack:** PostgreSQL 17 • Debian 13 • systemd • `pg_hba.conf`

<br/>

> *"Uptime is a feature. Detection is a discipline."*

<br/>

<a href="https://github.com/FabianCH20"><img src="https://img.shields.io/badge/back%20to-profile-000000?style=for-the-badge&logo=github&logoColor=00FF41"/></a>

</div>

<img src="https://capsule-render.vercel.app/api?type=waving&color=0:0D0208,100:000000&height=120&section=footer" width="100%"/>
