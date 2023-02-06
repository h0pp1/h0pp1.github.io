---
title: '[TIL] Mass Assignment Vulnerability in Spring Boot'
author: hoppi
date: 2023-02-06 15:13:00 + 0000
categories: [Web]
tags: [mass_assignment, spring_boot, dreamhack]
---
Yesterday, I solved [Dreampring](https://dreamhack.io/wargame/challenges/551/), web challenge in dreamhack.  
This application was built with `Spring Boot` which is commonly used in everywhere.  
In this game, the keyword is `Mass Assignment Vulnerability`. Today I'm going to talk about this vulnerability.  
<br/>

# What is Mass Assignment Vulnerability?
***
Software frameworks sometime allow developers to automatically bind HTTP request parameters into program code variables or objects. This Character of software frameworks can lead to unintended way. To put it Simply, Attacker can add new parameters in request so overwrites objects or new variable.  

In Spring MVC, its also called `Auto-Binding`. And there is alternative name of Mass Assignment.  
- Mass Assignment: Ruby on Rails, NodeJS
- Auto-Binding: Spring MVC, ASP NET MVC
- Object injection: PHP
<br/>

# Example
***
I will explain based on Spring MVC. Suppose there is a form for registering user's account.  
```html
<form>
     <input name="username" type="text">
     <input name="password" type="text">
     <input name="email" text="text">
     <input type="submit">
</form> 
```
{: file='regster.jsp'}
<br/>

And there is Model that the form binding to:  
```java
public class UserModel {
   private String username;
   private String password;
   private String email;
   private boolean isAdmin;

   //Getters & Setters
}
```
{: file='UserModel.java'}  
<br/>

And there is Controller which is handling HTTP request.
```java
...
@RequestMapping(value = "/register", method = RequestMethod.POST)
public String submit(UserModel user) {
   userService.register(user);
   return "successPage";
}
```
{: file='RegisterController.java'}  
<br/>

User's normally HTTP request like this:
```plaintext
POST /register
...
username=hoppi&password=guest1234&email=email@example.com
```
<br/>

But Attacker can add `isAdmin` parameter(attribute) which is need to access admin form or page.  
```plaintext
POST /register
...
username=attacker&password=guest1234&email=email@example.com&isAdmin=true
```
<br/>

# Mitigation
***
The best way preventing this vulnerability is setting only attribute which is editable by user when you design the Model.  

In Spring MVC, You can use `@InitBinder` to restrict attribute that is binding to model or object. Like is:
```java
@Controller
public class UserController
{
    @InitBinder
    public void initBinder(WebDataBinder binder, WebRequest request)
    {
        binder.setAllowedFields(["username","password","email"]);
    }
...
}
```
<br/>

Or Using blacklisting way.  
```java
@Controller
public class UserController
{
   @InitBinder
   public void initBinder(WebDataBinder binder, WebRequest request)
   {
      binder.setDisallowedFields(["isAdmin"]);
   }
...
}
```

# Reference
***
- [https://cheatsheetseries.owasp.org/cheatsheets/Mass_Assignment_Cheat_Sheet.html](https://cheatsheetseries.owasp.org/cheatsheets/Mass_Assignment_Cheat_Sheet.html)