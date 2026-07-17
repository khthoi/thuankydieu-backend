# thuankydieu-backend

Backend API cho website **thuankydieu.vn** — trang công khai quảng bá và bán cuốn sách *"Sức khỏe trong lòng bàn tay"* (Đông Y sỹ Trần Tấn Phước), gồm: bán sách giấy & PDF, thư viện đọc online, và hỏi đáp AI dựa trên nội dung sách (không bịa đặt, có kiểm duyệt).

Xây bằng **NestJS 11** (TypeScript).

## Chức năng (theo thiết kế — đang xây dựng)

- **Bán hàng:** đơn hàng sách giấy (COD hoặc chuyển khoản QR, giao qua GHN), bán PDF, bán gói lượt hỏi AI.
- **Thư viện:** đọc thử ~25% miễn phí; mua PDF thì mở khoá đọc 100% theo tài khoản.
- **Hỏi đáp AI:** 10 câu miễn phí mỗi thiết bị, sau đó đăng nhập và mua lượt; câu trả lời chỉ dựa trên nội dung sách, có lớp bộ nhớ câu trả lời đã được người duyệt.
- **Quản trị:** quản lý đơn/sản phẩm/bài viết, duyệt câu trả lời AI, chỉnh tham số cấu hình — thiết kế cho một người vận hành.

## Trạng thái

🚧 **Mới khởi tạo khung** — chưa có module nghiệp vụ. Thiết kế chi tiết (phân tích nhu cầu, FDD, ERD 15 bảng) nằm trong bộ tài liệu nội bộ của dự án, không kèm trong repo này.

## Yêu cầu

- Node.js ≥ 20 (đang dev trên v22)
- npm

## Cài đặt & chạy

```bash
npm install

# chạy dev (watch) — quy ước chạy port 3001 để tránh đụng frontend Next (3000)
PORT=3001 npm run start:dev      # macOS/Linux
$env:PORT=3001; npm run start:dev # Windows PowerShell

# build & chạy production
npm run build
npm run start:prod
```

## Lệnh khác

```bash
npm run lint      # eslint --fix
npm run format    # prettier
npm run test      # unit test (jest)
npm run test:e2e  # e2e test
```

## Cấu hình môi trường

Bí mật/cấu hình đặt trong file `.env` (đã gitignore — không commit). Sẽ bổ sung `.env.example` khi bắt đầu có biến thật (chuỗi kết nối CSDL, khoá dịch vụ AI…).

## Repo liên quan

- Frontend: [thuankydieu-frontend](https://github.com/khthoi/thuankydieu-frontend) — Next.js 16.
