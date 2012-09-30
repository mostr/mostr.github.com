---
layout: post
title: Tricky verification of final methods with Mockito
date: 2012-09-30 21:02
comments: true
categories: java mockito unittest
---

So here it is, my first blog post. Anyway, let's stop all those celebrations and let's go straight to the topic.

We developers test our code in order to deliver high quality software, right? If you code java, there is a chance you use Mockito as your framework of choice for providing your tests with test doubles objects. As stated in documentation, Mockito doesn't work with `final` methods, which effectively means that you can neither stub nor verify those methods. 
But what actually happens when you attempt to verify final method call? Let's say you didn't notice this method was declared final.  It turns out that your tests can pass. Moreover, the method you try to verify doesn't really have to be called in your code. I'll show you the case similar to one we had recently when there was method in our codebase that was declared final and not being aware of that we were trying to mock it.

Let's say team is coding users registration feature as below:

``` java UserRegistrationService.java
    public class UserRegistrationService {
    
        private UsersRepository usersRepository;
    
        public UserRegistrationService(UsersRepository usersRepository) {
            this.usersRepository = usersRepository;
        }
    
        public void registerUser(User newUser) {
            if(newUser.hasEmailDomainBanned()) {
                throw new SpamProtectionException("Email address domain is blacklisted");
            }
            // missing call to usersRepository.add()
        }
    
        public static class SpamProtectionException extends RuntimeException{    
            public SpamProtectionException(String message) {
                super(message);
            }
        }
        
    }
```

`UsersRepository` was defined by your teammate as follows and contains **final** method, because of... I don't know what. Anyway:

``` java UsersRepository.java
    public class UsersRepository {
    
        // beware: method declared final!
        public final void add(User newUser) {
            System.out.println("persistent storage call will be here");
        }
    
    }
```    

And here goes the first test (with JUnit and statically imported Mockito methods) to verify if registered user is passed to persist by calling `add` method on `usersRepository`. But if you take a look at `UserRegistrationService` you can spot that one, tiny detail (aka bug). There is no method on `usersRepository` called inside `registerUser`.

``` java UserRegistrationServiceTest.java (first test)
    public class UserRegistrationServiceTest {
    
        private UsersRepository usersRepository;
        private UserRegistrationService registrationService;
    
        @Before
        public void setUp() throws Exception {
            usersRepository = mock(UsersRepository.class);
            registrationService = new UserRegistrationService(usersRepository);
        }
    
        @Test
        public void shouldStoreRegisteredUser() throws Exception {
            User newUser = new User("John Doe", "user@gmail.com");
            registrationService.registerUser(newUser);
            verify(usersRepository).add(newUser);
        }

    }
```

Can you guess what test result shows? I'd say that this test should result with error yelling at me that it can't spy final methods (as `add` one) or at least the verification should fail as there is no call to `add` method in the code. But it turns out that the bar is green. Also this test seems to invoke real object's method as it prints out 
``` text
    persistent storage call will be here
```

To be true, this test would fail if only there were any prerequisities not configured for this test that would result with exception or so (e.g. database call, resource bundle access etc). But as long as this target method can execute with no exception this test passes.

Ok, werid a bit, but because this test passes, everithing looks good so let's add another one. Now let's test that it rejects registration of user whose email domain was blacklisted. In order to do this we invoke registration of user with invalid email address and we expect both: exception to be thrown and that no method was called on `usersRepository`. This test can be coded as below:

``` java UserRegistrationServiceTest.java (second test)
    @Test(expected = UserRegistrationService.SpamProtectionException.class)
    public void shouldNotRegisterUserIfEmailIsBlacklisted() throws Exception {
        User newUser = new User("John Doe", "user@spam.com");
        try {
            registrationService.registerUser(newUser);
        } finally {
            verifyZeroInteractions(usersRepository);
        }
    }
```

It turns out that when you run both tests together, the first one still succeeds as before, but the last one fails with the following message:

``` text
    org.mockito.exceptions.misusing.UnfinishedVerificationException: 
    Missing method call for verify(mock) here:
    -> at pl.michalostruszka.mockito.UserRegistrationServiceTest.shouldStoreRegisteredUser(UserRegistrationServiceTest.java:31)
    
    Example of correct verification:
        verify(mock).doSomething()
    
    Also, this error might show up because you verify either of: final/private/equals()/hashCode() methods.
    Those methods *cannot* be stubbed/verified.
    
        at pl.michalostruszka.mockito.UserRegistrationServiceTest.setUp(UserRegistrationServiceTest.java:23)
```
Whoaaa! It says that there was unfinished verification and when you click the line with arrow `->` in your IDE it highlights exacly this line with verification from first test. It is really helpful as it gives you valuable hint what may be wrong (and it was not the case in Mockito versions eariler than 1.8 as far as I know). But there is one more issue: you get this error only when you play with this specific mock - in this case this is `verifyZeroInteractions` call. This is because Mockito doesn't seem to verify its mocks state on test case end, but only at their next usage. You know, you should not rely on order in which test methods are executed, so it may happen that even with those two tests we still get green bar for both (if executed in different order). 

Test cases shown above are set up without specific JUnit runner and with `Mockito.mock()` method call to define mock.
I've been playing with that a bit more and found that if you run this test class with `MockitoJUnitRunner` both test would succeed with no warning message that there was something wrong. You don't even have to change this mock initialization from `mock()` method call to `@Mock` annotation. 

To sum up, if you use method that is final and try to verify this method's call you may end up with passing test even if the code you exercise is not correct. It may happen no matter how many times this method was exercised and how many times you expected it to be called. This also happens especially when verification is the last action invoked on this mock object during given test run. Mockito will do its best to let you know that there is unfinished verification giving you clear hint about what's wrong, but only when you run your tests without Mockito-specific runner. If you don't, you should be aware that when you get real implementation being executed on previously mocked object, you may have missed the fact that its method was declared final.

I'd say that coding even such simple feature test-first would probably save some time on investigating what was wrong. It is because your freshly written test would pass with no implementation in place, which is clearly suspucious. Doing it test-after, you just fine-tune test cases to pass according to existing implementation that may not be correct.

There was [Mockito issue](http://code.google.com/p/mockito/issues/detail?id=54&can=1&q=final%20method) raised in 2009 which led to implementation of those "unfinished verification" hints on failure, but the case described above seems to be present still in current mockito version which is 1.9.0. 


If you'd like to play with this code, I put maven project for this post [here on github](https://github.com/mostr/mockito-tricky-verification) with few commits containing respective project phases with each change described.