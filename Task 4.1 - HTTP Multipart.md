# HTTP Multipart và Upload Pipeline

## Bố cục nội dung

```
HTTP Multipart và Upload Pipeline
│
├── 1. Ôn HTTP cơ bản
│   ├── POST Request
│   ├── Header
│   ├── Body
│   ├── Content-Type
│   └── Content-Length
│
├── 2. multipart/form-data
│   ├── Form Upload hoạt động thế nào
│   ├── Boundary
│   ├── Part
│   ├── Content-Disposition
│   │   ├── name
│   │   └── filename
│   ├── Content-Type
│   └── Binary Data
│
├── 3. Burp Suite
│   ├── Intercept Upload
│   ├── Phân tích Request từng dòng
│   ├── Đổi filename
│   ├── Đổi Content-Type
│   ├── Quan sát Response
│   └── Repeater thử nhiều trường hợp
│
├── 4. PHP Upload Pipeline
│   ├── php.ini
│   ├── $_FILES
│   ├── upload_tmp_dir
│   ├── move_uploaded_file()
│   └── Debug var_dump($_FILES)
│
├── 5. Tổng kết Pipeline
│   ├── Browser
│   ├── HTTP Request
│   ├── Web Server
│   ├── PHP
│   ├── Temporary File
│   ├── move_uploaded_file()
│   └── Upload Folder
│
└── 6. Checklist
    ├── Giải thích request upload
    ├── Vẽ pipeline upload
    └── Trả lời câu hỏi tự kiểm tra
```

## Ôn tập về HTTP Request

### HTTP Request gồm những gì?

```
POST /post/comment HTTP/2
Host: 0a8000a403df11ce804617cd00ed0080.web-security-academy.net
Cookie: session=rje5dl4d3Fqd0ZpMVZy2brvDDR70kWqt
Content-Length: 136
Cache-Control: max-age=0
Sec-Ch-Ua: "Chromium";v="149", "Not)A;Brand";v="24"
Sec-Ch-Ua-Mobile: ?0
Sec-Ch-Ua-Platform: "Windows"
Accept-Language: en-US,en;q=0.9
Upgrade-Insecure-Requests: 1
Content-Type: application/x-www-form-urlencoded
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/149.0.0.0 Safari/537.36
Origin: https://0a8000a403df11ce804617cd00ed0080.web-security-academy.net
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Referer: https://0a8000a403df11ce804617cd00ed0080.web-security-academy.net/post?postId=5
Accept-Encoding: gzip, deflate, br
Priority: u=0, i

csrf=HyJhzJ68OS2doU5IBLSfsmaBzNGulvF9&postId=5&comment=AAAAAA&name=Minh&email=admin%40juice-sh.op&website=http%3A%2F%2Fwww.juiceshop.com
```

Một HTTP Request luôn có cấu trúc:

```
Request Line

Headers

(blank line)

Body
```

Vậy, có thể phân tích request mẫu như sau:

```
Request line là: POST /post/comment HTTP/2  

//Tập các dòng header nằm ngay bên dưới request line đến trước dòng trắng
Host:
User-Agent:
Content-Type:
Content-Length:
...   

(blank line) // Dòng trắng ngăn cách header và body

Body là: csrf=HyJhzJ68OS2doU5IBLSfsmaBzNGulvF9&postId=5&comment=AAAAAA&name=Minh&email=admin%40juice-sh.op&website=http%3A%2F%2Fwww.juiceshop.com
```

### Request Line

Lấy request line từ request mẫu là: `POST /upload HTTP/1.1`, có thể phân tích được như sau:

1. Method: POST
2. URI: /upload (cũng là API mà server có)
3. Phiên bản HTTP: HTTP/1.1

Trong upload file thì request gần như luôn dùng POST. Vì phương thức POST sẽ có Body, gửi được dữ liệu lớn, gửi được file.

### Header

Header là metadata.

