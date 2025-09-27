# local_passport_auth_service — Hướng dẫn chạy & kiểm thử API (Postman)

Ứng dụng minh hoạ xác thực **Local Strategy** với **Passport.js** dùng **session** của `express-session` và lưu người dùng bằng **MongoDB/Mongoose**. Server lắng nghe cổng **3000** và mount các route dưới tiền tố **/auth**. 

---

## 1) Thành phần & cấu trúc

- `app.js`: 
  - Parse JSON/urlencoded, cấu hình **express-session** (`secret`, `resave:false`, `saveUninitialized:false`). 
  - Khởi tạo **passport** (`initialize` + `session`) và gọi hàm cấu hình `initPassport(passport)`.
  - Kết nối MongoDB: `mongodb://127.0.0.1:27017/passport_local_demo`.
  - Mount route: `app.use("/auth", authRoutes)`; **listen 3000**. 

- `config/passport.js`:
  - Định nghĩa **LocalStrategy**: tìm user theo `username`, kiểm tra mật khẩu bằng `user.isValidPassword(password)`; `serializeUser` lưu `user.id` vào session; `deserializeUser` truy vấn lại user. 

- `routes/auth.js`:
  - `POST /register` — tạo user mới.
  - `POST /login` — `passport.authenticate("local")` (nếu đúng, trả về `{ message, user }`).
  - `GET /logout` — `req.logout()` và trả `{ message }`.
  - `GET /profile` — yêu cầu đã đăng nhập (`req.isAuthenticated()`), trả `{ message, user }`. 

- `models/User.js`:
  - Schema `User { username: unique, password }`.
  - Hook `pre('save')` **hash mật khẩu** bằng `bcryptjs`.
  - Method `isValidPassword(password)` so sánh bằng `bcrypt.compare`. 

- `package.json`: khai báo dependencies: `express`, `express-session`, `mongoose`, `passport`, `passport-local`, `bcryptjs` (và `ejs` nếu cần view). 

---

## 2) Chuẩn bị môi trường

- **Node.js ≥ 18**
- **MongoDB** chạy cục bộ (mặc định URI trong `app.js`)
- **Postman** (hoặc `curl`) để kiểm thử

Cài dependency:

```bash
npm install
```

(Phụ thuộc nằm trong `package.json`) 

---

## 3) Chạy ứng dụng

```bash
node app.js
# Console: "MongoDB connected"
# Console: "Server running on http://localhost:3000"
```
Ứng dụng dùng session để lưu trạng thái đăng nhập của người dùng. 
<img width="1431" height="469" alt="image" src="https://github.com/user-attachments/assets/380e5763-a309-4233-aa60-25f7dc2c5664" />

---

## 4) Hướng dẫn kiểm thử với Postman

**Base URL:** `http://localhost:3000/auth`

> Quan trọng: Gửi các request **trong cùng một phiên Postman** để Postman tự lưu **cookie session** (Passport dựa vào session để nhận diện đăng nhập).

### 4.1 Đăng ký — `POST /register`
- **URL:** `http://localhost:3000/auth/register`
- **Body → raw → JSON:**
```json
{ "username": "admin", "password": "12345" }
```
- **Kết quả mong đợi:** 
```json
{ "message": "User registered successfully" }
```
<img width="1274" height="759" alt="image" src="https://github.com/user-attachments/assets/00528ef1-d6a2-4d63-b4e5-f3102b85122e" />

(Mật khẩu sẽ được **hash** trước khi lưu). 

> Nếu trùng `username` hoặc thiếu trường bắt buộc, API trả lỗi `400`. 
<img width="1282" height="819" alt="image" src="https://github.com/user-attachments/assets/71eea182-605f-48d6-8038-8b0ee4d1b338" />

