---
name: spring-mvc-to-boot-migrator
description: Migrates a Spring MVC application to Spring Boot, converting XML config to auto-configuration, restructuring the project, and replacing container deployment with embedded. Use when modernizing a legacy Spring app, when moving off a standalone servlet container, or when the user has web.xml and wants application.yml.
license: Apache-2.0
metadata:
  category: "code-analysis"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "test-guided-migration-assistant, build-ci-migration-assistant"
---

# Spring MVC → Boot Migrator

Spring Boot is Spring MVC with conventions, embedded server, and auto-configuration. The migration isn't a rewrite — it's **deleting XML and letting Boot infer it.**

## What changes and what doesn't

| Concern               | Spring MVC (classic)              | Spring Boot                                | Migration effort |
| --------------------- | --------------------------------- | ------------------------------------------ | ---------------- |
| `@Controller` classes | `@Controller`, `@RequestMapping`  | Same — unchanged                           | Zero             |
| `@Service`, `@Repository` | Same annotations              | Same — unchanged                           | Zero             |
| Bean wiring (XML)     | `<bean id=... class=...>`         | `@Configuration` + `@Bean`, or just `@Component` | High — this is the work |
| DispatcherServlet setup| `web.xml`                        | Auto-configured — delete it                | Delete           |
| Server                | Deploy WAR to Tomcat              | Embedded Tomcat — `java -jar`              | pom.xml change   |
| Properties            | Multiple `.properties` via `<context:property-placeholder>` | `application.yml` with profiles | Consolidate |
| View resolver         | `<bean class="InternalResourceViewResolver">` | `spring.mvc.view.prefix=...` property | One-liner |
| Static resources      | `<mvc:resources mapping=...>`     | `/static`, `/public` auto-served           | Move files       |

## Migration order — keep it running at every step

Boot can coexist with XML config. Migrate incrementally:

1. **Add Boot starter, keep everything else.** App still works, now it's a Boot app that happens to read XML.
2. **Replace one XML block with `@Configuration`.** Test. Repeat.
3. **Delete `web.xml`** once DispatcherServlet XML config is gone.
4. **Switch WAR → JAR** once you're confident.

## Step 1 — Bootify the build

**Maven:**

```xml
<parent>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-parent</artifactId>
  <version>3.x.x</version>
</parent>

<dependencies>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
  </dependency>
  <!-- Keep existing deps; remove spring-webmvc, spring-context — the starter brings them -->
</dependencies>
```

Remove explicit `spring-*` versions — let the parent BOM manage them. Version conflicts here are → `dependency-resolver` territory.

## Step 2 — Add the main class, import existing XML

```java
@SpringBootApplication
@ImportResource("classpath:applicationContext.xml")   // bridge — delete later
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

`@ImportResource` lets Boot load your old XML. App runs under Boot with zero config migrated. Verify everything works before touching XML.

## Step 3 — XML → @Configuration, block by block

**XML:**

```xml
<bean id="dataSource" class="org.apache.commons.dbcp2.BasicDataSource">
  <property name="url" value="${db.url}"/>
  <property name="username" value="${db.user}"/>
</bean>

<bean id="txManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
  <constructor-arg ref="dataSource"/>
</bean>
```

**Boot — option A: explicit `@Configuration`:**

```java
@Configuration
public class DatabaseConfig {
    @Bean
    public DataSource dataSource(@Value("${db.url}") String url,
                                 @Value("${db.user}") String user) {
        BasicDataSource ds = new BasicDataSource();
        ds.setUrl(url);
        ds.setUsername(user);
        return ds;
    }
    // txManager — Boot auto-configures DataSourceTransactionManager if a DataSource bean exists. DELETE.
}
```

**Boot — option B: auto-config (preferred when it applies):**

```yaml
# application.yml
spring:
  datasource:
    url: ${DB_URL}
    username: ${DB_USER}
```

No Java at all. `spring-boot-starter-jdbc` sees these properties, auto-configures Hikari (not DBCP — **behavior change**, flag it).

## Common XML → Boot mappings

| XML                                          | Boot                                                        |
| -------------------------------------------- | ----------------------------------------------------------- |
| `<context:component-scan base-package=...>`  | `@SpringBootApplication` scans its own package + subpackages. If your packages are elsewhere: `@ComponentScan(basePackages=...)` |
| `<mvc:annotation-driven/>`                   | Auto — delete                                               |
| `<mvc:resources mapping="/static/**" location="/assets/"/>` | Move files to `src/main/resources/static/` |
| `<bean class="...ViewResolver">`             | `spring.mvc.view.prefix` / `.suffix` properties             |
| `<tx:annotation-driven/>`                    | Auto — `@Transactional` just works                          |
| `<context:property-placeholder location=...>`| `application.yml` + `@Value` or `@ConfigurationProperties`  |
| Filters in `web.xml`                         | `@Bean FilterRegistrationBean<YourFilter>` or just `@Component` on the filter |
| Servlets in `web.xml`                        | `@Bean ServletRegistrationBean<...>`                        |

## Step 4 — Delete the bridge

Once every `<bean>` is migrated: delete `@ImportResource`, delete `applicationContext.xml`, delete `web.xml`. Run tests.

## Gotchas

| Gotcha                                    | Symptom                                    | Fix                                         |
| ----------------------------------------- | ------------------------------------------ | ------------------------------------------- |
| Package layout                            | Beans not found                            | `@SpringBootApplication` scans its own package down — move main class up, or add `@ComponentScan` |
| Auto-config conflict                      | Two `DataSource` beans                     | Your XML defined one, Boot auto-configured another. `@Primary`, or exclude the auto-config: `@SpringBootApplication(exclude=DataSourceAutoConfiguration.class)` |
| Connection pool switch                    | Different perf characteristics             | Boot defaults to Hikari. If you need DBCP, declare it explicitly. |
| Context path                              | URLs all 404                               | Was `/myapp` in the servlet container; now `/`. Set `server.servlet.context-path=/myapp` or fix your links. |
| JSP                                       | Whitelabel error page                      | JSP + embedded Tomcat is fiddly — `spring-boot-starter-tomcat` provided scope, `tomcat-embed-jasper`, and keep WAR packaging. Or migrate to Thymeleaf. |

## Do not

- **Do not** migrate everything at once. `@ImportResource` exists so you can do it a bean at a time.
- **Do not** blindly trust auto-config. It picks defaults — Hikari over DBCP, Jackson over Gson. Usually better, occasionally a behavior change. Test.
- **Do not** leave both XML and Java config defining the same bean. That's two definitions — which wins is load-order-dependent.
- **Do not** migrate to Boot 2 and Boot 3 in one hop from Spring 4 MVC. Boot 3 is Jakarta EE (`javax.*` → `jakarta.*`). Do MVC → Boot 2, then Boot 2 → 3.

## Output format

```
## Current state
Spring version: <x>
Config style: <XML | JavaConfig | mixed>
Packaging: <WAR on <container>>

## Target
Boot version: <x>
Packaging: <executable JAR>

## Migration steps
| Step | XML removed | Java/YAML added | Test checkpoint |
| ---- | ----------- | --------------- | --------------- |

## Behavior changes
<connection pool, context path, anything auto-config does differently — FLAG THESE>

## Files
### pom.xml (diff)
### Application.java (new)
### application.yml (new)
### Deleted: <web.xml, applicationContext.xml, ...>
```
