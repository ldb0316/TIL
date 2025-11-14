### Get
데이터를 조회할 때 사용

### Post
데이터를 전송할 때 사용(등록, 로그인등)

### Put
데이터를 전체 교체할 때 사용(수정)

### Patch
데이터를 부분 교체할 때 사용(수정)

### Delete
데이터를 삭제할 때 사용

## Bind Annotation

### @ModelAttribute
``` text
쿼리 파라미터나 form에 의한 데이터(application/x-www-form-urlencoded)를 java 객체에 바인딩 할 때 사용한다.
클라이언트가 JSON을 본문으로 보내면 바인딩 할 수 없다. 
form과 쿼리 파라미터에 같은 파라미터명이 존재하면 URL에 있는 쿼리 파라미터가 우선 적용된다. 
Get 에서 사용한다. 
```

### @RequestBody
``` text
요청 본문(JSON 타입 데이터 등)을 java 객체에 자동 바인딩 할 때 사용한다.
Get 요청의 경우 본문이 없기 때문에 RequestBody를 사용하면 쿼리 파라미터를 받아 바인딩할 수 없다. 
java DTO에 setter가 없어도 Jackson 라이브러리 등의 컨버터가 내부적으로 데이터를 바인딩하며, 기본 생성자만 있어도 동작한다.
객체지향 설계에서 불변성을 유지하기 위해 Setter 사용 지양 가능 
POST, PUT, PATCH 에서 사용한다. 
```