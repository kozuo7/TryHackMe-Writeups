# Password Security Assessment

## Description

Marco Bianchi, a systems administrator, recently deployed several internal services, including a firewall console, employee portal, social platform, and SSH access to critical infrastructure.

Due to tight deadlines and operational pressure, Marco reused weak, predictable, and pattern-based passwords across multiple systems.

The objective of this assessment was to identify weaknesses in Marco’s authentication practices by gathering information from the exposed services and using it to create targeted password guesses.

---

## Reconnaissance

I started by running an Nmap scan against the target:

```bash
nmap -sC -sV 10.144.187.131
```

The scan identified the following open ports:

```text
22/tcp    SSH
5000/tcp  Challenge interface
5001/tcp  FirewallOS console
5002/tcp  Careers and employee portal
5003/tcp  Social media platform
```

Port 22 was running an SSH service on Ubuntu. The exact SSH and operating system version should be taken directly from the Nmap output.

I first visited the challenge interface as instructed:

```text
http://10.144.187.131:5000/
```

The page explained the objective of each level. It also stated that blind brute-force attacks were out of scope.

This meant that using large generic password lists without any prior investigation was not allowed. However, targeted password guessing based on information discovered during reconnaissance was permitted.

---

## Level 1 – Default Credentials

The first level stated that the FirewallOS console had been left with default credentials.

I navigated to the login page:

```text
http://10.144.187.131:5001/login
```

Since the challenge specifically mentioned default credentials, I started by manually testing common administrative username and password combinations:

```text
admin / admin
admin / password
admin / password123
admin / admin12345
```

The successful credentials were:

```text
Username: admin
Password: admin12345
```

This level demonstrated the risk of leaving default or easily guessed credentials active on an administrative service. An attacker would not need advanced tools to compromise the account because the credentials could be discovered through a few manual guesses.

---

## Level 2 – Company Keyword Password

For the second level, I navigated to the careers website running on port 5002:

```text
http://10.144.187.131:5002/
```

The website contained several company-related keywords beneath the phrase **“Build the Future with Engineering.”**

I recorded the following words:

```text
innovation
excellence
security
digital
cloud
future
talent
```

I then clicked the **Employee Login** button. The login page automatically filled in the username:

```text
marco
```

Because Marco was known to use weak and predictable passwords, I manually tested the company keywords collected from the public website.

The successful credentials were:

```text
Username: marco
Password: excellence
```

Marco had used a publicly displayed company keyword as his password. Anyone who visited the company website could have gathered the same word and attempted it against the employee login portal.

---

## Level 3 – Password Based on Personal Information

The third level required me to determine Marco’s password for the social platform:

```text
http://social.thm:5003/
```

The level instructed me to derive the password from Marco’s personal information.

I returned to the employee portal on port 5002 and reviewed Marco’s public employee profile. The following information was exposed:

```text
First name: Marco
Nickname: Marky
Surname: Bianchi
Birthdate: February 14, 1995
Department: IT Operations
```

I initially attempted to manually create password variations using Marco’s name, nickname, surname, and birthdate. However, none of my first guesses were successful.

At this point, I reviewed another writeup to understand how others approached the challenge. The method used in that writeup was more complicated than necessary for the earlier levels, but it introduced me to a useful tool for this level: the **Common User Passwords Profiler**, also known as **CUPP**.

CUPP creates targeted password lists using personal information about a specific person. Unlike a generic wordlist, the generated passwords are based on details such as names, nicknames, partners, pets, birthdays, and important dates.

I launched CUPP in interactive mode:

```bash
cupp -i
```

I entered the information collected from Marco’s employee profile, including his:

```text
First name
Surname
Nickname
Birthdate
```

After processing the information, CUPP generated a targeted password list named:

```text
marco.txt
```

I then used Hydra to test the generated passwords against the social platform’s HTTP login form:

```bash
hydra -l marco -P marco.txt -s 5003 10.144.187.131 \
http-post-form "/login:username=^USER^&password=^PASS^:F=Invalid" -V
```

Command breakdown:

```text
-l marco
```

Uses `marco` as the username.

```text
-P marco.txt
```

Uses the password list generated by CUPP.

```text
-s 5003
```

Targets the web service running on port 5003.

```text
http-post-form
```

Tells Hydra that the login form submits credentials using an HTTP POST request.

```text
/login
```

Specifies the login endpoint.

```text
username=^USER^&password=^PASS^
```

Defines where Hydra should insert each username and password attempt.

```text
F=Invalid
```

Tells Hydra that a response containing the word `Invalid` represents a failed login attempt.

Hydra successfully identified the password:

```text
Username: marco
Password: Bianchi2495
```

The password used Marco’s surname and a number pattern based on his personal information.

This level demonstrated how personal information exposed through employee directories or social media can be used to generate targeted password lists. Even when a password is not immediately obvious, automated profiling tools can create realistic variations based on a victim’s known details.

---

## Level 4 – Recovering the Original Filename

The fourth level explained that Marco had recently uploaded a new profile picture to the social platform.

For privacy and storage consistency, the application renamed uploaded files using the SHA-256 hash of the original filename and stored them using the following format:

```text
SHA256(original_filename).png
```

The objective was to identify the original filename of Marco’s profile picture.

After logging into Marco’s social media account, I right-clicked his profile picture and inspected the element using the browser’s developer tools.

The image source revealed the following URL:

```text
http://10.144.187.131:5003/uploads/d34a569ab7aaa54dacd715ae64953455d86b768846cd0085ef4e9e7471489b7b.png
```

I removed the `.png` extension and extracted the SHA-256 hash:

```text
d34a569ab7aaa54dacd715ae64953455d86b768846cd0085ef4e9e7471489b7b
```

