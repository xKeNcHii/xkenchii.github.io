---
title: "DSTA BrainHack CDDC 2026: Vuln Chain to Platform Compromise"
date: 2026-05-26 21:00:00 +0800
categories: [CTF, Disclosure]
tags: [ctf, cddc-2026, responsible-disclosure, aws, ecs, ecr, imds, ssrf, puppeteer, svg]
author: Koh Xuan Qi
excerpt: >-
  A six-step chain that pivoted Nexus Reports' SVG foreignObject bug into
  a cross-challenge image read on the shared CDDC 2026 platform. Disclosed
  responsibly, withdrew from the event.
---

> {:.prompt-warning}
> **TL;DR.** Nexus Reports is a headless-Chromium challenge whose intended
> bug class is "SVG sanitiser bypass into Puppeteer DevTools." During the
> 2026-05-16 round, JS execution inside that Puppeteer Chromium turned out
> to be enough to reach the EC2 metadata service, steal the host's
> `ecsInstanceRole` STS credentials, and pull container images for **other**
> challenges from ECR. One of which carried its flag as a plain file inside
> the image :o. Reported to the organisers, withdrew from the CTF that same
> evening.

---

## The intended bug

Okay so Nexus Reports renders user-supplied SVG with Puppeteer plus
headless Chromium 147. The challenge's `sanitize.js` walks the SVG tree
but it has this subtle slip: when it hits a `foreignObject` element it
returns early *without recursing into descendants*. Smuggle an XHTML
`<body onload="…">` inside `<foreignObject>` and arbitrary JS runs in the
Puppeteer page at origin `about:blank`. The intended end-state is to use
that JS to talk to Chromium's own DevTools port (which is launched with
`--remote-debugging-port=9222 --remote-allow-origins=* --disable-web-security`,
generous to a fault) and read an admin API key off disk. That part is
totally fine, honestly it's a clean teaching-quality bug and props to
whoever wrote it :).

## The chain

What's *not* fine is what's reachable from "JS in the Puppeteer page."
Six steps, end-to-end in about an hour:

1. **JS execution.** `foreignObject` plus `<body onload>` inside the
   sanitiser-bypassing SVG, arbitrary script runs in the rendered page.
2. **Internal recon.** Synchronous XHRs from that page enumerate what's
   reachable. `http://169.254.169.254/` answers without a token (IMDSv1 is
   enabled and the hop limit isn't enforced), and the ECS agent's
   introspection port at `http://172.17.0.1:51678/` is open from inside
   the container. Already not great :o.
3. **Credentials.** `GET /latest/meta-data/iam/security-credentials/ecsInstanceRole`
   returns JSON with a live STS access key, secret, and session token for
   `arn:aws:sts::<account-id>:assumed-role/ecsInstanceRole/i-<redacted>`,
   valid for roughly six hours.
4. **Co-tenant enumeration.** `GET 172.17.0.1:51678/v1/tasks` lists every
   ECS task running on the same EC2 host, with each task's ECR image
   reference. So at this point I literally have the exact image names for
   the other challenges scheduled alongside Nexus Reports xD.
5. **Cross-challenge ECR.** `ecsInstanceRole` is allowed
   `ecr:GetAuthorizationToken`, `ecr:BatchGetImage`, and
   `ecr:GetDownloadUrlForLayer` on every `cddc2026-challs/*` repository,
   not just Nexus Reports. Pulling any other challenge's image is just a
   normal Docker registry interaction signed with the harvested STS creds.
6. **Flag as a file.** One of the other challenges ships its flag as
   `jail/flag` baked directly into the image layer. `tar -xzf` on the
   blob, `cat jail/flag`, done. Flag value: `CDDC2026{REDACTED}`. No
   exploitation of the *other* challenge's intended vulnerability was
   needed, the image itself is the source of truth.

## Why the chain works

Each link is individually reasonable. The chain only exists because they
compose. The two challenge-side issues (the `foreignObject` validator
returning without recursing, and Puppeteer's promiscuous DevTools flags)
are the **intended** teaching surface and should stay. The other four
are platform-side:

- **IMDSv1 with no hop-limit** on the EC2 host means a container can
  reach metadata at all.
- **A shared `ecsInstanceRole`** with ECR pull on the entire
  `cddc2026-challs/*` namespace turns one container's compromise into a
  cross-tenant image read.
- **The ECS agent introspection port** exposed on the docker bridge
  gateway hands out the list of co-tenants for free.
- **Flags packaged as plain files inside container images** means a
  pull *is* the solve.

Close any one of those four and step 6 doesn't fall out. The full
root-cause breakdown, indicators a defender can pivot from, and the
access-key prefix plus time window the organisers can match against
CloudTrail are in the private disclosure.

## Disclosure

I sent the full report (reproduction steps, identifiers, IAM principal,
CloudTrail pivots, and a remediation list) to the CDDC Support Team the
same day, and asked them to invalidate the points associated with the
recovered flag. The artefact is from a challenge I hadn't been working
on, and the chain isn't the intended solve for it. I stopped playing
the CTF after sending the report.

The Support Team got back to me really quickly. They investigated,
applied remediations, and confirmed from their ECR pull logs that no
other suspicious pulls were made against the affected repositories
during the competition. So the chain wasn't exploited by anyone else,
which was honestly the bit I was most nervous about :). They also chose
to **keep** the team's points for the affected challenge as
acknowledgement of the disclosure rather than invalidating them, which
was very classy of them.

![CDDC Support Team response on the disclosure ticket](/assets/img/posts/cddc-2026/disclosure-thread.png)
_CDDC Support Team's response on the disclosure ticket._

## Closing

I'm posting this in redacted form because the chain is a useful study
in how a clean, single-purpose challenge bug can get amplified into a
platform-level finding by the surrounding infra: IMDS, IAM scope, and
where flags physically live. The chain was automated end-to-end with a
Claude Code agent under my supervision. The analysis, the call to stop,
and the disclosure are mine. Big thanks to the CDDC 2026 organisers,
who have the full reproduction privately.
