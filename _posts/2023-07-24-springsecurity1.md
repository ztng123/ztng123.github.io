---
layout: post
title: 🔒 Spring Security 로그인
tags : [spring]
---

&nbsp;

&nbsp;

사용자 정의 계정에서 로그인 하는 방법 알아보기

---

## ✏️ 의존성 추가

&nbsp;

```gradle
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-security' 
    implementation 'org.springframework.boot:spring-boot-starter-thymeleaf'

```

&nbsp;

- Spring Security 의존성 추가

&nbsp;

- HTML 템플릿 사용하기 위해 Thymeleaf 의존성 추가

&nbsp;

<img width="355" alt="스크린샷 2023-07-24 오전 11 09 58" src="https://github.com/spring-projects/spring-security-samples/assets/53010592/cfaccfd8-1ee5-4bcd-8cd4-c6c4c5b32e34">

&nbsp;

이제 인증 받은 사용자만 접근이 가능해졌고 위는 폼 로그인 방식

모든 요청은 인증이 되어야 자원에 접근이 가능하고 인증 방식은 폼 로그인 방식과 httpBasic로그인 방식을 제공한다

&nbsp;

&nbsp;

## 📄 커스텀 페이지

기본페이지와 로그인페이지 커스텀하기

<img width="251" alt="스크린샷 2023-07-24 오후 1 21 38" src="https://github.com/spring-projects/spring-security-samples/assets/53010592/ff553f46-0c3c-4651-a82f-5d62e8e20bc0">


templates 폴더 안에 html 파일 생성하기


&nbsp;

&nbsp;

1. index.html

&nbsp;

<img width="251" alt="스크린샷 2023-07-24 오후 12 27 07" src="https://github.com/spring-projects/spring-security-samples/assets/53010592/12cac81e-e241-4fd3-9e72-6ae71dbe995a">

&nbsp;

``` html
<html xmlns:th="https://www.thymeleaf.org">
<head>
    <title>Hello Security</title>
</head>
<body>
<div>
    <h1>Hello Security</h1>
    <form th:action="@{/logout}" method="post">
        <input type="submit" value="Logout"/>
    </form>
</div>
</body>
</html>
```

&nbsp;

&nbsp;


2. login.html

&nbsp;

<img width="1371" alt="스크린샷 2023-07-24 오후 12 11 03" src="https://github.com/spring-projects/spring-security-samples/assets/53010592/1c4dd028-0617-488d-ae55-bb51e193fa9e">

&nbsp;

``` html
<html xmlns:th="https://www.thymeleaf.org">
<head>
    <title>Custom Log In Page</title>
</head>
<body>
<h1>Custom Log In Page</h1>
<div>
    <form name="f" th:action="@{/login}" method="post">
        <fieldset>
            <legend>Please Login</legend>
            <div th:if="${param.error}" class="alert alert-error">Invalid
                username and password.</div>
            <div th:if="${param.logout}" class="alert alert-success">You
                have been logged out.</div>
            <label for="username">Username</label> <input type="text"
                                                          id="username" name="username" /> <label for="password">Password</label>
            <input type="password" id="password" name="password" />
            <div class="form-actions">
                <button type="submit" class="btn">Log in</button>
            </div>
        </fieldset>
    </form>
</div>
</body>
</html>

```

&nbsp;

&nbsp;


## 📝 컨트롤러 작성

&nbsp;

1. IndexController

&nbsp;

"/" 경로로 접근할 때 "index" 뷰 보여줌

&nbsp;

```java
@Controller
public class IndexController {

    @GetMapping("/")
    public String index() {
        return "index";
    }

}

```

2. LoginController

&nbsp;

"/login" 경로로 접근할 때 "login" 뷰 보여줌

&nbsp;

```java
@Controller
public class LoginController {

    @GetMapping("/login")
    String login() {
        return "login";
    }

}

```

&nbsp;

&nbsp;

## 🔧 Spring Security 초기화

&nbsp;

Spring Security의 설정과 필터 체인을 초기화

&nbsp;

``` java
public class SecurityWebApplicationInitializer extends AbstractSecurityWebApplicationInitializer {

    @Override
    protected boolean enableHttpSessionEventPublisher() {
        return true;
    }

}
```

&nbsp;

- 'AbstractSecurityWebApplicationInitializer' 클래스는 Spring Security 초기화 작업 수행
- 'enableHttpSessionEventPublisher()' 메서드가 오버라이드되어 있어 HttpSessionEventPublisher를 활성화
- HttpSessionEventPublisher를 활성화하면 로그인 및 로그아웃과 같은 세션 이벤트를 더욱 쉽게 처리 가능

&nbsp;

