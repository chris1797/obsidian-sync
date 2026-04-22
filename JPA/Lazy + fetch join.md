### "컬렉션을 하나 이상 즉시 로딩하는 것은 권장하지 않는다"

컬렉션을 `@OneToMany`, `@ManyToMany` 등의 관계로 선언하면서 `fetch = FetchType.EAGER`로 설정하지 말라는 뜻이다.

## ✅ 왜 EAGER 컬렉션이 문제일까?

### 1️⃣ **[[N+1 문제]] 문제 발생 가능성**

- 즉시 로딩은 엔티티가 조회될 때 연관된 컬렉션들도 _같이_ 쿼리해서 가져오게 되기 때문에 N+1 문제가 발생할 가능성이 큼.
- 특히 컬렉션이 많거나 크면 쿼리 수가 폭증함.

### 2️⃣ JPA가 컬렉션을 JOIN해서 조회하지 못함 (LIMIT, OFFSET 문제)

- JPA에서는 컬렉션 `fetch join`을 하게 되면 중복 데이터로 인해 페이징이 불가능해짐.
```sql
-- 잘못된 페이징
SELECT p FROM Post p JOIN FETCH p.comments
```
- `Post` 는 10건인데, `comments`가 여러 개라면 결과가 수십건이 될 수 있음.
- `Page<Post>` 와 같은 페이징 조회가 아예 불가능해짐.

### 3️⃣ 하나 이상의 EAGER 컬렉션 → JPA가 어떤 방식으로 조회할지 예측 불가

- Hibernate에서는 **하나의 컬렉션만 Fetch Join 가능**하고, **여러 컬렉션을 EAGER로 설정하면 기본적으로 `Cartesian product`(곱집합)** 문제가 생김.

### ✅ 결론: 컬렉션은 Lazy 로딩을 기본으로 쓰자.
