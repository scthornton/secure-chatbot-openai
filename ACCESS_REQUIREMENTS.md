# Prisma AI Runtime Security - Access Requirements

## Overview

This document outlines the network access requirements for customers using **Prisma AI Runtime Security** for model security scanning, runtime protection, and red teaming activities.

---

## Network Requirements

### Outbound Internet Access (Required)

All Prisma AIRS components require **outbound HTTPS access** to Palo Alto Networks cloud services. No inbound connections are required.

**Key Point:** Prisma AIRS is a cloud-based service - your application makes API calls TO Palo Alto Networks, not the reverse.

---

## API Endpoints by Region

### United States (US)

**Primary API Endpoint:**
```
https://service.api.aisecurity.paloaltonetworks.com
```

**Required for:**
- AI Runtime Security scanning (prompt/response analysis)
- Model security scanning
- Policy enforcement
- Security profile management

**Ports:**
- TCP 443 (HTTPS)

---

### Europe (Germany)

**Primary API Endpoint:**
```
https://service-de.api.aisecurity.paloaltonetworks.com
```

**Required for:**
- EU customers requiring data residency in Germany
- Same functionality as US endpoint

**Ports:**
- TCP 443 (HTTPS)

---

## Required Domains for Whitelisting

### Core Services (REQUIRED)

If your environment uses firewall rules or proxy servers, whitelist these domains:

| Domain | Port | Purpose |
|--------|------|---------|
| `service.api.aisecurity.paloaltonetworks.com` | 443 | US API endpoint (primary) |
| `service-de.api.aisecurity.paloaltonetworks.com` | 443 | EU API endpoint (Germany) |
| `api.paloaltonetworks.com` | 443 | Authentication and licensing |
| `certificate.paloaltonetworks.com` | 443 | Certificate validation |
| `certificatetrusted.paloaltonetworks.com` | 443 | Trusted certificate services |

### Certificate Validation (REQUIRED)

| Domain | Port | Purpose |
|--------|------|---------|
| `ocsp.paloaltonetworks.com` | 80 | Certificate revocation checking (OCSP) |
| `crl.paloaltonetworks.com` | 80 | Certificate revocation list |
| `ocsp.godaddy.com` | 80 | Third-party certificate validation |

### Model Security (if using Model Security features)

| Domain | Port | Purpose |
|--------|------|---------|
| `prod.us.secure-ai.paloaltonetworks.com` | 443 | Model security scanning service |
| `model-scanner.paloaltonetworks.com` | 443 | Model artifact analysis |

---

## IP Whitelisting

### Cloud Service IPs

Palo Alto Networks uses **dynamic cloud infrastructure** with load balancing and auto-scaling. Static IP whitelisting is **not recommended** and may cause service disruptions.

**Recommended Approach:** Whitelist by **FQDN** (Fully Qualified Domain Name) instead of IP addresses.

**Why?**
- IPs change frequently due to cloud infrastructure updates
- Global load balancing routes traffic to nearest data center
- Maintenance and scaling events change IP assignments

**If FQDN whitelisting is not possible:**
- Contact Palo Alto Networks support for current IP ranges
- Note: IP ranges are subject to change without notice
- Consider using proxy authentication instead

---

## Proxy Configuration

### Corporate Proxy Servers

If your environment requires all internet traffic to go through a corporate proxy:

**Option 1: System Proxy (Recommended)**

Set system environment variables:

```bash
export HTTP_PROXY="http://proxy.example.com:8080"
export HTTPS_PROXY="http://proxy.example.com:8080"
export NO_PROXY="localhost,127.0.0.1"
```

The Python `requests` library (used by our chatbot) automatically uses these proxies.

**Option 2: Application-Specific Proxy**

Configure proxy in your application code:

```python
import requests

proxies = {
    'http': 'http://proxy.example.com:8080',
    'https': 'http://proxy.example.com:8080',
}

response = requests.post(
    "https://service.api.aisecurity.paloaltonetworks.com/v1/scan/sync/request",
    headers=headers,
    json=payload,
    proxies=proxies  # Add proxy configuration
)
```

**Option 3: Authenticated Proxy**

If your proxy requires authentication:

```bash
export HTTP_PROXY="http://username:password@proxy.example.com:8080"
export HTTPS_PROXY="http://username:password@proxy.example.com:8080"
```

---

## Firewall Rules

### Recommended Firewall Configuration

**Outbound Rules (ALLOW):**

```
Source: Application servers/workstations running Prisma AIRS
Destination: *.paloaltonetworks.com
Port: TCP 443 (HTTPS)
Protocol: HTTPS
Action: ALLOW
```

**Specific Rules (if wildcard not allowed):**