```
Host: 0a8000a403df11ce804617cd00ed0080.web-security-academy.net
// Host cho biết request này gửi đến server nào

Cookie: session=rje5dl4d3Fqd0ZpMVZy2brvDDR70kWqt
// Có thể server dùng cookie này để xác thực, nhận biết user hoặc phân quyền 

Content-Length: 136
// Length = 136 -> Server đọc 136 byte của body

Cache-Control: max-age=0
Sec-Ch-Ua: "Chromium";v="149", "Not)A;Brand";v="24"
Sec-Ch-Ua-Mobile: ?0
Sec-Ch-Ua-Platform: "Windows"
Accept-Language: en-US,en;q=0.9
Upgrade-Insecure-Requests: 1

Content-Type: application/x-www-form-urlencoded
// Là header quan trọng nhất, báo hiệu cho server là body được mã hóa theo chuẩn nào
// Content-Type: application/x-www-form-urlencoded -> body có dạng key=value&key=value
// Content-Type: multipart/form-data -> body upload file -> server sẽ chọn multipart parser để đọc.

User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/149.0.0.0 Safari/537.36
// Có thể server log thông tin này

Origin: https://0a8000a403df11ce804617cd00ed0080.web-security-academy.net
// Request được sinh từ website nào

Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
// Browser muốn nhận định dạng như thế nào

Sec-Fetch-Site: same-origin
// Cho server biết request này đến từ đâu

Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Referer: https://0a8000a403df11ce804617cd00ed0080.web-security-academy.net/post?postId=5
// Browser đang ở đâu trước khi gửi request này

Accept-Encoding: gzip, deflate, br
// Server có thể nén response (vì có gzip)

Priority: u=0, i
```

### Body

Body là nơi chứa dữ liệu thực.

Từ request mẫu, thu được body là

```
csrf=HyJhzJ68OS2doU5IBLSfsmaBzNGulvF9

&postId=5

&comment=AAAAAA

&name=Minh

&email=admin%40juice-sh.op

&website=http%3A%2F%2Fwww.juiceshop.com
```

Do request có header: `Content-Type: application/x-www-form-urlencoded`, nên server sẽ parse thành

| Key     | Value                            |
| ------- | -------------------------------- |
| csrf    | HyJhzJ68OS2doU5IBLSfsmaBzNGulvF9 |
| postId  | 5                                |
| comment | AAAAAA                           |
| name    | Minh                             |
| email   | admin@juice-sh.op                |
| website | http://www.juiceshop.com         |

### POST vs GET

GET thường không có body -> sẽ truyền dữ liệu vào param. Nhưng HTTP cũng không cấm GET chứa body

```
GET /search?q=cat
```

POST

```
POST /login

username=minh&password=123
```

Upload file luôn dùng POST vì cần gửi dữ liệu nhị phân.

### Content-Type

Mỗi type khác nhau thì server sẽ đọc và xử lý theo cách khác nhau.

| Content-Type                          | Dữ liệu trong Body | Ví dụ Body                                     | Parser phía Server (PHP)                              | Trường hợp sử dụng                                 |
| ------------------------------------- | -------------------- | ------------------------------------------------ | ------------------------------------------------------ | ------------------------------------------------------- |
| `application/x-www-form-urlencoded` | Dữ liệu Form       | `username=minh&password=123456`                | Form Parser (`$_POST`, `Request.Form`)             | Form HTML (Login, Register, Search, Comment)            |
| `multipart/form-data`               | Form + File          | Các Part được ngăn cách bằng`boundary` | Multipart Parser (`$_POST` + `$_FILES`)            | Upload file, gửi nhiều file hoặc file kèm dữ liệu |
| `application/json`                  | JSON                 | `{"username":"minh","password":"123456"}`      | JSON Parser (`json_decode()`, `@RequestBody`, ...) | REST API, Mobile API                                    |
| `text/plain`                        | Văn bản thuần     | `Hello World`                                  | Raw Text Parser                                        | API nhận chuỗi văn bản, Webhook                     |
| `application/xml`                   | XML                  | `<user><name>Minh</name></user>`               | XML Parser                                             | SOAP, API cũ, XXE                                      |
| `application/octet-stream`          | Dữ liệu nhị phân | Binary Data                                      | Raw Binary Parser                                      | Upload firmware, file nhị phân, download file         |
| `application/pdf`                   | File PDF             | Binary Data                                      | Binary/File Parser                                     | Upload hoặc tải tài liệu PDF                        |
| `application/zip`                   | File ZIP             | Binary Data                                      | Binary/File Parser                                     | Upload file ZIP, giải nén (Zip Slip)                  |
| `image/jpeg`                        | Ảnh JPEG            | Binary Data                                      | Image Parser                                           | Upload hoặc hiển thị ảnh JPG                        |
| `image/png`                         | Ảnh PNG             | Binary Data                                      | Image Parser                                           | Upload hoặc hiển thị ảnh PNG                        |
| `image/gif`                         | Ảnh GIF             | Binary Data                                      | Image Parser                                           | Upload hoặc hiển thị ảnh GIF                        |
| `image/svg+xml`                     | Ảnh SVG (XML)       | `<svg>...</svg>`                               | XML/Image Parser                                       | Upload SVG (cần chú ý XSS)                           |
| `text/html`                         | HTML                 | `<html>...</html>`                             | HTML Parser                                            | Trang web trả về cho trình duyệt                    |
| `text/css`                          | CSS                  | `body { color: red; }`                         | CSS Parser                                             | Stylesheet                                              |
| `application/javascript`            | JavaScript           | `alert("Hello");`                              | JavaScript Engine                                      | File JavaScript                                         |

