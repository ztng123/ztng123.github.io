---
layout: post
title: 🗝️ Spring Security 비밀번호
tags : [spring,MySQL]
---


&nbsp;

&nbsp;

비밀번호 저장 메서드는 총 3가지로 In-memory,JDBC,LDAP 방식에 대해 알아보자

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

``` gradle
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

데이터베이스 연결과 상호작용을 위해 DataSource 설정

&nbsp;

```java
@Bean
    public DataSource dataSource(){
        DriverManagerDataSource dataSource = new DriverManagerDataSource();
        dataSource.setDriverClassName("com.mysql.cj.jdbc.Driver");
        dataSource.setUrl("jdbc:mysql://localhost:3306/jdbc?characterEncoding=UTF-8&serverTimezone=Asia/Seoul");
        dataSource.setUsername("root");
        dataSource.setPassword("12345678");
        return dataSource;

    }
```


&nbsp;

&nbsp;


📌 데이터베이스 URL 설정

&nbsp;

```Java
dataSource.setUrl("jdbc:mysql://localhost:[포트번호]/[name]?characterEncoding=UTF-8&serverTimezone=Asia/Seoul");
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

위의 사용자들로 로그인 시도했을 때 성공 

&nbsp;

&nbsp;

## 🗂️ LDAP

&nbsp;

LDAP (Lightweight Directory Access Protocol)는 조직에서 사용자 정보와 인증 서비스를 위한 중앙 저장소로 자주 사용

먼저 LDAP 서버를 설정해야하는데 Embedded UnboundID Server,Embedded ApacheDS Server 사용이 가능하다

&nbsp;

&nbsp;

#### Embedded UnboundID Server

&nbsp;

📌 의존성 추가

&nbsp;

```Gradle
depenendencies {
 runtimeOnly "com.unboundid:unboundid-ldapsdk:6.0.9"
}
```
&nbsp;

&nbsp;


📌 LDAP 서버 구성

&nbsp;

```Java
@Bean
public EmbeddedLdapServerContextSourceFactoryBean contextSourceFactoryBean() {
    return EmbeddedLdapServerContextSourceFactoryBean.fromEmbeddedLdapServer();
}

```


인메모리 LDAP 서버를 자동으로 시작하게 함

&nbsp;

&nbsp;

📌 LDAP 서버 수동 구성

&nbsp;

```Java
@Bean
UnboundIdContainer ldapContainer() {
 return new UnboundIdContainer("dc=springframework,dc=org",
    "classpath:users.ldif");
}
```

ldif 파일을 사용하여 설정

&nbsp;

&nbsp;


📌 users.ldif 작성

&nbsp;

<img width="218" alt="스크린샷 2023-07-26 오후 9 09 54" src="https://github.com/spring-projects/spring-security-samples/assets/53010592/25478ef7-8887-41db-b5c7-6368e3323916">

&nbsp;
resources 폴더 안에 작성
&nbsp;

```ldif
In the following samples, we expose users.ldif as a classpath resource to initialize the embedded LDAP server with two users, user and admin, both of which have a password of password:
users.ldif
dn: ou=groups,dc=springframework,dc=org
objectclass: top
objectclass: organizationalUnit
ou: groups

dn: ou=people,dc=springframework,dc=org
objectclass: top
objectclass: organizationalUnit
ou: people

dn: uid=admin,ou=people,dc=springframework,dc=org
objectclass: top
objectclass: person
objectclass: organizationalPerson
objectclass: inetOrgPerson
cn: Rod Johnson
sn: Johnson
uid: admin
userPassword: password

dn: uid=user,ou=people,dc=springframework,dc=org
objectclass: top
objectclass: person
objectclass: organizationalPerson
objectclass: inetOrgPerson
cn: Dianne Emu
sn: Emu
uid: user
userPassword: password

dn: cn=user,ou=groups,dc=springframework,dc=org
objectclass: top
objectclass: groupOfNames
cn: user
uniqueMember: uid=admin,ou=people,dc=springframework,dc=org
uniqueMember: uid=user,ou=people,dc=springframework,dc=org

dn: cn=admin,ou=groups,dc=springframework,dc=org
objectclass: top
objectclass: groupOfNames
cn: admin
uniqueMember: uid=admin,ou=people,dc=springframework,dc=org

```

LDIF 파일을 사용하여 스프링 애플리케이션에서 내장된 LDAP 서버를 초기화하고, "user"와 "admin" 두 개의 사용자를 등록하며, 그룹 정보를 설정

&nbsp;

&nbsp;


#### Embedded ApacheDS Server

&nbsp;

📌 의존성 추가

&nbsp;

```gradle
depenendencies {
 runtimeOnly "org.apache.directory.server:apacheds-core:1.5.5"
 runtimeOnly "org.apache.directory.server:apacheds-server-jndi:1.5.5"
}
```

&nbsp;

&nbsp;


📌 LDAP 서버 구성

&nbsp;

```Java
@Bean
ApacheDSContainer ldapContainer() {
 return new ApacheDSContainer("dc=springframework,dc=org",
    "classpath:users.ldif");
}
```

&nbsp;

&nbsp;


---
출처:[Spring-security-sample](https://github.com/spring-projects/spring-security-samples/tree/main/servlet/java-configuration/authentication/username-password/jdbc),[Spring docs](https://docs.spring.io/spring-security/reference/index.html)

&nbsp;

&nbsp;