### 4.2 Đăng nhập — `POST /login`
- **URL:** `http://localhost:3000/auth/login`
- **Body → raw → JSON:**
```json
{ "username": "admin", "password": "12345" }
```
- **Xử lý:** `passport.authenticate("local")` sẽ kiểm tra tồn tại user và đúng mật khẩu. 
- **Kết quả mong đợi:**
```json
{ "message": "Logged in successfully", "user": { "_id": "...", "username": "admin", ... } }
```
<img width="1280" height="932" alt="image" src="https://github.com/user-attachments/assets/cb068784-e9ca-4b4b-a383-568a785fc1f8" />

Sau khi đăng nhập thành công, session được tạo và gắn với request hiện tại.

> Nếu sai thông tin, middleware xác thực sẽ trả lỗi 401 (mặc định của Passport Local). 
<img width="1274" height="712" alt="image" src="https://github.com/user-attachments/assets/fb9c2595-3bf1-4e32-b9c2-6210cddd82a0" />

### 4.3 Truy cập trang cá nhân — `GET /profile`
- **URL:** `http://localhost:3000/auth/profile`
- **Yêu cầu:** đã đăng nhập (cùng session).
- **Kết quả mong đợi (đã đăng nhập):**
```json
{ "message": "Profile data", "user": { "_id": "...", "username": "admin", ... } }
```
<img width="1277" height="865" alt="image" src="https://github.com/user-attachments/assets/be7fd407-747d-4d27-b037-0b298964d5ab" />

- **Khi chưa đăng nhập:** 
```json
{ "message": "Not authenticated" }
```
kèm mã **401**. 
<img width="1277" height="718" alt="image" src="https://github.com/user-attachments/assets/6b0c1b95-a4bc-4136-8982-ba67dfd4e3dc" />

### 4.4 Đăng xuất — `GET /logout`
- **URL:** `http://localhost:3000/auth/logout`
- **Kết quả mong đợi:**
```json
{ "message": "Logged out" }
```
<img width="1277" height="723" alt="image" src="https://github.com/user-attachments/assets/9d14cb36-db71-4735-a0ea-71740b504b35" />

Sau khi logout, session bị huỷ; gọi lại `/profile` sẽ nhận **401**. 
<img width="1277" height="718" alt="image" src="https://github.com/user-attachments/assets/a69e3a42-57f6-4294-8993-19009291d519" />

---

## 5) Ghi chú bảo mật & cấu hình

- **Session secret:** `mysecretkey` chỉ dùng cho môi trường học tập; khi triển khai thực tế hãy đổi sang biến môi trường mạnh. 
- **Cookie bảo mật:** session cookie là HttpOnly; khi triển khai HTTPS nên bật `cookie.secure = true` (hiện không thiết lập ở `app.js`). 
- **Hash mật khẩu:** thực hiện ở hook `pre('save')` với `bcrypt.hash`, xác minh bằng `bcrypt.compare`. 
- **Luồng Passport:** LocalStrategy → `serializeUser` (lưu `user.id`) → `deserializeUser` (nạp lại user). 

---

## 6) Xử lý sự cố

- **Không giữ phiên đăng nhập trong Postman:** hãy gửi `/login` và đảm bảo request tiếp theo dùng **cùng session/cookie**.
- **`401 Not authenticated` ở `/profile`:** chưa đăng nhập hoặc session hết hạn/đã huỷ; login lại. 
- **Lỗi kết nối MongoDB:** khởi chạy MongoDB cục bộ và kiểm tra URI `mongodb://127.0.0.1:27017/passport_local_demo`. 
- **Cổng 3000 bận:** dừng tiến trình khác hoặc thay đổi cổng trong `app.js`. 

---

## 7) Tài liệu tham chiếu nhanh (mã nguồn)

- Cấu hình server, session, passport khởi tạo, kết nối DB: `app.js`.  
- Chiến lược Passport Local & (de)serialize: `config/passport.js`.   
- Endpoint đăng ký/đăng nhập/logout/profile: `routes/auth.js`.  
- Mô hình người dùng và hash mật khẩu: `models/User.js`.   
- Danh sách dependencies: `package.json`. 

---
