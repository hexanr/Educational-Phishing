# Deployment & Configuration Guide

## Overview

This document covers the system requirements, installation procedure, configuration options, and verification steps required to deploy PhishGuard in a local educational environment. The application is intended for single-machine, offline deployment only. It is not designed or tested for public-facing or networked multi-user deployment.

---

## System Requirements

| Component | Minimum Version | Notes |
|---|---|---|
| PHP | 7.4 | 8.x is supported and recommended |
| MySQL | 5.7 | MariaDB 10.3 or later is also supported |
| Web server | Apache 2.4 / Nginx 1.18 | Apache with mod_php is the simplest setup |
| Browser | Any modern browser | Required only for the frontend interface |

All components must be running on the same machine. No external network connectivity is required during operation.

---

## Installation

### Step 1 — Copy project files to the web server root

```bash
cp -r "Educational Phishing" /var/www/html/
cd "/var/www/html/Educational Phishing"
```

On Windows with XAMPP, place the folder in `C:\xampp\htdocs\` instead.

### Step 2 — Create and populate the database

Three options are provided. Use whichever matches your environment.

**Option A: phpMyAdmin**

1. Open phpMyAdmin and create a new database named `phishing` with character set `utf8mb4`.
2. Select the database, navigate to the Import tab, and load `Database/phishing_educational.sql`.

**Option B: MySQL command line (single command)**

```bash
mysql -u root -p < Database/phishing_educational.sql
```

**Option C: Interactive MySQL session**

```bash
mysql -u root -p
```

```sql
CREATE DATABASE phishing CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
USE phishing;
SOURCE '/var/www/html/Educational Phishing/Database/phishing_educational.sql';
```

### Step 3 - Check the configuration

No example file is needed. `Configuration/config.php` already defaults to the standard XAMPP values:

```php
define('DB_HOST', 'localhost');
define('DB_NAME', 'phishing');
define('DB_USER', 'root');
define('DB_PASS', '');
```

You can override them with the environment variables `DB_HOST`, `DB_NAME`, `DB_USER` and `DB_PASS`.

The demo account is created by the SQL file: username `demo_user`, password `test123`. For learning use only.

### Step 4 — Set directory permissions

```bash
chmod 755 /var/www/html/Educational Phishing
chmod 644 "/var/www/html/Educational Phishing"/Frontend/*.php
chmod 644 "/var/www/html/Educational Phishing"/Backend/*.php
chmod 644 "/var/www/html/Educational Phishing"/Database/*.sql

mkdir -p "/var/www/html/Educational Phishing"/logs
mkdir -p "/var/www/html/Educational Phishing"/temp
chmod 755 "/var/www/html/Educational Phishing"/logs
chmod 755 "/var/www/html/Educational Phishing"/temp
```

The `logs/` and `temp/` directories must be writable by the web server process. If the web server runs as `www-data`, you may need:

```bash
chown -R www-data:www-data logs/ temp/
```

### Step 5 — Verify the installation

Open a browser and navigate to:

```
http://localhost/Educational%20Phishing/Frontend/warning.php
```

If the page loads without errors, proceed to the verification checks below. If you encounter an error, refer to the Troubleshooting section.

---

## Verification Checklist

Run through the following checks before any demonstration session.

**Database**

```bash
mysql -u root -p phishing -e "SHOW TABLES;"
```

Expected output should list `users`, `security_logs`, and any other tables defined in the SQL schema.

**Application flow**

Navigate through each page in sequence and confirm the expected behavior:

1. `Frontend/warning.php` — Consent screen loads. Accept and Decline buttons are present and functional.
2. `Frontend/index_educational.html` — Login form loads. Submitting the form redirects to the next page without errors.
3. `Frontend/thank_you_educational.html` — Disclosure and debrief content is displayed.
4. `Frontend/protection_guide.html` — Reference content loads completely.

**Logging**

After submitting the test form, verify that a log entry was written:

```bash
tail -n 10 "/var/www/html/Educational Phishing"/logs/security.log
```

A timestamped entry should appear for the submission event. If the file is empty or absent, check the permissions on the `logs/` directory.

**Test credentials**

Use the following credentials for all testing and demonstration purposes. Do not use real credentials at any point.

```
Username: demo_user
Password: test123
```

---

## Session Procedure

The following procedure is recommended for each instructional session.

**Before the session.** Verify all checklist items above. Clear the `logs/` directory and reset the database submissions table if the application has been used previously:

```sql
TRUNCATE TABLE security_logs;
```

**During the session.** Do not reveal that the login page is simulated before participants submit the form. The perceptual training value of the simulation depends on participants encountering it without prior framing.

After participants submit the form and reach `thank_you_educational.html`, pause and walk through the following with the group:

1. What visual or behavioral indicators were present that could have identified the page as fraudulent.
2. What data was captured by the backend and how it was stored.
3. Which security controls in the backend code correspond to which vulnerabilities.
4. What controls are absent from the current implementation and what attacks those absences leave open.

Close the session with `protection_guide.html` as a reference document for participants to retain.

**After the session.** Delete or truncate all submission data from the database. If participants may have entered real credentials by mistake, inform them immediately and advise them to change those credentials on the relevant service.

---

## Security Controls Reference

The following controls are implemented in the backend. They are documented here for reference during code-level instruction.

**Prepared statements** separate query structure from user-supplied data, preventing SQL injection regardless of input content:

```php
$stmt = $conn->prepare("SELECT * FROM users WHERE username = ?");
$stmt->bind_param("s", $username);
$stmt->execute();
```

**BCRYPT password hashing** stores credentials as one-way hashes. The cost factor of 12 makes brute-force attacks computationally expensive:

```php
$hashed = password_hash($password, PASSWORD_BCRYPT, ['cost' => 12]);
```

**CSRF token validation** ensures that form submissions originate from the application's own interface and not from a third-party site:

```php
if (!hash_equals($_SESSION['csrf_token'], $_POST['csrf_token'])) {
    die('Invalid CSRF token.');
}
```

**Input validation** rejects values that do not conform to the expected format before they reach any database query or response:

```php
if (!preg_match('/^[a-zA-Z0-9_]{3,20}$/', $username)) {
    die('Invalid username format.');
}
```

**Secure HTTP headers** instruct the browser to reject common injection and framing attacks at the transport layer:

```php
header('X-Content-Type-Options: nosniff');
header('X-Frame-Options: DENY');
header('X-XSS-Protection: 1; mode=block');
```

---

## Troubleshooting

**"Connection failed" or database error on load**
MySQL is not running, or the credentials in `Configuration/config.php` do not match the database configuration. Verify that MySQL is active (`systemctl status mysql`) and that the values in `Configuration/config.php` are correct.

**"Table doesn't exist"**
The SQL file was not imported successfully. Re-run the import and check the MySQL output for errors. Confirm that the correct database is selected before running the import.

**HTTP 404 on any page**
The project files are not in the directory that the web server is configured to serve. Verify that the folder is in the correct document root and that the path in the browser matches the folder name.

**Blank page with no error**
PHP error reporting may be suppressed. Add the following line temporarily at the top of the relevant PHP file to surface the error:

```php
ini_set('display_errors', 1);
error_reporting(E_ALL);
```

Remove these lines after diagnosing the issue.

**Log file not being written**
The `logs/` directory is not writable by the web server process. Run `chmod 755 logs/` and, if necessary, `chown www-data:www-data logs/`.

**Form submission does not redirect**
The CSRF token may be missing from the session, which typically indicates a session configuration issue. Verify that `session_start()` is called at the top of the form handler and that PHP sessions are enabled in `php.ini`.

---

**Last updated:** 2025
**License:** Educational use only. Deployment against uninformed users is prohibited.