### Tại sao Upload File không dùng JSON?

Ví dụ:

```JSON
{
    "file":"????"
}
```

JSON không được thiết kế để truyền dữ liệu nhị phân trực tiếp. Muốn nhét file vào JSON thường phải mã hóa Base64, làm dữ liệu tăng khoảng 33% và tốn thêm CPU để mã hóa/giải mã.

Trong khi đó, multipart/form-data được thiết kế riêng để truyền dữ liệu nhị phân trực tiếp, gửi nhiều trường (field) và nhiều file trong cùng một request, không cần chuyển đổi nội dung file.

Đó là lý do các form upload truyền thống trên web sử dụng multipart/form-data.

Lưu ý: Một số API hiện đại vẫn gửi file trong JSON bằng Base64 hoặc gửi trực tiếp luồng nhị phân (application/octet-stream), nhưng đó là các thiết kế API riêng, không phải cách mà form HTML hoạt động mặc định.

### Luồng HTTP khi Upload

```
   User chọn file
   │
   ▼
   Browser đọc file
   │
   ▼
   Đóng gói thành HTTP Body
   │
   ▼
   Thêm Header
   (Content-Type)
   │
   ▼
   POST Request
   │
   ▼
   Web Server
```

## multipart/form-data

### Mục tiêu

Sau phần này, cần trả lời được

* Form là gì?
* `enctype` dùng để làm gì?
* Vì sao upload file phải dùng `multipart/form-data`?
* `boundary` là gì?
* Browser tạo request upload như thế nào?

### Tài liệu tham khảo