```
# AI Runtime Security API (US)
Source: Your network
Destination: service.api.aisecurity.paloaltonetworks.com
Port: TCP 443
Action: ALLOW

# AI Runtime Security API (EU)
Source: Your network
Destination: service-de.api.aisecurity.paloaltonetworks.com
Port: TCP 443
Action: ALLOW

# Authentication & Licensing
Source: Your network
Destination: api.paloaltonetworks.com
Port: TCP 443
Action: ALLOW

# Certificate Services
Source: Your network
Destination: certificate.paloaltonetworks.com
Port: TCP 443
Action: ALLOW

# OCSP (Certificate Revocation)
Source: Your network
Destination: ocsp.paloaltonetworks.com, crl.paloaltonetworks.com
Port: TCP 80
Action: ALLOW
```

**Inbound Rules:**

❌ **NO INBOUND RULES REQUIRED** - Prisma AIRS is outbound-only

---

## Authentication & Authorization

### API Key Requirements

**What you need:**
- **API Key** (`PANW_AI_SEC_API_KEY`)
- **Profile Name** (`PANW_AI_SEC_PROFILE_NAME`)

**How to obtain:**
1. Log in to Prisma Cloud console
2. Navigate to: **Settings → Access Keys**
3. Generate new API key for AI Runtime Security
4. Copy key and profile name (you won't see the key again)

**Security Best Practices:**
- Store API keys in secure vault (HashiCorp Vault, AWS Secrets Manager)
- Never commit API keys to version control
- Rotate API keys quarterly
- Use separate keys for dev/staging/production
- Apply principle of least privilege (minimum required permissions)

### IAM Permissions

**Minimum Required Permissions:**
- `ai-runtime-security:scan` - Scan prompts/responses
- `ai-runtime-security:read-profile` - Read security profiles
- `ai-runtime-security:read-results` - Read scan results

**Full Access Permissions** (for administrators):
- `ai-runtime-security:*` - All AI Runtime Security operations
- `ai-model-security:*` - Model scanning operations
- `ai-security:admin` - Profile and policy management

---

## Testing Network Connectivity

### Quick Connectivity Test

Run this from your application server/workstation:

```bash
# Test US endpoint
curl -I https://service.api.aisecurity.paloaltonetworks.com

# Expected: HTTP 401 Unauthorized (proves connectivity, auth required)
# NOT Expected: Connection timeout, DNS resolution failure

# Test EU endpoint
curl -I https://service-de.api.aisecurity.paloaltonetworks.com

# Test certificate services
curl -I https://certificate.paloaltonetworks.com
curl -I http://ocsp.paloaltonetworks.com
```

**Successful output:**
```
HTTP/2 401
date: ...
content-type: application/json
```

**Failed output (network blocked):**
```
curl: (7) Failed to connect to service.api.aisecurity.paloaltonetworks.com port 443: Connection refused
```

### Full API Test

```bash
# Test with valid API key
export PANW_AI_SEC_API_KEY="your-api-key"
export PANW_AI_SEC_PROFILE_NAME="your-profile"

curl --location 'https://service.api.aisecurity.paloaltonetworks.com/v1/scan/sync/request' \
  --header 'Content-Type: application/json' \
  --header "x-pan-token: $PANW_AI_SEC_API_KEY" \
  --data '{
    "tr_id": "test-connection",
    "ai_profile": {
      "profile_name": "'$PANW_AI_SEC_PROFILE_NAME'"
    },
    "contents": [{
      "prompt": "Hello, test connection"
    }]
  }'
```

**Expected:** JSON response with `category`, `action` fields

---

## Common Access Issues

### Issue 1: Connection Timeout

**Symptom:**
```
requests.exceptions.ConnectionError: Connection timeout
```

**Root Causes:**
- Firewall blocking outbound HTTPS to `*.paloaltonetworks.com`
- Proxy server not configured
- VPN/network connectivity issues

**Solutions:**
1. Verify firewall allows HTTPS to required domains
2. Configure proxy if required (see Proxy Configuration section)
3. Test basic connectivity: `curl -I https://service.api.aisecurity.paloaltonetworks.com`
4. Check corporate network restrictions

---

### Issue 2: DNS Resolution Failure

**Symptom:**
```
requests.exceptions.ConnectionError: Failed to resolve hostname
```

**Root Causes:**
- DNS server cannot resolve `*.paloaltonetworks.com`
- Corporate DNS filtering
- Split-tunnel VPN configuration

**Solutions:**
1. Test DNS: `nslookup service.api.aisecurity.paloaltonetworks.com`
2. Use public DNS temporarily: `8.8.8.8` or `1.1.1.1`
3. Contact network team to add DNS entries

---

### Issue 3: SSL/TLS Certificate Errors

**Symptom:**
```
requests.exceptions.SSLError: certificate verify failed
```

**Root Causes:**
- Corporate SSL inspection/interception
- Outdated CA certificate bundle
- Self-signed corporate certificates

**Solutions:**
1. **Corporate SSL inspection:** Add corporate CA to trust store
2. **Outdated certificates:** Update `certifi` package: `pip install --upgrade certifi`
3. **Temporary bypass** (NOT recommended for production):

```python
import requests
import urllib3
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

response = requests.post(url, headers=headers, json=payload, verify=False)
```

---

### Issue 4: 401 Unauthorized

**Symptom:**
```
HTTP 401 Unauthorized
```

**Root Causes:**
- Invalid API key
- Expired API key
- Wrong profile name
- API key lacks required permissions

**Solutions:**
1. Verify API key is correct: `echo $PANW_AI_SEC_API_KEY`
2. Regenerate API key in Prisma Cloud console
3. Check profile name matches exactly (case-sensitive)
4. Verify IAM permissions include `ai-runtime-security:scan`

---

### Issue 5: 403 Forbidden

**Symptom:**
```
HTTP 403 Forbidden
```

**Root Causes:**
- API key valid but lacks permissions
- Profile does not exist
- Account subscription expired

**Solutions:**
1. Verify profile exists: Check in Prisma Cloud console
2. Check API key permissions: Settings → Access Keys → View Permissions
3. Verify subscription is active: Settings → Licensing

---

## Regional Data Residency

### Choosing the Right Region

**US Region:**
- Default for most customers
- Lower latency for North/South America
- Endpoint: `service.api.aisecurity.paloaltonetworks.com`

**EU Region (Germany):**
- Required for GDPR data residency requirements
- Lower latency for Europe/Middle East/Africa
- Endpoint: `service-de.api.aisecurity.paloaltonetworks.com`

**Switching Regions:**

Update your base URL:

```python
# US Region
BASE_URL = "https://service.api.aisecurity.paloaltonetworks.com"

# EU Region
BASE_URL = "https://service-de.api.aisecurity.paloaltonetworks.com"
```

Note: Profiles and API keys are region-specific. Create separate credentials for each region.

---

## Red Teaming & Security Testing

### Additional Considerations for Red Teaming

**Traffic Volume:**
- Red team exercises may generate high API call volumes
- Monitor usage to avoid rate limiting
- Contact support for rate limit increases if needed

**Test Environments:**
- Use separate API keys for production vs. red team testing
- Configure separate security profiles for testing
- Tag red team traffic with distinct transaction IDs

**Data Handling:**
- Red team test data is scanned but not stored long-term
- PII/sensitive data in test prompts is still detected
- Use synthetic/sanitized data for testing

---

## Support & Escalation

### Getting Help

**Network/Access Issues:**
1. **Verify connectivity:** Run connectivity tests (see Testing section)
2. **Check firewall logs:** Look for blocked connections to `*.paloaltonetworks.com`
3. **Contact network team:** Provide required domains list
4. **Open support case:** Include connectivity test results

**Palo Alto Networks Support:**
- Portal: https://support.paloaltonetworks.com
- Email: support@paloaltonetworks.com
- Phone: Contact your account team

**Documentation:**
- Technical docs: https://docs.paloaltonetworks.com
- API reference: https://pan.dev

---

## Quick Reference Card

### Essential Information

**API Endpoints:**
- US: `https://service.api.aisecurity.paloaltonetworks.com`
- EU: `https://service-de.api.aisecurity.paloaltonetworks.com`

**Required Ports:**
- TCP 443 (HTTPS) - API communication
- TCP 80 (HTTP) - Certificate validation

**Firewall Rules:**
- Outbound: ALLOW `*.paloaltonetworks.com:443`
- Inbound: NONE (no inbound required)

**Authentication:**
- API Key: Store in `PANW_AI_SEC_API_KEY`
- Profile: Store in `PANW_AI_SEC_PROFILE_NAME`

**Connectivity Test:**
```bash
curl -I https://service.api.aisecurity.paloaltonetworks.com
# Expected: HTTP 401 (proves connectivity works)
```

---

## Summary

**Key Takeaways:**

✅ **Outbound HTTPS only** - No inbound connections required
✅ **Whitelist by FQDN** - Not by IP address (IPs change)
✅ **Minimal ports** - Just TCP 443 for APIs, TCP 80 for cert validation
✅ **Cloud-based** - No on-premises infrastructure needed
✅ **Proxy-friendly** - Works with corporate proxies (HTTP_PROXY env var)
✅ **Region-specific** - Choose US or EU based on data residency needs

**For customers:**
- Share required domains list with network team
- Request outbound HTTPS access to `*.paloaltonetworks.com:443`
- Configure proxy if required
- Test connectivity before deployment

**No special IP whitelisting or VPN required** - standard internet access is sufficient.
