Below are concise, copy‑pasteable steps to refresh your MV via host crontab using docker exec.

Prerequisites
- MV exists and is initially populated once.
- Unique index exists to allow CONCURRENTLY.
- Create a SECURITY DEFINER function so readonly_user can trigger refresh safely:

```sql
-- Run as MV owner or superuser
CREATE OR REPLACE FUNCTION reporting.refresh_report_mv() RETURNS void
LANGUAGE plpgsql
SECURITY DEFINER
AS $$
BEGIN
  REFRESH MATERIALIZED VIEW CONCURRENTLY reporting.report_mv;
END
$$;

ALTER FUNCTION reporting.refresh_report_mv() SET search_path = pg_catalog, reporting;
GRANT EXECUTE ON FUNCTION reporting.refresh_report_mv() TO readonly_user;
```

Option 1 — crontab using readonly_user via docker exec
1) Quick and simple (uses env var for password)
- Replace placeholders: <container_name>, mydb, readonly_user, yourpass.
- Every 15 minutes:
```
*/15 * * * * docker exec -e PGPASSWORD='yourpass' <container_name> \
  psql -U readonly_user -d mydb -v ON_ERROR_STOP=1 \
  -c "SELECT reporting.refresh_report_mv();" >> /var/log/report_mv_refresh.log 2>&1
```

2) More secure (avoid password in crontab) with .pgpass inside the container
- Inside the container, create /var/lib/postgresql/.pgpass (or the home dir of the user you’ll run psql as):
```
localhost:5432:mydb:readonly_user:yourpass
```
- Permissions:
```
docker exec <container_name> sh -lc "printf 'localhost:5432:mydb:readonly_user:yourpass\n' > /var/lib/postgresql/.pgpass && chmod 600 /var/lib/postgresql/.pgpass"
```
- Crontab entry (no password in crontab):
```
*/15 * * * * docker exec <container_name> \
  psql -h localhost -U readonly_user -d mydb -v ON_ERROR_STOP=1 \
  -c "SELECT reporting.refresh_report_mv();" >> /var/log/report_mv_refresh.log 2>&1
```

Option 2 — crontab as postgres superuser (no function)
If you prefer to avoid SECURITY DEFINER and let cron run as DB superuser:
- This typically works if local peer auth is allowed in the container.
```
*/15 * * * * docker exec -u postgres <container_name> \
  psql -d mydb -v ON_ERROR_STOP=1 \
  -c "REFRESH MATERIALIZED VIEW CONCURRENTLY reporting.report_mv;" >> /var/log/report_mv_refresh.log 2>&1
```

Test it once manually
- Run the cron command once in your shell to verify:
```
docker exec -e PGPASSWORD='yourpass' <container_name> \
  psql -U readonly_user -d mydb -c "SELECT reporting.refresh_report_mv();"
```
- Check logs:
```
tail -n 200 /var/log/report_mv_refresh.log
```
- Validate updates:
```sql
SELECT COUNT(*) FROM reporting.report_mv;
```

Notes
- Replace <container_name> with your Postgres container (e.g., db or pg).
- If your host is Windows, use Task Scheduler or run cron inside WSL; the docker exec command stays the same.
- Keep the function’s search_path locked down (as shown) and only grant EXECUTE to roles that should refresh.
