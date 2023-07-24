---
layout: post
title: ⚙️ Spring Security 설정
tags : [spring]
---


&nbsp;

&nbsp;

요청 기반 권한 설정과 인가에 대해 알아보기

---

## 🚪 HTTP 요청에 대한 권한 설정

&nbsp;

```java

http.authorizeRequests((authorize) -> authorize.anyRequest().authenticated())

```

authorizeRequests() 메서드를 사용하여 모든 요청을 인증된 사용자만 허용

&nbsp;

&nbsp;

#### 엔드포인트에 대한 권한 설정

&nbsp;

```java
@Bean
SecurityFilterChain web(HttpSecurity http) throws Exception {
 http
  .authorizeHttpRequests((authorize) -> authorize
   .requestMatchers("/endpoint").hasAuthority('USER')
   .anyRequest().authenticated()
  )
        // ...

 return http.build();
}
```

&nbsp;

"/endpoint" 엔드포인트를 USER 권한을 가진 사용자만 접근 가능하도록 설정
다른 모든 요청에 대해서는 인증만 되어 있다면 접근 허용

&nbsp;

&nbsp;

#### 테스트로 권한 검증하기

&nbsp;

```java
@WithMockUser(authorities="USER")
@Test
void endpointWhenUserAuthorityThenAuthorized() {
    this.mvc.perform(get("/endpoint"))
        .andExpect(status().isOk());
}

@WithMockUser
@Test
void endpointWhenNotUserAuthorityThenForbidden() {
    this.mvc.perform(get("/endpoint"))
        .andExpect(status().isForbidden());
}

@Test
void anyWhenUnauthenticatedThenUnauthorized() {
    this.mvc.perform(get("/any"))
        .andExpect(status().isUnauthorized())
}


```

&nbsp;

&nbsp;

## 🎯 요청 매칭

Spring Security에서 요청을 매칭하는 방법은 모든 요청을 매칭하거나 URI 패턴을 사용하여 매칭
URI 패턴 매칭은 Ant를 사용하거나 정규표현식을 사용한다

&nbsp;


#### Ant

&nbsp;


"/resource" 디렉토리 아래의 모든 엔드포인트를 매칭할 때

&nbsp;

``` java
http
    .authorizeHttpRequests((authorize) -> authorize
        .requestMatchers("/resource/**").hasAuthority("USER")
        .anyRequest().authenticated()
    )

```

요청에서 경로 값을 추출하여 사용할 수 있음

&nbsp;

``` java
http
    .authorizeHttpRequests((authorize) -> authorize
        .requestMatchers("/resource/{name}").access(new WebExpressionAuthorizationManager("#name == authentication.name"))
        .anyRequest().authenticated()
    )

```

"/resource/{name}" 패턴의 엔드포인트에 접근하는 경우, 요청에서 "name" 파라미터 값을 추출하여 해당 값이 사용자의 이름과 일치할 때에만 접근을 허용

&nbsp;

&nbsp;

#### 정규표현식

**를 사용하여 하위 디렉토리를 더 엄격하게 매칭하고 싶을 때 유용

``` java
http
    .authorizeHttpRequests((authorize) -> authorize
        .requestMatchers(RegexRequestMatcher.regexMatcher("/resource/[A-Za-z0-9]+")).hasAuthority("USER")
        .anyRequest().denyAll()
    )

```

정규 표현식인 "RegexRequestMatcher"를 사용하여 알파벳과 숫자로만 이루어지고 "USER"권한을 가진 사용자만 접근이 가능하도록 설정

#### HTTP 메서드별로 권한 매칭

&nbsp;

권한 부여를 읽기 또는 쓰기 권한으로 구분하여 설정하고자 할 때 유용

&nbsp;

``` java
http
    .authorizeHttpRequests((authorize) -> authorize
        .requestMatchers(HttpMethod.GET).hasAuthority("read")
        .requestMatchers(HttpMethod.POST).hasAuthority("write")
        .anyRequest().denyAll()
    )

```
&nbsp;

 "GET" 요청에 대해 읽기 권한을 요구하고, "POST" 요청에 대해 쓰기 권한을 요구하며, 그 외의 다른 요청은 모두 거부


&nbsp;

&nbsp;

## 🔑 요청 인가하기

&nbsp;

요청이 매칭되면 여러가지 방법으로 해당 요청을 인가할 수 있다

&nbsp;

> `permitAll`: 요청에 대한 인가가 필요 없으며 public 엔드포인트
> 인증(Authentication)은 세션으로부터 가져오지 않음

&nbsp;

> `denyAll`: 요청은 어떠한 경우에도 허용되지 않음
> 인증(Authentication)은 세션으로부터 가져오지 않음

&nbsp;

> `hasAuthority`: 요청은 인증(Authentication)이 주어진 값과 일치하는 GrantedAuthority를 가져옴

&nbsp;

> `hasRole`: hasAuthority의 단축키로, 자동으로 "ROLE_" 접두사를 붙여주는 역할

&nbsp;

> `hasAnyAuthority`: 요청은 인증(Authentication)이 주어진 여러 값 중 하나와 일치하는 GrantedAuthority를 가져옴

&nbsp;

> `hasAnyRole`: hasAnyAuthority의 단축키로, 자동으로 "ROLE_" 접두사를 붙여주는 역할

&nbsp;

>`access`: 사용자 정의 AuthorizationManager를 사용하여 요청에 대한 접근 권한을 결정

&nbsp;

각 규칙은 선언된 순서대로 고려된다

&nbsp;

&nbsp;


---

출처 : [Spring-security-sample](https://github.com/spring-projects/spring-security-samples/tree/main/servlet/spring-boot/java/authentication/username-password/user-details-service/custom-user),[Spring docs](https://docs.spring.io/spring-security/reference/index.html)

&nbsp;

&nbsp;
