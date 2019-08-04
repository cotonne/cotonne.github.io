---
layout: post
title:  "Deep dive into Spring Security"
date:   2019-08-04 08:42:27 +0100
categories: java spring security
---

During a long time, I was lost with Spring Security. How does it works? Why should we do this or that?
This series of posts aim to clarify how Spring Security works and what are the mechanisms in place.
We will also see how Spring Security 5 is integrated with Spring Boot 2.
I will mainly focus on **authentication** (basic, OAuth2, Kerberos...) and **authorization** (ACL, RBAC).
I will also cover **MVC** and **Reactive** controllers. Both are fundamental and incompatibles differences.

As a pet project, you can go to [Spring Initializr](https://start.spring.io/) and create a project with this module:

 * spring-boot-starter-security
 * spring-boot-starter-web

Then, just create an **HelloController**.

# What happens when I boot my application the first time?

Let's boot our project with `mvn spring-boot:run`. You can access [hello at http://localhost:8080/hello](http://localhost:8080/hello). And magic (as a lot of things with Spring)! You have been redirected to a form to login. 
But, we did not create this form. And, how can I sign-in?

Let's check with **cURL**:
```
curl -v http://localhost:8080/hello
*   Trying 127.0.0.1...
* TCP_NODELAY set
* Connected to localhost (127.0.0.1) port 8080 (#0)
> GET /hello HTTP/1.1
> Host: localhost:8080
> User-Agent: curl/7.55.1
> Accept: */*
> 
< HTTP/1.1 401 
< Set-Cookie: JSESSIONID=CA6F50EC9970A53481C25A5B3650ADD7; Path=/; HttpOnly
< WWW-Authenticate: Basic realm="Realm"
< X-Content-Type-Options: nosniff
< X-XSS-Protection: 1; mode=block
< Cache-Control: no-cache, no-store, max-age=0, must-revalidate
< Pragma: no-cache
< Expires: 0
< X-Frame-Options: DENY
< Content-Type: application/json;charset=UTF-8
< Transfer-Encoding: chunked
< Date: Sun, 04 Aug 2019 12:17:37 GMT
< 
* Connection #0 to host localhost left intact
{"timestamp":"2019-08-04T12:17:37.375+0000","status":401,"error":"Unauthorized","message":"Unauthorized","path":"/hello"}
```

We do have a lot of interesting things in place (security headers like **X-XSS-Protection**, **X-Frame-Options**...). However, we have not been redirected. Spring Security is smart enough to do the difference between a call from the browser and a call for accessing the API. It relies on the **Accept** header sent in the request:

```
curl -v http://localhost:8080/hello -H 'Accept: application/xhtml+xml'
...
< HTTP/1.1 302
...
< Location: http://localhost:8080/login
```

Great! But, still where is the password? If you look at the logs, you will see something like this:

```
INFO --- [           main] .s.s.UserDetailsServiceAutoConfiguration : 

Using generated security password: 11329144-b697-4a5c-bc7f-46c5943fec86
```

Spring Security enforces a great security principle: **[Secure by default](https://en.wikipedia.org/wiki/Secure_by_default)**.
Which means that Spring Security will enable security controls that protects you.
So, you will have to disable it voluntarily by yourself.

But, how did I know that the user was **user**? First, the [doc states it](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-security.html). Also, you can look in the Spring source code of the class [User](org.springframework.boot.autoconfigure.security.SecurityProperties.User):

```java
public static class User {
	/**
	 * Default user name.
	 */
	private String name = "user";

	/**
	 * Password for the default user name.
	 */
	private String password = UUID.randomUUID().toString();
```

This configuration has been enabled due to the absence of beans which are in charge of doing the authentication.

```java
@ConditionalOnMissingBean({ AuthenticationManager.class, AuthenticationProvider.class, UserDetailsService.class })
public class UserDetailsServiceAutoConfiguration {
```

The configuration adds a class which implements `UserDetailsService`. It has only one method which, given a username, returns the whole user with his details (password, ...).

```java
public interface UserDetailsService {
	UserDetails loadUserByUsername(String username) throws UsernameNotFoundException;
}
```

Great! But, where do we do the authentication (the action of validating that the user has the identity he claims)? Let's continue to dive into the logs. We enable DEBUG by setting **logging.level.org.springframework.security=DEBUG**
in the file *application.properties*.

```
DEBUG --- [           main] eGlobalAuthenticationAutowiredConfigurer : Eagerly initializing {org.springframework.boot.autoconfigure.security.servlet.WebSecurityEnablerConfiguration=org.springframework.boot.autoconfigure.security.servlet.WebSecurityEnablerConfiguration$$EnhancerBySpringCGLIB$$d7f4fb37@6ada9c0c}
...
DEBUG --- [           main] s.s.c.a.w.c.WebSecurityConfigurerAdapter : Using default configure(HttpSecurity). If subclassed this will potentially override subclass configure(HttpSecurity).
...
INFO --- [           main] o.s.s.web.DefaultSecurityFilterChain     : Creating filter chain: any request, [org.springframework.security.web.context.request.async.WebAsyncManagerIntegrationFilter@574a89e2,
org.springframework.security.web.context.SecurityContextPersistenceFilter@5a1c3cb4, 
org.springframework.security.web.header.HeaderWriterFilter@142213d5, 
org.springframework.security.web.csrf.CsrfFilter@20b9d5d5, 
org.springframework.security.web.authentication.logout.LogoutFilter@3a66e67e, 
org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter@623ebac7, 
org.springframework.security.web.authentication.ui.DefaultLoginPageGeneratingFilter@2f5b8250, 
org.springframework.security.web.authentication.ui.DefaultLogoutPageGeneratingFilter@1e1e9ef3, 
org.springframework.security.web.authentication.www.BasicAuthenticationFilter@79a04e5f, 
org.springframework.security.web.savedrequest.RequestCacheAwareFilter@56637cff, 
org.springframework.security.web.servletapi.SecurityContextHolderAwareRequestFilter@48b4a043, 
org.springframework.security.web.authentication.AnonymousAuthenticationFilter@3dd31157, 
org.springframework.security.web.session.SessionManagementFilter@2630dbc4, 
org.springframework.security.web.access.ExceptionTranslationFilter@53cf9c99, 
org.springframework.security.web.access.intercept.FilterSecurityInterceptor@3b920bdc]
```

# How Spring Security is enabled?

With this log, we are now able to answer this question! When the application boot, `org.springframework.boot.autoconfigure.security.servlet.WebSecurityEnablerConfiguration`is found.
According to the documentation, if the user forgets to add the annotation, it is added automatically. It is designed to prevent the user for making a mistake.

This configuration is activated by another configuration: `org.springframework.boot.autoconfigure.security.servlet.SecurityAutoConfiguration`. SecurityAutoConfiguration also enables
the configuration for Spring Data and `org.springframework.boot.autoconfigure.security.servlet.SpringBootWebSecurityConfiguration`. 
**SpringBootWebSecurityConfiguration** provides a default **org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter**.
And, finally, **WebSecurityConfigurerAdapter** defines the configuration:

```java
	protected void configure(HttpSecurity http) throws Exception {
		logger.debug("Using default configure(HttpSecurity). If subclassed this will potentially override subclass configure(HttpSecurity).");

		http
			.authorizeRequests()
				.anyRequest().authenticated()
				.and()
			.formLogin().and()
			.httpBasic();
	}
```

# How do we authenticate an user?

The authentication is done by the **AuthenticationManager**. The default one created by **WebSecurityConfigurerAdapter** is **org.springframework.security.authentication.ProviderManager**.

By default, the check is done by `org.springframework.security.authentication.dao.DaoAuthenticationProvider` which is
*An AuthenticationProvider implementation that retrieves user details from a UserDetailsService*. 
Everything connects together!

`org.springframework.security.authentication.AccountStatusUserDetailsChecker`, the abstract class of **DaoAuthenticationProvider** does a set of checks before verifying if the user is correct:

 * Is the account locked?
 * Is the account enabled?
 * Is the account expired?

So, Spring Security gives us a fine-graine control over the state of the user account. We should not have to 
re-implement that.

Once done, the authentication can append in `org.springframework.security.authentication.dao.DaoAuthenticationProvider#additionalAuthenticationChecks`.

```java
if (!passwordEncoder.matches(presentedPassword, userDetails.getPassword())) {
	throw new BadCredentialsException(messages.getMessage(
			"AbstractUserDetailsAuthenticationProvider.badCredentials",
			"Bad credentials"));
}
```

The comparison of expected password and provided password is done by a PasswordEncoder.
The default password encoder is `org.springframework.security.crypto.password.NoOpPasswordEncoder`.
However, it is not a safe as it is sensible to time. When you compare two strings in Java,
the **equals** function returns a result as soon as one character is different.
An attacker can use that to test the expected password.
Don't worry! In Spring, other password encoders are safe. For example, you can have a look at **BCryptPasswordEncoder**. It called **equalsNoEarlyReturn** which is secure function for comparing strings.

# Spring Security Filter

Last part we have in our log is **Filter**. As you can see, we have a lot of **Filter**. 
Spring Security is based on the **"[Secure Chain of responsibility Pattern](https://www.jpcert.or.jp/research/2009/SecureDesignPatterns-E_091104.pdf).
In this pattern, every filter has a small responsibility and checks if it can do something.

They are build in `org.springframework.security.config.annotation.web.configuration.WebSecurityConfiguration#springSecurityFilterChain`.

Once a request is received, it goes through filters until it gets stop (for example, because you are not authenticated) or until the end.

In the log, we see that there is a filter called `org.springframework.security.web.authentication.www.BasicAuthenticationFilter`.
Here is an extract of the code of this filter:

```java
public class BasicAuthenticationFilter extends OncePerRequestFilter {
	@Override
	protected void doFilterInternal(HttpServletRequest request,
			HttpServletResponse response, FilterChain chain)
					throws IOException, ServletException {

		String header = request.getHeader("Authorization");

		if (header == null || !header.toLowerCase().startsWith("basic ")) { 
		    // If we don't find a header "Authorization: basic", it might be another method so, let's continue
		    // With OAuth2, it can be "Authorization: Bearer"
			chain.doFilter(request, response);
			return;
		}

		try {
			String[] tokens = extractAndDecodeHeader(header, request);
			assert tokens.length == 2;

			String username = tokens[0];

			if (authenticationIsRequired(username)) {
			    // Have a look to org.springframework.security.web.authentication.www.BasicAuthenticationFilter#authenticationIsRequired
			    // as a lot of interesting case are described which might implement the need to re-authenticate
			    
				UsernamePasswordAuthenticationToken authRequest = new UsernamePasswordAuthenticationToken(	username, tokens[1]);
				authRequest.setDetails(this.authenticationDetailsSource.buildDetails(request));

                // Authenticate the user
				Authentication authResult = this.authenticationManager.authenticate(authRequest);

				// In case of success, no exception should be thrown
				// We will see in another post what is SecurityContextHolder.getContext()
				SecurityContextHolder.getContext().setAuthentication(authResult);

				this.rememberMeServices.loginSuccess(request, response, authResult);

				onSuccessfulAuthentication(request, response, authResult);
			}

		} catch (AuthenticationException failed) {
		    ...
            // Fail! Stop here!
			return;
		}

        // Let's continue with the others filters as one might blocked the user
        // Maybe because of an invalid CSRF token
		chain.doFilter(request, response);
	}
```

You can use the debugger from your IDE and set a breakpoint in this function. You can trigger it with

```bash
$ curl -v http://localhost:8080/hello -H "Authorization: basic $(echo -n 'user:<YOUR SPRING PASSWORD FROM THE LOG>' | base64)"
```

# Conclusion

In this post, we have seen some of the basic parts of Spring Security:

 * What happens when you start your Spring app with Spring Security
 * How Spring Security protects your application if you make mistakes
 * The default behavior of authentication
 * Spring Filters

In the next articles, we will talked about:

 * Customizing Security in Spring Security
 * Authorization
 * Advanced parts like custom filters
 * Testing with Spring Security

And we will try to answer to this question: "How secure is Spring Security?"