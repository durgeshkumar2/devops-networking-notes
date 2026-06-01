# How Email Servers Actually Work Behind the Scenes
### *(The article that finally made it click for me)*

**Durgesh Kumar** — Software Engineer | Embedded Systems | Computer Vision | DevOps

*May 27, 2026*

---

I thought I understood the internet.

I knew how domains worked. I knew DNS. I knew what an A record was, what a CNAME did, how nameservers pointed your domain to your hosting. I had set up websites, configured servers, played with Linux. I felt comfortable.

Then someone asked me: "How does email actually work?"

And I realized — I had no idea.

Not really. Not the deep part. Not the part that explains why spoofing happens, why SPF exists, why enterprises spend millions running their own mail infrastructure, or why a single DNS misconfiguration can silently kill your entire company's email.

So I went down the rabbit hole. And this is what I found.

---

## PART 1: The Wild West Days of Email

Email is older than the modern internet. It was built in the 1970s on a network called ARPANET — a small, trusted community of researchers and academics. Nobody was worried about hackers or fraud. Everyone on the network basically knew each other.

So when SMTP — Simple Mail Transfer Protocol — was designed, it was built on pure trust.

The idea was simple: you connect to a mail server, you say who you are, you say who you're sending to, and you hand over the message. That's it. The server just... believes you.

There was no verification. No authentication. No "prove you are who you say you are."

Which meant anyone could open a connection to a mail server and type:

```
FROM: ceo@google.com
TO: employee@yourcompany.com
SUBJECT: Urgent - Wire Transfer Needed
```

And the server would happily deliver it.

This is called **email spoofing**. And for decades, it was completely trivial to do. You didn't need to hack anything. You didn't need special tools. The protocol itself allowed it by design.

This became one of the most exploited vulnerabilities in internet history. Phishing attacks. CEO fraud. Fake invoices. Impersonation attacks. All built on the same foundation: SMTP trusted everyone.

The internet eventually had to fix this. But fixing it wasn't simple — because you can't just change a protocol that billions of people are already using. You have to add layers on top of it.

Those layers are SPF, DKIM, and DMARC. But we'll get there.

First, let's understand how email actually works today.

---

## PART 2: The Anatomy of a Modern Email System

Let's say you work at a company called Acme Corp. Your domain is `acmecorp.com`. You have email addresses like `info@acmecorp.com` and `sales@acmecorp.com`.

Here's what the full picture looks like:

- **Domain** — `acmecorp.com`. You own this. It's registered with a registrar like GoDaddy or Namecheap. Everything revolves around this.

- **Mail Hosting** — Someone has to actually store and process your email. This could be Google Workspace, Microsoft 365, Zoho Mail, or your own server. This is where mailboxes live.

- **Mailboxes** — The actual email accounts. `info@acmecorp.com` is a mailbox. `sales@acmecorp.com` is a mailbox. Each one is essentially a storage folder on a server somewhere.

- **Mail Server** — The software/infrastructure that handles sending, receiving, and storing mail. When you use Google Workspace, Google's servers are your mail servers. If you self-host, you run this yourself.

- **SMTP (Simple Mail Transfer Protocol)** — The protocol used to **SEND** email. When you hit "Send," your mail client talks SMTP to your mail server. When your mail server delivers to another server, it's also using SMTP.

- **IMAP / POP3** — The protocols used to **RETRIEVE** email. When your mail app checks for new messages, it's using IMAP (or the older POP3). IMAP keeps mail on the server. POP3 downloads and removes it.

- **Webmail** — A browser-based interface to access your mailbox. Gmail is webmail. Outlook Web is webmail. You're just accessing your mailbox through a website instead of a desktop app.

- **Mail Clients** — Desktop or mobile apps like Outlook, Thunderbird, or Apple Mail. They connect to your mail server using IMAP/SMTP.

Now let's see how all of this actually moves.

---

## PART 3: How Email Travels Across the Internet

You open Gmail. You type an email to your client at `clientname@outlook.com`. You hit Send.

