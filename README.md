# wp2shell-PoC

> **⚠ This tool is created solely for educational or bug bounty purpose only. Unauthorized use outside of controlled environments is strictly prohibited.**


## Overview

A proof-of-concept for the wp2shell vulnerability chain affecting WordPress Core, combining CVE-2026-63030 and CVE-2026-60137. The project demonstrates the interaction between the REST API Batch route confusion vulnerability and a WP_Query SQL injection, resulting in an unauthenticated path to full WordPress compromise and remote code execution (RCE).

**Read the full advisory [here](https://wp2shell.com/)**

# How it works

**wp2shell** is a pre-authentication RCE chain in WordPress core, combining CVE-2026-63030 (route confusion in the batch REST endpoint) and CVE-2026-60137 (SQL injection in WP_Query).

**The route confusion:** `/wp-json/batch/v1` processes multiple sub-requests via parallel `$matches` and `$validation` arrays indexed by position. A sub-request with a malformed path (e.g., `http://:`) is appended to `$validation` but not `$matches` due to a `continue` statement, desynchronizing the arrays. Later requests are dispatched under the handler meant for the *next* request, bypassing schema validation and permission checks.

**The SQL injection:** Two nested batch calls exploit this. The outer batch bypasses the method allow-list (normally blocking GET). The inner batch delivers a scalar `author_exclude` string to `GET /wp/v2/posts` - the desync routes it past validation, and `WP_Query` interpolates the unsanitized string directly into SQL, yielding a UNION-based blind injection.

**Cache poisoning:** The SQLi returns forged `WP_Post` objects, which WordPress caches in-memory. These fake posts contain `[embed]` shortcodes that cause WordPress to create real `oembed_cache` database rows from the fake references.

**Changeset escalation:** Using the SQLi, the attacker forges a `customize_changeset` post in-memory with `"user_id": 1` in its JSON. A cycle-detection gadget triggers `wp_update_post()` without overwriting `post_content`, preserving the attacker's payload. Applying the changeset temporarily assumes the administrator's identity.

**Hook re-entry:** A fabricated post with status `parse` and type `request` fires the `parse_request` hook, replaying the entire batch request with the assumed admin role. This time, a `POST /wp/v2/users` sub-request succeeds, creating a new admin account.

**Code execution:** The attacker logs in as the created admin and uploads a malicious plugin to run arbitrary commands.

# Affected versions

| Version| Status |
|---------------|--------|
| WordPress 6.9.0 – 6.9.4 | Vulnerable |
| WordPress 7.0.0 – 7.0.1 | Vulnerable |
| WordPress 6.9.5 | Fixed |
| WordPress 7.0.2+ | Fixed |


# Usage

**To use this PoC, the only requirement is Python 3.8+.**

Run it from the repository directory to perform a vulnerability check:

```bash
wp2shell.py http://victim.com
```

### Check Mode (default)

Performs a single vulnerability check. Sends a benign batch marker probe that detects the route confusion bug without executing SQLi payloads. A vulnerable target returns HTTP 207 with the error pattern `parse_path_failed`, `block_cannot_read`, and `rest_batch_not_allowed`.

Use `--confirm-sqli` to also send an active SQLi confirmation payload. The confirmation tries UNION reflection first, then falls back to timing-based probes.

 **Check single target (default mode)**
 
```bash
wp2shell.py http://target.com
```

**Check with explicit mode**

```bash
Check with explicit mode
wp2shell.py http://target.com --check
```

**Check with SQLi confirmation**

```bash
wp2shell.py http://target.com --check --confirm-sqli
```


### Read Mode - Extract Data Through SQL Injection

Extracts data from the database using the pre-authentication SQL injection. By default uses `--technique auto`, which tries available methods in this order:

- **union** - forges a fake `WP_Post` row via UNION and reads its title back from the REST response as `||HEX(value)||`. One request per value. Fastest.
- **error** - uses `EXTRACTVALUE`/`UPDATEXML` to leak ~15 bytes per request. Works when the target reflects MySQL errors (e.g., `WP_DEBUG_DISPLAY` on).
- **blind** - boolean binary search, ~8 requests per character. Reads the `X-WP-Total` header as the true/false signal. Works even when no data is reflected.

Force a specific technique with `--technique union|error|blind`. These read paths are read-only and do not write to the database.


**Server fingerprint (default query)**

```bash
wp2shell.py http://target.com --read
```

**Dump logins and password hashes**

```bash
wp2shell.py http://target.com --read --preset users
```

**Custom SQL query**

```bash
wp2shell.py http://target.com --read --query "SELECT @@version"
```

**Force blind technique**

```bash
wp2shell.py http://target.com --read --technique blind --query "SELECT user_login FROM wp_users LIMIT 1"
```

**Extract with error-based technique**

```bash
wp2shell.py http://target.com --read --technique error --query "SELECT user_pass FROM wp_users LIMIT 1"
```

### Shell Mode 

Executes commands on the target server. Works in two modes:

With credentials (logs in as existing admin and uploads plugin shell):


**Execute specific command**

```bash
wp2shell.py http://target.com --shell --user admin --password '<recovered>' --cmd id
```

**Interactive shell**

```bash
wp2shell.py http://target.com --shell --user admin --password '<recovered>' --interactive
Without credentials (pre-auth RCE - runs the full SQLi→admin bridge, logs in as generated admin, then uploads plugin shell):
```

**Execute single command**

```bash
wp2shell.py http://target.com --shell --cmd id
```

**Interactive shell**

```bash
wp2shell.py http://target.com --shell --interactive
```

The plugin webshell is uploaded with a random path and a per-run token. The uploaded webshell is removed automatically. When the pre-auth bridge creates an administrator, that generated account is removed automatically after the shell session finishes.


**All flags list:**

| Flag | Description |
| :--- | :--- |
| `--check` | Run vulnerability check (default mode if no other mode specified) |
| `--read` | Extract data via SQL injection |
| `--shell` | Execute commands on the server |
| `--query` | Custom SQL query for read mode |
| `--preset` | Predefined query preset (`users`, `config`, `versions`) |
| `--technique` | SQLi extraction technique: `union`, `error`, `blind`, or `auto` (default) |
| `--confirm-sqli` | Send SQLi confirmation payload after check |
| `--cmd` | Command to execute in shell mode (default: `id`) |
| `--interactive`, `-i` | Interactive shell mode |
| `--user` | Admin username for authenticated shell |
| `--password` | Admin password for authenticated shell |
| `--proxy` | HTTP/HTTPS proxy (e.g., `http://127.0.0.1:8080`) |
| `--timeout` | Request timeout in seconds (default: 30) |
| `--verbose`, `-v` | Verbose output |


# References:
1. https://slcyber.io/research-center/exploit-brokers-pay-500000-for-a-wordpress-rce-i-found-one-with-gpt5-6/
2. https://www.picussecurity.com/resource/blog/cve-2026-63030-and-cve-2026-60137-wp2shell-wordpress-rce-explained

# Disclaimer

This tool is created solely for educational or bug bounty purpose only. Unauthorized use outside of controlled environments is strictly prohibited.











