# Neighbor — TryHackMe Writeup

## Overview

**Room:** Neighbor  
**Category:** Web Security  
**Vulnerability:** IDOR  
**Difficulty:** Beginner  

This room focused on identifying an **IDOR** vulnerability in a web application. The solve was simple, but it was a good example of why source code inspection, URL parameter testing, and understanding authorization logic matter.

## Pre-Lab Hypothesis 

Before joining the room, the description strongly hinted at an IDOR vulnerability, the wording mentiooned finding secrets on a neighbor's logged-in page, so my first thought was that the application would expose another user's data through a controllable reference.

My initial guess was that there would be a parameter in the URL, like ?id=

## Initial Observation

The application opened to a login page. On the page, there was a message suggesting that `Ctrl+U` could be used to register.

After pressing `Ctrl+U`, the browser opened the page source. Inside the source code, I found the following comment:

```html
Don't have an account? Use the guest account! (<code>Ctrl+U</code>)</p>
<!-- use guest:guest credentials until registration is fixed. "admin" user account is off limits!!!!! -->
```

This revealed two important pieces of information:

- A valid guest account existed
- An admin account also existed

At first, I focused too much on the part saying the `admin` account was off limits. My first thought was to try logging in as `admin` using common passwords.

I also briefly thought about whether tools like Hydra or Burp Suite would be useful, but after looking back at the source code, the better clue was the exposed guest credentials.

```text
guest:guest
```

## Logging In

Using the credentials from the source code, I logged in as the guest user.

```text
Username: guest
Password: guest
```

After logging in, I noticed that the URL contained a user parameter:

```text
?user=guest
```

Since the room was hinting toward IDOR, this parameter immediately stood out.

## Testing for IDOR

The application appeared to be using the `user` parameter to decide which user's page should be displayed.

Since the source code had already mentioned an `admin` account, I changed the URL from:

```text
?user=guest
```

to:

```text
?user=admin
```

After changing the parameter, the application loaded the admin user's page and displayed the flag.

## Vulnerability Explanation

This was an **Insecure Direct Object Reference** vulnerability.

The application allowed me to authenticate as a low-privileged user, but it did not properly check whether that user was authorized to access another user's page.

The issue was not weak passwords or brute forcing. The issue was that the server trusted a user-controlled parameter.

```text
?user=admin
```

A normal guest user should not be able to access the admin page just by modifying a value in the URL.

## Authentication vs Authorization

This room was a good example of the difference between authentication and authorization.

```text
Authentication = Can you log in?
Authorization = Are you allowed to access this specific resource?
```

In this case, authentication worked because I was logged in as `guest`.

Authorization failed because the application allowed the `guest` user to access the `admin` page.

## Root Cause

The root cause was a missing server-side authorization check on the `user` GET parameter.

The application likely trusted the value of the `user` parameter to decide what account page to show, instead of checking whether the currently logged-in user was allowed to access that account.

## Impact

In this lab, the impact was access to the admin page and the flag.

In a real application, this type of vulnerability could expose sensitive user information, private account data, admin-only pages, invoices, messages, settings, or other restricted resources.

## Remediation

The application should not trust the `user` parameter directly.

A safer approach would be to:

- Load user data based on the authenticated session
- Perform server-side authorization checks
- Deny access if the logged-in user does not have permission
- Avoid exposing sensitive resources through predictable user-controlled parameters

Before returning any user data, the server should verify that the current user is allowed to access the requested resource.

## Lessons Learned

This room was straightforward, but it reinforced an important beginner web security lesson: do not overcomplicate the target before checking the basics.

My first instinct was to focus on the `admin` login and think about brute forcing. The better move was to slow down, reread the source code, use the provided guest credentials, and inspect how the application handled logged-in users.