Here's exactly what happens:

```
Your Browser / Mail App
        |
        | (SMTP - port 587 or 465)
        ↓
  Your Mail Server
  (e.g., Google's SMTP server for Gmail)
        |
        | "Where do I deliver this?"
        | Performs DNS MX Lookup for outlook.com
        ↓
     DNS System
  Returns: "Send mail to outlook-com.mail.protection.outlook.com"
        |
        ↓
  Microsoft's Receiving Mail Server
  (Checks SPF, DKIM, DMARC)
        |
        ↓
  Recipient's Inbox
```

That's the full journey. Let's zoom into the most important step that most people never think about: the MX lookup.

---

## PART 4: MX Records — The Internet's Mail Routing System

When Google's mail server wants to deliver your email to `someone@outlook.com`, it doesn't just guess where to send it. It asks DNS.

Specifically, it looks up the **MX record** for `outlook.com`.

MX stands for **Mail Exchanger**. It's a DNS record that says: "If you want to send email to this domain, here's which server to talk to."

Every domain that wants to receive email must have at least one MX record. Without it, no one can send you email.

A real MX record looks like this:

```
Domain: acmecorp.com
MX Record:
  Priority: 10   mail.acmecorp.com
  Priority: 20   mail-backup.acmecorp.com
```

The priority number matters. **Lower number = higher priority.** Mail servers try the lowest number first. If that server is down, they try the next one. This is how basic redundancy works at the DNS level.

When you sign up for Google Workspace and point your domain to it, Google asks you to add their MX records to your DNS:

```
Priority: 1    aspmx.l.google.com
Priority: 5    alt1.aspmx.l.google.com
Priority: 10   alt2.aspmx.l.google.com
```

This tells the entire internet: "Mail for this domain goes to Google's servers."

One DNS change. That's it. That's how you redirect your entire company's email to a new provider.

But MX records only solve routing. They don't solve trust. They don't stop someone from spoofing your domain. That's where SPF comes in.

---

## PART 5: SPF — Teaching the Internet Who Can Send Your Mail

It's 2003. Email spoofing is rampant. Anyone can send email claiming to be from `ceo@google.com` or `noreply@paypal.com`. Phishing is exploding.

A group of engineers had a simple idea: What if a domain could publish a list of servers that are allowed to send email on its behalf?

That's **SPF. Sender Policy Framework.**

An SPF record is a DNS TXT record on your domain that says: "These are the only servers authorized to send email from `@acmecorp.com`."

It looks like this:

```
acmecorp.com  TXT  "v=spf1 include:_spf.google.com ~all"
```

Breaking this down:

- `v=spf1` — This is an SPF record
- `include:_spf.google.com` — Google's servers are allowed to send for us
- `~all` — Any other server is "soft fail" (suspicious but not hard reject)

When an email arrives claiming to be from `info@acmecorp.com`, the receiving mail server does this:

1. Looks up the SPF record for `acmecorp.com`
2. Gets the list of authorized sending servers
3. Checks: did this email come from one of those servers?
4. If yes → **passes SPF**. If no → **fails SPF**.

If a scammer tries to send email claiming to be from your domain, from their own random server, SPF check will fail. The receiving server can then decide what to do with it.

But SPF had a limitation. It checks the server IP — but it doesn't check whether the message content was tampered with in transit. That's where DKIM comes in.

---

## PART 6: DKIM — Putting a Tamper-Proof Seal on Every Email

Here's a scary thought: email travels across multiple servers before it reaches the recipient. What's stopping someone in the middle from modifying the content?

Nothing, actually. Without DKIM, someone with the right network access could intercept your email and change "Please send $500" to "Please send $50,000" before it reaches the recipient. The recipient would have no way to know.

**DKIM — DomainKeys Identified Mail** — solves this using cryptographic signatures.

Here's how it works:

Your mail server has a **private key** (kept secret). A corresponding **public key** is published in your DNS as a TXT record.