I searched for online SHA-256 lookup tools and tested the hash using several websites. Some of the tools did not return a result.

I eventually used the following website:

```text
https://10015.io/tools/sha256-encrypt-decrypt
```

The website returned the matching input:

```text
family
```

Therefore, the original filename was:

```text
family
```

It is important to note that SHA-256 was not actually decrypted. SHA-256 is designed to be a one-way hashing algorithm.

The original value was most likely discovered because the word `family` was already present in a precomputed hash database or was tested as part of a dictionary lookup. Common and predictable inputs are still vulnerable to this type of recovery even when a strong hashing algorithm is used.

---

## Level 5 – Predictable SSH Password Pattern

The final level stated that Marco had revealed his password-generation pattern through a post on the social platform.

The task was to use that information to create a targeted wordlist or set of password candidates and gain access to the SSH service using the username:

```text
marco
```

From Marco’s social media post, I identified the following password format:

```text
Capitalized company keyword + year + exclamation mark
```

Earlier in the challenge, Marco had used the company keyword:

```text
excellence
```

Using the newly discovered password pattern, I created the following candidate:

```text
Excellence2024!
```

The password was constructed by:

```text
Company keyword: excellence
Capitalized keyword: Excellence
Appended year: 2024
Special character: !
```

I tested `Excellence2024!`, but it was unsuccessful.

I returned to Marco’s post and reviewed it again. The tags beneath the post contained five company-related keywords:

```text
security
excellence
innovation
digital
cloud
```

Using Marco’s disclosed password format, I converted each keyword into a possible SSH password:

```text
Security2024!
Excellence2024!
Innovation2024!
Digital2024!
Cloud2024!
```

I used `2024` because Marco had explicitly mentioned that year in his post. If none of those candidates had worked, my next step would have been to test other personally relevant years, such as his birth year:

```text
1995
```

I connected to the SSH service using:

```bash
ssh marco@10.144.187.131
```

I started with the first password candidate:

```text
Security2024!
```

The password worked on the first attempt.

The successful SSH credentials were:

```text
Username: marco
Password: Security2024!
```

This level demonstrated that even a password containing uppercase letters, numbers, and a special character can still be weak when it follows a predictable pattern.

The password may appear complex, but every part of it could be discovered from public information:

```text
Security  -> Public company keyword
2024      -> Year disclosed in Marco’s post
!         -> Special character revealed as part of his pattern
```

---

## Findings

The assessment identified several weaknesses in Marco’s authentication practices.

### 1. Default Administrative Credentials

The FirewallOS console used the following credentials:

```text
admin / admin12345
```

Default and commonly guessed credentials allow attackers to gain access without needing to perform extensive reconnaissance or exploitation.

### 2. Password Based on Public Company Information

Marco used the following company keyword as a password:

```text
excellence
```

The word was publicly displayed on the company’s careers website and could easily be discovered by an attacker.

### 3. Password Based on Personal Information

Marco’s social media password was:

```text
Bianchi2495
```

The password was generated using his surname and personal information exposed through his employee profile.

### 4. Predictable Filename

Marco’s profile picture originally used the filename:

```text
family
```

Although the application stored the SHA-256 hash of the filename, the original value was recoverable because it was a common dictionary word.

### 5. Publicly Disclosed Password Pattern

Marco disclosed enough information on social media to reconstruct his SSH password:

```text
Security2024!
```

The use of capitalization, a year, and a special character did not provide meaningful protection because the overall structure was predictable.

### 6. Repeated Password-Generation Habits

Although Marco did not use the exact same password for every service, he repeatedly relied on:

```text
Default credentials
Company terminology
Personal information
Important years
Common formatting patterns
```

An attacker who compromised one account could use the information discovered there to predict passwords for Marco’s other accounts.

---

## Recommendations

### Use Unique Random Passwords

Marco should use a unique, randomly generated password for every account. Passwords should not be based on names, birthdays, company terms, departments, hobbies, or other public information.

### Use a Password Manager

A trusted password manager would allow Marco to generate and store strong passwords without needing to remember predictable patterns.

### Change Default Credentials

All default usernames and passwords should be changed immediately after deploying a service.

### Enable Multi-Factor Authentication

Multi-factor authentication should be enabled for administrative portals, employee accounts, social platforms, and remote access services whenever possible.

### Protect SSH Access

Password-based SSH authentication should be disabled where possible and replaced with public-key authentication.

Additional protections could include:

```text
Restricting SSH access by IP address
Using firewall rules
Disabling direct root login
Using Fail2ban
Monitoring failed login attempts
```

### Implement Rate Limiting

Web applications should limit repeated login attempts to reduce the effectiveness of automated password-guessing tools such as Hydra.

### Reduce Public Information Exposure

Employee profiles and social media posts should not expose information that could help attackers create passwords, security question answers, or targeted wordlists.

### Avoid Predictable Password Patterns

Adding a capital letter, year, and exclamation mark does not make a password secure when the base word and formatting pattern are predictable.

---

## Conclusion

This assessment demonstrated how an attacker could compromise multiple services by combining basic reconnaissance with targeted password guessing.

Marco’s accounts were vulnerable because his passwords were based on default credentials, public company keywords, personal information, and predictable formatting rules.

The attack did not require a massive blind brute-force campaign. Each password attempt was guided by information collected from the exposed services.

The most important lesson from this assessment is that password complexity alone does not guarantee security. A password such as `Security2024!` satisfies many traditional complexity requirements, but it remains weak because its structure can be predicted.

Using unique random passwords, a password manager, multi-factor authentication, SSH keys, rate limiting, and reduced public information exposure would significantly improve Marco’s authentication security.
````
