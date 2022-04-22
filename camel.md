## springboot的方式集成camel 和 hawtio

```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>

        <dependency>
            <groupId>io.hawt</groupId>
            <artifactId>hawtio-springboot</artifactId>
            <version>2.9.1</version>
            <exclusions>
                <exclusion>
                    <artifactId>log4j-to-slf4j</artifactId>
                    <groupId>org.apache.logging.log4j</groupId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
            <groupId>org.apache.camel.springboot</groupId>
            <artifactId>camel-management-starter</artifactId>
            <version>3.16.0</version>
        </dependency>
        <!--相关依赖-->
```

```yaml
#配置文件
management:
  endpoints:
    web:
      exposure:
        include:
          - hawtio
          - jolokia

hawtio:
  authenticationEnabled: false
```

```java
package com.jd.jnos.baize.config;

import org.springframework.boot.actuate.autoconfigure.jolokia.JolokiaEndpoint;
import org.springframework.boot.actuate.autoconfigure.security.servlet.EndpointRequest;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.authentication.builders.AuthenticationManagerBuilder;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.builders.WebSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.web.csrf.CookieCsrfTokenRepository;

@Configuration
public class BaizeSecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.inMemoryAuthentication()
                .passwordEncoder(new BCryptPasswordEncoder())
                .withUser("baize") // 添加用户admin
                .password(new BCryptPasswordEncoder().encode("Baize123#@!"))
                .roles("ADMIN");// 添加角色为admin，user
    }

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests()
                .antMatchers("/actuator/**").hasRole("ADMIN")
                .anyRequest().authenticated()
                .and()
                .formLogin().and()
                .httpBasic().and()
                .csrf().csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
                .ignoringRequestMatchers(EndpointRequest.to(JolokiaEndpoint.class));
    }

    @Override
    public void configure(WebSecurity webSecurity)  {
        webSecurity.ignoring().antMatchers("/api/**", "/system/**");
    }
}

```

spring security的WebSecurity和HttpSecurity区别

https://stackoverflow.com/questions/62677271/httpsecurity-permitall-and-websecurity-ignoring-functions-for-un-auth-urls

