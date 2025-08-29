# Upptime Monitoring 403 Error - Issue Analysis & Solutions

## Executive Summary

The RiskSentinel status page shows all endpoints as "down" with 403 errors, despite displaying 100% historical uptime. This document outlines the root cause and provides practical solutions using existing tools and free Cloudflare plans.

## Issue Description

### Current Status
- **Symptom**: All monitored endpoints return HTTP 403 (Forbidden) errors
- **Uptime Display**: Shows 100% uptime despite "down" status
- **Manual Testing**: APIs work correctly via browser and curl
- **Impact**: Status page incorrectly shows complete outage

### Root Cause Analysis

The issue stems from **Cloudflare's Bot Fight Mode** blocking GitHub Actions requests:

```json
{
  "action": "managed_challenge",
  "ruleId": "bot_fight_mode",
  "source": "botFight",
  "clientASNDescription": "MICROSOFT-CORP-MSN-AS-BLOCK",
  "clientAsn": "8075",
  "userAgent": "Upptime/1.0"
}
```

**Key Points:**
1. Cloudflare identifies GitHub Actions (Microsoft AS8075) as potential bot traffic
2. Bot Fight Mode cannot be bypassed using standard WAF custom rules
3. The monitoring requests never reach your actual servers
4. Historical uptime remains 100% because this is a recent configuration/detection change

## Available Solutions (Free Tier Compatible)

### Solution 1: Disable Bot Fight Mode
**Implementation:**
1. Navigate to Cloudflare Dashboard → Security → Bots
2. Toggle Bot Fight Mode to OFF
3. Save changes

**Pros:**
- Immediate fix
- No code changes required
- Zero cost

**Cons:**
- Reduces bot protection for entire domain
- May increase unwanted bot traffic
- Not recommended for production sites

**When to Use:** Temporary testing or low-traffic sites with minimal bot concerns

---

### Solution 2: Cloudflare Worker with Secret Header
**Implementation:**

1. **Create Worker** in Cloudflare Dashboard:
```javascript
export default {
  async fetch(request, env, ctx) {
    const SECRET_HEADER = 'X-Uptime-Secret';
    const SECRET_VALUE = env.UPTIME_SECRET || 'generate-strong-secret-here';
    
    if (request.headers.get(SECRET_HEADER) === SECRET_VALUE) {
      const newHeaders = new Headers(request.headers);
      newHeaders.delete(SECRET_HEADER);
      
      return await fetch(new Request(request.url, {
        method: request.method,
        headers: newHeaders,
        body: request.body
      }), {
        cf: { cacheTtl: 0 }
      });
    }
    
    return fetch(request);
  }
}
```

2. **Deploy Worker** to routes:
   - `risksentinel.ai/*`
   - `api.risksentinel.ai/*`

3. **Add secret to GitHub**:
   ```bash
   gh secret set UPTIME_SECRET --body="your-generated-secret"
   ```

4. **Update .upptimerc.yml**:
   ```yaml
   sites:
     - name: RiskSentinel API - Health Check
       url: https://risksentinel.ai/health
       headers:
         - "X-Uptime-Secret: $UPTIME_SECRET"
   ```

**Pros:**
- Maintains Bot Fight Mode protection
- Selective bypass only for monitoring
- Works with free Cloudflare plan (100k requests/day)
- Secure with rotating secrets

**Cons:**
- Requires Worker setup
- Adds complexity
- Worker requests count toward free tier limit

**When to Use:** Best balance of security and functionality

---

### Solution 3: Cloudflare Page Rules Workaround
**Implementation:**

1. **Create specific monitoring paths**:
   - Add `/status-check` endpoint to your application
   - Make it return same data as health endpoint

2. **Create Page Rule**:
   - URL: `*risksentinel.ai/status-check`
   - Settings: Security Level → Essentially Off
   - Save and Deploy

3. **Update Upptime configuration** to use new endpoint

