---
layout: post
title: 🗝️ Spring Security 비밀번호
tags: [spring, MySQL]
---

&nbsp;

&nbsp;

비밀번호 저장 메서드는 총 3가지로 In-memory,JDBC,LDAP 방식이 있고
In-memroy와 JDBC 방식에 대해 알아보기

---

## 💾 In- memory

&nbsp;

애플리케이션 자체의 메모리에 사용자 정보를 저장하는 간단한 방법

&nbsp;

```java
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

## 🚪 JDBC

&nbsp;

데이터베이스를 사용하여 사용자 인증과 권한 부여를 처리

&nbsp;

#### 의존성 추가

&nbsp;

build.gradle에 의존성 추가

&nbsp;

```gradle
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-security'
    implementation 'org.springframework.boot:spring-boot-starter-jdbc'
    implementation 'com.mysql:mysql-connector-j'

}
```

&nbsp;

&nbsp;

📌 Artifact ID 변경

&nbsp;

```gradle
implementation 'mysql:mysql-connector-java'
```

&nbsp;

처음 의존성 추가할 때 위와 같이 추가했는데 mysql:mysql-connector-java를 찾을 수 없다는 오류가 발생

현재 사용하고 있는 Spring-Boot 3.1.2 와 호환되지 않는건가 Dependency 찾아보니 Artifact ID가 변경된 것이었다

&nbsp;

<img width="854" alt="스크린샷 2023-07-25 오후 5 21 54" src="https://github.com/spring-projects/spring-security-samples/assets/53010592/f110e7eb-fd08-4d48-995c-64e92e557caa">

&nbsp;
바뀐 것으로 수정했더니 정상적으로 의존성 추가되었다

&nbsp;

&nbsp;

#### 데이터베이스 스키마 생성

&nbsp;
📌 데이터베이스 생성

&nbsp;

```sql
mysql> CREATE DATABASE jdbc;
```

&nbsp;

&nbsp;

📌 데이터베이스 선택

&nbsp;

```sql
mysql> USE jdbc;
```

&nbsp;

&nbsp;

📌 테이블 생성

&nbsp;

```sql
mysql> CREATE TABLE users(
    -> username VARCHAR(50) NOT NULL PRIMARY KEY,
    -> password VARCHAR(100) NOT NULL,
    -> enabled boolean not null
    -> );
Query OK, 0 rows affected (0.05 sec)

mysql> CREATE TABLE authorities(
    -> username VARCHAR(50) NOT NULL,
    -> authority VARCHAR(50) NOT NULL,
    -> constraint fk_authorities_users foreign key(username) references users(username)
    -> );
Query OK, 0 rows affected (0.03 sec)

mysql> CREATE UNIQUE INDEX ix_auth_username on authorities (username,authority);
Query OK, 0 rows affected (0.04 sec)
```

`CREATE UNIQUE INDEX` ix_auth_username on authorities (username,authority): 'username'과 'authority' 열의 조합을 고유하게 만들어 중복 데이터를 방지

&nbsp;

&nbsp;

#### DataSource

&nbsp;

데이터베이스 연결과 상호작용을 위해 application.properties 작성

&nbsp;

```properties
spring.datasource.driver-class-name= com.mysql.cj.jdbc.Driver
spring.datasource.url=jdbc:mysql://localhost:3306/[DB이름]?characterEncoding=UTF-8&serverTimezone=Asia/Seoul
spring.datasource.username=[유저이름]
spring.datasource.password=[비밀번호]
```

&nbsp;

&nbsp;

📌 데이터베이스 URL 설정

&nbsp;

```Java
dataSource.setUrl("jdbc:mysql://localhost:[포트번호]/[DB이름]?characterEncoding=UTF-8&serverTimezone=Asia/Seoul");
```

포트번호 확인하고 name에 자신의 데이터베이스 이름으로 수정

&nbsp;

- MySQL 포트번호 확인

```sql
mysql> SHOW VARIABLES LIKE 'port';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| port          | 3306  |
+---------------+-------+
1 row in set (0.02 sec)
```

&nbsp;

&nbsp;

📌 사용자 설정

&nbsp;

```Java
dataSource.setUsername("username");
dataSource.setPassword("password");
```

데이터베이스에 접속할 계정이름과 비밀번호 설정

&nbsp;

&nbsp;

#### JDBCUserDetailManager

&nbsp;

```Java
@Bean
    UserDetailsManager users(DataSource dataSource) {
        UserDetails user = User.builder()
                .username("user")
                .password("{bcrypt}$2a$10$GRLdNijSQMUvl/au9ofL.eDwmoohzzS7.rmNSJZ.0FxO/BTk76klW")
                .roles("USER")
                .build();
        UserDetails admin = User.builder()
                .username("admin")
                .password("{bcrypt}$2a$10$GRLdNijSQMUvl/au9ofL.eDwmoohzzS7.rmNSJZ.0FxO/BTk76klW")
                .roles("USER")
                .build();
        JdbcUserDetailsManager users = new JdbcUserDetailsManager(dataSource);
        users.createUser(user);
        users.createUser(admin);
        return users;
    }
