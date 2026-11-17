[breach-narrative.md](https://github.com/user-attachments/files/34946967/breach-narrative.md)
PHP2Shell: The Hartwell Breach: Web Server Compromise, Start to Finish

## Summary

An attacker operating from `192.168.1.100` scanned a public-facing web server, found a weakness in how it handled file uploads, and used it to plant a webshell named `/health-check.php`. From there they ran commands as the web server's service account (`www-data`), mapped the network, read the WordPress configuration file, dumped the user table with password hashes, tried and failed to shut down the host's security monitoring, and set up a scheduled callback for persistence. Data left the server through the webshell's own HTTP responses. There is no evidence the attacker moved to any other host.

> **Note on times:** Times are shown as recorded in the case file. Entries 5 and 6 (09:49 PM and 01:36 PM) are not in strict clock order, so they may reflect different days or log sources.

---

## Phase 1: Reconnaissance

**03:00 PM: The scanner appears.** Activity begins from `192.168.1.100`, the source of everything that follows.

**03:08 PM: Nmap.** The attacker uses `Nmap` to probe the server and enumerate what it exposes. This is active vulnerability scanning (T1595, T1595.002).

## Phase 2: Initial Access

**03:19 PM: The error trail.** Errors referencing `session.upload_progress` show up. This is a PHP feature tied to tracking file uploads, and the errors suggest the attacker was probing or abusing the server's upload handling.

**05:06 PM: Exploit delivery.** The attacker sends a `POST` request with `multipart/form-data` and the boundary `------------------------xvmrl9egljKcUfer5nAbL3`. This is the exploit itself, delivered as a file upload against a public-facing application (T1190).

## Phase 3: Foothold

**09:49 PM: The payload lands.** A webshell is placed at `/health-check.php`. The name is innocuous-sounding, which helps it blend in with ordinary site files (T1505.003).

**01:36 PM: The operator.** Commands now execute as `www-data`, the unprivileged account the web server runs under. This limits what the attacker can do, but it is enough to read the web application's files and database credentials (T1059.004).

**02:02 PM: The hidden trigger.** The webshell responds to a custom HTTP header, `X-HMG-Debug`. Requests without it look like normal traffic, so the header acts as a gate that keeps the shell quiet from casual scanning. It is a useful pivot for hunting.

**02:38 PM: Encoded orders.** The attacker's commands are obfuscated with `ROT13` and `base64`, which hides their intent from simple log review and signature matching (T1027).

## Phase 4: Discovery

**04:50 PM: Network mapping.** The attacker runs `ip addr show`, `ip neigh`, and `ss -tlnp` to learn the server's interfaces, neighboring hosts, and listening services (T1016).

**05:17 PM: Monitoring awareness.** The attacker checks for `splunk`, `sysmon`, and `auditd`, working out what is watching the machine (T1518.001).

## Phase 5: Credential Access

**05:29 PM: The config file.** The attacker reads `wp-config.php`, which holds the WordPress database connection details.

**05:41 PM: The database dump.** Using those details, they pull `wp_users` and the `user_pass` column, obtaining password hashes (T1555).

## Phase 6: Defense Evasion

**05:53 PM: Targets for silence.** The attacker tries to stop `sysmon`, `auditd`, and `SplunkForwarder` (T1562.001).

**07:18 PM: Success or failure? Failure.** The stop commands ran as `www-data`, an unprivileged service account without root or sudo rights, so `systemctl` and `auditctl` were refused. The monitoring stayed up, which is why the rest of this activity is visible.

## Phase 7: Persistence

**07:35 PM: The scheduled callback.** The attacker adds a `crontab` entry that redirects output (`2>`) and calls `.cache-helper`, so the implant runs on a schedule even if the webshell is removed (T1053.003).

**07:46 PM: The hidden implant.** The implant sits at `/dev/shm/.cache-helper`. `/dev/shm` is memory-backed shared storage, and the leading dot and generic name make the file easy to overlook (T1036.005).

## Phase 8: Exfiltration

**07:50 PM: Data out.** The attacker had the data base64-encoded on the server, then pulled it back through the webshell's own HTTP responses. There was no separate upload or outbound transfer (T1041).

## Phase 9: Scope Check

**07:54 PM: Lateral movement? No.** Every command ran locally on the web server as `www-data`. The attacker did discovery (`ip addr show`, `ip neigh`, `ss -tlnp`) and looked for SSH keys (`find /home -name authorized_keys -o -name id_rsa`), but no `ssh`, `scp`, `nmap`, `nc` or `ping` to another 192.168.1.x host appears. The `curl` to 127.0.0.1 was loopback, so there is no evidence of lateral movement (T1021 was considered and not observed).

---

## Outcome

| Question | Answer |
|----------|--------|
| How did they get in? | Exploit against the public-facing web application via a multipart file upload |
| What did they leave behind? | Webshell at `/health-check.php`, cron callback, implant at `/dev/shm/.cache-helper` |
| What did they take? | WordPress config details and `wp_users` data, including `user_pass` hashes |
| Did they blind the defenders? | No. Stop attempts on `sysmon`, `auditd`, and `SplunkForwarder` were refused |
| How did data leave? | Through the webshell's HTTP responses, with no separate transfer |
| Did they spread? | No evidence of lateral movement |

---

## Indicators of Compromise (9)

| Type | Value |
|------|-------|
| indicator | `192.168.1.100` |
| indicator | `Nmap` |
| indicator | `session.upload_progress` |
| indicator | `/health-check.php` |
| indicator | `www-data` |
| pivot | `X-HMG-Debug` |
| indicator | `wp-config.php` |
| indicator | `crontab, 2&gt, .cache-helper` |
| pivot | `/dev/shm/.cache-helper` |

## MITRE ATT&CK Mapping (14)

| Technique | Name | Where it appears |
|-----------|------|------------------|
| T1595 | Active Scanning | Phase 1 |
| T1595.002 | Vulnerability Scanning | Phase 1 |
| T1190 | Exploit Public-Facing Application | Phase 2 |
| T1505.003 | Web Shell | Phase 3 |
| T1059.004 | Unix Shell | Phase 3 |
| T1027 | Obfuscated Files or Information | Phase 3 |
| T1016 | System Network Configuration Discovery | Phase 4 |
| T1518.001 | Security Software Discovery | Phase 4 |
| T1555 | Credentials from Password Stores | Phase 5 |
| T1562.001 | Disable or Modify Tools | Phase 6 |
| T1053.003 | Cron | Phase 7 |
| T1036.005 | Match Legitimate Name or Location | Phase 7 |
| T1041 | Exfiltration Over C2 Channel | Phase 8 |
| T1021 | Remote Services | Phase 9 (checked, not observed) |
