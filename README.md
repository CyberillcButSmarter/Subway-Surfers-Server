Subway Surfers
=====

This is custom server code for old versions of Subway Surfers with now defunct servers. It is written in Python 3 and intended to be used on HTTP server with mod-wsgi installed.

You need to redirect hoodrunner.kiloo.com domain to your server and alias scripts on the server as follows:
   * /hr_dailyquests.php -> hr_dailyquests.py
   * /onlinesettings.php -> onlinesettings.py

The following server end points are known to exist in old versions of the game:

   * /hr_dailyquests.php - provides words for Daily Challenge/Word Hunt. Request/response format remained completely unchanged at least up until 2020.
   * /onlinesettings.php - initially just informed the game of new versions. In later versions provides unlock/expiry times for characters and boards and informs the game of active discounts.
   * /register.php - registers the user in the leaderboards system.
   * /report.php - uploads high score to the leaderboards.
   * /friends.php - synchronizes friends list between Facebook and the game.
   * /scores.php - fetches high scores of the user's friends.
   * /poke.php - pokes the specified friend?..
   * /brag.php - brags about the new high score to the user's friends.

Most of these got replaced with "2" versions over time, i.e. hr_dailyquests2.php, onlinesettings2.php, register2.php, probably due to format changes.

## Hosting and hooking it to a decompiled Unity client

The `DailyWord` script in older Subway Surfers builds posts a random `key` to:

* `http://hoodrunner.kiloo.com/hr_dailyquests.php`

and expects a semicolon-separated response in this exact shape:

* `dayIndex;word;year;sha1;month;day;hour;minute;second;expireSeconds`

This repository already implements that contract in `hoodrunner/hr_dailyquests.py`, including the same secret key/hash flow that your decompiled script uses.

### 1) Host the WSGI endpoints

You can run this repo behind Apache + `mod_wsgi` (as intended by this project).

1. Install Apache + mod_wsgi + Python 3 packages (examples below).
2. Copy this repo to your server (for example `/opt/subway/Subway-Surfers-Server`).
3. Configure URL aliases so the original game paths resolve to these files:
   * `/hr_dailyquests.php` -> `hoodrunner/hr_dailyquests.py`
   * `/onlinesettings.php` -> `hoodrunner/onlinesettings.py`

Install commands by OS/version:

```bash
# Ubuntu 20.04 / 22.04 / 24.04, Debian 11/12
sudo apt update
sudo apt install -y apache2 libapache2-mod-wsgi-py3 python3

# Fedora / RHEL / Rocky / AlmaLinux 8+
sudo dnf install -y httpd mod_wsgi python3

# CentOS 7 (EOL, only if you already run it)
sudo yum install -y httpd mod_wsgi python3

# Arch Linux
sudo pacman -S --needed apache mod_wsgi python

# macOS (Homebrew)
brew update
brew install httpd mod_wsgi
```

macOS note: Homebrew `httpd` runs on port `8080` by default. If you need port `80`,
edit the Homebrew Apache config and ensure permissions/firewall rules allow it.

Enable/start service:

```bash
sudo systemctl enable --now apache2  # Debian/Ubuntu
# or
sudo systemctl enable --now httpd     # RHEL/Fedora/CentOS/Arch

# macOS (Homebrew services)
brew services start httpd
```

Minimal Apache vhost example:

```apache
<VirtualHost *:80>
    ServerName hoodrunner.kiloo.com

    WSGIScriptAlias /hr_dailyquests.php /opt/subway/Subway-Surfers-Server/hoodrunner/hr_dailyquests.py
    WSGIScriptAlias /onlinesettings.php /opt/subway/Subway-Surfers-Server/hoodrunner/onlinesettings.py

    <Directory /opt/subway/Subway-Surfers-Server/hoodrunner>
        Require all granted
    </Directory>
</VirtualHost>
```

Then restart Apache.

### 2) Point the game to your server

You have two common options:

* **No APK changes:** point `hoodrunner.kiloo.com` DNS (or local hosts file for testing) to your server IP.
* **Patched client:** edit the decompiled string URL in `DailyWord` from `http://hoodrunner.kiloo.com/hr_dailyquests.php` to your own domain (for example `http://your-domain.com/hr_dailyquests.php`).

For Android local testing, if DNS is inconvenient, patching the URL in smali/C# is usually easiest.

### 3) Keep hash compatibility with your decompiled script

Your `DailyWord` code validates this SHA-1 input sequence:

`rawData[0] + rawData[1] + rawData[2] + key + secret + rawData[4] + rawData[5] + rawData[6] + rawData[7] + rawData[8] + rawData[9]`

`hoodrunner/hr_dailyquests.py` uses the same shared secret and ordering, so checksum verification passes as long as:

* your app sends `key` in POST form data,
* your endpoint returns all 10 fields separated by `;`,
* server clock is sane (UTC-based expiry timing).

### 4) Quick endpoint check

From your server shell, test the endpoint directly:

```bash
curl -X POST -d "key=ABCD123EF" http://127.0.0.1/hr_dailyquests.php
```

You should get a single line with 10 semicolon-separated values.

### 5) Common gotchas with old clients

* The client calls **HTTP**, not HTTPS, in your decompiled snippet.
* Some devices/networks block custom cleartext traffic; if needed, patch client URL and network settings accordingly.
* Word UI was built around short words (project comment notes >5 letters can look wrong).

### 6) If you want this running without Apache

You can also wrap these WSGI apps with Gunicorn/uWSGI + Nginx, but keep the same public paths (`/hr_dailyquests.php`, `/onlinesettings.php`) and plaintext response format.


### 7) Can this be hosted on GitHub Pages?

Short answer: **No, not by itself**.

GitHub Pages is static hosting (HTML/CSS/JS files). This project needs server-side Python/WSGI to:

* accept POST requests,
* read the `key` parameter,
* generate time-based values,
* compute SHA-1 checksums dynamically.

So use a host that can run Python processes (VPS, Railway/Render/Fly.io, your own machine, etc.).
You *can* still keep the code on GitHub, but the running endpoint must be deployed somewhere that supports backend code.
