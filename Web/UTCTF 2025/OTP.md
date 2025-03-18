

In this challenge, we can register users with a secret. We can then "Look up a specific pairing," which returns the XOR result of their secrets. For example, if a user registers with secret "a" and another with secret "b", the pairing function returns 3, since:
```py
ord('a') = 97  
ord('b') = 98  
97 XOR 98 = 3
```
Quick script to test that:
```py
import requests
import random
import re

url = "http://challenge.utctf.live:3725/index.php"

randomuser1 = f"user{random.randint(1000, 9999)}"
randomuser2 = f"user{random.randint(1000, 9999)}"

data1 = {
    "username": randomuser1,
    "password": "a"
}
data2 = {
    "username": randomuser2,
    "password": "b"
}
data3 = {
    "username1": randomuser1,
    "username2": randomuser2
}

r1 = requests.post(url, data=data1)
r2 = requests.post(url, data=data2)
r3 = requests.post(url, data=data3)

regex_data = re.search(r"The pairing for (\w+) and (\w+) is: (\d+)", r3.text)
print(regex_data.group(3))
```
```
3
```


Since we can pair any registered secret with "flag", I wrote a script that iterates through all possible characters, selecting the one that returns the lowest XOR value at each position.

```py
import requests
import re
import string
url = "http://challenge.utctf.live:3725/index.php"
user_value = 0
def send_request(currentFlag, char):
    global user_value
    randomuser = f"user{user_value}"
    user_value += 1
    data1 = {
        "username": randomuser,
        "password": currentFlag + char
    }
    data2 = {
        "username1": randomuser,
        "username2": "flag"
    }
    r1 = requests.post(url, data=data1)
    r2 = requests.post(url, data=data2)
    regex_data = re.search(r"The pairing for (\w+) and (\w+) is: (\d+)", r2.text)
    pairing_value = int(regex_data.group(3))
    print(f"Trying {currentFlag + char}: Pairing Value = {pairing_value}")
    return pairing_value

flag = "utflag{"
pairing_value = 9999
chars = string.ascii_letters + string.digits + '}' + '_'
while(pairing_value > 0):
    min_pairing_value_current = 9999
    current_position_char = ''
    for char in chars:
        pairing_value = send_request(flag, char)
        if pairing_value < min_pairing_value_current:
            min_pairing_value_current = pairing_value
            current_position_char = char
    flag += current_position_char
    pairing_value = min_pairing_value_current
print(flag)
```
This script is quite slow, in the event I give this script to chatgpt to add 10 workers
```py
while pairing_value > 0:
    min_pairing_value_current = 9999
    current_position_char = ''

    with concurrent.futures.ThreadPoolExecutor(max_workers=10) as executor:
        future_to_char = {executor.submit(send_request, flag, char): char for char in chars}

        for future in concurrent.futures.as_completed(future_to_char):
            char, temp_pairing_value = future.result()
            if temp_pairing_value < min_pairing_value_current:
                min_pairing_value_current = temp_pairing_value
                current_position_char = char

    flag += current_position_char
    pairing_value = min_pairing_value_current
    print(f"Current flag progress: {flag}")
```
Returned flag: `utflag{On3_sT3P_4t_4_t1m3}`