When you send an email, your mail server:
1. Takes a hash of the email headers and body
2. Encrypts that hash with the private key
3. Attaches the encrypted signature to the email header

When the receiving server gets the email:
1. It looks up your public key from DNS
2. It decrypts the signature using the public key
3. It independently computes the hash of the headers and body
4. It compares: does the hash it computed match the hash in the signature?

If **yes** → the email was not tampered with, and it genuinely came from a server that has your private key.

If **no** → the email was either modified in transit, or it's fake.

A DKIM DNS record looks like this:

```
google._domainkey.acmecorp.com  TXT  "v=DKIM1; k=rsa; p=MIGfMA0GCS..."
```

The `"google"` part is the **selector** — it lets you have multiple DKIM keys for different services.

So now we have SPF verifying the sending server, and DKIM verifying the message integrity. Should be enough, right?

Almost. There was still one problem left.

---

## PART 7: DMARC — The Policy Layer That Ties It All Together

SPF and DKIM are great. But they had no coordination. They just produced pass/fail results. What was the receiving server supposed to DO with those results?

Some servers quarantined suspicious mail. Some rejected it. Some let it through anyway. There was no standard. So attackers just tried different techniques until something worked.

**DMARC — Domain-based Message Authentication, Reporting and Conformance** — was the missing piece.

DMARC does two things:

1. It tells receiving servers exactly what to do when SPF or DKIM fails.
2. It sends you reports so you can see what's happening to your domain's email.

A DMARC record looks like this:

```
_dmarc.acmecorp.com  TXT  "v=DMARC1; p=reject; rua=mailto:dmarc@acmecorp.com"
```

The `"p="` is the **policy**. Three options:

- `p=none` — Monitor only. Don't take action. Just send me reports. *(Good for when you're first setting up — you want to watch before you act.)*
- `p=quarantine` — If DMARC fails, put the email in spam/junk.
- `p=reject` — If DMARC fails, reject the email entirely. Do not deliver it.

If `acmecorp.com` has `p=reject`, then anyone trying to spoof `@acmecorp.com` will have their email silently rejected by every major mail provider. The attack is neutralized.

This is the full trust stack:

```
SPF   → Verifies: Did this come from an authorized server?
DKIM  → Verifies: Was this message tampered with?
DMARC → Policy:   What do we do if either check fails?
```

Together, these three records transformed email from an honor-system protocol into a verifiable, accountable communication system.

---

## PART 8: DNS — The Silent Control Center of Email

By now you've noticed something: almost every email security mechanism we discussed lives in DNS.

| Record | Purpose |
|--------|---------|
| MX records | Where to deliver email |
| SPF records | Who can send email |
| DKIM records | Public key for signature verification |
| DMARC records | What to do when things fail |

DNS isn't just for websites. For email, DNS is literally the control center. If someone gets control of your DNS, they can:

- Redirect all incoming mail to their own server
- Remove SPF and DKIM records, allowing spoofing
- Change MX records to intercept your email

This is why DNS security — DNSSEC, locked registrar accounts, two-factor authentication on your registrar — is **critical email security**.

One DNS change can break your entire email system. One DNS change can fix it. It's the single most powerful lever in email infrastructure.

---

## PART 9: Cloud Mail vs. Self-Hosted — The Big Decision

Every organization eventually faces this question: Do we use a cloud mail provider, or do we run our own mail servers?

Let's look at both sides honestly.

### The Cloud Side: Google Workspace, Microsoft 365, Zoho

When a company uses Google Workspace, here's what Google handles for them:

- Physical servers in multiple data centers
- 99.9%+ uptime SLAs
- Spam and malware filtering
- Automatic updates and security patches
- Terabytes of storage
- Integrated calendar, video, documents, chat
- Mobile apps, webmail, API access
- SPF, DKIM, DMARC configuration support
- Compliance tools and audit logs

The cost? Around **$6 to $18 per user per month** depending on the tier.

For a 50-person company, that's $300 to $900 per month. In exchange, you have zero servers to maintain, zero downtime events to handle, and a team of thousands of Google engineers keeping your email running.