```

사용자이름,bcrypt로 암호화된 비밀번호,역할을 데이터베이스에 저장

&nbsp;

&nbsp;

📌 테이블 내용 조회

&nbsp;

```sql
mysql> SELECT * FROM users;
+----------+----------------------------------------------------------------------+---------+
| username | password                                                             | enabled |
+----------+----------------------------------------------------------------------+---------+
| admin    | {bcrypt}$2a$10$GRLdNijSQMUvl/au9ofL.eDwmoohzzS7.rmNSJZ.0FxO/BTk76klW |       1 |
| user     | {bcrypt}$2a$10$GRLdNijSQMUvl/au9ofL.eDwmoohzzS7.rmNSJZ.0FxO/BTk76klW |       1 |
+----------+----------------------------------------------------------------------+---------+

mysql> SELECT * FROM authorities;
+----------+-----------+
| username | authority |
+----------+-----------+
| admin    | ROLE_USER |
| user     | ROLE_USER |
+----------+-----------+
```

&nbsp;

&nbsp;

## 🔍 JDBC 예제

&nbsp;

#### 👤 회원가입

&nbsp;

<img width="247" alt="스크린샷 2023-08-02 오후 9 23 50" src="https://github.com/codingspecialist/-Springboot-Security-OAuth2.0-V3/assets/53010592/6a69d3ab-c78a-4d34-b1a4-faea6b7187b4">

&nbsp;

사용자이름과 비밀번호로 회원가입

&nbsp;

📌 DTO

&nbsp;

사용자로부터 입력받은 정보를 저장

&nbsp;

```Java
@Getter
@Setter
public class DTO {
    private String username;
    private String password;

}
```

&nbsp;

&nbsp;

📌 User

&nbsp;

테이블에 대응하는 엔티티 클래스로 데이터베이스의 한 행에 대응
UserDetails로 사용자 인증 정보의 관리

&nbsp;

```Java
@Entity
@Data
@Getter
@Setter
@Table(name = "users")
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@NamedQuery(name = "User.findByUsername", query = "SELECT u FROM User u WHERE u.username = :username")
public class User implements UserDetails {

    @jakarta.persistence.Id
    @Column(name = "id")
    @GeneratedValue(strategy = IDENTITY)
    private Long id;

    @Column(name = "username", updatable = false, unique = true)
    private String username;

    @Column(name = "password")
    private String password;

    @Column(name = "enabled")
    private boolean enabled;

    @Builder
    public User(String username, String password, boolean enabled) {
        this.username = username;
        this.password = password;
        this.enabled = true;

    }

    public Long getId(){
        return id;
    }
    public void setPassword(String password) {
        this.password = password;
    }

    public boolean getEnabled() {
        return enabled;
    }


    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {

        return List.of(new SimpleGrantedAuthority("user"));
    }

    //UserDetails

    @Override
    public String getPassword() {
        return password;
    }


    @Override
    public String getUsername() {
        return username;
    }


    @Override
    public boolean isAccountNonExpired() {
        return true;
    }


    @Override
    public boolean isAccountNonLocked() {
        return true;
    }


    @Override
    public boolean isCredentialsNonExpired() {
        return true;
    }


    @Override
    public boolean isEnabled() {
        return true;
    }
}
```

&nbsp;

&nbsp;

📌 UserRepository

&nbsp;

User 엔티티를 조회, 저장, 삭제하는데 필요한 메서드들을 제공

&nbsp;

```Java
public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByUsername(String username);

    void deleteByUsername(String username);
}
```

&nbsp;

> `extends JpaRepository<'Entity 클래스', 'Entity 아이디타입 >`:save(T entity),findById(ID id),findAll(),delete(T entity) 메서드 제공
> 필요한 쿼리 메서드를 선언하면 Spring Data JPA가 해당 메서드를 자동으로 구현

