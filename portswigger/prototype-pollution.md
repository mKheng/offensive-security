# Prototype Pollution

Source: https://www.notion.so/3314d37069a880148212d66e83e78f5a

## Khái Niệm

Prototype pollution xảy ra khi attacker có thể inject property vào prototype của object JavaScript, ví dụ `Object.prototype`. Khi prototype đã bị pollute, mọi object kế thừa từ nó có thể bị ảnh hưởng.

Ví dụ merge function nguy hiểm:

```javascript
function merge(target, source) {
  for (let key in source) {
    if (typeof source[key] === 'object') {
      if (!target[key]) target[key] = {};
      merge(target[key], source[key]);
    } else {
      target[key] = source[key];
    }
  }
}
```

Payload:

```javascript
let malicious = JSON.parse('{"__proto__": {"isAdmin": true}}');
merge({}, malicious);

let user = {};
console.log(user.isAdmin); // true
```

## Ba Thành Phần Exploit

### Source

Input cho phép poison prototype object:

- Query string: `?__proto__[polluted]=true`
- JSON body
- `location.hash`
- Parser recursive/deparam

### Sink

Nơi giá trị bị attacker kiểm soát có thể gây hành vi nguy hiểm:

- `innerHTML`
- `eval()`
- `document.write()`
- `script.src`
- Template engine server-side

### Gadget

Property được app đọc rồi đưa vào sink mà không sanitize. Gadget là cầu nối giữa source và sink.

Ví dụ:

```javascript
let scriptUrl = config.transport_url;
scriptElement.src = scriptUrl;
```

Nếu attacker pollute được `Object.prototype.transport_url = "data:,alert(1)//"`, sink có thể kích hoạt.

## Detect

- Inject `__proto__[testprop]=123`.
- Kiểm tra `Object.prototype.testprop`.
- Dùng DOM Invader trong Burp Browser.
- Dùng tools như `ppfuzz`, `ppmap`, Server-Side Prototype Pollution Scanner.

## Payload Mẫu

```json
{"__proto__": {"polluted": "yes"}}
```

```json
{"constructor": {"prototype": {"polluted": "yes"}}}
```

Client-side XSS:

```json
{"__proto__": {"innerHTML": "<img src=x onerror=alert(1)>"}}
```

## Phòng Chống

- Dùng `Object.create(null)` cho object không cần prototype.
- Filter key nguy hiểm: `__proto__`, `constructor`, `prototype`.
- Dùng `Map` thay plain object khi phù hợp.
- Dùng `Object.hasOwn()` khi iterate.
- Validate input bằng schema như Joi hoặc Zod.