For 99% of businesses — startups, SMBs, even large companies — this is the obvious, correct choice.

### The Self-Hosted Side: Why Some Organizations Run Their Own Mail

Then there are organizations where cloud is not an option.

Defence agencies. Intelligence services. Banks in certain jurisdictions. Healthcare systems handling classified patient data. Government ministries. Large enterprises in heavily regulated industries.

Their reasons are legitimate and serious:

- **Data Sovereignty** — Their emails cannot physically sit on servers owned by an American company in Virginia. Legal, regulatory, or national security requirements demand data stays within their own infrastructure or even their own buildings.

- **Air-Gapped Networks** — Some organizations operate networks that are completely disconnected from the internet. A defence system that processes classified communications cannot use Gmail. Period. Mail must run entirely on internal infrastructure.

- **Full Control** — In a cloud setup, you're trusting the provider. You're trusting Google won't read your emails, won't be compelled to hand over data, won't have a breach. Some organizations can't take that trust risk.

- **Regulatory Compliance** — GDPR, HIPAA, local data protection laws, government security classifications — some of these legally prohibit storing sensitive communications on third-party infrastructure.

For these organizations, self-hosting isn't a preference. It's a requirement.

---

## PART 10: What Running Your Own Mail Server Actually Requires

This is where people learn how complex real infrastructure is.

Running a production mail server for an enterprise isn't "install Postfix on a Linux box." That might work for a hobby project. For an organization with 500 employees depending on email for their operations, it looks something like this:

- **Hardware / Servers:** Multiple physical or virtual Linux servers. At minimum: primary mail server, secondary/backup mail server, spam filter server, possibly a dedicated archiving server.

- **Mail Server Software:** Postfix (SMTP), Dovecot (IMAP), plus additional software for webmail (Roundcube, SOGo), spam filtering (SpamAssassin, Rspamd), virus scanning (ClamAV), DKIM signing, and management interfaces.

- **Storage Infrastructure:** Mail data needs reliable, fast storage. Enterprise organizations typically use RAID arrays — multiple disks working together so that if one disk dies, no data is lost. RAID 1 mirrors data across two disks. RAID 5/6 distributes data with parity across several disks. RAID 10 combines mirroring and striping for both performance and redundancy.

- **Backups:** This is where most people confuse three different concepts. Let me explain them clearly.

---

## PART 11: Backup vs. Snapshot vs. Redundancy

These three terms get mixed up constantly. They are completely different things.

**Redundancy** is about uptime. It means having duplicate systems running simultaneously, so if one fails, another takes over instantly.

> *Example: Two mail servers running in parallel. Server 1 handles 50% of connections, Server 2 handles the other 50%. If Server 1 crashes at 2 PM on a Tuesday, Server 2 instantly handles everything. No downtime. No one notices. This is redundancy.*

**Snapshot** is a point-in-time copy of a system's state, taken quickly, stored on the same infrastructure.

> *Example: At 9 AM, a snapshot is taken of your mail server's disk. At 11 AM, a bad software update corrupts your mailbox database. You roll back to the 9 AM snapshot and you're restored in minutes — losing only 2 hours of changes. This is fast recovery from system failures.*

**Backup** is a copy of your data stored in a separate, physically different location, taken on a regular schedule.

> *Example: Every night at midnight, all mailbox data is copied to an external backup server in a different building (or a different city). If your entire data center catches fire, you restore from last night's backup. You lose 24 hours of data in the worst case, but you didn't lose everything. This is disaster recovery.*

| | Redundancy | Snapshot | Backup |
|---|---|---|---|
| **Purpose** | Uptime | Fast rollback | Disaster recovery |
| **Location** | Parallel systems | Same infrastructure | Separate location |
| **Recovery time** | Instant | Minutes | Hours |

**Redundancy keeps you running. Snapshots let you roll back quickly. Backups save you when everything else fails.**

An enterprise mail system needs all three.

---

## PART 12: The Full Infrastructure Picture

