---
layout: post
title:  "Deep dive into Spring Security - part 2"
date:   2019-10-24 08:42:27 +0100
categories: gdb 
---

Understanding Spring Security - part 2

In the [last post about Spring](https://cotonne.github.io/java/spring/security/2019/08/04/spring-security-1.html), 
we talked about how Spring Security managed **Authentication**. Today, we are going to see how Spring managed Authorization.

For this post, we are going to use a [Spring project with ACL](https://github.com/cotonne/security/tree/master/demo-acl).

# Trying to access an unauthorized resource

Let's analyze what happens when we try to access http://localhost:8080/is-allowed-as-user, 
http://localhost:8080/is-also-allowed-as-user and 
http://localhost:8080/isallowed-as-admin endpoints with the credentials: user/password.
This two endpoints are doing authorization based on a RBAC matrix.

Depending on your code, Spring is going to allow access to resources in different ways.

## FilterSecurityInterceptor

You can control access to resource by defining authorization in your configuration : 

```java
public class SecurityConfiguration extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
                .authorizeRequests()
                    .antMatchers("/is-also-allowed-as-user").hasRole("ADMIN")
                .and()
                .formLogin().and().httpBasic();
    }
    ...
}
```

If you try to access the resource (with `curl http://user:password@localhost:8080/is-also-allowed-as-user), you will get this log;

```
o.s.s.w.u.matcher.AntPathRequestMatcher  : Checking match of request : '/is-also-allowed-as-user'; against '/is-also-allowed-as-user'
o.s.s.w.a.i.FilterSecurityInterceptor    : Secure object: FilterInvocation: URL: /is-also-allowed-as-user; Attributes: [hasRole('ROLE_USER')]
o.s.s.w.a.i.FilterSecurityInterceptor    : Previously Authenticated: org.springframework.security.authentication.UsernamePasswordAuthenticationToken@4428690f: Principal: org.springframework.security.core.userdetails.User@36ebcb: Username: user; Password: [PROTECTED]; Enabled: true; AccountNonExpired: true; credentialsNonExpired: true; AccountNonLocked: true; Granted Authorities: ROLE_USER; Credentials: [PROTECTED]; Authenticated: true; Details: org.springframework.security.web.authentication.WebAuthenticationDetails@380f4: RemoteIpAddress: 0:0:0:0:0:0:0:1; SessionId: 4AE0767E72F180252CFC8479DEDF553F; Granted Authorities: ROLE_USER
o.s.s.access.vote.AffirmativeBased       : Voter: org.springframework.security.web.access.expression.WebExpressionVoter@403e8a59, returned: 1
o.s.s.w.a.i.FilterSecurityInterceptor    : Authorization successful
```

**org.springframework.security.web.access.intercept.FilterSecurityInterceptor** is the same kind of filter as in the authentication part with the FilterChain.
FilterSecurityInterceptor also extends **org.springframework.security.access.intercept.AbstractSecurityInterceptor** which is also used for the second type of access control.
AbstractSecurityInterceptor has a function called **beforeInvocation** which called the **AccessDecisionManager**. AccessDecisionManager then asks "Voters" if they agree or not.

Before going deeper into AccessDecisionManager and Voters, let's see a second authorization mode.

## Expression-based Access Control

You can used **@PreAuthorize** or **@PostAuthorize** to control access to the resource:

```java
@RestController
public class HelloController {
  @GetMapping(path = "is-allowed-as-user")
  @PreAuthorize("hasRole('ROLE_USER')")
  public String is_allowed_as_user() {
    return "yes";
  }
  ...
}
```

You need to enable such annotations with:

```
@EnableGlobalMethodSecurity(
  prePostEnabled = true, 
  securedEnabled = true)
)
```

At the boot time, Spring detects and "caches" methods enclosed with those annotations.

```
PrePostAnnotationSecurityMetadataSource: @org.springframework.security.access.prepost.PreAuthorize(value="hasRole('ROLE_USER')") found on specific method: public java.lang.String com.example.demoacl.HelloController.isallowed_as_user()
DelegatingMethodSecurityMetadataSource : Caching method [CacheKey[com.example.demoacl.HelloController; public java.lang.String com.example.demoacl.HelloController.isallowed_as_user()]] with attributes [[authorize: 'hasRole('ROLE_USER')', filter: 'null', filterTarget: 'null']]
```

When you try to access the endpoint as user (you can use `curl -s  http://user:password@localhost:8080/is-allowed-as-user`), Spring checks if you are allowed to acces the resource :

```
o.s.s.a.i.a.MethodSecurityInterceptor    : Secure object: ReflectiveMethodInvocation: public java.lang.String com.example.demoacl.HelloController.isallowed_as_user(); target is of class [com.example.demoacl.HelloController]; Attributes: [[authorize: 'hasRole('ROLE_USER')', filter: 'null', filterTarget: 'null']]
o.s.s.a.i.a.MethodSecurityInterceptor    : Previously Authenticated: org.springframework.security.authentication.UsernamePasswordAuthenticationToken@442b7c85: Principal: org.springframework.security.core.userdetails.User@36ebcb: Username: user; Password: [PROTECTED]; Enabled: true; AccountNonExpired: true; credentialsNonExpired: true; AccountNonLocked: true; Granted Authorities: ROLE_USER; Credentials: [PROTECTED]; Authenticated: true; Details: org.springframework.security.web.authentication.WebAuthenticationDetails@957e: RemoteIpAddress: 127.0.0.1; SessionId: null; Granted Authorities: ROLE_USER
o.s.s.access.vote.AffirmativeBased       : Voter: org.springframework.security.access.prepost.PreInvocationAuthorizationAdviceVoter@5cbd422e, returned: 1
o.s.s.a.i.a.MethodSecurityInterceptor    : Authorization successful
```

Here, we have **MethodSecurityInterceptor** which also extends **AbstractSecurityInterceptor**. So, access decision is using the same mechanism as the FilterInterceptor.
However, we can notice that the "Voters" is different.

Let's see how this mechanism works.

# Taking decisions

Spring bases his decision for granting access on two important concepts: **AccessDecisionManger** and **Voter**.

Here is a [good example of how to customize it](https://www.baeldung.com/spring-security-custom-voter).

## AccessDecisionManager

As stated in the [Spring Reference](https://docs.spring.io/spring-security/site/docs/current/reference/html/authorization.html): *The AccessDecisionManager [...] is responsible for making final access control decisions*.

AccessDecisionManager asks its Voters their vote. Then, given the number of votes for/abstain/against, there are different ways to decide if we can grant access or not:
 - AffirmativeBased: *Denies access only if there was a deny vote AND no affirmative votes.*
 - UnanimousBased: if anyone deny, just reject.
 - ConsensusBased: The code gives us a good understanding of the decision:
```java
public void decide(Authentication authentication, Object object,
			Collection<ConfigAttribute> configAttributes) throws AccessDeniedException {
                
	for (AccessDecisionVoter voter : getDecisionVoters()) {
		int result = voter.vote(authentication, object, configAttributes);

		switch (result) {
		case AccessDecisionVoter.ACCESS_GRANTED:
			grant++;
			break;
		case AccessDecisionVoter.ACCESS_DENIED:
			deny++;
			break;
		default:
			break;
		}
	}

	if (grant > deny) {
		return;
	}

	if (deny > grant) {
		throw new AccessDeniedException(messages.getMessage(
				"AbstractAccessDecisionManager.accessDenied", "Access is denied"));
	}

	if ((grant == deny) && (grant != 0)) {
		if (this.allowIfEqualGrantedDeniedDecisions) {
			return;
		}
		else {
			throw new AccessDeniedException(messages.getMessage(
					"AbstractAccessDecisionManager.accessDenied", "Access is denied"));
		}
	}
```

So, decision rules are :
 * **grant > deny** : allow
 * **deny > grant**: reject
 * **deny == grant**: depend on your configuration

## Voter

Voters implements the interface **AccessDecisionVoter** which has a function **vote**.
So, you can create your own voter. For example, you might decide to allow access to a resource based on IP reputation for example. By the way, Spring offers ip filtering (with .antMatchers().hasIpAddress("allowed.ip")).

There are different kinds of voters:
 - WebExpressionVoter: used mainly for the FilterSecurityInterceptor. If the URL is matching the authorization and the role of the principal, you can access the resource
 - PreInvocationAuthorizationAdviceVoter#vote calls ExpressionBasedPreInvocationAdvice  which evaluate @PreAuthorize(<CONTENT>) with DefaultMethodSecurityExpressionHandler
 - RoleVoter and RoleHierarchyVoter: an interesting one. Let's analyzing it!
 
## RoleVoter and RoleHierarchyVoter

RoleVoter voter allows you to access the resource if you have a role which matches the allowed role.
In our example, we have allowed any principal with the role "USER" to access */is-also-allowed-as-user*.

```
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
                .authorizeRequests()
                    .antMatchers("/is-also-allowed-as-user").hasRole("USER")
        ...
    }
```

So, if you try to access the resource with `curl http://user:password@localhost:8080/is-also-allowed-as-user`,
you will get a successful response. However, let's say that we have a MANAGER role, which is an extended user.
If i try to access the resource with the manager account (`curl http://user:password@localhost:8080/is-also-allowed-as-user`), you get a 401 error (Unauthorized).

One solution is to use `hasAnyRole("USER", "MANAGER")` to allow access. But, you will need to remember the link
between roles USER and MANAGER. Here comes RoleHierarchyVoter. RoleHierarchyVoter gives you the opportunity to
define this hierarchy.

Defining Role Hierarchy doesn't seem to be obvious. Based on this [StackOverflow answer](https://stackoverflow.com/a/30461132):

```java
public class SecurityConfiguration extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
                .authorizeRequests()
                    .expressionHandler(webExpressionHandler())
                    .antMatchers("/is-also-allowed-as-user").hasRole("USER")
        ...
    }

    private SecurityExpressionHandler<FilterInvocation> webExpressionHandler() {
        RoleHierarchyImpl roleHierarchy = new RoleHierarchyImpl();
        roleHierarchy.setHierarchy("ROLE_MANAGER > ROLE_USER");
        DefaultWebSecurityExpressionHandler defaultWebSecurityExpressionHandler = new DefaultWebSecurityExpressionHandler();
        defaultWebSecurityExpressionHandler.setRoleHierarchy(roleHierarchy);
        return defaultWebSecurityExpressionHandler;
    }
    ...
}
```

The important part is: **ROLE_MANAGER > ROLE_USER**. It means: "MANAGER" is included in "USER" role.
(Full reference [here](https://docs.spring.io/spring-security/site/docs/current/reference/html/authorization.html#authz-hierarchical-roles)).
Note that Spring likes to [prefix with "ROLE" everywhere"](https://docs.spring.io/autorepo/docs/spring-security/current/reference/htmlsingle/#appendix-faq-role-prefix).

## Fine-graine access control: ACL

We have seen how to define an RBAC with two roles: USER and MANAGER.
Spring also offers [ACL](https://docs.spring.io/spring-security/site/docs/current/reference/html/authorization.html#domain-acls).

With ACL, you can define who owns a resource, who can see per user or per group...
Spring provides a [default database schema](https://docs.spring.io/spring-security/site/docs/current/reference/html/authorization.html#domain-acls).

In the project, you will find:
 - [An example for the configuration](https://github.com/cotonne/security/blob/master/demo-acl/src/main/java/com/example/demoacl/AclConfiguration.java)
 - The [schema](https://github.com/cotonne/security/blob/master/demo-acl/src/main/resources/acl-schema.sql) and [data with two users](https://github.com/cotonne/security/blob/master/demo-acl/src/main/resources/acl-data.sql)
 - [Tests to validate the configuration](https://github.com/cotonne/security/blob/master/demo-acl/src/test/java/com/example/demoacl/DemoAclApplicationTests.java)

# Conclusion

So, we see how the access control is implemented in Spring. It uses the same centralized system as the authentication.
Also, it offers a lot of possibilites and options so you can rely fine tune your security.

We can see that Spring does authentication **before** authorization, which is the good way of doing it.

```
2019-10-24 14:11:35.896 DEBUG 21930 --- [nio-8080-exec-1] o.s.s.w.a.www.BasicAuthenticationFilter  : Authentication success: org.springframework.security.authentication.UsernamePasswordAuthenticationToken@eb81d3: Principal: org.springframework.security.core.userdetails.User@31c90fad: Username: manager; Password: [PROTECTED]; Enabled: true; AccountNonExpired: true; credentialsNonExpired: true; AccountNonLocked: true; Granted Authorities: ROLE_MANAGER; Credentials: [PROTECTED]; Authenticated: true; Details: org.springframework.security.web.authentication.WebAuthenticationDetails@957e: RemoteIpAddress: 127.0.0.1; SessionId: null; Granted Authorities: ROLE_MANAGER
...
2019-10-24 14:11:35.913 DEBUG 21930 --- [nio-8080-exec-1] o.s.s.w.a.i.FilterSecurityInterceptor    : Authorization successful

```

You just have to properly configure it and Spring manages authorization in a secure way.