* [datatracker.ietf.org/doc/html/rfc7578](https://datatracker.ietf.org/doc/html/rfc7578)
* [www.geeksforgeeks.org/html/define-multipart-form-data](https://www.geeksforgeeks.org/html/define-multipart-form-data/)
* [www.w3schools.com/Tags/att_form_enctype.asp](https://www.w3schools.com/Tags/att_form_enctype.asp)

### HTML Form

Form là thành phần của HTML dùng để thu thập dữ liệu từ người dùng và gửi về server.

Ví dụ:

```HTML
<form action="/login" method="POST">
    <input type="text" name="username">
    <input type="password" name="password">

    <button>Login</button>
</form>
```

Người dùng nhập

```
username = minh
password = 123456
```

Browser sẽ gửi

```
POST /login

username=minh&password=123456
```

### Form Upload

Khi có upload file

```HTML
<form action="/upload"
      method="POST"
      enctype="multipart/form-data">

    <input type="file" name="avatar">

    <button>Upload</button>

</form>
```

Browser không thể chỉ gửi `avatar=cat.jpg` vì chuỗi "cat.jpg" chỉ là tên của file, không phải dữ liệu của file.

Server nhận được: `cat.jpg` thì không thể khôi phục lại ảnh, vì server không có quyền truy cập vào ổ đĩa của máy người dùng.

Do đó, browser phải:

1. Mở file người dùng đã chọn.
2. Đọc toàn bộ dữ liệu (bytes) của file.
3. Gửi cả metadata và nội dung file trong request multipart/form-data.

Giả sử client chọn file `cat.jpg` để upload

File này gồm:

```
Tên file: cat.jpg

Nội dung: FF D8 FF E0 00 10 4A 46 49 46 ... (là tập các byte)
```

Browser sẽ gửi cả hai:

```
Metadata
──────────────
filename = cat.jpg

Content-Type = image/jpeg

Raw bytes
──────────────
FF D8 FF E0 00 10 4A 46 49 46 ...
```

### enctype

#### enctype là gì?

enctype = Encoding Type

enctype quyết định browser sẽ mã hóa dữ liệu Form như thế nào trước khi gửi.

Ví dụ"

* Nếu `<form enctype="application/x-www-form-urlencoded">` thì body sẽ chuyển thành dạng `username=minh&password=123`
* Nếu `<form enctype="multipart/form-data">` thì body sẽ có dạng:

```
Part 1

Part 2

Part 3

// Mỗi Part là một field.
```

#### Các loại enctype

| enctype                           | Dùng khi             | Body                    |
| --------------------------------- | --------------------- | ----------------------- |
| application/x-www-form-urlencoded | Form thông thường  | key=value&key=value     |
| multipart/form-data               | Upload file           | Chia thành nhiều Part |
| text/plain                        | Debug, rất ít dùng | Văn bản thuần        |

#### Mối liên hệ giữa enctype và Content-Type

`enctype` là cách browser mã hóa dữ liệu form trước khi gửi, còn `Content-Type` là header thực tế trên HTTP request. Khi form dùng `enctype="multipart/form-data"`, browser sẽ tạo body theo dạng multipart và đồng thời gửi header:

```text
Content-Type: multipart/form-data; boundary=----abc123
```

Nói cách khác:

- `enctype` quyết định kiểu mã hóa dữ liệu phía client
- `Content-Type` là thông tin mà server đọc được trên wire
- Với upload file, hai khái niệm này liên quan chặt chẽ với nhau

#### Tại sao upload file phải dùng multipart?

Ví dụ: `avatar=cat.jpg` thì server chỉ biết `cat.jpg` nhưng không biết: kích thước, dữ liệu, bytes, ảnh

Browser cần gửi:

```
Tên file

↓

Kiểu file

↓

Binary

↓

Field khác
```

`application/x-www-form-urlencoded` không làm được như vậy.

### Multipart

#### multipart là gì?

`multi` = nhiều, `part` = phần => body sẽ chia thành nhiều phần.

Ví dụ:

```
POST /upload HTTP/1.1

Content-Type: multipart/form-data;
boundary=----abc123

------abc123

Content-Disposition:
form-data;
name="username"

minh

------abc123

Content-Disposition:
form-data;
name="avatar";
filename="cat.jpg"

Content-Type: image/jpeg

(binary)

------abc123--
```

#### Mỗi Part gồm những gì?

Mỗi Part có cấu trúc giống một HTTP Request thu nhỏ.

```
Part
├── Header
├── Blank Line
└── Body
```

Ví dụ

```
Content-Disposition:
form-data;
name="username"

(blank)

minh
```

Hoặc

```
Content-Disposition:
form-data;
name="avatar";
filename="cat.jpg"

Content-Type:
image/jpeg

(blank)

(binary)
```

> Lưu ý: mỗi part có thể khai báo Content-Type hoặc không. Nếu không khai báo, type mặc định là text/plain (theo RCF 7578)

#### Content-Disposition

Đây là header bắt buộc trong mỗi part.

Ví dụ:

```text
Content-Disposition: form-data; name="avatar"; filename="cat.jpg"
```

Ý nghĩa:

- `form-data`: cho biết đây là dữ liệu của HTML Form
- `name="avatar"`: tên của field trong form, tương ứng với input như `<input type="file" name="avatar">`
- `filename="cat.jpg"`: tên file do client khai báo

Lưu ý quan trọng:

- `filename` không phải đường dẫn
- `filename` không phải nội dung thật của file
- `filename` không đáng tin, vì client có thể tự chỉnh

Ví dụ server sẽ nhận được như sau:

- PHP: `$_FILES["avatar"]`
- ASP.NET: `Request.Form.Files["avatar"]`

Pentester thường thử sửa giá trị này để kiểm tra validation, ví dụ:

- `filename="shell.php"`
- `filename="shell.php.jpg"`
- `filename="../../../shell.php"`

#### Boundary

Boundary là chuỗi dùng để phân tách các part trong body.

Ví dụ:

```text
------abc123
```

Server sẽ đọc `------abc123` là dấu bắt đầu của part mới. Nếu gặp `------abc123--` thì hiểu đó là boundary cuối cùng và kết thúc body.

Browser sinh boundary gần như ngẫu nhiên, ví dụ:

```text
------WebKitFormBoundaryX9N1D2G5P
```

Nhằm tránh trường hợp chuỗi boundary trùng với nội dung thực của file.

#### Browser tạo Multipart như thế nào?

```text
User chọn file
   │
   ▼
Browser đọc file
   │
   ▼
Đọc bytes của file
   │
   ▼
Sinh Boundary ngẫu nhiên
   │
   ▼
Ghép từng Part
   │
   ▼
Thêm Header
(Content-Type: multipart/form-data; boundary=...)
   │
   ▼
Gửi HTTP Request
```

#### Server xử lý Multipart như thế nào?

```text
HTTP Request
   │
   ▼
Đọc Header
   │
   ▼
Content-Type = multipart/form-data
   │
   ▼
Multipart Parser
   │
   ▼
Đọc Boundary
   │
   ▼
Tách từng Part
   ├──────────────┐
   ▼              ▼
Header         Body
   │              │
   └──────┬───────┘
          ▼
Nếu part có filename → File → $_FILES
          │
          ▼
Nếu part không có filename → Text Field → $_POST
```

#### Binary Data

Sau header của part, sẽ có một dòng trắng rồi đến body của part. Đây mới là phần dữ liệu thật của file.

Ví dụ:

```text
Content-Disposition: form-data; name="avatar"; filename="cat.jpg"
Content-Type: image/jpeg

FFD8FFE000104A46... (raw bytes)
```

Ở đây:

- `filename="cat.jpg"` chỉ là metadata
- `FFD8FFE000104A46...` mới là nội dung thật của file

Phần này còn được gọi là body của part, hoặc raw binary data.

#### Hình tổng quát

```text
HTML Form
   │
   ▼
Browser
   │
   ▼
Đọc enctype
   │
   ▼
multipart/form-data
   │
   ▼
Sinh Boundary
   │
   ▼
Ghép từng Part
   ├──────────────┐
   ▼              ▼
Header         Binary Data
   │              │
   └──────┬───────┘
          ▼
HTTP Request
          ▼
Web Server
          ▼
Multipart Parser
          ├──────────────┐
          ▼              ▼
Text Fields      Files
          ▼              ▼
         $_POST         $_FILES
```

#### Pentester cần chú ý

Một file part thường bị sửa ở 3 chỗ:

- `Content-Disposition` → đổi `filename`
- `Content-Type` → đổi từ `image/jpeg` sang `application/octet-stream`
- `Binary Data` → thay bằng payload PHP, HTML hoặc JavaScript, hoặc dùng polyglot

Đây chính là những điểm thường bị khai thác trong các bài upload bypass.

## Luồng upload file với PHP

> Chi tiết về PHP Core, hãy xem tài liệu 4.2.

### Browser gửi file

Khi người dùng chọn file và submit form:

```html
<form action="/upload" method="POST" enctype="multipart/form-data">
    <input type="file" name="avatar">
    <button>Upload</button>
</form>
```

Browser sẽ gửi một request `POST` có body dạng multipart, trong đó có phần file và phần metadata.

### Server nhận request và PHP phân tích

Khi request tới web server, PHP sẽ:

1. đọc header `Content-Type: multipart/form-data`
2. phân tích từng part trong body
3. nếu có file thì tạo một file tạm trên server
4. đưa thông tin của file vào mảng `$_FILES`

Thông thường PHP sẽ tạo ra một cấu trúc như sau:

```php
$_FILES['avatar'] = [
    'name'     => 'cat.jpg',
    'type'     => 'image/jpeg',
    'tmp_name' => '/tmp/phpYg4k92',
    'error'    => 0,
    'size'     => 15420,
];
```

### Code PHP xử lý tiếp

Sau khi có `$_FILES`, code PHP có thể:

- kiểm tra lỗi upload
- kiểm tra extension / MIME / kích thước
- di chuyển file từ thư mục tạm sang thư mục lưu trữ thật bằng `move_uploaded_file()`

Ví dụ:

```php
if (isset($_FILES['avatar']) && $_FILES['avatar']['error'] === 0) {
    $tmp = $_FILES['avatar']['tmp_name'];
    $dest = './uploads/' . $_FILES['avatar']['name'];
    move_uploaded_file($tmp, $dest);
}
```

### 4. Điểm quan trọng cho pentester

- `name` và `type` có thể bị client kiểm soát, nên không nên tin tuyệt đối
- `tmp_name` là đường dẫn file tạm do server tạo, nên đây là dữ liệu đáng tin hơn
- nếu code không kiểm tra kỹ, attacker có thể upload file độc hại và dẫn tới bypass upload hoặc RCE.