# Module 4 – Lesson 4.1  
# Spring Security Fundamentals (Basic Authentication & Role-Based Authorization)

---

## Lesson Overview

In all the previous modules, our `simple-crm` application has been completely **open**. Any user who knows the API endpoints can create, update, or delete customer data. While this is acceptable for learning purposes, it is **not acceptable in real-world applications**.

In this lesson, we introduce **Spring Security**, the standard security framework used in Spring-based applications. Spring Security allows us to control **who** can access our application and **what** they are allowed to do.

This lesson focuses on **foundational security concepts** and applies them directly to the existing **Simple CRM project** built in the Spring Boot module. We will start with **basic authentication**, then gradually enhance it using **role-based authorization**, and finally introduce **password encoding** to follow security best practices.

The goal of this lesson is not to secure everything at once, but to **understand the security model clearly** and apply it incrementally in a way that students can follow and extend confidently.

---

## Lesson Duration

**Approximate Duration:** 3 hours

- Conceptual explanations and walkthroughs: ~1.5 hours  
- Guided implementation: ~1 hour  
- Hands-on student activity and discussion: ~30 minutes  

---

## Lesson Objectives

By the end of this lesson, students will be able to:

- Explain why application-level security is required for backend APIs
- Understand the role of Spring Security in a Spring Boot application
- Differentiate between authentication and authorization
- Configure basic authentication using Spring Security
- Apply role-based authorization to a REST endpoint
- Encode passwords securely using a password encoder
- Secure a simple endpoint in the existing `simple-crm` project
- Extend security rules to other endpoints as a hands-on exercise

---

## Prerequisites

Before starting this lesson, students should already be comfortable with:

- Spring Boot project structure
- REST controllers and endpoints
- Service and repository layers
- The `simple-crm` application created in earlier lessons
- Basic Maven dependency management

---

## Part 1: Why Do We Need Security?

So far, our API looks something like this:

```
GET  /customers
POST /customers
PUT  /customers/{id}
DELETE /customers/{id}
```

Anyone can call these endpoints using tools like Postman, curl, or even a browser.

### Problems with an Unsecured API

