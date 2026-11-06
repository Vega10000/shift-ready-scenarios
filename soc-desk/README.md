# A Day in the Life: SOC Analyst (Tier 1)

![Shift Ready: SOC Desk](https://img.shields.io/badge/Shift%20Ready-SOC%20Desk-1F6F78?style=for-the-badge) ![Scenes: 8 of 8](https://img.shields.io/badge/Scenes-8%20of%208-3D4A5C?style=for-the-badge) ![Status: Complete](https://img.shields.io/badge/Status-Complete-2F7A3E?style=for-the-badge)

> Interview-prep scenarios from **Shift Ready**, a scenario-based practice game. Each scene is a realistic moment from a SOC Analyst (Tier 1)'s shift at a fictional company: the situation, the call I'd make, and why. This is structured practice, not a real incident.

## The shift at a glance

| Time | Scene | Key lesson |
|---|---|---|
| 08:00 | Shift handoff | Triage order is about risk, not deadlines. |
| 09:10 | The brute-force alert | The classic gotcha: raw failure counts flag harmless noise. |
| 09:35 | What can 'deploy' do? | After a confirmed login, check three things: groups (id), exact admin rights (sudo -l), and whether those rights were used (sudo lines in auth.log). |
| 09:55 | Did web-03 phone home? | Read outbound traffic three ways: who (destination reputation and age), how often (many small, regular connections = beaconing), and how much (large bytes_out = possible exfiltration). |
| 10:30 | Escalate or close? | Lead with severity, then what happened, impact, what you already checked, and what you recommend. |
| 12:40 | Phishing report from a client | Phishing triage is about scope: received, clicked, entered credentials. |
| 14:15 | Vulnerability scan results | Prioritize by severity, exposure and threat activity together. |
| 16:45 | Your handoff | Good handoffs are short and complete: each item has a status, a time and the next action. |

## Skills practiced

- Alert triage and prioritization by risk
- Confirming a breach in Splunk: failed vs. successful logins
- Post-compromise checks: account rights, persistence, outbound traffic
- Writing a clear escalation for Tier 2
- Scoping a phishing report
- Prioritizing vulnerabilities by severity and exposure
- Shift handoffs

## Scene by scene

### 08:00 · Shift handoff

**Situation:** You log in at a managed detection and response (MDR) provider. The overnight analyst left a handoff note. Before touching the queue, you need to know what is still open.

```text
HANDOFF 07:55  overnight -> day
- INC-4412  phishing click, user jlee  STATUS: contained, awaiting reimage
- INC-4415  impossible travel, svc-backup  STATUS: open, NOT investigated
- Splunk fwd on DC02 dropped at 03:10, restored 03:40
- Client ACME asked for weekly summary by noon
```

**The question:** What do you look at first?

**My call:** INC-4415. It is open, not investigated, and involves a service account.

**Why:** Triage order is about risk, not deadlines. Open beats contained, privileged beats normal, and a logging gap is context you carry into the investigation.

**How I'd explain it in an interview:** At the start of a shift I read the handoff and sort by risk: anything open and uninvestigated on a privileged account goes first. I also note any logging gaps, because they change how much I can trust what I see.

### 09:10 · The brute-force alert

**Situation:** An alert fires on a Linux server. This looks a lot like the SSH brute-force investigation I wrote up in my home lab.

```text
index=linux sourcetype=linux_secure "Failed password" earliest=-60m
| stats count by src_ip, user
| sort -count

src_ip          user      count
203.0.113.45    root       812
203.0.113.45    admin      344
198.51.100.7    jdavis       3

index=linux sourcetype=linux_secure "Accepted password" src_ip=203.0.113.45
> 1 event  09:04:51  user=deploy
```

**The question:** What is the most important line on this screen?

**My call:** The single "Accepted password" for user deploy from the same IP.

**Why:** The classic gotcha: raw failure counts flag harmless noise. The signal is a success from the same IP after the failures. Escalate, and recommend disabling the deploy account and blocking the IP.

**How I'd explain it in an interview:** In my home lab I investigated an SSH brute-force in Splunk. I counted failed logins by source IP and user, then checked whether any of those IPs later logged in successfully. A success after a burst of failures is what turns noise into an incident.

### 09:35 · What can 'deploy' do?

**Situation:** The attacker logged in as deploy at 09:04:51. Before you escalate, you need to know what that account is allowed to do on web-03, and whether the attacker already used those rights. You run three checks on the host.

```text
$ id deploy
uid=1002(deploy) gid=1002(deploy) groups=1002(deploy),27(sudo),999(docker)

$ sudo -l -U deploy
User deploy may run the following commands on web-03:
    (ALL : ALL) NOPASSWD: ALL

$ grep sudo /var/log/auth.log | grep deploy
09:06:12 sudo: deploy : COMMAND=/usr/sbin/useradd -m -s /bin/bash sysupdate
09:06:40 sudo: deploy : COMMAND=/usr/bin/tee -a /home/sysupdate/.ssh/authorized_keys
09:07:05 sudo: deploy : COMMAND=/usr/bin/crontab -e
```

**The question:** What do these results tell you?

**My call:** deploy has full root rights with no password, and the attacker already used them: a new account (sysupdate), an SSH key for it, and a cron edit. That is persistence. Severity goes up.

**Why:** After a confirmed login, check three things: groups (id), exact admin rights (sudo -l), and whether those rights were used (sudo lines in auth.log). New accounts, added SSH keys and cron edits are persistence: ways back in after you change the password. Disabling deploy alone is no longer enough; sysupdate and its key must go too. On Windows the same questions are answered with whoami /groups, net localgroup administrators, and events 4672, 4720 and 4732.

**How I'd explain it in an interview:** After a successful login I check what the account can do and whether it used it: groups, sudo rights, and the sudo entries in the auth log. On Windows I'd look at event 4672 for admin logons and 4720 or 4732 for new accounts and group changes. Anything like a new user or an added SSH key tells me the attacker set up persistence.

### 09:55 · Did web-03 phone home?

**Situation:** Now the second question: since 09:04:51, has web-03 (10.0.5.23) been connecting to anything outside the company? You search the firewall logs, filtering out internal traffic.

```text
index=firewall src_ip=10.0.5.23 earliest="10/31/2026:09:04:51"
| where NOT cidrmatch("10.0.0.0/8", dest_ip)
| stats count, sum(bytes_out) as bytes_out by dest_ip, dest_port
| sort -bytes_out

dest_ip          dest_port  count   bytes_out
198.51.100.200   443           41   1,842,330,112
91.92.250.17     4444         388          96,204
151.101.1.69     443            6          18,450
185.125.190.36   80             3           4,102

# reputation: 151.101.1.69 = Fastly CDN   185.125.190.36 = Ubuntu updates
#             91.92.250.17 = no history    198.51.100.200 = registered 6 days ago
```

**The question:** Which destinations do you flag?

**My call:** 91.92.250.17 on port 4444 and 198.51.100.200 on 443.

**Why:** Read outbound traffic three ways: who (destination reputation and age), how often (many small, regular connections = beaconing), and how much (large bytes_out = possible exfiltration). Rule out known-good destinations like CDNs and update servers by checking reputation, not by guessing. Also check DNS logs for lookups of odd domains. With active C2 and possible data theft, the recommendation becomes: isolate web-03 from the network now.

**How I'd explain it in an interview:** For outbound activity I pull firewall, proxy and DNS logs for the host from the time of the login, filter out internal traffic, and look at destination reputation, connection frequency and bytes sent. Many small regular connections suggest a beacon to a control server; a large upload to a new destination suggests exfiltration. Either one means I recommend isolating the host.

### 10:30 · Escalate or close?

**Situation:** You are Tier 1. Your runbook says: close false positives with notes; escalate anything with evidence of compromise to Tier 2 within 15 minutes.

**The question:** Put your escalation note in the right order.

**My call:**

1. Severity: Critical. Escalating to Tier 2 per runbook (15-minute SLA)
2. What happened: 1,156 failed SSH logins from 203.0.113.45, then 1 successful login as 'deploy' at 09:04:51
3. Impact so far: deploy has passwordless root on web-03; attacker created user 'sysupdate', added an SSH key and edited cron; host is beaconing to 91.92.250.17:4444 and sent 1.8 GB to 198.51.100.200
4. What I did: confirmed the login in auth logs, reviewed sudo rights and sudo history, pulled firewall logs since 09:04:51, checked reputation on each external IP
5. Recommended next step: isolate web-03 from the network, disable 'deploy' and 'sysupdate', block all three external IPs, start data-loss review

**Why:** Lead with severity, then what happened, impact, what you already checked, and what you recommend. Tier 2 should not have to re-do your work. Clear, short, evidence first.

**How I'd explain it in an interview:** When I escalate, I lead with severity, then the facts, what I already checked, and what I recommend. In my apprenticeship I documented findings for a professional audience, so I'm used to writing for someone who has to act on it.

### 12:40 · Phishing report from a client

**Situation:** A user at a client forwards an email: "Your Microsoft 365 password expires today. Keep current password: [link]". The link goes to micros0ft-login-secure.com.

**The question:** Which is the strongest set of first actions?

**My call:** Search mail logs for the same sender and subject, find who else received it, check proxy logs for clicks to that domain, and request the domain be blocked.

**Why:** Phishing triage is about scope: received, clicked, entered credentials. Each answer leads to a different action. Reporting users are your best sensor, so thank them.

**How I'd explain it in an interview:** For a reported phish I look for scope: who else got it, who clicked, and whether anyone entered credentials. Then contain by blocking the domain and pulling the message from inboxes.

### 14:15 · Vulnerability scan results

**Situation:** Your team lead asks you to review this week's scan summary before the patch meeting.

```text
Host        CVE / Finding                          CVSS  Internet-facing
web-03      OpenSSH outdated (known RCE)          9.8   yes
hr-app      TLS 1.0 enabled                        5.3   no
print-02    SNMP default community string          7.5   no
laptop-118  Missing browser update                 8.1   no
```

**The question:** Which do you put at the top of the patch list?

**My call:** web-03, critical and internet-facing (and it was just brute-forced).

**Why:** Prioritize by severity, exposure and threat activity together. It's the same logic I used in my Nessus home-lab assessments.

**How I'd explain it in an interview:** I don't just sort by CVSS. I weigh severity, whether the system is internet-facing, and whether we're seeing active attacks against it. In my lab I ran Nessus and Nmap and wrote up findings that way.

### 16:45 · Your handoff

**Situation:** End of shift. The evening analyst is taking over.

**The question:** Which handoff note is best?

**My call:** "INC-4415: escalated to T2 at 10:41, web-03 isolated, deploy and sysupdate disabled, data-loss review in progress with T2. INC-4412: reimage confirmed 13:20, closed. DC02 forwarder stable since 03:40. ACME summary sent 11:50."

**Why:** Good handoffs are short and complete: each item has a status, a time and the next action.

**How I'd explain it in an interview:** I write handoffs so the next person can act without asking me anything: ticket, status, time, next step.

## Reflections

**Which scene surprised you most, and what did it teach you?**

The scene that surprised me most is the importance of a good hand-off.  Good hand-offs are short and complete: each item has a status, a time and the next action.  I learned to write my hand-offs in a way that the next person can act on without having to ask me anything, leaving no gaps that would cause them to change how they trust what they see and have to redo the work for lack of completeness.

**Pick one scene and explain the right call in your own words, the way you'd say it in an interview.**

Escalate or close?  When assessing and documenting the reasoning behind the decision of whether to escalate or close a security threat, the right call is to lead with severity, then with the facts, assess the impact,  then state what I already checked, and use this as the basis of my recommendation for next steps.

**How would you use what you practiced here on day one of a real SOC Analyst (Tier 1) job?**

On day one of a SOC Analyst Tier 1 job, I would use what I practiced here to be able to find a security breach, determine what an attacker can do with permissions and groups and whether they have taken any further action, such as having setup persistence, made outbound connections and to what extent, i.e. frequency and volume.

## Next steps

- Practice these answers out loud with a 2-minute timer.
- Back each scene with hands-on lab evidence where possible (home lab, Splunk, TryHackMe).
