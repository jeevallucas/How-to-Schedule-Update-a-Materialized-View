



Here are two reliable ways to auto-refresh your materialized view in Docker. Since your user is readonly_user, we’ll use a SECURITY DEFINER function so they can trigger refresh safely.

# Option A: Use pg_cron inside PostgreSQL (recommended)
1) Enable pg_cron in the container (requires restart)
- With docker-compose:
```yaml
services:
  db:
    image: postgres:16
    command:
      - "postgres"
      - "-c"
      - "shared_preload_libraries=pg_cron"
      - "-c"
      - "cron.database_name=mydb"     # database where jobs run
    environment:
      - POSTGRES_DB=mydb
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=yourpass
    volumes:
      - ./init:/docker-entrypoint-initdb.d
```
- Or plain docker run:
```bash
docker run -d --name pg \
  -e POSTGRES_DB=mydb -e POSTGRES_PASSWORD=yourpass -e POSTGRES_USER=postgres \
  -p 5432:5432 postgres:16 \
  -c shared_preload_libraries=pg_cron \
  -c cron.database_name=mydb
```

2) In the DB (run as superuser/owner), install pg_cron:
```sql
CREATE EXTENSION IF NOT EXISTS pg_cron;
```

3) Create a SECURITY DEFINER function to refresh your MV (owned by the MV owner):
```sql
-- Create or ensure a role that owns the MV
-- ALTER MATERIALIZED VIEW reporting.report_mv OWNER TO reporting_owner;

CREATE OR REPLACE FUNCTION reporting.refresh_report_mv() RETURNS void
LANGUAGE plpgsql
SECURITY DEFINER
AS $$
BEGIN
  -- Use CONCURRENTLY if your unique index is in place
  REFRESH MATERIALIZED VIEW CONCURRENTLY reporting.report_mv;
END
$$;

-- Harden search_path for safety
ALTER FUNCTION reporting.refresh_report_mv() SET search_path = pg_catalog, reporting;

-- Allow readonly_user to call it
GRANT EXECUTE ON FUNCTION reporting.refresh_report_mv() TO readonly_user;
```

4) Schedule with pg_cron
- As the user who should own the job (can be readonly_user if they can execute the function):
```sql
-- Every 15 minutes
SELECT cron.schedule('report_mv_15min', '*/15 * * * *', $$SELECT reporting.refresh_report_mv();$$);

-- View jobs
SELECT * FROM cron.job;
```

Notes:
- CREATE EXTENSION requires superuser.
- The job runs in database mydb (set by cron.database_name).
- The function runs with the owner’s privileges (SECURITY DEFINER), so readonly_user does not need REFRESH rights.