Here's what a complete self-hosted enterprise mail infrastructure looks like:

```
                       INTERNET
                           |
              [ Firewall / DDoS Protection ]
                           |
               [ Load Balancer / HA Proxy ]
                  /                    \
        [ Primary Mail Server ]    [ Secondary Mail Server ]
        (Postfix + Dovecot)        (Hot Standby)
                  |
        [ Spam Filter Layer ]
        (Rspamd / SpamAssassin)
                  |
        [ Virus Scanner ]
        (ClamAV)
                  |
        [ Mail Storage - RAID Array ]
                  |
        [ Backup Server - nightly sync ]
                  |
        [ Archiving Server - compliance ]
```

**Supporting Infrastructure:**
- Monitoring: Grafana, Prometheus, Zabbix
- Log Management: ELK Stack
- DNS: Internal DNS + External authoritative DNS
- Certificate Management: SSL/TLS certs, renewal
- DKIM key management
- Patch management
- Incident response procedures

And who manages all of this?

A dedicated team. System administrators. DevOps engineers. Network engineers. Security engineers. A team that is on-call, that handles incidents at 3 AM, that plans capacity, that manages upgrades, that monitors for threats, that writes and tests disaster recovery procedures.

This is why small and medium companies don't self-host. Not because they couldn't technically figure it out — but because the ongoing operational cost and complexity is enormous. The talent required is expensive. The risk of something going wrong, and going wrong badly, is real.

Cloud providers have armies of engineers solving these problems at scale. For most organizations, it makes no sense to replicate that internally.

But for a defence ministry protecting classified communications? The cost is justified. The alternative — trusting an external provider — is unacceptable.

---

## PART 13: Key Lessons From This Rabbit Hole

After going through all of this, here's what I think actually matters:

**1. Email was never designed to be secure.** It was designed for trust. Every security mechanism we use today — SPF, DKIM, DMARC — is a layer built on top of a 50-year-old protocol that assumed everyone was honest. That's a fragile foundation, and it explains why email fraud is still one of the most common attack vectors.

**2. DNS is everything.** You cannot understand email infrastructure without deeply understanding DNS. MX records route mail. SPF lives in DNS. DKIM public keys live in DNS. DMARC policies live in DNS. If your DNS is wrong, broken, or compromised, your email is broken or compromised. Full stop.

**3. SPF + DKIM + DMARC together is non-negotiable.** If you manage a domain and you don't have all three configured and tested, you are vulnerable. Your domain can be spoofed. Your users can be phished in your name. Configure them. Test them. Set DMARC to reject.

**4. Cloud vs. self-hosted is a real architectural decision.** It's not just about cost. It's about control, compliance, risk tolerance, and operational capability. For most organizations, cloud is the right answer. For some, it's not. Understanding why helps you think clearly about infrastructure tradeoffs in general.

**5. "Simple" infrastructure is never simple at scale.** Email looks simple from the outside — you send a message, someone receives it. But underneath: cryptographic signing, multi-server delivery, DNS propagation, spam filtering, disaster recovery, high availability, security monitoring. Every "simple" service you use is hiding enormous complexity.

---

## Why This Matters for Infrastructure Engineers

If you're building a career in DevOps, system administration, cloud engineering, or any kind of infrastructure role — email infrastructure is a microcosm of everything you'll deal with professionally.

DNS management. Security protocols. High availability design. Backup and recovery strategy. Linux server administration. Monitoring and observability. Regulatory compliance. Cost-benefit analysis of build vs. buy.

**Email touches all of it.**

Understanding how it works doesn't just make you better at email. It makes you better at thinking about how complex distributed systems are designed, how trust is established across the internet, and how modern infrastructure handles the gap between "working in theory" and "reliable under real-world conditions."

The internet runs on protocols that were built decades ago, patched with layers of security, operated by millions of systems that have to agree on conventions they were never originally designed for. Email is one of the clearest examples of this.

And once you understand it — really understand it — you start seeing the same patterns everywhere.