- Anyone can read sensitive customer data
- Anyone can modify or delete records
- There is no accountability (we don't know who made a change)
- This violates basic security and compliance standards

In real systems, we need answers to questions like:
- Who is making this request?
- Is this user authenticated?
- Is this user allowed to perform this action?

Notice that the third problem is not about attackers at all. Even with a completely trustworthy set of users, an application with no concept of identity has no way to record who did what. Once a customer record is deleted, there is no way to find out who deleted it. Adding security is therefore not only about keeping bad actors out, it is also about knowing who is inside.

This is where **Spring Security** comes in.

---

## Part 2: What Is Spring Security?

Spring Security is a powerful and flexible framework that provides:

- Authentication (verifying identity)
- Authorization (verifying permissions)
- Protection against common security vulnerabilities

### How It Works

Spring Security works by placing a **security filter chain** in front of your application. Every incoming HTTP request passes through this chain **before** it reaches your controllers.

At a high level, the flow looks like this:

```
Client Request
   ↓
Spring Security Filter Chain
   ↓
Controller
   ↓
Service → Repository
```

A filter is simply a piece of code that sits in the path of a request and gets to inspect it, change it, or stop it, before the request continues on. A chain is several of these arranged in order, each handing the request to the next. Spring Security registers around fifteen filters of its own, and a request has to survive all of them before Spring MVC even decides which controller method to call.

The important consequence is that **security is not something your controller does**. Your `CustomerController` has no idea security exists. It contains no `if (userIsLoggedIn)` checks and no role conditions. All of that lives in the filter chain, configured in one place, which is why you can secure an entire application without editing a single controller. It also means that if a request fails a security check, your controller code never runs at all.

> **Note:** Because security runs in the filter chain, a `401` or `403` never reaches your controller — which means your existing `@RestControllerAdvice` and `ErrorResponse` format do **not** apply to security errors. Spring Security returns its own default response instead.

---

## Part 3: Authentication vs Authorization

These two concepts are often confused, so let's be very clear.

### Authentication

Authentication answers the question:

**Who are you?**

Examples:
- Username and password
- Token validation
- OAuth login

Authentication establishes identity, and nothing more. It has no opinion about what you are allowed to do. A successful authentication produces exactly one result: the system now knows which user is making this request. Think of it as the security guard at the building entrance checking that your pass is genuine. He confirms you are who you claim to be, and that is the end of his job.

In this lesson, we will start with **basic authentication using username and password**.

### Authorization

Authorization answers the question:

**Are you allowed to do this?**

Examples:
- Only ADMIN users can delete data
- Only authenticated users can view data
- Different roles have different permissions

Authorization happens **after** authentication and depends on it entirely. You cannot decide what someone is permitted to do until you know who they are. Continuing the building analogy, the guard has confirmed your pass, but your pass still will not open the server room door. Same person, same verified identity, different permission.

In this lesson, we will use **role-based authorization**.

### The Two Status Codes

This distinction shows up directly in the HTTP status codes, and it is worth learning to read them as a diagnosis rather than just an error.

- **401 Unauthorized** — you have not proven who you are. Authentication failed, or you sent no credentials at all. The fix is to send valid credentials.
- **403 Forbidden** — we know exactly who you are, and you are still not allowed to do this. Authentication succeeded, authorization failed. Sending the same credentials again will never help; you need a different role.

When something goes wrong later in this lesson, the first question to ask is which of these two you received, because they point at completely different problems.

---

## Part 4: Adding Spring Security to `simple-crm`

### Step 1: Add Spring Security Dependency

Open `pom.xml` and add the Spring Security starter:

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

After adding this dependency, restart the application.

### What Happens Immediately?

Once Spring Security is added:
- All endpoints are secured by default
- Spring Security generates a default user
- A password is printed in the console at startup

Try accessing:
```
GET http://localhost:8080/customers
```

You will now receive:
```
401 Unauthorized
```

This confirms that Spring Security is active.

Pause on what just happened. You wrote no configuration, no rules and no users, yet your entire API is now locked. This is Spring Security's **deny by default** principle: the moment it is on the classpath, everything is protected, and it is your job to open things up deliberately. The alternative design, where nothing is protected until you remember to protect it, is how endpoints get accidentally exposed in real systems. Spring chooses the safer default and makes you argue your way out of it.

> **Note:** Any MockMvc tests written in Lesson 3.19 will now return `401` and fail when you run `mvn test`. Testing secured endpoints is out of scope for this lesson.

---

## Part 5: Logging In for the First Time

Before we write any configuration, we need to actually get into the API.

### Option A: The Generated Password

When the application starts, Spring Security creates **one** user:

- Username: `user`
- Password: a random value printed in the console

Scroll up in your terminal and look for a line like this:

```
Using generated security password: 8f3a2b91-4c7d-4e2a-9b11-0d5e6f7a8c12
```

Copy that value. In **Postman**, open the **Authorization** tab, choose **Basic Auth**, and enter `user` as the username and the copied value as the password. In a **browser**, a login popup will appear — enter the same two values.

> **Important:** This password is regenerated on every restart, and with DevTools enabled, on every reload. If you suddenly get `401` again after saving a file, this is why.

### Option B: Set Your Own in `application.properties`

Because the generated password keeps changing, set a fixed one instead:

```properties
spring.security.user.name=admin
spring.security.user.password=admin123
spring.security.user.roles=ADMIN
```

Restart the application. The generated password line no longer appears, and you can now log in with `admin` / `admin123` every time.

> **Note:** These properties are only used when no user store is defined in Java. Once we create a `UserDetailsService` bean in Part 11, they are ignored completely.

> **Note:** A plain-text password in a properties file is not acceptable in production. We address this in Part 10.

---

## Part 6: Default Spring Security Behavior

By default, Spring Security:
- Enables HTTP Basic authentication
- Secures all endpoints
- Creates a default user with a generated password

### What Is HTTP Basic Authentication?

HTTP Basic is a standard, defined way of putting a username and password onto an HTTP request. The client joins them with a colon, as `username:password`, encodes that with Base64, and sends the result in a header:

```
Authorization: Basic dXNlcjpwYXNzd29yZA==
```

Postman builds this header for you when you fill in the Basic Auth tab. Spring Security's Basic authentication filter reads the header, decodes it, looks up the user and checks the password.

Two properties of this scheme explain everything else about it. First, **Base64 is encoding, not encryption**. Anyone who can see the request can decode it back to the plain password instantly. There is no secret involved. This is why HTTP Basic is only acceptable over HTTPS, where the whole request is encrypted in transit. Second, **there is no login step that gets remembered**. The username and password are sent again on every single request, which is why you must set the Authorization tab on each request in Postman rather than logging in once.

This default behavior is **not suitable for production**, but it helps us understand how security intercepts requests.

---

## Part 7: Creating a Custom Security Configuration

To control security behavior, we create a **Security Configuration class**.

### Step 1: Create SecurityConfig

Create a new class in the `config` folder:

```
src/main/java/.../config/SecurityConfig.java
```

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

}
```

`@Configuration` marks this as a class where Spring should look for `@Bean` methods, the same way you used it for `AppConfig` in earlier lessons. `@EnableWebSecurity` switches on Spring Security's web support and tells Boot to step back and use your configuration instead of its defaults.

This class will define:
- Which endpoints are secured
- How authentication works
- What roles are required

### Imports Used in This Class

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.HttpMethod;
import org.springframework.security.config.Customizer;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.core.userdetails.User;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.provisioning.InMemoryUserDetailsManager;
import org.springframework.security.web.SecurityFilterChain;
```

