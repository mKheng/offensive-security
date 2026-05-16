# JSON Web Token

## JWT Là Gì?

JWT là định dạng chuẩn để gửi dữ liệu JSON được ký giữa các hệ thống. JWT thường dùng cho authentication/authorization.

Cấu trúc:

```text
Header.Payload.Signature
```

- Header: thuật toán ký, ví dụ `alg`, và loại token `typ`.
- Payload: claims như user id, role, exp.
- Signature: chữ ký đảm bảo integrity.

Payload chỉ là base64url encoding, không phải encryption. Không đặt password hoặc secret vào payload.

## JWT, JWS, JWE, JWK, JWA

- JWT: container claims.
- JWS: signed JWT, đảm bảo integrity.
- JWE: encrypted JWT, đảm bảo confidentiality.
- JWK: JSON Web Key.
- JWA: JSON Web Algorithms.

## Attack Vectors

### Unverified Signature

Server đọc payload nhưng không verify chữ ký. Attacker sửa claim như:

```json
{"sub": "administrator"}
```

### `alg: none`

Một số thư viện cũ hoặc cấu hình sai có thể chấp nhận token không ký.

### Algorithm Confusion

Server dùng RSA (`RS256`) nhưng attacker đổi sang HMAC (`HS256`) và dùng public key làm HMAC secret.

Quy trình:

1. Lấy public key từ JWKS hoặc certificate.
2. Đổi `alg` sang `HS256`.
3. Ký token bằng public key như secret.
4. Server verify nhầm và chấp nhận.

### Weak Secret

Nếu dùng HS256 với secret yếu, có thể crack:

```bash
hashcat -m 16500 jwt.txt wordlist.txt
```

Sau khi có secret, attacker tự ký token mới.

### Header Injection

Các vector đáng chú ý:

- `kid` injection nếu server dùng `kid` để đọc file hoặc query DB.
- `jwk` header injection nếu server tin public key do token tự cung cấp.
- `jku` spoofing nếu server fetch key từ URL attacker kiểm soát.

## Tools

- Burp Suite JWT Editor.
- `jwt_tool`.
- `hashcat` hoặc `john` cho weak secret.

## Phòng Chống

- Luôn verify signature.
- Không chấp nhận `alg: none`.
- Pin thuật toán phía server, không tin `alg` từ token.
- Không dùng secret yếu.
- Validate `kid`, `jwk`, `jku` nghiêm ngặt.
- Dùng allowlist JWKS host nếu cần remote key.