&nbsp;

&nbsp;

📌 AppConfig

&nbsp;

비밀번호를 암호화할 때 사용

&nbsp;

```Java
@Configuration
public class AppConfig {

   @Bean
   public PasswordEncoder passwordEncoder() {
       return new BCryptPasswordEncoder();
   }
}
```

&nbsp;

Service/Controller에서 @Autowired 등을 통해 주입해 사용

&nbsp;

&nbsp;

📌 UserService

&nbsp;

사용자 정보를 데이터베이스에 저장

&nbsp;

```Java

@Service

public class UserService {


    private final UserRepository userRepository;

    private final PasswordEncoder passwordEncoder;

    @Autowired
    public UserService(UserRepository userRepository, PasswordEncoder passwordEncoder) {
        this.userRepository = userRepository;
        this.passwordEncoder = passwordEncoder;
    }


    @Transactional
    public long save(DTO dto) {
        User userToSave = User.builder()
                .username(dto.getUsername())
                .password(passwordEncoder.encode(dto.getPassword()))
                .enabled(true)
                .build();
        User savedUser = userRepository.save(userToSave);
        return savedUser.getId();

    }
}
```

DTO를 매개변수로 받아, 사용자의 username과 password를 가져오고 password는 passwordEncoder를 사용해 암호화된 형태로 변환
이후 User 객체를 생성하고, 이 객체를 userRepository.save()를 통해 데이터베이스에 저장

> `@Transactional` 은 이 메서드를 하나의 트랜잭션으로 처리하도록 지시해 이 메서드 내에서 일어나는 여러 데이터베이스 작업들이 모두 성공적으로 완료되거나, 실패 시 모두 롤백되도록 보장

&nbsp;

&nbsp;

📌 Controller

&nbsp;

웹 요청을 받아 처리하는 컨트롤러

&nbsp;

```Java
@Controller
public class UserController {

    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

    @PostMapping("/joinIt")
    public String join(@ModelAttribute DTO dto) {
        userService.save(dto);
        return "redirect:/login";
    }

    @GetMapping("/join")
    public String join(){
        return "join";
    }

    @GetMapping("/index")
    public String index(){
        return "index";
    }

    @GetMapping("/login")
    public String login(){
        return "login";
    }

}

```

사용자가 회원 가입 폼에서 입력한 정보를 DTO 객체로 받아서 등록
완료되면 로그인 페이지로 리다이렉트

&nbsp;

&nbsp;

📌 WebSecurityConfig

&nbsp;

웹 요청에 대한 보안 설정을 정의하고 사용자 인증을 처리하는 방법을 설정

&nbsp;

```Java

@RequiredArgsConstructor
@Configuration
public class WebSecurityConfig {

    private final UserDetailService userService;
    private final PasswordEncoder passwordEncoder;

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
                .authorizeRequests(authorize -> authorize
                        .requestMatchers(HttpMethod.GET, "/login", "/join").permitAll()
                        .requestMatchers(HttpMethod.POST, "/joinProc","/updatePassword").permitAll()
                        .anyRequest().authenticated())
                .formLogin((form) -> form
                        .loginPage("/login")
                        .permitAll())
                .logout((logout) -> logout
                        .logoutSuccessUrl("/login"))
                .csrf(AbstractHttpConfigurer::disable) //CSRF 공격 방어 기능을 비활성화
                .build();
    }
}
```

> `SecurityFilterChain` : 인증 및 권한 검사와 같은 보안 관련 처리를 수행

&nbsp;

&nbsp;

#### 🔐 로그인

&nbsp;

<img width="492" alt="스크린샷 2023-08-02 오후 9 31 25" src="https://github.com/codingspecialist/-Springboot-Security-OAuth2.0-V3/assets/53010592/426253f9-8bf5-4cdd-8607-ccbc5af1a039">

&nbsp;

로그인 페이지

&nbsp;

<img width="211" alt="스크린샷 2023-08-02 오후 9 24 13" src="https://github.com/codingspecialist/-Springboot-Security-OAuth2.0-V3/assets/53010592/b62454a5-a1cd-41d6-9c38-16cd1c87be11">

