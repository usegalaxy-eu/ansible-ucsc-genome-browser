Ansible Role: UCSC Genome Browser
=================================

Installs and configures the [UCSC Genome Browser](https://genome.ucsc.edu/),
a web-based tool for visualizing genomic data. It works on RHEL-based
distributions (see [meta/main.yml](./meta/main.yml) for details).

## Requirements

None.

## Role Variables

Ansible variables are listed below (except for
`ucsc_genome_browser_assemblies_env`), along with default values (see `defaults/main.yml`):

```yaml
ucsc_genome_browser_setup_script: https://raw.githubusercontent.com/ucscGenomeBrowser/kent/master/src/product/installer/browserSetup.sh
ucsc_genome_browser_setup_script_checksum: "{{ undef() }}"  # e.g. "sha256:e47cd21c479a53d3157a4cd54f867dc26420d5f825d0b6954257e3ee3054882d"
ucsc_genome_browser_setup_script_path: /usr/local/bin/browserSetup.sh  # install and execute setup script from this path

ucsc_genome_browser_offline_mode: false
# Offline mode: data is loaded only from the local Mysql database and file system.
# On-the-fly mode: data is loaded from UCSC when not present locally.

ucsc_genome_browser_assemblies: all  # e.g. ["mm10", "wuhCor1"]
ucsc_genome_browser_assemblies_udr: false
ucsc_genome_browser_assemblies_update_enabled: false
ucsc_genome_browser_assemblies_update_schedule: monthly
ucsc_genome_browser_assemblies_update_assemblies: "{{ ucsc_genome_browser_assemblies }}"

ucsc_genome_browser_cgi_update_enabled: true
ucsc_genome_browser_cgi_update_schedule: weekly

# periodic trash directory cleanups are managed by a systemd timer
ucsc_genome_browser_clean_enabled: true
ucsc_genome_browser_clean_schedule: daily

# custom tracks unused for this many hours are removed by a systemd timer
ucsc_genome_browser_custom_tracks_clean_enabled: true
ucsc_genome_browser_custom_tracks_clean_schedule: daily
ucsc_genome_browser_custom_tracks_clean_age: 2880  # 120 days

# user data (`hgcentral`, `customTrash`, trash `ct/`) is backed up by a systemd timer
ucsc_genome_browser_backup_enabled: true
ucsc_genome_browser_backup_schedule: daily
ucsc_genome_browser_backup_dir: /var/backups/ucsc-genome-browser
ucsc_genome_browser_backup_keep_days: 15

# short API URLs, e.g. /list/ucscGenomes
ucsc_genome_browser_api_short_urls_enabled: true

# rate-limit hyperactive clients (bots/scrapers) with the bottleneck server
ucsc_genome_browser_bottleneck_enabled: false
ucsc_genome_browser_bottleneck_version: v505_branch.1  # kent git ref
ucsc_genome_browser_bottleneck_path: /usr/local/bin/bottleneck
ucsc_genome_browser_bottleneck_host: localhost
ucsc_genome_browser_bottleneck_port: 17776
ucsc_genome_browser_bottleneck_subnets: ["127.0.0.1/32", "::1/128"]  # allowed to connect
ucsc_genome_browser_bottleneck_penalty: 75  # ms per access
ucsc_genome_browser_bottleneck_recovery: 10  # ms recovered per idle second
ucsc_genome_browser_bottleneck_except: []  # never throttled

# Plausible analytics, off when the domain is empty
ucsc_genome_browser_plausible_domain: ""  # e.g. genome-browser.galaxyproject.eu
ucsc_genome_browser_plausible_server: https://plausible.galaxyproject.eu
```

Use `ucsc_genome_browser_setup_script` to pin the installer script to a
specific version or to run a custom installer script. The variable
`ucsc_genome_browser_setup_script_checksum` can be used to verify the
integrity of the script; leave it blank to skip the verification. The
setup script is installed to `ucsc_genome_browser_setup_script_path`. After
the setup is complete, the script is still called on a schedule to execute
recurrent upkeep jobs.

The UCSC Genome Browser can operate in two modes: online and offline. In
online mode, data is loaded from UCSC when not present locally. In offline
mode, data is loaded only from the local MySQL database and file system. The
operation mode can be set using `ucsc_genome_browser_offline_mode`. The
default is the online mode.

Text in the browser's track images is rendered anti-aliased with FreeType.
The installer script turns this on in `hg.conf` (`freeType=on`) and
downloads the fonts it needs into `htdocs/urw-fonts`, where they are kept up
to date by the software updates. The fonts are looked up at
`../htdocs/urw-fonts`, relative to the CGI directory. If `HTDOCDIR` is not
next to `CGIBINDIR`, point `freeTypeDir` to the fonts, or set `freeType` to
`off` to go back to the browser's built-in bitmap fonts, using
`ucsc_genome_browser_hg_conf_settings`:

```yaml
ucsc_genome_browser_hg_conf_settings:
  freeTypeDir: /srv/ucsc/htdocs/urw-fonts
```

The genome assemblies to install can be customized using the
`ucsc_genome_browser_assemblies` variable. By default, all available
assemblies are installed.

Genome assembly downloads use plain `rsync` over TCP by default. Set
`ucsc_genome_browser_assemblies_udr` to `true` to use UDR (fast UDP) transfers
instead.

Genome assemblies can be kept up to date using a `systemd` timer, which is
disabled by default. Set `ucsc_genome_browser_assemblies_update_enabled` to
`true` to enable it. Note that the installer script's `update` command cannot
update the assemblies without also updating the UCSC Genome Browser software,
therefore **every assembly update also involves a software update**. MariaDB is
stopped while the assembly update is in progress; the UCSC Genome Browser is
unavailable until the update finishes. Pick a quiet time via
`ucsc_genome_browser_assemblies_update_schedule`.

A second `systemd` timer keeps the UCSC Genome Browser software up to date by
running the installer script's `cgiUpdate` command. It updates the CGIs, the
htdocs and the shared tables in `hgcentral` (BLAT servers, assembly list), but
not the genome assemblies. `hg.conf` and `hg.conf.local` are left untouched.
Set `ucsc_genome_browser_cgi_update_enabled` to `false` to disable it, and
`ucsc_genome_browser_cgi_update_schedule` to any valid `OnCalendar` value to
change how often it runs.

Another `systemd` timer removes old files from the trash directory by running
the installer script's `clean` command. Set `ucsc_genome_browser_clean_enabled`
to `false` to disable it, and `ucsc_genome_browser_clean_schedule` to any valid
`OnCalendar` value to change how often it runs.

Custom tracks are stored in the `customTrash` database, with small pointer
files in the trash `ct/` directory, which the trash cleaner skips. A further
timer removes custom tracks unused for
`ucsc_genome_browser_custom_tracks_clean_age` hours (default 120 days, so that
saved sessions keep their custom tracks) using UCSC's `dbTrash`. Set
`ucsc_genome_browser_custom_tracks_clean_enabled` to `false` to disable it.

A further timer backs up the user data: user accounts and saved sessions
(`hgcentral`), custom tracks (`customTrash`) and the custom track files in the
trash `ct/` directory. The genome assemblies are not backed up, as they can be
downloaded again, nor are the big `hgcentral` tables that the software update
reloads from UCSC. Backups are written to `ucsc_genome_browser_backup_dir` and
kept for `ucsc_genome_browser_backup_keep_days` days. Point it to storage off
the host. Set `ucsc_genome_browser_backup_enabled` to `false` to disable it.
To restore, run `zcat databases-<date>.sql.gz | mysql` and extract
`ct-<date>.tar.gz` into the trash directory.

The [REST API](https://genome.ucsc.edu/goldenPath/help/api.html) is served by
the `hubApi` CGI program at `/cgi-bin/hubApi/`. The variable
`ucsc_genome_browser_api_short_urls_enabled` (defaults to `true`), enables
short URLs for the API via Apache rewrite rules, so that the API endpoints
`/list`, `/getData`, `/findGenome` and `/search` are also served at `/` and
cross-origin requests are allowed. For example, with short URLs enabled,
`/list/ucscGenomes` answers the same as `/cgi-bin/hubApi/list/ucscGenomes`.

Settings in `ucsc_genome_browser_hg_conf_private_settings` (empty by default)
are written to `hg.conf.private`, which only Apache can read, and included
from `hg.conf`. Use it for secrets. For example, to enable
[OAuth login](https://genome.ucsc.edu/goldenPath/help/mirrorManual.html) with
GitHub (redirect URI `https://<host>/cgi-bin/hgLogin`):

```yaml
ucsc_genome_browser_hg_conf_settings:
  login.https: "on"
  login.oauth.providers: github
  login.oauth.github.clientId: Ov23xxxxxxxx
ucsc_genome_browser_hg_conf_private_settings:
  login.cookieSalt: "{{ vault_login_cookie_salt }}"  # long random string
  login.oauth.github.clientSecret: "{{ vault_github_client_secret }}"
```

Set `ucsc_genome_browser_bottleneck_enabled` to `true` to rate-limit hyperactive
clients (bots, scrapers) with `bottleneck`, a daemon that the CGIs ask how long
to delay each client. UCSC publishes no binary, so the role builds it from the
kent source at `ucsc_genome_browser_bottleneck_version` and runs it as a
`systemd` service. Behind a reverse proxy, Apache must see the real client IPs
(e.g. via `mod_remoteip`).

Set `ucsc_genome_browser_plausible_domain` to count page views with
[Plausible](https://plausible.io/) at `ucsc_genome_browser_plausible_server`.
The role adds a small script via the hg.conf `addJs` key, which sends only the
URL path, never the query string. Downloads (Table Browser output files and
file links) are sent as `File Download` events. Static pages such as the help
pages are not counted.

## Dependencies

The `community.general` collection.

## Example Playbook

```yaml
- hosts: all
  roles:
    - role: usegalaxy_eu.ucsc_genome_browser
      become: true
```

## License

[GPLv3](LICENSE.md)

Please contribute back to this role by submitting pull requests or reporting issues.

## Author Information

This role was created in 2026 by [José Manuel Domínguez](https://github.com/domgz).
