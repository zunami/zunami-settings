# Tailscale DNS Issues After Switching from ISC DHCP to Dnsmasq

## Problem Description

After switching from **ISC DHCP v4** to **Dnsmasq** in OPNsense, Tailscale connections work perfectly **outside the LAN** but fail **inside the LAN**. Local servers that were previously accessible via Tailscale become unreachable from devices within the home network.

### Symptoms
- ✅ Tailscale works perfectly when outside the home network
- ❌ Tailscale connections fail when inside the LAN
- ❌ Local servers (NAS, Docker containers, etc.) not reachable via Tailscale hostnames from LAN devices
- ✅ Direct IP connections still work

### Environment
- **Firewall:** OPNsense with Dnsmasq, Unbound DNS, and AdGuard Home
- **Network:** Home lab with multiple servers and Docker containers
- **VPN:** Tailscale for remote access
- **DNS Setup:** Dnsmasq → Unbound → AdGuard Home → Public DNS

---

## Root Cause

The switch from ISC DHCP to Dnsmasq changed how **local DNS resolution** works. Dnsmasq now handles both DHCP and local DNS, but it doesn't know how to forward Tailscale-specific DNS queries (`.ts.net` domain) to the Tailscale Magic DNS server.

### DNS Chain Before (ISC DHCP):
```
Device → ISC DHCP (hostnames) → Unbound → AdGuard → Public DNS
         ✅ Tailscale names worked
```

### DNS Chain After (Dnsmasq):
```
Device → Dnsmasq → Unbound → AdGuard → Public DNS
         ❌ Tailscale names not forwarded correctly
```

---

## Solution Overview

We need to configure **Split-DNS** at three levels:
1. **Tailscale Admin Console** - Tell Tailscale which DNS to use for local domains
2. **Dnsmasq (OPNsense)** - Forward `.ts.net` queries to Tailscale Magic DNS
3. **AdGuard Home** - Forward `.ts.net` queries to Tailscale Magic DNS

---

## Step-by-Step Fix

### 1. Configure Split-DNS in Tailscale Admin Console

#### 1.1 Access Tailscale Admin Console
1. Go to https://login.tailscale.com
2. Log in with your Tailscale account
3. Click on **DNS** (top navigation)

#### 1.2 Configure Local DNS Nameserver
1. Scroll to **"Global nameservers"** or **"Nameservers"**
2. Find your custom nameserver (your OPNsense/firewall IP, e.g., `192.168.1.1`)
3. Click the **three dots (•••)** next to the IP
4. Select **"Edit nameserver"**
5. Enable **"Restrict to domain"** toggle (should turn blue)
6. In the **"Domain"** field, enter: `yourdomain.local` (replace with your local domain)
7. **Disable** "Use with exit node" (should be gray/off)
8. Click **"Save"**

**Result:** Your local DNS will now only be used for queries ending with `.yourdomain.local`