&nbsp;

인덱스 페이지

&nbsp;

📌 UserDetailService

&nbsp;

사용자 인증 과정에서 사용자의 자격 증명

&nbsp;

```Java
@RequiredArgsConstructor
@Service
public class UserDetailService implements UserDetailsService {

    private final UserRepository userRepository;



    @Override
    public UserDetails loadUserByUsername(String username){
        return userRepository.findByUsername(username)
                .orElseThrow(() -> new IllegalArgumentException(username));
    }
}
```

&nbsp;

> `loadUserByUsername(String username)` :UserDetailsService 인터페이스의 메서드
> 입력받은 username에 해당하는 사용자 정보를 데이터베이스에서 조회하고, 이 정보를 UserDetails 형태로반환

&nbsp;

> `UserDetails` :사용자 이름, 비밀번호, 권한 등의 사용자 세부정보

&nbsp;

&nbsp;

📌 WebSecurityConfig

&nbsp;

사용자가 입력한 정보를 검증하고 인증 상태를 관리

&nbsp;

```Java

@RequiredArgsConstructor
@Configuration
public class WebSecurityConfig {

    private final UserDetailService userService;
    private final PasswordEncoder passwordEncoder;

    // . . .



    @Bean
    public DaoAuthenticationProvider daoAuthenticationProvider() throws Exception{
        DaoAuthenticationProvider daoAuthenticationProvider = new DaoAuthenticationProvider();

        daoAuthenticationProvider.setUserDetailsService(userService);
        daoAuthenticationProvider.setPasswordEncoder(passwordEncoder);

        return daoAuthenticationProvider;
    }

}
```

&nbsp;

> `DaoAuthenticationProvider` : Spring Security에서 제공하는 인증 제공자
> 사용자 자격 증명 (사용자 이름,비밀번호 등)을 검증하는 역할

&nbsp;

&nbsp;

#### ✏️ 비밀번호 수정

&nbsp;

<img width="331" alt="스크린샷 2023-08-02 오후 9 24 54" src="https://github.com/codingspecialist/-Springboot-Security-OAuth2.0-V3/assets/53010592/66a22290-69f0-45dd-a7dd-27ad58c4c112">

&nbsp;

📌 UserService

&nbsp;

```Java
@Service
public class UserService {

    // ...

    public User updatePassword(long id, String newPassword) {
        User user = userRepository.findById(id)
                .orElseThrow(() -> new IllegalArgumentException("Invalid user Id:" + id));

        user.setPassword(passwordEncoder.encode(newPassword));

        return userRepository.save(user);
    }
}
```

&nbsp;

주어진 ID의 사용자를 찾아 해당 사용자의 비밀번호를 새 비밀번호로 업데이트하고, 업데이트된 사용자 정보를 데이터베이스에 저장

&nbsp;

&nbsp;

📌 Controller

&nbsp;

```Java
@Controller
public class UserController {

    private final UserService userService;

    @Autowired
    public UserController(UserService userService) {
        this.userService = userService;
    }

    //. . .

    @GetMapping("/editPassword")
    public String showEditPasswordForm(Model model) {
        //현재 로그인한 사용자의 정보를 가져와서 해당 정보를 Model에 추가
        Object principal = SecurityContextHolder.getContext().getAuthentication().getPrincipal();
        if (principal instanceof UserDetails) {
            String username = ((UserDetails) principal).getUsername();
            User user = userService.findByUsername(username).orElse(null);
            if (user != null) {
                model.addAttribute("user", user);
                return "updatePassword"; //비밀번호 수정 폼을 보여주는 뷰를 반환
            }
        }
        return "redirect:/"; //사용자 정보가 없다면 메인 페이지로 리다이렉트
    }


    @PostMapping("/updatePassword")
    public String updatePassword(@ModelAttribute("user") User user, @RequestParam("newPassword") String newPassword) {
        //요청 파라미터에서 사용자 정보와 새 비밀번호를 가져와서 userService를 통해 사용자의 비밀번호를 수정
        userService.updatePassword(user.getId(), newPassword);
        return "redirect:/";//완료되면 메인 페이지로 리다이렉트
    }
}
```

&nbsp;

&nbsp;

#### 👣 회원 탈퇴

&nbsp;

&nbsp;

📌 UserService

