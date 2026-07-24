# Responding to an Active WordPress Compromise: Root Cause, Toolkit, and a Process That Wouldn't Die

**Date:** 2026-07-23 / 2026-07-24
**Platform:** WordPress 7.0.0 (patched to 7.0.2), Plesk on AWS Lightsail

This is the technical version. There's a more readable, story-shaped version of this on [my blog](https://jimbevard.com/responding-to-an-active-wordpress-compromise-a-root-cause-walkthrough/) if you want the narrative without the IOCs.

## Root cause

The site was running WordPress **7.0.0**, vulnerable to a pre-auth remote code execution bug publicly nicknamed **wp2shell**:

- **CVE-2026-63030** — logic flaw in the WordPress REST API batch processor (`/wp-json/batch/v1`): validation and execution happen in separate loops, allowing request confusion
- **CVE-2026-60137** — SQL injection via the `author__not_in` parameter of `WP_Query`

Chained together, unauthenticated RCE. Fixed in 7.0.2 (and 6.9.5 for the 6.9.x branch). Public PoCs exist; it's on CISA's KEV catalog (added 2026-07-21).

The rogue admin accounts created during exploitation confirmed the toolkit directly:

| Username | Email | Created |
|---|---|---|
| `wpenginebot` | wpenginebot@wpengine.com | 2026-07-19 |
| `w2s_df79d519c056` | w2s_df79d519c056@google.local | 2026-07-21 |
| `yun_11` | yun_11@wp2shell.invalid | 2026-07-21 |
| `w2s_339c3d4dc4d3` | w2s_339c3d4dc4d3@shellcode.lol | 2026-07-22 |

`w2s` = wp2shell. The domain `wp2shell.invalid` on one of the accounts isn't subtle.

## The access-control trick

Both the site root and ten per-directory `.htaccess` files (one in each directory holding a planted backdoor) had been modified. The interesting part: this wasn't a crude "deny everything" file. Plesk auto-deploys a standard hardening template across every subdirectory on a shared box, of the shape:

```apache
<FilesMatch ".(py|exe|php)$">
Order allow,deny
Deny from all
</FilesMatch>
<FilesMatch "^(about.php|radio.php|index.php|content.php|lock360.php|admin.php|wp-login.php)$">
Order allow,deny
Allow from all
</FilesMatch>
```

That base allow-list (`about.php`, `radio.php`, `index.php`, `content.php`, `lock360.php`, `admin.php`, `wp-login.php`) is Plesk's own legitimate default, present unmodified in thousands of directories server-wide. The attacker's modification just appended their own filenames to the *existing* list:

```
wp-l0gin.php, wp-the1me.php, wp-scr1pts.php, wp-admin.php, mah.php, jp.php, ext.php
```

Note the character substitutions (`wp-l0gin.php` with a zero, not an "o"). This is why a generic scanner produces thousands of false positives on this pattern (every legitimate Plesk `.htaccess` matches the shape) while still hiding the ten *actually* malicious ones in plain sight. A general "does this file look like an access-control bypass" heuristic can't tell the difference; you need to know the attacker's specific filenames to separate the two.

## What was planted (IOC list)

All quarantined, not deleted:

| Path | What it was |
|---|---|
| `wp-includes/blocks/video/wp-scr1pts.php` | Webshell, fake "Monarx Security Site Analyzer" banner, self-decoding `eval()` payload |
| `wp-content/themes/twentyseventeen/inc/wp-login.php` | Same signature, inactive default theme |
| `wp-content/themes/twentytwentyone/assets/2index.php` | Same signature |
| `wp-content/plugins/folders/includes/wp-the1me.php` | Same signature, inactive plugin |
| `wp-content/uploads/2023/03/wp-l0gin.php` | Same signature, in `uploads/` (should never execute PHP) |
| `httpdocs/3a8b7298/about.php` | Same signature, random top-level directory |
| `wp-content/plugins/8c8d11b6a865b5699c727a497f61ae3f/` | Hash-named fake plugin wrapping a real, unmodified bbPress install plus ~17 additional webshell scripts (`ag.php`, `get.php`, `rm.php`, `byp.php`, `ahax.php`, `ahax2.php`, `ms.php`, `ms-w.php`, `ms-elp.php`, `login.php`, `acfuck.php`, `403.php`, `upgrade.php`, `new.php`, `nv.php`, `see.php`, `un.php`, `bye.php`, `log.php`, `links.php`) |
| `index.php` | Visitor-profiling/cloaking redirect. Sent visitor IP/UA/referrer to `us512-v352.usamazon3[.]click` (base64-decoded from source), could silently swap in remote page content |
| `wp-includes/class-wp.php` | Raw `include` injected into `WP::main()` (runs on every request), pointed at a disguised cache-path trigger file |
| `.../swv/74014` | The self-heal trigger for `index.php` — compared live content against a stored encoded copy, silently rewrote it back on mismatch |
| `.../swv/86511` | `base64_decode(str_rot13(...))`-encoded copy of the malicious `index.php`, the source `74014` restored from |
| `wp-includes/js/dist/index.php` | Remote code loader, fetched and `include`d PHP from `51la.icw7[.]xyz/a2.txt` |
| `wp-includes/blocks/admin.php` (also found on staging) | Full RCE backdoor, Japanese-language comments, triggered via `$_REQUEST['ac']/['path']/['api']/['t']`, referenced `51la.vo4[.]xyz/f.txt` and `thebevardstudio.com/admin.php?520` as a `shell_link` |
| `wp-includes/html-api/radio.php` | Fake "WP Super Cache" plugin header, obfuscated `eval()` chain, fake attribution to `github.com/cAT3VWynuiL7CRgr` |
| `wp-content/uploads/omgf/elementor-gf-robotoslab/index.php` | Same fake-plugin-header kit as `radio.php` above |
| `wp-content/plugins/host-webfonts-local/mu-plugin/radio.php` | Second full webshell/file-manager panel + generic remote-code loader (C2 URL passed as a request param, not hardcoded — evades static string scans) |
| `wp-content/plugins/host-webfonts-local/mu-plugin/lock360.php` | Companion file, same location |
| `wp-content/cache/page/3cc/content.php` | Same toolkit family |
| `test.thebevardstudio.com/admin.php`, `/index.php`, `/wp-includes/blocks/index.php` | Same kit, independently planted on the staging subdomain |

