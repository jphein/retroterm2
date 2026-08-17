# RETROTERM 2

**The pocket terminal for your AI coding agents.** A 2010 Motorola Droid 2 — Android 2.2, slide-out
QWERTY — turned into a passwordless SSH console into a fleet of [Claude Code](https://claude.com/claude-code)
agents running in `tmux` on a homelab box.

It is a joke wrapped around a real project. The "product launch" is the joke. The engineering is
real, and it was more involved than it should have been — a phone this old can't finish a modern
SSH handshake, and the SSH app that runs on it ships a signature bug. Both had to be fixed.

🌐 **Landing page:** https://jphein.github.io/retroterm2/

Not affiliated with Motorola or Anthropic. Nothing here is for sale. It was in a drawer.

---

## What it actually is

- **Hardware:** Motorola Droid 2 (A955), 1 GHz OMAP3630, 512 MB RAM, 3.7″ 480×854, 5-row physical keyboard, rooted.
- **OS:** Android 2.2 "Froyo" (2010).
- **Client:** [VX ConnectBot](https://f-droid.org/packages/sk.vx.connectbot/) 1.7.1 — the fork maintained for old Android.
- **The other end:** an SSH box running `tmux`, where each window is a Claude Code agent session.

The result: open one saved host, `tmux attach`, and a sixteen-year-old phone shares the exact
session as everything modern on the network.

## Why it's harder than "install an app"

Four real problems. None solved by the app store.

### 1. A door the phone can knock on

A 2011-era SSH client only offers SHA-1-era key exchange, which modern OpenSSH refuses — the
handshake dies **before** login. Rather than weaken the real SSH port, run a **second** sshd on a
side port that *additionally* accepts the legacy algorithms. RSA host key only (old clients choke
on ECDSA).

```
# /etc/ssh/sshd_config_legacy   —  systemd: a second unit, ExecStart=… -f this file
Port 2222
HostKey /etc/ssh/ssh_host_rsa_key
KexAlgorithms +diffie-hellman-group14-sha1,diffie-hellman-group-exchange-sha1,diffie-hellman-group1-sha1
HostKeyAlgorithms +ssh-rsa
PubkeyAcceptedAlgorithms +ssh-rsa
Ciphers +aes128-ctr,aes128-cbc,3des-cbc
MACs +hmac-sha1
PasswordAuthentication yes
```

Threat model note: this is LAN-only, password over an encrypted channel, a phone from 2010.
SHA-1 KEX here is fine. Do not open it to the internet.

### 2. A client that signs correctly

The stock ConnectBot build produced RSA signatures the server rejected as **"invalid format"** —
key auth could never succeed, no matter the key. **VX ConnectBot** is the fork whose signer is
repaired. Install it; leave the original alone (its keys can't be exported anyway).

### 3. Getting the key onto it (rooted, no file-picker path)

No cloud sync, no working import UI. With root, write the key **directly into ConnectBot's SQLite
database**. ConnectBot stores an unencrypted private key as **PKCS#8 DER** and the public key as
**X.509 SubjectPublicKeyInfo DER**, `encrypted=0`, in the `pubkeys` table; a `hosts` row links to
it by `pubkeyid`.

```sh
# on your workstation
ssh-keygen -t rsa -b 2048 -m PEM -N "" -f droid            # droid (PEM), droid.pub (OpenSSH)
openssl pkcs8 -topk8 -nocrypt -in droid -outform DER -out priv.der
openssl rsa -in droid -pubout -outform DER -out pub.der

# pull the app DBs (root), edit with sqlite3, push back, chown to the app uid
sqlite3 pubkeys "INSERT INTO pubkeys(nickname,type,private,public,encrypted,startup,confirmuse,lifetime)
                 VALUES('droid','RSA',x'$(xxd -p priv.der|tr -d \\n)',x'$(xxd -p pub.der|tr -d \\n)',0,1,0,0);"
# then a hosts row with pubkeyid = that _id, and — CRITICAL — fill the nullable columns:
sqlite3 hosts   "INSERT INTO hosts(nickname,protocol,username,hostname,port,pubkeyid,
                 usekeys,useauthagent,stayconnected,wantsession,compression,color,fontsize,encoding,delkey)
                 VALUES('homelab','ssh','you','your-server',2222,1,
                 'true','no','false','true','false','gray',10,'UTF-8','del');"
```

Authorize the **public** half on the server (`~/.ssh/authorized_keys`, ideally `from="<lan>/24"`).

> ⚠️ **The one-hour gotcha:** a hand-injected `hosts` row leaves columns the UI normally fills as
> `NULL`. ConnectBot calls `.equals()` on `useauthagent` — so it authenticates *perfectly* and then
> throws a `NullPointerException` in `finishConnection` **before requesting the shell**. The server
> logs `Accepted publickey` but never `Starting session: shell`. Fill `useauthagent` (`'no'`) and the
> other nullable text columns.

### 4. Doing it all remotely, and *proving* it

The whole setup was driven over a USB cable with no eyes on the phone:

- **See the screen:** `su -c 'cat /dev/graphics/fb0'` → raw framebuffer → `convert -size 480x854 -depth 8 BGRA:… -rotate 270 out.png`.
- **Drive the UI:** `adb shell input keyevent`/`input text` (Froyo has no `input tap`).
- **Verify server-side**, the only line that counts: a real login shell on a pty, key-authed, no password.

> ⚠️ **Second gotcha:** ConnectBot caches its DB in memory. It must be **fully killed** — not
> backgrounded — before it re-reads an edited database. `su -c 'kill -9 <pid>'`, confirm dead, relaunch.

## Verifying it worked

```
# server: this is success — a shell, by key, no password
$ who
you   pts/0   (10.x.x.x)          # a login shell from the phone
$ journalctl -u ssh-legacy | grep -E 'Accepted publickey|Starting session: shell'
```

## Disclaimer

A weekend project, documented honestly. LAN-only. Use a `from=` restriction on the authorized key.
The old side-port and legacy algorithms are a deliberate, scoped trade for one vintage device — not
a template for exposing anything to the internet.

## License

GPLv3 — see [LICENSE](LICENSE). Matches the realm-sigil family.