---

## Part 8: In-Memory Authentication (For Learning)

For learning purposes, we will use **in-memory users**.

### Why In-Memory Users?

- Simple to set up
- No database required
- Easy to understand authentication flow

In-memory means the users are created in Java code when the application starts and held in an ordinary object in memory. There is no user table, no registration endpoint and no persistence. Restart the application and you get the same two hardcoded users back, because they are written into the source code.

This is **not** how production systems work, but it is perfect for learning. It lets us focus on how rules and roles behave without the distraction of a user table, and the piece we replace later turns out to be small and well isolated.

---

## Part 9: Configuring Basic Authentication

Add the following method inside `SecurityConfig`:

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {

    http
        .csrf(csrf -> csrf.disable())
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/customers/**").authenticated()
            .anyRequest().permitAll()
        )
        .httpBasic(Customizer.withDefaults());

    return http.build();
}
```

### Reading This Method Line by Line

`http` is a builder. Each line adds one piece of configuration, and `http.build()` at the end turns all of it into the actual filter chain object that Spring places in front of your application.

`.csrf(csrf -> csrf.disable())` turns off CSRF protection. CSRF, Cross-Site Request Forgery, is a browser attack. If you are logged into a site in one tab and a malicious site in another tab submits a request to it, your browser helpfully attaches your session cookie, and the server sees a valid logged-in request that you never intended to make. Spring's defence is a secret token that must accompany every write request, which the attacker's site cannot read. We disable it here because that attack depends on credentials being attached automatically, and our client is Postman, sending credentials explicitly on every call. Left enabled, every POST, PUT and DELETE from Postman would be rejected with 403 and nothing would be gained.

`.authorizeHttpRequests(...)` is where you declare which URLs need what. Everything inside the lambda is a list of rules, evaluated **top to bottom, first match wins**. This ordering is the single most common source of confusion, so state it plainly: once a request matches a rule, no later rule is consulted.

`.requestMatchers("/customers/**").authenticated()` is the first rule. `requestMatchers` selects which requests it applies to, and `.authenticated()` says the caller must be logged in. Note that it does not mention an HTTP method, so GET, POST, PUT and DELETE on those paths are all covered by this one line, and it does not care which role you hold, only that you proved who you are.

The `/**` at the end matters. A single `*` matches exactly one path segment, so `/customers/*` covers `/customers/5` but not `/customers/5/interactions`. A double `**` matches any number of segments, which is why `/customers/**` catches everything beneath that path.

`.anyRequest().permitAll()` is the catch-all for anything that did not match above. There can only ever be one `anyRequest()` and it must be last, because it matches everything that reaches it, so nothing could ever get past it to a later rule. It is also not optional. Leave it out and requests matching none of your rules have no defined outcome, and Spring will fail rather than guess.

`.httpBasic(Customizer.withDefaults())` switches on the Basic authentication filter, which is the part that actually reads the username and password off the request. `Customizer.withDefaults()` simply means "use the standard settings." Without this line your rules would exist but there would be no way to log in.

A useful way to hold the two halves apart: **`authorizeHttpRequests` decides what is protected, `httpBasic` decides how you prove who you are.** Two separate jobs, and you need both.

> **Note:** `httpBasic()` must be written as `httpBasic(Customizer.withDefaults())`. The older no-argument version was removed in Spring Security 7, which ships with Spring Boot 4.

> **Note:** If your `CustomerController` is mapped at `/api/customers`, use that path in the matchers instead.

### Mixing Open and Protected Endpoints

Locking everything down is rarely what a real API wants. A far more common shape is that reads are open and writes are protected. To express that, add the HTTP method as the first argument to `requestMatchers`:

```java
.authorizeHttpRequests(auth -> auth
    .requestMatchers(HttpMethod.GET, "/customers/**").permitAll()
    .requestMatchers(HttpMethod.POST, "/customers/**").authenticated()
    .requestMatchers(HttpMethod.PUT, "/customers/**").authenticated()
    .requestMatchers(HttpMethod.DELETE, "/customers/**").authenticated()
    .anyRequest().permitAll()
)
```

Now `GET /customers` and `GET /customers/5` work with no credentials at all, while POST, PUT and DELETE return 401 unless a valid username and password are sent. The same path can therefore behave differently depending on the method used to call it.

The three outcomes available to any rule are worth naming now, because we use all of them in Part 12. `permitAll()` means open to anyone, with no credentials. `authenticated()` means you must log in, but any valid user will do. From Part 12 onwards we add `hasRole(...)`, which means you must log in **and** hold that specific role.

> **Important (Postman):** To test an endpoint you have made `permitAll()`, set the Authorization tab to **No Auth**. Clearing the username and password fields is not enough — Basic Auth still sends an empty header, the authentication filter runs *before* the authorization rules, and it rejects the request with 401 before `permitAll()` is ever reached. Sending bad credentials to an open endpoint fails, while sending none succeeds. That looks backwards until you remember that authentication happens first and authorization second.

---

## Part 10: Password Encoding

Storing passwords as plain text is **dangerous**. If anyone ever reads your database, your configuration file or your source code, every account is immediately compromised. Worse, because people reuse passwords, you have also handed over their accounts on other systems.

Spring Security therefore refuses to work with raw passwords at all and requires them to be encoded.

### Hashing, and Why It Is One-Way

Encoding a password here means **hashing** it. A hash function takes the password and produces a fixed-length scrambled value, and the process only runs in one direction. You can turn `admin123` into a hash, but you cannot take the hash and work backwards to recover `admin123`.

That raises an obvious question: if the password cannot be recovered, how does login work? The answer is that the system never recovers it. When a user logs in, their submitted password is hashed again and the two hashes are compared. If they match, the password was correct. The stored value is never turned back into a password, which is why a stolen database gives an attacker hashes rather than passwords.

### Create a Password Encoder Bean

```java
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder();
}
```

### Why BCrypt?

BCrypt is the standard choice in the Spring world for two reasons beyond simply being a hash function.

The first is **salting**. A plain hash always produces the same output for the same input, so two users who both chose `password123` would end up with identical stored hashes, and attackers can precompute hashes for millions of common passwords and simply look yours up. BCrypt defeats this by generating a random value, called a salt, for every single password and mixing it in before hashing. The same password hashed twice produces two completely different results. The salt is stored alongside the hash, so verification still works, and precomputed tables become useless. A quick way to see this for yourself: encode the same password twice and compare the output.

The second is that BCrypt is **deliberately slow**. Ordinary hash functions are designed to be fast, which is exactly what an attacker wants when trying billions of guesses. BCrypt includes a work factor that makes each hash take a noticeable amount of time. A real login is unaffected by a few hundred milliseconds. An attacker guessing their way through a stolen database very much is.

---

## Part 11: Creating Users with Roles

Now let's define users and roles.

```java
@Bean
public UserDetailsService userDetailsService(PasswordEncoder passwordEncoder) {

    UserDetails user = User.builder()
        .username("user")
        .password(passwordEncoder.encode("password"))
        .roles("USER")
        .build();

    UserDetails admin = User.builder()
        .username("admin")
        .password(passwordEncoder.encode("admin123"))
        .roles("ADMIN")
        .build();

    return new InMemoryUserDetailsManager(user, admin);
}
```

### What `UserDetailsService` Actually Is

`UserDetailsService` is an interface with a single method: given a username, return the details of that user, or fail if there is no such user. That is its entire job.

Spring Security depends on this interface rather than on any particular storage, and that is what makes the design flexible. The authentication filter does not know or care where users come from. It simply asks whoever implements this interface. Today we hand it `InMemoryUserDetailsManager`, which holds the two users we just built. Later you can hand it an implementation that reads from a database table, and **not one line of your security rules has to change**. This is the same dependency-injection thinking from Lesson 3.14, applied to authentication.

Notice that the password is passed through `passwordEncoder.encode(...)` before it is stored, never in plain text. Spring injects the `PasswordEncoder` bean you created in Part 10 into this method as a parameter.

> **Note:** `build()` returns a `UserDetails`, not a `User`. `User` is only the builder.

### The `ROLE_` Prefix

Spring Security stores permissions as **authorities**, which are just strings. Roles are a convention layered on top of that: a role is simply an authority whose name begins with `ROLE_`.

This is why `.roles("ADMIN")` actually stores the authority `ROLE_ADMIN`. The `roles()` method adds the prefix for you, and `hasRole("ADMIN")` adds it back when checking, so the two line up and everything works. The trap appears when you mix the two styles. `hasAuthority("ADMIN")` does **not** add a prefix, so it looks for an authority literally called `ADMIN`, does not find it because the stored value is `ROLE_ADMIN`, and denies the request with a 403 that is hard to explain. Stay with `roles()` and `hasRole()` together and you will not hit this.

Once this bean exists, the `spring.security.user.*` properties from Part 5 are ignored. You may remove them.

---

## Part 12: Role-Based Authorization (RBAC)

Now let's enhance security by applying **role-based rules**.

Update the security rules inside `securityFilterChain`. These rules **replace** the ones written earlier; you are editing the same `authorizeHttpRequests` block, not adding a second one.

```java
.authorizeHttpRequests(auth -> auth
    .requestMatchers(HttpMethod.GET, "/customers/**").hasAnyRole("USER", "ADMIN")
    .requestMatchers(HttpMethod.POST, "/customers/**").hasRole("ADMIN")
    .requestMatchers(HttpMethod.PUT, "/customers/**").hasRole("ADMIN")
    .requestMatchers(HttpMethod.DELETE, "/customers/**").hasRole("ADMIN")
    .anyRequest().authenticated()
)
```

### What This Means

- Both USER and ADMIN can view customers
- Only ADMIN can create, update, or delete customers
- All other requests require authentication

`hasRole("ADMIN")` requires that one role. `hasAnyRole("USER", "ADMIN")` passes if the user holds either. Both imply authentication, since a role cannot be checked until the user is known.

The final line has changed from `permitAll()` to `authenticated()`, and that change matters more than it looks. It means every endpoint in the application that you have not explicitly named now requires a login. In `simple-crm` that covers anything outside the customer paths, and, more importantly, it covers **any endpoint you add in future**. This is called failing closed: forget to write a rule for a new controller next month and it is protected by default rather than exposed by accident. Writing `permitAll()` there instead would produce the opposite, and much more dangerous, behaviour.

### Rules Follow URLs, Not Classes

Your Interaction endpoints, such as `POST /customers/{id}/interactions`, are already covered by the rules above. They were never mentioned anywhere, but their paths begin with `/customers/`, so `/customers/**` matches them, and the method-specific rules apply to them too.

This is worth pausing on. Spring Security matches on **URL patterns, not on classes or packages**. It has no idea which controller handles a path. If someone later moves those endpoints to `/interactions` in a controller of their own, they will silently stop matching these rules and fall through to `.anyRequest()`, quietly losing their ADMIN-only protection. Restructuring URLs is therefore a security change, even when no security code is touched.

---

## Part 13: Testing with Postman

In Postman, set credentials under the **Authorization** tab → **Basic Auth**. Remember that credentials are per request; set them once on the collection and leave each request as "Inherit auth from parent" to avoid retyping them.

### Test as USER
- Username: `user`
- Password: `password`
- Try:
  - GET `/customers` → ✅ Allowed
  - DELETE `/customers/{id}` → ❌ 403 Forbidden

### Test as ADMIN
- Username: `admin`
- Password: `admin123`
- Try:
  - GET `/customers` → ✅ Allowed
  - DELETE `/customers/{id}` → ✅ Allowed

### Test with No Credentials
- GET `/customers` → ❌ 401 Unauthorized

This demonstrates **both authentication and authorization on the same endpoint**. The contrast between the USER delete (403) and the anonymous read (401) is the clearest illustration of the difference: one caller was identified and refused, the other was never identified at all.

---

## Part 14: Hands-On Activity

### Add a MANAGER Role

Your CRM now has two roles, but real organisations rarely stop at two. Add a third.

**Step 1.** In `SecurityConfig`, add a third user to the `UserDetailsService`:
- Username: `manager`
- Password: `manager123`
- Role: `MANAGER`

**Step 2.** Update the rules in `securityFilterChain` so that:
- Everyone logged in (USER, MANAGER, ADMIN) can **read** customers
- MANAGER and ADMIN can **update** customers
- Only ADMIN can **create** or **delete** customers

**Step 3.** Verify your work in Postman against all three users. Your results should match this table exactly:

| Request | user | manager | admin |
|---|---|---|---|
| GET `/customers` | ✅ 200 | ✅ 200 | ✅ 200 |
| GET `/customers/{id}` | ✅ 200 | ✅ 200 | ✅ 200 |
| POST `/customers` | ❌ 403 | ❌ 403 | ✅ 201 |
| PUT `/customers/{id}` | ❌ 403 | ✅ 200 | ✅ 200 |
| DELETE `/customers/{id}` | ❌ 403 | ❌ 403 | ✅ 200 |

**Step 4.** Also confirm that sending **no credentials** to any of these returns `401`, not `403`, and be ready to explain why the two differ.

---

## Part 15: Troubleshooting

**`There is no PasswordEncoder mapped for the id "null"`**  
Your `PasswordEncoder` bean is missing, or a password was stored without being encoded.

**Still getting 401 after setting the properties**  
Restart the application. Also check there is no `UserDetailsService` bean overriding them.

**401 on an endpoint you made `permitAll()`**  
Postman is still sending credentials and they are invalid. Set that request's Authorization tab to **No Auth**.

**403 when you expected access**  
Check the role spelling, and remember the `ROLE_` prefix rule from Part 11. Also check rule order — an earlier rule may have matched first.

**Generated password keeps changing**  
That is expected. Use the `application.properties` approach from Part 5.

---

## Part 16: Key Takeaways

- Spring Security intercepts requests in a filter chain, before controllers run
- Security is deny by default; you open access up deliberately
- Authentication verifies identity (401), authorization verifies permissions (403)
- Rules are evaluated top to bottom, first match wins, and `anyRequest()` must be last
- `authorizeHttpRequests` decides what is protected; `httpBasic` decides how you prove who you are
- Passwords must always be hashed, and BCrypt salts every one individually
- Rules match URLs, not classes, so changing a path changes its security

---

## What's Next?

In the next lesson, we will:
- Integrate security with persistent users
- Explore token-based authentication concepts
- Prepare the application for production-grade security

---