A single MD5 password hash, `d90ffde1b2e6b317292bff7c565c5e69`, recurred across multiple otherwise-distinct-looking backdoors — same toolkit, redeployed with variations, not several unrelated actors.

**C2 infrastructure observed** (defanged):
`usamazon3[.]click`, `51la.vo4[.]xyz`, `51la.icw7[.]xyz`, `c.zvo4[.]xyz`, `c2.icw7[.]com`, `c.zvo1[.]xyz`, `45[.]11[.]57[.]159`

## The part that made this hard: a process, not a file

After the first cleanup pass (core patched, backdoors removed, credentials rotated), the site was compromised again within hours. `.htaccess` and `index.php` kept reverting, including while the entire site was locked down at the web-server level (`Deny from all` on every `.php` file).

That was the actual signal: if nothing can execute through Apache, whatever's rewriting the files isn't going through Apache anymore.

```
$ ps -u thebevardstudio.com_r6nrz3v5fp -o pid,etime,cmd
thebeva+  300720  00:17  /opt/plesk/php/8.2/bin/php /var/www/vhosts/thebevardstudio.com/httpdocs/l.php
```

A standalone PHP CLI process, launched once via a webshell call, running outside Apache entirely. It had already deleted `l.php` from disk immediately after launch, so no file-based scan would ever find it, only checking what was actually *running* did.

```
$ kill -9 300720
```

No respawn after that. The live attacker IP making the triggering webshell calls (`198.35.44.241`) was blocked at the firewall separately.

## Remediation, in order

1. Patch WordPress core to 7.0.2 (downloaded fresh from wordpress.org directly, not from the staging copy, which turned out to have its own independently-planted backdoor)
2. Kill the rogue process; confirm no respawn under sustained monitoring, not a single check
3. Remove the four confirmed-malicious admin accounts; downgrade one ambiguous account (an expired legitimate contractor) to no-privilege rather than deleting it, after confirming with the site owner
4. Rotate: WP admin passwords, DB password, SFTP/system account password, WP secret keys/salts
5. Quarantine all backdoor files/directories (moved, not deleted) for later reference
6. Re-verify with `wp core verify-checksums` against the official WordPress.org release rather than trusting a visual diff
7. Full malware scan pass (ImunifyAV) as a second opinion — caught the 10 companion `.htaccess` files and two additional staging webshells my manual sweep missed, at the cost of ~5,600 false positives on the generic Plesk `.htaccess` pattern that had to be triaged out
8. Block the attacker's source IP at the firewall level directly, in addition to fail2ban's automatic (and in this case, self-inflicted-on-testing) banning

## Why the false positives mattered

Worth noting explicitly: an automated scanner and a manual review disagreed in both directions here. ImunifyAV caught real backdoors a manual `grep`-based sweep missed (the companion `.htaccess` files, because the malicious content wasn't a *new* string, it was an *addition* to legitimate content). But it also flagged one theme file repeatedly as a backdoor dropper that, on manual inspection (checking for the injection-boundary pattern used elsewhere, checking escaping, checking against the vendor's actual code structure), was legitimate, if unusually `$_POST`-heavy, contact-form handling code. Neither tool alone was sufficient. The scanner found what grep couldn't see; the manual review filtered out what the scanner couldn't distinguish from a false pattern match.
