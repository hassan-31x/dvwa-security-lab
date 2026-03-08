
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