# NOTES — BYOL Node.js + Express on Lambda

## Strategy đã chọn

**Strategy A — `serverless-http` adapter**

## Lý do chọn

- Thay đổi code ít nhất: chỉ thêm 1 file `lambda.js` (~3 dòng) và 1 npm dependency, hoàn toàn không chạm vào `app.js`.
- `serverless-http` là thư viện phổ biến, được maintain tốt, hỗ trợ Express out-of-the-box — không cần hiểu sâu về event format của API Gateway.
- Giữ đúng nguyên tắc của bài: framework code (`app.js`) hoàn toàn "Lambda-unaware", adapter nằm riêng ở `lambda.js`.
- So với Strategy C (Lambda Web Adapter), Strategy A không cần thêm Lambda Layer và không phụ thuộc vào shell script — đơn giản hơn cho môi trường workshop.
- So với Strategy D (roll your own), tránh được ~30–80 dòng boilerplate xử lý event → req → res thủ công, giảm nguy cơ bug (đặc biệt path stripping, body parsing).

## Cách implement

```
lambda.js          ← entrypoint mới, wrap app bằng serverless-http
app.js             ← không thay đổi gì
template.yaml      ← Handler: lambda.handler (đã fill TODO)
package.json       ← thêm "serverless-http" vào dependencies
```

`lambda.js`:
```js
const serverless = require('serverless-http');
const app = require('./app');
exports.handler = serverless(app);
```

## Cold start đo được

Đo thực tế trên AWS Lambda (arm64, nodejs22.x, 512 MB, region `us-west-2`):

| Lần đo | Init Duration | Duration | Memory Used |
|--------|--------------|----------|-------------|
| 1      | 278.94 ms    | 32.40 ms | 93 MB       |
| 2      | 301.23 ms    | 16.80 ms | 95 MB       |

**Trung bình cold start: ~290 ms** — nằm trong khoảng ước tính 200–400 ms của Strategy A.

> Lấy từ CloudWatch REPORT lines:
> ```
> sam logs --stack-name byol-node-express --region us-west-2
> ```

**API URL:** `https://j41ku3ijlk.execute-api.us-west-2.amazonaws.com`

## So sánh với các strategy khác (Q4.6)

| Strategy | Cold start ước tính | Độ phức tạp | Ghi chú |
|----------|---------------------|-------------|---------|
| A — serverless-http | 200–400 ms | Thấp | **Đã chọn** |
| B — @vendia/serverless-express | 200–400 ms | Thấp | Fork của A, tương đương |
| C — Lambda Web Adapter | +200 ms over native | Trung bình | Zero JS change nhưng cần Layer + shell script |
| D — Roll your own | Phụ thuộc implementation | Cao | Dễ bug, không nên dùng cho production |
