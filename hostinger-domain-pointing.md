

# Why NOT Change Nameservers?

If Google Workspace emails are already configured (for example 100 company email accounts), changing nameservers directly to Hostinger can break:

* MX records
* SPF records
* DKIM
* DMARC
* Google verification

This can stop company email delivery.

Instead of changing nameservers, use:

```text
A Record DNS Pointing
```

This is safer and more professional.

---

# Architecture

Before configuration:

```text
Domain Registrar: ResellerClub
Hosting: Hostinger
Emails: Google Workspace
```

After configuration:

```text
Website Traffic → Hostinger
Emails → Google Workspace
DNS Management → ResellerClub
```

---

# Step 1: Get Hostinger Website IP Address

Login to Hostinger.

Go to:

```text
Websites
↓
Select Website
↓
Hosting Plan / Website Details
```

Find:

```text
Website IP Address
```

Example:

```text
178.16.xxx.xxx
```

This is the IP address where website traffic will point.

---

# Step 2: Open DNS Management in ResellerClub

Login to ResellerClub.

Go to:

```text
Domain List
↓
Select Domain
↓
DNS Management
```

You will see DNS records.

Important:

**Do not delete existing email records.**

Do NOT touch:

```text
MX
TXT
SPF
DKIM
DMARC
google-site-verification
```

These are required for Google Workspace emails.

---

# Step 3: Point Main Domain to Hostinger

Add or update an A Record.

Example configuration:

```text
Type: A
Host/Name: @
Value / Points To: 178.16.xxx.xxx
TTL: Default
```

Meaning:

```text
example.com
→ Hostinger website server
```

---

# Step 4: Configure WWW Version

Add or update a CNAME record.

Example:

```text
Type: CNAME
Host/Name: www
Value: @
TTL: Default
```

or

```text
Value: example.com
```

Meaning:

```text
www.example.com
→ example.com
```

---

# Step 5: Configure Subdomain (Optional)

If website is hosted on a subdomain such as:

```text
app1.example.com
```

Add:

```text
Type: A
Host/Name: app1
Value / Points To: 178.16.xxx.xxx
TTL: Default
```

Meaning:

```text
app1.example.com
→ Hostinger website
```

---

# Step 6: Save Changes

Save DNS settings.

Do not panic if website does not open immediately.

DNS propagation takes time.

Typical timing:

```text
5 minutes to 24 hours
```

Usually:

```text
15 minutes to 2 hours
```

---

# Step 7: Verify Website

Open:

```text
https://yourdomain.com
```

or

```text
https://app1.yourdomain.com
```

Website should load from Hostinger.

Emails should continue working normally because Google Workspace records were untouched.

---

# DNS Safety Checklist

Before clicking Save:

* Keep MX records untouched
* Keep SPF untouched
* Keep DKIM untouched
* Keep DMARC untouched
* Keep Google verification untouched
* Only edit website A record and WWW CNAME

---

# Final Result

Working setup:

```text
Domain Registrar → ResellerClub
Website Hosting → Hostinger
Emails → Google Workspace
DNS Management → ResellerClub
```

Result:

```text
GitHub CI/CD
→ Hostinger deploy
→ Domain points correctly
→ Emails remain safe
```
