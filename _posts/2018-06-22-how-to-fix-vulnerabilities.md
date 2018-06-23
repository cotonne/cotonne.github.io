---
layout: post
title:  "How to fix vulnerabilities? (The 'Path Transversal' case)"
date:   2018-06-22 08:42:27 +0100
categories: secure design methodology craft
---

Some days ago, I was with a client. An audit has revealed some weaknesses. As a good habit, he wanted to fix them. But, correcting weaknesses are not so easy, even more, if the codebase is a legacy one.

In this article, I will take about the resolution of one of the weaknesses: path transversal. We will see how to treat the problem. Then let's find what is the root cause of this vulnerability. Some technics can help us in preventing the apparition of such vulnerability.

# What is a Path transversal vulnerability?

Imagine that you have a webpage design used as a download page. You can store your file in a remote server ("/a/b" for example). Your user provides the file and a name ("data" for example). The application saves the file into the remote file **/a/b/data**.

But what if he provides a value like "../../etc/paswd" as a name. The application will saves the file as **/a/b/../../etc/passwd**. If we normalize the path, we got **/etc/passwd**. So, given the fact that your application can have root access, you can manipulate important configuration files from your server.

# Fix it!

The first thing to do is to fix the problem and to fix it. How can we fix it in an efficient way? How can we be sure that the vulnerability will not come back?

## Test it!

You need to have different test cases. You can use the [OWASP Path Transversal](https://www.owasp.org/index.php/Testing_Directory_traversal/file_include_(OTG-AUTHZ-001)) as a source of inspiration.

Given your architecture, you will need to test
 - for different platforms (Windows, Unix, ...)
 - for different encodings

## Solutions

Let's see the different solutions with a secure coding approach.

### Check input

You can modify the input to make it safe to use. Different approaches are possible.
You can
 - remove all characters which are in a list (blacklist approach)
 - remove all characters which are **not** in a list (whitelist approach)

```java
public class SanitizerShould {
    @Test
    public void sanitize_a_path_blacklist() {
        String unsafeFileName = "/test.txt";
        String sanitizedFileName = sanitizeWithBlackList(unsafeFileName);
        assertEquals("test.txt", sanitizedFileName);
    }

    private String sanitizeWithBlackList(String unsafeFileName) {
        return unsafeFileName.replaceAll("/", "");
    }

    @Test
    public void sanitize_a_path_whitelist() {
        String unsafeFileName = "/test.txt";
        String sanitizedFileName = sanitizeWithWhiteList(unsafeFileName);
        assertEquals("test.txt", sanitizedFileName);
    }

    private String sanitizeWithWhiteList(String unsafeFileName) {
        return unsafeFileName.replaceAll("[^\\w\\d.]", "");
    }
}
```

I will advise you to have a whitelist approach. In fact, you can have unknown cases, so you need as restrictive as you can. Even with this approach, this is not bullet-proof so be careful.

You can mix this approach:
 - A blacklist of dangerous characters (like /). If you find any, the application can reject the request and generates an alert. It is not a normal and expected value that the front can send.
 - A whitelist of valid characters 

### Check the folder

With the previous example, we expect files to be in the folder **/a/b**. So, we can check the folder before doing any action.

```java
    @Test
    public void validate_folder() {
        String unsafeFileName = "../test.txt";
        boolean isSaveInTheCorrectFolder = validateFolder("/a/b/", unsafeFileName);
        assertFalse(isSaveInTheCorrectFolder);
    }

    private boolean validateFolder(String folder, String unsafeFileName) {
        try {
            File finalFile = new File(folder + unsafeFileName);
            System.out.println(finalFile.getCanonicalPath());
            return finalFile.getCanonicalPath().startsWith(folder);
        } catch (IOException e) {
            return false;
        }
    }
```

Now that we have fixed this vulnerability, we need to understand why we let it go to production.
We want to find the root cause of the problem.

# Why does it happen to me?

Different principles from secure by design principles should have been applied 
 - **Trust cautiously**: we have not imagined that the user or someone could send use invalid or dangerous values
 - **Least privilege**: our application has too many rights. The application should not have been able to access other folders except the upload folder. Even in this case, we need to check the rights of a user to access a file (**Complete Mediation**)
 - **Defense in depth**: we should have designed our application to have many lines of defense. For example, we can use some controls in the code **and** a specific user to restrict rights of the application
 - **KISS**: the codebase was complex. Security vulnerabilities are a pay off of technical debt.  You need to constantly invest in the remediation of the debt even for end-of-life applications. New vulnerabilities are discovered every day. So even if your application is safe today, it doesn't mean it will be safe tomorrow.

# Preventing the unknown unknown

Given this analysis, we can start to imagine solutions which can prevent future and unknown vulnerabilities from being possible.

## Trust cautiously

Don't think that you can't receive dangerous data. Your application needs to have a kind of "input phobia" as input can be harmful to your application.

If you lack ideas of dangerous data, have a look at existing checklists. The [OWASP XSS Filter Evasion Cheat Sheet](https://www.owasp.org/index.php/XSS_Filter_Evasion_Cheat_Sheet) can be a source of inspiration.

You can also try contests like [CTF](https://en.wikipedia.org/wiki/Capture_the_flag#Computer_security) and challenges around obfuscation or security enigmas to mold your mind. Training with professionals can also help you improve as a developer

## Least privilege

To limit rights, there are a lot of different ways:
 - application-specific: you implement a way to check and restrict rights. You can develop an RBAC or ABAC system. Yet, you need to be careful as it is a sensitive and complex functionality. Reuse as much as you can safe, tested and trusted software.
 - framework-specific: some frameworks have built-in solutions for such problems. For example, Java has a special file called ["Java Security Policy"](https://docs.oracle.com/javase/7/docs/technotes/guides/security/PolicyFiles.html) where you define fine-grained rights. By default, it denies access to resources **default deny**. So you need to be specific. If not, you can have the following exception

```
Exception in thread "main" java.security.AccessControlException: access denied ("java.io.FilePermission" "<a_file>" "write")
```

 - system-specific: you can create a specific user and group with no write or read access to any folder except the upload one.
 - distribution-specific: have a look to SELinux or AppArmor for Linux. They harden a system.

## KISS

A good way to keep your codebase in a good shape is to look at the practices from the craft community: TDD, Clean Code, Craftsmanship spirit and so on.

To not lost security rules, like input validation rules for path, you can define a class designed to encapsulate a dangerous input:
```java
    private class SafeFile {
        private final String name;

        private SafeFile(String name) {
            this.name = name;
        }

        public static SafeFile enforce(String name) {
            if(!name.matches("<dangerous characters>"))
                throw new SecurityException();
            return new SafeFile(name);
        }
    }
```


Once a vulnerability is found, a fix should be made. Don't panic and take time to do it properly.
After the problem is resolved, you can start a post-mortem analysis to improve yourself.
Whenever applying secure by design principles reduces the risk of a new vulnerability.
