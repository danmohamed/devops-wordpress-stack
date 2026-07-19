# Troubleshooting Log

Real issues encountered while building this stack, and how each was diagnosed and fixed. Kept here as a reference for future debugging and to document the actual problem-solving that went into this project.

---

## 1. MySQL Authentication Errors

**Symptom:** WordPress couldn't connect to the database container — connection was refused or authentication failed even though credentials in `.env` looked correct.

**Cause:** Mismatch between the MySQL authentication plugin and the method WordPress's PHP driver expected. Newer MySQL versions default to `caching_sha2_password`, which isn't always compatible with older PHP MySQL clients bundled in some WordPress images.

**Fix:** Ensured the WordPress image used (`php8.2-fpm` variant) had an up-to-date `mysqli`/`pdo_mysql` driver, and verified the MySQL user was created with a compatible auth plugin. Confirmed connectivity by exec-ing into the WordPress container and testing the connection manually before relying on WordPress's own install wizard to surface the error.

**Lesson:** When a DB connection fails in a container setup, test the connection directly from inside the app container first — it isolates whether the problem is network/DNS related or purely an auth/driver mismatch.

---

## 2. Nginx Symlink Issues

**Symptom:** Nginx reverse proxy config wasn't loading — requests weren't being routed to the WordPress backend, and Nginx gave no clear error on reload.

**Cause:** In the original manual LEMP setup (pre-Docker), the config lived in `sites-available` and needed to be symlinked into `sites-enabled` for Nginx to actually load it. The symlink had either not been created or pointed to the wrong path after moving files around.

**Fix:** Recreated the symlink correctly and verified with `nginx -t` before reloading. When migrating to Docker, this whole class of problem was eliminated by mounting configs directly into `conf.d/`, which Nginx loads automatically — no symlinking required.

**Lesson:** The Docker migration wasn't just about containerization — it removed a whole category of manual-setup fragility (symlinks, file permissions, path drift) that existed in the WSL/manual LEMP version.

---

## 3. WordPress URL Misconfiguration

**Symptom:** After migrating from the manual LEMP setup to Docker, the site loaded but all links, assets, and redirects pointed to the old host/URL — resulting in broken CSS, broken navigation, and login redirect loops.

**Cause:** WordPress stores its `siteurl` and `home` values directly in the database (`wp_options` table). These don't update automatically when the environment or hostname changes — they persisted from the original manual install.

**Fix:** Updated the `siteurl` and `home` values directly in the database to match the new Docker-based host:
```sql
UPDATE wp_options SET option_value = 'http://localhost' WHERE option_name IN ('siteurl', 'home');
```
Confirmed the fix by clearing the browser cache and reloading — assets and links resolved correctly.

**Lesson:** WordPress "remembers" its environment in the database, not just in config files. Any time you move a WordPress install to a new host/domain, check `wp_options` before assuming the app config is the problem.

---

## 4. Credential Syntax Errors in `.env`

**Symptom:** The MySQL container was failing to start, or starting with an empty/default database, with no obviously descriptive error in the logs.

**Cause:** Malformed syntax in the `.env` file — specifically values containing special characters (like `#` or spaces) that Docker Compose's env parser interpreted incorrectly, silently truncating or misreading the variable.

**Fix:** Quoted all values in `.env` and avoided characters that could be misinterpreted as comments or delimiters. Verified the actual values Docker Compose was passing into the container using:
```bash
docker compose config
```
This command prints the fully resolved compose file, which made the bad variable immediately visible.

**Lesson:** `docker compose config` is the fastest way to check what environment variables are actually reaching your containers — much faster than guessing from container logs alone.

---

## General Debugging Approach

Across all four issues, the same pattern helped:
1. Don't trust the surface-level error — get inside the container and test the failing piece directly (DB connection, config load, resolved env vars).
2. Check persisted state (database values, symlinks) separately from application config — WordPress and Nginx both keep state outside the obvious config files.
3. Use each tool's built-in validation (`nginx -t`, `docker compose config`) before assuming the problem is somewhere more complex.
