---
description: 하이버네이트의 `@SQLRestriction`에 대해
---

# JPA 전역에 쿼리 제한을 적용하는 법

#### `@Where` 에서 `@SQLRestriction` 으로

* use `@SQLRestriction`

```
//  
// Source code recreated from a .class file by IntelliJ IDEA  
// (powered by FernFlower decompiler)  
//  
  
package org.hibernate.annotations;  
  
import java.lang.annotation.ElementType;  
import java.lang.annotation.Retention;  
import java.lang.annotation.RetentionPolicy;  
import java.lang.annotation.Target;  
  
/** @deprecated */  
@Target({ElementType.TYPE, ElementType.METHOD, ElementType.FIELD})  
@Retention(RetentionPolicy.RUNTIME)  
@Deprecated(  
  since = "6.3"  
)  
public @interface Where {  
  String clause();  
}
```

JPA를 사용하면서 Soft Delete를 적용하면, 노출 범위를 제한하기 위해 삭제된 게시물은 조회하지 않는 제한이 필요하다.

* 여기에 `@Where` 이라는 어노테이션으로 원하는 제한을 걸 수 있었는데, Hibernate 6.3부터 Deprecated되었다고 나와있다.

```
@Entity
 @Where(clause = "status <> 'DELETED'")
 class Document {
     ...
     @Enumerated(STRING)
     Status status;
     ...
 }
```

기존에는 위와 같이 제한을 걸었다.

공식문서

https://docs.jboss.org/hibernate/orm/6.5/javadocs/org/hibernate/annotations/Where.html

* `SQLRestriction` 을 사용하라는 문구.

사용방법은 이전과 99퍼센트 유사하다.

```
 @Entity
 @SQLRestriction("status <> 'DELETED'")
 class Document {
     ...
     @Enumerated(STRING)
     Status status;
     ...
 }
```

***

#### 그런데.. 이거 그냥 적용해도 되나?

* 공식문서에 나와있는 내용을 한번 살펴보고 가자.

> Note that `@SQLRestriction`s are always applied and cannot be disabled. Nor may they be parameterized. They're therefore _much_ less flexible than [filters](https://docs.jboss.org/hibernate/orm/6.5/javadocs/org/hibernate/annotations/Filter.html).

* `@SQLRestriction` 을 사용하면 비활성화가 불가능 하다는 말이다.

**이 친구가 참 편리한데, 문제가 바로 Entity에 박아놓은 순간 해제가 안된다는 점이다.**

이전 프로젝트에서 soft-delete를 적용하고 `@Where` 을 사용하여 제한을 걸었던 적이 있다.

* 어느날 기획팀에서 걸려온 요청 : 음 그 지웠던 것도 확인할 수 있게 해주세요! :tada:

해당 요구사항을 받아들이고 아차싶었다. 이미 배포는 `@Where` 을 사용하여 배포가 되어있고, 테스트코드들도 이에 맞춰서 작성되어있다. 모든 서비스 코드는 JPA 엔티티를 도메인 객체로 사용하고 있었다.

* **JPA 설정을 바꾸니 테스트가 터진다.**

결국 JPQL에서 쿼리로 제한을 직접 거는 방향으로 수정했다. 단순하게 사용자에게 보여지는 내용만 생각한다면 괜찮지 않을까 생각이 들 수 있지만, 어드민까지 생각하면 해당 어노테이션은 다소 위험한 것 같다.

그렇기에 문서에서도 적어놓은 것 같다.

***

**배운 점**

* JPA가 서비스 로직에 많이 침투해있었다.
  * 도메인 객체와 JPA 객체를 분리하는 것도 앞으로는 고려해볼 것 같다.
* 함부로 전역적인 제한을 걸지않도록 조심하자.
  * 당장은 필요없어도 나중에 필요할 수 있다
