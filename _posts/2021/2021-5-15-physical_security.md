---
layout: post
title: Physical Security
tags:
- security
- meatspace
- watercooler
- cinema
- locksmithing
---

I love a good caper film.  When I was a wee lad my father introduced me to [The Great Train Robbery](https://www.imdb.com/title/tt0079240/).  Over the years I've seen several memorable others [Rififi](https://en.wikipedia.org/wiki/Rififi), [A World Without Thieves](https://www.imdb.com/title/tt0439884/), several by [David Mamet](https://www.imdb.com/name/nm0000519/?ref_=tt_ov_dr) like [The Spanish Prisoner](https://www.imdb.com/title/tt0120176/).  Just the other day I watched [Inside Man](https://www.imdb.com/title/tt0454848/).

Numerous vulnerabilities like [Spectre](https://en.wikipedia.org/wiki/Spectre_(security_vulnerability)), [Man-in-the-middle](https://en.wikipedia.org/wiki/Man-in-the-middle_attack), [SQL injection](https://en.wikipedia.org/wiki/SQL_injection), [keyloggers](https://en.wikipedia.org/wiki/Keystroke_logging), [phishing](https://en.wikipedia.org/wiki/Phishing) and a legion of others have loomed over the IT sphere for decades.  Seems like every month there's a new, "largest ever", high profile security incident.

The never ending struggle has given rise to a bevy of best-practices and technologies: [asymmetric cryptography](https://en.wikipedia.org/wiki/Public-key_cryptography), [penetration testing](https://en.wikipedia.org/wiki/Penetration_test), [salting hashed passwords](https://en.wikipedia.org/wiki/Salt_(cryptography)), numerous VM enhancements (like [SEV](https://developer.amd.com/sev/)), [two-factor authentication](https://en.wikipedia.org/wiki/Multi-factor_authentication) and countless others.

## Locks and Lock Picking

Many people working in IT are accustomed to the idea of a (climate-controlled) server room with limited access.  Anyone that's needed to restore root/administrator access (and perhaps keeps a bootable USB drive of associated tools on their person) understands the fundamental risk.

In most cases, these are protected by locks controlled via keypad, access card, or biometrics.  But the pre-cursor to them, the humble pin tumbler lock, is still found in a variety of settings like doors and cabinets that likewise protect equipment.

About a year ago I became curious about lock picking.  I can't remember which movie I was watching, but someone is locked in a basement and manages to remove their restraints and escape- seemed like a useful skill to have.

Numerous resources exist for budding locksmiths such as [The Complete Book of Locks and Locksmithing](https://www.amazon.com/gp/product/B01LZ44ZD5/ref=kinw_myk_ro_title).  It covers the design of several types of locks, how to duplicate keys and design a master-key system, and lockpicking among other topics relevant in the trade.  Additionally, there are an assortment of educational tools such as practice locks:

![](/assets/lock_practice.png)

After which you can graduate to over-the-counter locks available in any hardware store:

![](/assets/lock_real.png)

Most surprising to me was that even with inexpensive, entry-level picks, anything less than "high security" locks take little practice.  __NB__: possession of lockpicks by unlicensed individuals may [not be legal where you are](https://en.wikipedia.org/wiki/Lock_picking#Legal_status).

Continuing with my cinematic theme, [The Next Three Days](https://www.imdb.com/title/tt1458175/) features Russel Crowe's failed attempt to use a ["bump key"](https://en.wikipedia.org/wiki/Lock_bumping).  A variety of lockpicking using a specially cut key for particular locks.

## Does Physical Security Really Exist?

Over the years, researchers have demonstrated borderline outlandish, largely theoretical attacks that don't even require physical access.  For example, using [hard-drive acoustics](https://arstechnica.com/information-technology/2016/08/new-air-gap-jumper-covertly-transmits-data-in-hard-drive-sounds/), [screen brightness](https://cyber.bgu.ac.il/exfiltrating-data-from-air-gapped-computers-using-screen-brightness/), and several others.

And more recently, the article that inspired this post, [Researchers Can Duplicate Keys from the Sounds They Make in Locks](https://kottke.org/20/08/researchers-can-duplicate-keys-from-the-sounds-they-make-in-locks).