&nbsp;

```Java
@Service
public class UserService {
    // ...

    @Transactional
    public void deleteUserByUsername(String username) {
        userRepository.deleteByUsername(username);
    }
}

```

&nbsp;

&nbsp;

📌 Controller

&nbsp;

```Java
@Controller
public class UserController {

    private final UserService userService;

    @Autowired
    public UserController(UserService userService) {
        this.userService = userService;
    }

    //. . .

    @PostMapping("/deleteUser")
    public String deleteUser(HttpServletRequest request) {
        Object principal = SecurityContextHolder.getContext().getAuthentication().getPrincipal();
        if (principal instanceof UserDetails) {
            String username = ((UserDetails) principal).getUsername();
            userService.deleteUserByUsername(username);
            request.getSession().invalidate();
        }
        return "redirect:/login";
    }


}
```

&nbsp;

> `request.getSession().invalidate()` : 현재 세션을 무효화
> 삭제된 사용자가 로그인 상태를 유지하지 못하도록 하기 위해서 무효화 필요

&nbsp;

&nbsp;

#### ❗️ 로그인 ExceptionHandler

&nbsp;

📌 AuthenticationFailureHandler

&nbsp;

```java
public class UserAuthenticationFailureHandler implements AuthenticationFailureHandler {

    @Override
    public void onAuthenticationFailure(HttpServletRequest request, HttpServletResponse response,
                                        AuthenticationException exception) throws IOException, ServletException {

        response.sendRedirect("/login?error=" + exception.getMessage());
    }
```

&nbsp;

사용자를 로그인 페이지로 다시 리다이렉트하며 실패 이유 표시

&nbsp;

> `AuthenticationFailureHandler` : SpringSecurity에서 로그인 인증 실패 시 사용되는 핸들러

&nbsp;

&nbsp;

📌 WebSecurityConfig

```java
.formLogin((form) -> form
                        .loginPage("/login")
                        .failureHandler(UserAuthenticationFailureHandler)
                        .permitAll())
```

&nbsp;

로그인 인증이 실패했을 때 호출될 핸들러 설정

&nbsp;

&nbsp;

<img width="480" alt="스크린샷 2023-08-10 오후 5 41 36" src="https://github.com/ztng123/calendar/assets/53010592/8fad91b2-081d-49dc-8767-958dfa7f3647">

&nbsp;

&nbsp;

#### ❗️ 회원가입 ExceptionHandler

&nbsp;

📌 UserService

&nbsp;

```java
public long save(DTO dto) {
        if(userRepository.existsByUsername(dto.getUsername())){
            throw new UserAlreadyExistsException("username already exists");
        }
}
```

&nbsp;

사용자 이름이 이미 존재하는지 체크하고 이미 존재하면 UserAlreadyExistsException 예외가 발생

&nbsp;

&nbsp;

📌 UserAlreadyExistsException

&nbsp;

```java
public class UserAlreadyExistsException extends RuntimeException {
    public UserAlreadyExistsException(String message) {
        super(message);
    }
}
```

&nbsp;

예외가 발생하면 메세지가 출력

&nbsp;

> `RuntimeException` : unchecked exception의 주요 예로, 컴파일러가 이를 명시적으로 처리하도록 요구하지 않음
> Exception을 직접 상속받는 경우는 checked exception으로 throws를 사용하여 예외 선언 해야함

&nbsp;

&nbsp;

📌 UserController

&nbsp;

```java
@PostMapping("/joinProc")
    public String join(DTO dto) {
        try {
            userService.save(dto);
        } catch (UserAlreadyExistsException e) {
            return "redirect:/join?error=User already exists!";
        }
        return "redirect:/login";
    }
```

&nbsp;

UserAlreadyExistsException이 발생하면 "/join?error=User already exists!"로 리다이렉트

&nbsp;

&nbsp;

<img width="502" alt="스크린샷 2023-08-10 오후 5 33 08" src="https://github.com/ztng123/calendar/assets/53010592/18d4a391-b01d-4949-9079-30c6c4136205">

&nbsp;

&nbsp;

---

출처:[Spring-security-sample](https://github.com/spring-projects/spring-security-samples/tree/main/servlet/java-configuration/authentication/username-password/jdbc),[Spring docs](https://docs.spring.io/spring-security/reference/index.html)

&nbsp;

&nbsp;