&nbsp;

## ⚙️ Spring Security 설정

&nbsp;

```java
@Configuration
@EnableWebSecurity
public class SecurityConfiguration {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        
        http
                .authorizeRequests((authorize) -> authorize
                        .anyRequest().authenticated()
                )
                .formLogin((form) -> form
                        .loginPage("/login")
                        .permitAll()
                );
        
        return http.build();
    }

    
    @Bean
    public UserDetailsService userDetailsService() {
        UserDetails user = User.withDefaultPasswordEncoder()
                .username("user")
                .password("password")
                .roles("USER")
                .build();
        return new InMemoryUserDetailsManager(user);
    }
    

}


```

```java

http.authorizeRequests((authorize) -> authorize.anyRequest().authenticated())

```

authorizeRequests() 메서드를 사용하여 모든 요청을 인증된 사용자만 허용

- 커스텀 로그인 폼

``` java
http.formLogin((form) -> form.loginPage("/login").permitAll());
```

- 폼 로그인

``` java
http.formLogin(withDefaults());
```

- In-Memory 인증

``` java

@Bean
    public UserDetailsService userDetailsService() {
        UserDetails user = User.withDefaultPasswordEncoder()
                .username("user")
                .password("password")
                .roles("USER")
                .build();
        return new InMemoryUserDetailsManager(user);
    }
```


&nbsp;

&nbsp;

## 🔧 Spring MVC 초기화

&nbsp;

Spring MVC의 설정과 DispatcherServlet을 초기화 및 설정해 MVC 패턴을 쉽게 구성 할 수 있다

&nbsp;

> `DispatcherServlet` :  클라이언트로부터 들어오는 모든 요청을 처리하고 해당 요청에 대해 적절한 컨트롤러로 전달하는 중요한 컴포넌트

&nbsp;

``` java
public class MvcWebApplicationInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {

    @Override
    protected Class<?>[] getRootConfigClasses() {
        return null;
    }

    @Override
    protected Class<?>[] getServletConfigClasses() {
        return new Class[] { ApplicationConfiguration.class };
    }

    @Override
    protected String[] getServletMappings() {
        return new String[] { "/" };
    }

    @Override
    protected Filter[] getServletFilters() {
        return new Filter[] { new HiddenHttpMethodFilter() };
    }

}
```

&nbsp;

AbstractAnnotationConfigDispatcherServletInitializer 클래스는 Spring MVC 웹 애플리케이션을 구성하기 위한 여러 초기화 작업을 수행
👉 상속하면 자동으로 DispatcherServlet을 설정하고 Spring MVC 설정을 구성

&nbsp;

- Root Application Context의 설정 클래스 지정

``` java
@Override
    protected Class<?>[] getRootConfigClasses() {
        return null;
    }
```

null을 반환하고 있으므로 Root Application Context를 설정하지 않는 것을 의미
>`Root Application Context`: 보통 데이터베이스 연결, 서비스, 보안 등의 빈들을 정의하는 데 사용

&nbsp;

- Servlet Application Context의 설정 클래스를 지정

``` java
@Override
    protected Class<?>[] getServletConfigClasses() {
        return new Class[] { ApplicationConfiguration.class };
    }
```

ApplicationConfiguration.class를 반환하고 있으므로 ApplicationConfiguration 클래스에 정의된 빈들이 Servlet Application Context에 등록됨
>`Servlet Application Context`: 주로 컨트롤러와 뷰 리졸버 등과 같이 웹 계층 관련 빈들을 정의하는 데 사용

&nbsp;

- DispatcherServlet의 매핑을 지정

``` java
@Override
    protected String[] getServletMappings() {
        return new String[] { "/" };
    }
```

 "/"를 반환하고 있으므로 DispatcherServlet은 기본적으로 모든 요청에 대해 매핑

&nbsp;

- DispatcherServlet 앞에 적용할 필터들을 지정
  
``` java
 @Override
    protected Filter[] getServletFilters() {
        return new Filter[] { new HiddenHttpMethodFilter() };
    }
```

HiddenHttpMethodFilter를 반환하고 있으므로 HiddenHttpMethodFilter가 DispatcherServlet 앞에서 작동
>`HiddenHttpMethodFilter`: HTTP POST 메서드를 사용하여 DELETE, PUT, PATCH와 같은 추가 HTTP 메서드를 시뮬레이션하는 데 사용

&nbsp;

&nbsp;

## ⚙️ Spring MVC 설정

&nbsp;

웹 애플리케이션의 MVC 설정, ViewResolver 설정, 정적 리소스 처리 등을 처리

Spring MVC의 구성을 커스터마이즈 하는 메서드들을 오버라이드