#### 1.3 Verify MagicDNS is Enabled
1. In the same DNS tab, check if **MagicDNS** is enabled
2. You should see a button that says **"Disable MagicDNS..."** (meaning it's currently ON)
3. If it says **"Enable MagicDNS"**, click it to enable

#### 1.4 Clean Up Search Domains
1. Scroll down to **"Search Domains"**
2. Remove any invalid entries:
   - ❌ Remove IP addresses (e.g., `192.168.1.1`)
   - ❌ Remove overly specific machine names (e.g., `firewall.tail12345.ts.net`)
3. Keep only:
   - ✅ Your Tailscale network domain (e.g., `tail12345.ts.net`)
   - ✅ Your local domain (e.g., `yourdomain.local`)

---

### 2. Configure Dnsmasq in OPNsense

#### 2.1 Add Tailscale Domain Forwarding
1. Log into **OPNsense**
2. Go to **Services → Dnsmasq DNS → Domains**
3. Click the **[+]** button (Add)
4. Fill in the fields:
   - **Sequence:** `1`
   - **Domain:** `ts.net` (without leading dot!)
   - **IP Address:** `100.100.100.100` (Tailscale Magic DNS)
   - **Port:** *(leave empty)*
   - **Firewall Alias:** *(leave empty)*
   - **Description:** `Tailscale Magic DNS`
5. Click **"Save"**
6. Click **"Apply"** (green play button at top right)

#### 2.2 Add Local Domain Forwarding (Optional but Recommended)
1. Still in **Services → Dnsmasq DNS → Domains**
2. Click **[+]** again
3. Fill in:
   - **Sequence:** `2`
   - **Domain:** `yourdomain.local` (your local domain)
   - **IP Address:** `192.168.1.1` (your OPNsense/firewall IP)
   - **Description:** `Local Domain (Unbound/AdGuard)`
4. Click **"Save"** and **"Apply"**

#### 2.3 Restart Dnsmasq Service
1. Go to **Services → Dnsmasq DNS → Settings**
2. Click **"Restart Service"**

---

### 3. Configure AdGuard Home

#### 3.1 Access AdGuard Home
1. Open AdGuard Home (usually at `http://your-firewall-ip` or through OPNsense link)
2. Go to **Settings → DNS Settings**

#### 3.2 Add Upstream DNS Forwarders
1. Scroll to **"Upstream DNS servers"**
2. Click in the text field
3. **At the very top** of the list, add these two lines:

```
[/ts.net/]100.100.100.100
[/yourdomain.local/]192.168.1.1:5353
```

**Important Notes:**
- Replace `yourdomain.local` with your actual local domain
- Replace `192.168.1.1` with your OPNsense IP
- The `:5353` port is for Unbound DNS (check your Unbound configuration if unsure)
- These lines **MUST** be at the top, before any other DNS servers
- Do **NOT** add a line like `192.168.1.1:5353` without square brackets (this causes DNS loops!)

#### 3.3 Example Configuration
Your **Upstream DNS servers** should look like this:

```
[/ts.net/]100.100.100.100
[/yourdomain.local/]192.168.1.1:5353
https://dns.quad9.net:443/dns-query
tls://dns.quad9.net:853
https://security.cloudflare-dns.com:443/dns-query
tls://security.cloudflare-dns.com:853
... (other public DNS servers)
```

#### 3.4 Optional: Configure Private Reverse DNS
1. Scroll to **"Private reverse DNS resolvers"** (if available)
2. Add these lines:

```
[/100.in-addr.arpa/]100.100.100.100
[/1.168.192.in-addr.arpa/]192.168.1.1:5353
```

This allows reverse DNS lookups for Tailscale IPs and local IPs.

#### 3.5 Save and Clear Cache
1. Scroll to the bottom
2. Click **"Save"** (green button)
3. Scroll further down to **"DNS cache"**
4. Click **"Clear DNS cache"**

---

### 4. Optional: Firewall Rules (If Still Not Working)

If Tailscale still doesn't work after the above steps, add a firewall rule in OPNsense:

1. Go to **Firewall → Rules → LAN**
2. Click **[+]** (Add)
3. Configure:
   - **Action:** `Pass`
   - **Interface:** `LAN`
   - **Protocol:** `any`
   - **Source:** `LAN net`
   - **Destination:** Custom → `100.64.0.0/10` (Tailscale IP range)
   - **Description:** `Allow LAN to Tailscale`
4. Click **"Save"** and **"Apply Changes"**

---

## Testing

### Test 1: Tailscale Domain Resolution
From a device **inside your LAN** with Tailscale running:

```bash
nslookup yourserver.ts.net
```

**Expected Result:**
```
Server:  100.100.100.100
Address: 100.100.100.100#53

Name:    yourserver.ts.net
Address: 100.x.x.x
```

### Test 2: Local Domain Resolution
```bash
nslookup yourserver.yourdomain.local
```

**Expected Result:**
```
Server:  192.168.1.1
Address: 192.168.1.1#53

Name:    yourserver.yourdomain.local
Address: 192.168.1.x
```

### Test 3: Ping Test
```bash
ping yourserver.ts.net
```

**Expected Result:** You should see ping replies from the Tailscale IP (`100.x.x.x`)

### Test 4: Check AdGuard Query Log
1. Go to AdGuard Home → **Query Log**
2. Search for entries like `yourserver.ts.net`
3. You should see:
   - **Status:** "Forwarded"
   - **Upstream:** `100.100.100.100`

---

## How It Works

### Before the Fix (Problem):
```
LAN Device asks: "What is mynas.ts.net?"
  ↓
Dnsmasq: "I don't know this domain" → forwards to Unbound
  ↓
Unbound: "I don't know either" → forwards to AdGuard Home
  ↓
AdGuard Home: "Let me ask public DNS"
  ↓
Public DNS (Quad9/Cloudflare): "Unknown domain"
  ↓
❌ Result: "Server not found"
```

### After the Fix (Working):
```
LAN Device asks: "What is mynas.ts.net?"
  ↓
Dnsmasq: "Ah, .ts.net! I forward this to 100.100.100.100"
  ↓
Tailscale Magic DNS: "That's 100.x.x.x"
  ↓
✅ Result: Connection successful!
```

---

## Common Mistakes to Avoid

### ❌ Wrong: Adding Unbound as default upstream in AdGuard
```
192.168.1.1:5353
[/ts.net/]100.100.100.100
```
**Problem:** This creates a DNS loop (AdGuard → Unbound → AdGuard → ...)

### ✅ Correct: Using Split-DNS with square brackets
```
[/ts.net/]100.100.100.100
[/yourdomain.local/]192.168.1.1:5353
```
**Why:** Square brackets restrict the DNS server to specific domains only.

### ❌ Wrong: Using a leading dot in Dnsmasq domain
```
Domain: .ts.net
```

### ✅ Correct: No leading dot
```
Domain: ts.net
```

---

## Troubleshooting

### Issue: Still no connection after configuration

**Solution 1: Restart Services**
1. Restart Dnsmasq (OPNsense)
2. Restart AdGuard Home
3. Restart Tailscale on the client device

**Solution 2: Check DNS Chain**
Run this on a LAN device:
```bash
nslookup yourserver.ts.net 192.168.1.1
```
This forces the query through your local DNS.

**Solution 3: Check Tailscale Status**
```bash
tailscale status
```
Verify all devices show up with their `100.x.x.x` IPs.

### Issue: DNS loop (slow or failing queries)

**Cause:** Unbound is forwarding to AdGuard, and AdGuard is forwarding back to Unbound.

**Solution:** Check your **AdGuard Upstream DNS** configuration:
- Ensure the first line is NOT just `192.168.1.1:5353` without square brackets
- Use `[/yourdomain.local/]192.168.1.1:5353` instead

### Issue: Tailscale works, but local domains don't

**Solution:** Verify Unbound has the correct **Host Overrides** or **Domain Overrides** configured for your local domains.

---

## Additional Resources

- [Tailscale Split DNS Documentation](https://tailscale.com/kb/1054/dns)
- [Dnsmasq Manual](https://thekelleys.org.uk/dnsmasq/docs/dnsmasq-man.html)
- [AdGuard Home DNS Configuration](https://github.com/AdguardTeam/AdGuardHome/wiki/Configuration#upstream)

---

## Summary

**Problem:** Dnsmasq doesn't know about Tailscale's `.ts.net` domain after switching from ISC DHCP.

**Solution:** Configure Split-DNS at three levels:
1. **Tailscale:** Restrict local DNS to local domain only
2. **Dnsmasq:** Forward `.ts.net` to Tailscale Magic DNS (`100.100.100.100`)
3. **AdGuard Home:** Forward `.ts.net` to Tailscale Magic DNS

**Result:** 
- ✅ Tailscale works both inside and outside the LAN
- ✅ No DNS loops or conflicts
- ✅ Local and Tailscale domains properly separated
- ✅ Fast DNS resolution

---

## License

This guide is provided as-is under CC0 1.0 Universal (Public Domain). Feel free to use, modify, and share.

## Contributing

If you found this guide helpful or have suggestions for improvement, please open an issue or PR on GitHub!

---

**Last Updated:** 2025-10-28
**Tested On:** OPNsense 25.7, Dnsmasq, Unbound DNS, AdGuard Home, Tailscale
