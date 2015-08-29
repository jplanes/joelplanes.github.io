---
layout: post
title:  "Lambda expressions in java 8"
date:   2015-08-16 18:08:47
categories: java functional
comments: true
---
One of the greatest improvements of Java 8 are the lambda expressions, they let us write cleaner, more expressive and less boilerplate code. But, what are the key concepts behind the scene? is the jvm capable of execute our raw lambda expression as it is or the compiler has to transform it before? what does the compiler do in order to achieve that?
These are some of the questions I want to solve with this simple article.

### Functional interface
Before starting with the concept of a lambda expression we have to talk a little about what a `functional interface` is. In Java 8, a functional interface is an interface which has *an only unimplemented method*, I know it sounds **creepy** ... but yes, now you can implement some methods directly on the interface (with some constraints). For example, you can see the [Predicate](http://grepcode.com/file/repository.grepcode.com/java/root/jdk/openjdk/8-b132/java/util/function/Predicate.java) interface and notice that has some **default** methods which have already been implemented.

*So, as far as you provide an interface with **an only unimplemented method** you are good to go, and your interface would be considered as a functional interface as well.*

The best example is to take a look at the `Comparable` interface:

{% highlight java linenos %}
@FunctionalInterface
public interface Comparable<T> {
  int compareTo(T other);
}
{% endhighlight %}

### Old fashioned comparators
So, suppose we want to sort a list of `Users` by name using java 7 capabilities, and suppose a `User` looks like:

{% highlight java linenos %}
public class User {
  private String fullName;
  private Date birthDate;
  private Date lastLogIn;

  public User(String fullName, Date birthDate, Date lastLogIn) {
    this.fullName = fullName;
    this.birthDate = birthDate;
    this.lastLogIn = lastLogIn;
  }

  public String getFullName() {
    return fullName;
  }

  public Date getBirthDate() {
    return birthDate;
  }

  public Date getLastLogIn() {
    return lastLogIn;
  }
}
{% endhighlight %}

What we need to do is to create an anonymous class that implements `Comparator` and provide the specific implementaion of `compareTo(User user)` method:

{% highlight java linenos %}
List<User> users = new ArrayList<User>();
users.add(new User("john doe", new Date(), new Date()));
users.add(new User("eric clapton", new Date(), new Date()));

// creates an anonymous class that implements Comparator
Comparator<User> byName = new Comparator<User>() {
  @Override
  public int compare(User oneUser, User anotherUser) {
    return oneUser.getFullName().compareTo(anotherUser.getFullName());
  }
};

// apply the sorting rule
Collections.sort(users, byName);

for (User user : users) {
  System.out.println(user.getFullName());
}
{% endhighlight %}

As you can see, from line 6 to 11 we wrote a lot of java code with very poor expressiveness. Indeed, the only line that gives any sense to this code is the 9th.
So, why we do have to write a lot of boilerplate code? why don't the compiler just take care of that? Well, that is what java boys have been thinking about for the last few years and they [have come out with the solution](http://www.globalnerdy.com/wordpress/wp-content/uploads/2013/01/java-problem-factory.jpg) I'll explain.

### Comparators with steroids
Well, now lets try to rewrite this code but now using a lambda expression:

{% highlight java linenos %}
List<User> users = new ArrayList<User>();
users.add(new User("john doe", new Date(), new Date()));
users.add(new User("eric clapton", new Date(), new Date()));

Comparator<User> byName = (oneUser, anotherUser) -> oneUser.getFullName().compareTo(anotherUser.getFullName());

users.stream()
  .sorted(byName)
  .map(user -> user.getFullName())
  .forEach(System.out::println);
{% endhighlight %}

As you can see, now the code is not only more compact but also more expressive and easy to understand. We can focus in the pure business logic of what we wanted to do instead of being worry about how java works on background.
Every time you see a `FunctionalInterface` you can replace the anonymous class with a lambda expression.

Furthermore, this code can be written easier by using the new `Comparator helpers` and passing `User::getFullName` as parameter:

{% highlight java linenos %}
users.stream()
  .sorted(Comparator.comparing(User::getFullName))
  .map(user -> user.getFullName())
  .forEach(System.out::println);
{% endhighlight %}

If you didn't know, `User::getFullname` is a kind of 'pointer' to the `getFullName()` method. So, whith `sorted(Comparator.comparing(User::getFullName))` we are doing all the same it took almost 7 lines of code before but now the code seems to be cleaner and more expressive.


### Conclusion
So, we saw what a `FunctionalInterface` is, we also saw that every `FunctionalInterface` can be replaced as a lambda expression, and we saw the benefits of using lambda expressions into our code.
As a matter of fact, although the code seems to change a lot, what really happen behind the scene is that the compiler interpret our clean code and transform it into something ugly and very similar to the first example. So, we can conclude that a `FunctionalInterface` is something related to the compiler rather than the execution environment.

Thank you for taking the time to read this article, in [next posts](/java/functional/2015/08/17/java-lambda-validations.html) I will try to explain how you can implement a basic validation library using lambda expressions.