**Pros:**
- No Worker needed
- Simple configuration
- Uses existing Page Rules (3 free)

**Cons:**
- Requires application changes
- Limited to 3 rules on free plan
- May still be blocked by Bot Fight Mode

**When to Use:** When Worker solution isn't feasible

---

### Solution 4: Alternative Monitoring Endpoint
**Implementation:**

1. **Set up monitoring subdomain**:
   - Create `monitor.risksentinel.ai`
   - Point to same origin
   - Exclude from Bot Fight Mode

2. **Configure DNS**:
   ```
   Type: CNAME
   Name: monitor
   Target: risksentinel.ai
   Proxy: OFF (DNS only - bypasses Cloudflare)
   ```

3. **Update Upptime** to use `monitor.risksentinel.ai`

**Pros:**
- Complete Cloudflare bypass
- No Worker needed
- Simple DNS change

**Cons:**
- Exposes origin IP
- No DDoS protection for monitor subdomain
- Requires SSL certificate for subdomain

**When to Use:** When other solutions fail

---

### Solution 5: GitHub Actions IP Whitelist Alternative
**Implementation:**

Since Bot Fight Mode can't be bypassed with IP rules, create a monitoring proxy:

1. **Use GitHub Pages** as proxy:
   - Create simple HTML with JavaScript fetch
   - Host on GitHub Pages (free)
   - Have it make client-side requests

2. **Or use Cloudflare Worker** as proxy (see Solution 2)

**Pros:**
- Works around IP restrictions
- Free with GitHub

**Cons:**
- Complex setup
- May not work for authenticated endpoints

---

## Recommended Solution

**For immediate fix:** Solution 2 (Cloudflare Worker with Secret Header)

**Reasoning:**
- Maintains security (Bot Fight Mode stays on)
- Works with free Cloudflare plan
- Minimal code changes
- Can be implemented quickly
- Secure with proper secret management

## Implementation Checklist

- [ ] Generate secure secret: `openssl rand -hex 32`
- [ ] Create Cloudflare Worker
- [ ] Deploy Worker to both domains
- [ ] Add UPTIME_SECRET to GitHub Secrets
- [ ] Update .upptimerc.yml with secret header
- [ ] Commit and push changes
- [ ] Run `gh workflow run "Uptime CI"`
- [ ] Verify endpoints return 200 status
- [ ] Monitor for 24 hours to confirm stability

## Testing Commands

```bash
# Test with secret header (should work)
curl -H "X-Uptime-Secret: your-secret" https://risksentinel.ai/health

# Test without secret (should still be blocked)
curl https://risksentinel.ai/health

# Trigger Upptime check
gh workflow run "Uptime CI"

# View workflow logs
gh run list --workflow="Uptime CI"
gh run view [run-id] --log
```

## Long-term Considerations

1. **Monitor Worker usage**: Free tier allows 100,000 requests/day
2. **Rotate secrets quarterly**: Update both GitHub Secret and Worker
3. **Consider paid Cloudflare**: Super Bot Fight Mode offers better control
4. **Alternative monitors**: Consider UptimeRobot free tier as backup

## Troubleshooting

If issues persist after implementation:

1. **Check Worker logs** in Cloudflare Dashboard
2. **Verify secret** matches in GitHub and Worker
3. **Test endpoints** manually with curl
4. **Review GitHub Actions logs**: `gh run view --log`
5. **Ensure Worker routes** cover all domains
6. **Check for typos** in header names/values

## Conclusion

The 403 errors are caused by Cloudflare's Bot Fight Mode blocking GitHub Actions traffic. While this protection is valuable, it interferes with Upptime monitoring. The recommended Cloudflare Worker solution provides a secure bypass mechanism while maintaining protection against actual malicious bots.

The status page showing 100% uptime with "down" status is expected behavior - Upptime tracks historical availability, and these 403 errors are recent. Once fixed, the status will show "up" while maintaining the historical uptime percentage.