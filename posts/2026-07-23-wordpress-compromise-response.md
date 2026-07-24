---
title: "Responding to an Active WordPress Compromise: A Root-Cause Walkthrough"
date: 2026-07-23
tags: [incident-response, wordpress, security, infrastructure]
---

## TL;DR

A WordPress site I run on a small AWS box got hacked through a core vulnerability I hadn't patched yet. What started as "the site won't load" turned into a multi-hour mess involving a botched update, some rogue admin accounts, a handful of disguised backdoors, and the thing that actually made this hard: a background process that kept quietly undoing every fix I made. I'm writing this up mostly so future-me has a playbook instead of a blank page next time.

## The symptom

The site was throwing a fatal error on every single page load:

```
PHP Fatal error: Uncaught Error: Failed opening required
'.../wp-includes/compat-utf8.php' in .../wp-settings.php:35
```

My first read was that a routine core update had gotten interrupted partway through and left five files missing. I copied them back in from a staging copy of the site, the errors stopped, and I figured that was that.

It wasn't. "The update got interrupted" is a symptom, not an explanation, and I should have asked why before moving on.

## Round two: locked out, and the real digging begins

Not long after, I started getting locked out of both the site and the admin panel. Connections just refused, over and over. I chased it through a few layers: DNS was fine, the firewall was fine, the panel service itself was healthy. Eventually I found it. My own IP had gotten escalated into fail2ban's "recidive" jail, the repeat-offender jail that blocks everything, not just one service.

So why was I getting banned? Turned out WordPress's own Heartbeat feature was quietly pinging admin-ajax.php every couple of minutes from a browser tab I had open, and something on the server was rejecting those pings hard enough to trip the ban. That "something rejecting admin-ajax.php" is the thread that actually mattered.

## The real root cause

The site's .htaccess file had been rewritten to block basically all PHP execution except for a specific list of filenames. That's a pattern for letting an attacker's own planted files run while quietly breaking normal WordPress functionality, admin-ajax.php included, which is exactly what set off the fail2ban loop in the first place.

Checking the WordPress version against known vulnerabilities gave me the actual answer. The site was running 7.0.0, and there's a nasty unauthenticated remote code execution bug in that version, publicly nicknamed "wp2shell," actively being exploited in the wild and sitting on CISA's list of known exploited vulnerabilities. It's fixed in 7.0.2.

The confirmation was almost funny once I saw it. A few rogue admin accounts I found during cleanup were literally named things like "w2s_something." The attacker's own tooling had basically signed the work.

If there's one thing worth remembering from this whole mess, it's this: patch WordPress core first, always, before you touch anything else. Every backdoor, every rogue account, every weird plugin issue traced back to this one unpatched hole. None of the cleanup matters if the front door's still unlocked.

## What was actually left behind

Once I knew what I was looking for, a full sweep turned up quite a bit (all of it quarantined, none of it deleted, in case I needed to look at it again later):

A visitor-tracking script had been slipped into the front of index.php that profiled visitors and could silently swap in different page content depending on who was looking. Someone had injected a raw include statement directly into a core WordPress function, pointed at a hidden trigger file that hadn't been dropped yet. There were multiple standalone webshells tucked inside inactive plugin folders, dressed up with fake plugin headers, one of them literally borrowing a real security vendor's file banner to look legitimate. There was even a self-healing pair of files, one holding an encoded copy of the malicious index.php that would quietly rewrite it back the moment it noticed the file had been cleaned. And a few rogue admin accounts, one pretending to belong to a hosting provider I don't even use.

This wasn't a smash and grab. It was a toolkit, built with redundancy on purpose.

## The part that actually made this hard

After the first full cleanup, core patched, backdoors gone, every credential rotated, the site got hit again within a few hours. Every fix I made kept getting quietly undone, even while I had the entire site locked down so no PHP could execute at all through the web server.

That last part is what tipped me off. If nothing can run through the web server, whatever's rewriting these files isn't coming through the web server anymore. Sure enough, checking running processes turned up a standalone PHP process, launched once through a webshell call, now running completely on its own outside of Apache, periodically writing the malicious files right back. It had already deleted its own source file the moment it launched, so no file scan was ever going to catch it. Only looking at what was actually running did.

The lesson there: when a fix doesn't stick and you've already ruled out "did I mess up applying it," stop looking at files and start looking at what's running. A compromise that survives a full web-layer lockdown isn't a web-layer problem anymore.

## What actually fixed it, roughly in order

Patch WordPress core to the version with the fix, since that's the actual door that needs closing. Kill anything unexpected running under the site's system user. Pull any rogue admin accounts, and check the ambiguous ones with the actual site owner instead of guessing. Rotate every credential in the chain, WordPress passwords, database password, SFTP password, secret keys, on the assumption that anything sitting on disk had already been read. Quarantine anything suspicious instead of deleting it outright, since you'll probably need to go back and check it again. Re-verify core file integrity against the official checksums rather than trusting your own eyeballing of "looks fine." And don't assume a staging or backup copy is clean just because it's not the live site. In this case, the staging copy had its own, separately planted backdoor.

## What I'd do differently

I'd treat "no auto-updates in production" as a much stronger rule specifically for security patches. Feature updates can wait for a proper review pass, a pre-auth RCE can't. If something comes back after being fixed once, I'd jump straight to assuming there's still active, ongoing access rather than reapplying the same fix and hoping. I'd keep a real quarantine folder as a standing practice instead of improvising one mid-incident. And I'd keep running automated malware scanning, since it caught real things my own manual review missed, but I'd go in expecting a lot of noise on completely unrelated, harmless files that needs sorting through rather than trusting it blindly.

---

*This happened on a small side project server, nothing work related. But the actual approach, don't stop at the first explanation that sounds plausible, check that a fix actually holds instead of assuming it does, and take it seriously when something breaks your model of what should even be possible, is the same discipline I'd want in a Salesforce org migration or a production Flow incident. Different platform, same instinct.*
