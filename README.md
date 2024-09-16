# Java Security Day 1 Workshop

## Learning Objectives
- Build a simple API (just the GET endpoints)
- Prevent unauthorised users from accessing it
- Secure the application using a simple username and password login  

## Instructions

1. Fork this repository
2. Clone your fork to your machine
3. Open the project in IntelliJ

## Update `build.gradle` and Add Connection Details to `application.yml`

We're going to build a simple application that makes use of a few newer libraries from the Spring Framework that we haven't used before these will let us create a simple login page using the Thymeleaf library which gives us some HTML capabilities, managed by Spring, we'll also make use of Spring's Security libraies to enable us to secure the endpoints we want behind a login system.

To begin with add your usual connection details to `application.yml` if you are reusing a database instance you've used previously then clear out any existing tables.

Then change the contents of build.gradle to the following:

```groovy
plugins {
	id 'java'
	id 'org.springframework.boot' version '3.1.5'
	id 'io.spring.dependency-management' version '1.1.3'
}

group = 'com.booleanuk'
version = '0.0.1-SNAPSHOT'

java {
	sourceCompatibility = '17'
}

configurations {
	compileOnly {
		extendsFrom annotationProcessor
	}
}

repositories {
	mavenCentral()
}

dependencies {
	implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
	implementation 'org.springframework.boot:spring-boot-starter-security'
	implementation 'org.springframework.boot:spring-boot-starter-thymeleaf'
	implementation 'org.springframework.boot:spring-boot-starter-web'
	implementation 'org.thymeleaf.extras:thymeleaf-extras-springsecurity6'
	compileOnly 'org.projectlombok:lombok'
	developmentOnly 'org.springframework.boot:spring-boot-devtools'
	runtimeOnly 'org.postgresql:postgresql'
	annotationProcessor 'org.projectlombok:lombok'
	testImplementation 'org.springframework.boot:spring-boot-starter-test'
	testImplementation 'org.springframework.security:spring-security-test'
}

//tasks.named('bootBuildImage') {
//	builder = 'paketobuildpacks/builder-jammy-base:latest'
//}

tasks.named('test') {
	useJUnitPlatform()
}
```

We've added the new libraries and configured some of it so that they work together properly. With the addition of Spring Security if we run the project as it is, then we will not be able to access either of the endpoints we have as we will get a 401 error due to the fact that we haven't logged in. If you temporarily comment out the lines which refer to Spring Security you should be able to access the `/books` and `/books/1` endpoints as normal. Use TablePlus to add some books to your database and test out that everything works as expected.

Then uncomment the Spring Security lines and try to connect again. You should be able to access the tables as normal via TablePlus but the endpoints should now give you an Unauthorised error in Insomnia (or the browser). We're going to make a simple login system which will prevent unauthorised users from accessing the endpoints but allow those with the correct credentials to access them. The way we will do this today is **__NOT__** suitable for use in production and is merely to demonstrate some of the concepts we will use. Tomorrow we will implement a fully working log in system that could be used in production.

## Create a Login Page

In the `templates` folder in `src/main/resources` add a `login.html` file, with the following contents:

```html
<!DOCTYPE html>
<html xmlns="http://www.w3.org/1999/xhtml" xmlns:th="https://www.thymeleaf.org">
<head>
    <title>Spring Security Example </title>
</head>
<body>
<div th:if="${param.error}">
    Invalid username and password.
</div>
<div th:if="${param.logout}">
    You have been logged out.
</div>
<form th:action="@{/login}" method="post">
    <div><label> User Name : <input type="text" name="username"/> </label></div>
    <div><label> Password: <input type="password" name="password"/> </label></div>
    <div><input type="submit" value="Sign In"/></div>
</form>
</body>
</html>
```

This will create a simple login page using HTML that includes code from the Thymeleaf templating engine, which has integrations with Java. We will not be using this apart from here, so I'm not going to take any time to dig into it, but if you want to learn more about it and its use cases then you can do so at their website: [https://www.thymeleaf.org](https://www.thymeleaf.org). Essentially when we click on the `Sign In` button on the page that is displayed it will take the contents of the username and password text boxes and submit them to our API as part of a `POST` request.

## Add a Login Controller

We want to make it so that when we navigate to [http://localhost:4000/login](http://localhost:4000/login) we will see the login page. To make it show up we need to add a `LoginController` class to the controller folder (and then some other configuration to deal with clicking Sign In) we don't need a model or a repository.

Create a new `LoginController` class and add the following code to it:

```java
package com.booleanuk.library.controller;

import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.GetMapping;

@Controller
public class LoginController {
    @GetMapping("/login")
    String login() {
        return "login";
    }
}
```

Note: To display the page we are using a `GET` mapping as we want the page to show up, the `POST` mapping is handled separately (ie not in the Controller).

## Add the Security Configuration

The next (and final) part of our setup is to add new Java class to our project. I tend to create a new Package folder inside the `library` package called `config` and then add a new class called `SecurityConfiguration` to that. It doesn't actually matter what we call this, as we are going to annotate it in such a way that when we run the project, Spring will build this class and include it in the correct places to handle security related parts of the application. Create the new package and class and then add the following code to it:

```java
package com.booleanuk.library.config;


import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.core.userdetails.User;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.provisioning.InMemoryUserDetailsManager;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
@EnableWebSecurity
public class SecurityConfiguration {
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
                .authorizeHttpRequests((requests) -> requests
                        .requestMatchers("/books", "/books/*").authenticated()
                )
                .formLogin((form) -> form
                        .loginPage("/login").permitAll()
                )
                .logout((logout) -> logout.permitAll());
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

The class is annotated with `@Configuration` and `@EnableWebSecurity` which alongside using the `@Bean` annotation is enough to get this code added to the project when we run it. The first method `securityFilterChain` is what we use to build a pattern matching system that looks at the URL endpoints and decides whether we need to be authenticated to access it or not (it can do a lot more than this, but you will need to investigate it yourselves depending on the use cases you envisage it using). When it is called, we pass a `HttpSecurity` object that contains all of the necessary information about the HTTP request that has been made such as:

* What endpoint is it visiting? 
* Has the user been authenticated? 
* What request type is being made?
* ...

We then use dot notation to chain a number of method calls onto it, which attempt to match the request, the first one to do so will cause the rest to be ignored. In ours it starts by checking the path that the request has been made to, if it matches the `books` one (in either form) then it returns true and asks if the user has been authenticated. Assuming that they haven't it redirects them to the login page (which matches the second entry) and gets them to attempt to login. If they succeed then the user will be redirected to the `books` endpoint. If they fail it will redirect them again to the login page.

The second method is used to create an in-memory hard coded user for us to use to login. The password is not encode at all, and nothing is stored in our database at this point. 

**_DO NOT DO THIS IN PRODUCTION!!!!_**

We'll look at ways to store your users in a database with encrypted passwords tomorrow.

## Run the Project

That is all that is needed to make a basic application which has been secured, you should be able to access the `books` and the `books/1` endpoints as long as you add some data to the tables. This will work in the browser, but proved problematic for me when trying to use Insomnia.

## Exercises

Try and extend this to have other linked tables.

Can you make it work for `PUT`, `POST` and `DELETE` endpoints? 