&nbsp;

``` java
@EnableWebMvc
@Configuration
public class WebMvcConfiguration implements WebMvcConfigurer {

    @Override
    public void addViewControllers(ViewControllerRegistry registry) {
        registry.setOrder(Ordered.HIGHEST_PRECEDENCE);
    }

    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/resources/**")
                .addResourceLocations("classpath:/resources/")
                .setCachePeriod(31556926);
        registry.setOrder(Ordered.HIGHEST_PRECEDENCE);
    }

    @Bean
    public ViewResolver viewResolver(ISpringTemplateEngine templateEngine) {
        ThymeleafViewResolver resolver = new ThymeleafViewResolver();
        resolver.setTemplateEngine(templateEngine);
        resolver.setCharacterEncoding("UTF-8");
        return resolver;
    }

    @Bean
    public SpringTemplateEngine templateEngine(ITemplateResolver templateResolver) {
        SpringTemplateEngine engine = new SpringTemplateEngine();
        engine.setEnableSpringELCompiler(true);
        engine.setTemplateResolver(templateResolver);
        return engine;
    }

    @Bean
    public SpringResourceTemplateResolver templateResolver() {
        SpringResourceTemplateResolver resolver = new SpringResourceTemplateResolver();
        resolver.setPrefix("classpath:/templates/");
        resolver.setSuffix(".html");
        resolver.setTemplateMode(TemplateMode.HTML);
        return resolver;
    }

}
```

&nbsp;

- 특정 URL 경로에 대한 View(Controller)를 간단하게 등록

&nbsp;  

``` java
@Override
    public void addViewControllers(ViewControllerRegistry registry) {
        registry.setOrder(Ordered.HIGHEST_PRECEDENCE);
    }
```

registry.setOrder(Ordered.HIGHEST_PRECEDENCE)를 통해 등록된 ViewController의 우선순위를 가장 높게 설정

&nbsp;

- 정적 리소스(이미지, CSS, JavaScript 등)에 대한 핸들러를 등록

&nbsp;

``` java
@Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/resources/**")
                .addResourceLocations("classpath:/resources/")
                .setCachePeriod(31556926); //리소스 캐싱 기간 설정
        registry.setOrder(Ordered.HIGHEST_PRECEDENCE);
    }
```

"/resources/**" 경로에 해당하는 요청을 classpath의 "/resources/" 디렉토리에서 처리하도록 설정

&nbsp;

- ThymeleafViewResolver를 생성하고 템플릿 엔진인 Thymeleaf와 연결하는 빈을 등록

&nbsp;

``` java
@Bean
    public ViewResolver viewResolver(ISpringTemplateEngine templateEngine) {
        ThymeleafViewResolver resolver = new ThymeleafViewResolver();
        resolver.setTemplateEngine(templateEngine); //Thymeleaf 템플릿 엔진과 연결
        resolver.setCharacterEncoding("UTF-8");
        return resolver;
    }
```

&nbsp;

- SpringTemplateEngine을 생성하고 Thymeleaf 템플릿 엔진과 관련된 설정들을 수행하는 빈을 등록

&nbsp;

``` java
@Bean
    public SpringTemplateEngine templateEngine(ITemplateResolver templateResolver) {
        SpringTemplateEngine engine = new SpringTemplateEngine();
        engine.setEnableSpringELCompiler(true); //스프링 EL 컴파일러 활성화
        engine.setTemplateResolver(templateResolver); //템플릿 리졸버와 연결
        return engine;
    }
```

&nbsp;

- SpringResourceTemplateResolver를 생성하고 템플릿 파일들의 위치와 확장자, 템플릿 모드 등과 관련된 설정들을 수행하는 빈을 등록

&nbsp;

``` java
@Bean
    public SpringResourceTemplateResolver templateResolver() {
        SpringResourceTemplateResolver resolver = new SpringResourceTemplateResolver();
        resolver.setPrefix("classpath:/templates/"); //템플릿 파일들 디렉토리 지정
        resolver.setSuffix(".html"); //템플릿 파일 확장자 지정
        resolver.setTemplateMode(TemplateMode.HTML);
        return resolver;
    }
```

&nbsp;

&nbsp;

---

위 방법을 통해 커스텀한 로그인 페이지에서 계정이름 "user" 비밀번호 "password"로 로그인하는 것에 성공 !

&nbsp;
출처 : [Spring-security-sample](https://github.com/spring-projects/spring-security-samples/tree/main/servlet/spring-boot/java/authentication/username-password/user-details-service/custom-user),[Spring docs](https://docs.spring.io/spring-security/reference/index.html)
&nbsp;

&nbsp;
