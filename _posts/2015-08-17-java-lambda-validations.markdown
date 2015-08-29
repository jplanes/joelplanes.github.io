---
layout: post
title:  "Lambda validations with java 8"
date:   2015-08-17 18:08:47
categories: java functional
comments: true
---

I remember myself writing ugly code when trying to perform some server side validations over my entities. There are some frameworks in the market to solve this problem, but, does it really worth to use a hole framework / library to do such a simple thing like validate an entity? In the other hand, does it make sense to write a lot of painful validation code?

I think we can take advantage of the lambda expressions in java 8 and write our validations in a simple and elegant way without the need of have a lot of dependencies to third party components and 100% the ability to modify our code.

In this article I want to share with you a very simple but useful set of classes you can copy&paste in your proyect if you want to have simple validation features (based on 'keep it simple' concept).

In all this article we will be working with a `Person` entity which looks like:

**<u>NOTE</u>:** *you can see the source code of this post in [my github repository](https://github.com/jplanes/java-lambda-validations)*

{% highlight java linenos %}
public class Person {
  private String firstName;
  private String lastName;
  private String email;
  private int age;
  
  public Person(String firstName, String lastName, String email, int age) {
    this.firstName = firstName;
    this.lastName = lastName;
    this.email = email;
    this.age = age;
  }

  public String getFirstName() {
    return firstName;
  }

  public String getLastName() {
    return lastName;
  }

  public String getEmail() {
    return email;
  }

  public int getAge() {
    return age;
  }

}
{% endhighlight %}

We will also be using a `Validator` interface so we can switch both validators we will implement depending on the context:

{% highlight java linenos %}
public interface PersonValidator {
  void validate(Person person);
}
{% endhighlight %}

### Old fashioned validations
First of all, try to remember how you usually validate any kind of entity in java 7 or below. The approach may vary but your code might looks pretty much like this one:

{% highlight java linenos %}
public class OldFashionedPersonValidator implements PersonValidator {

  public void validate(Person person) {
    if(person.getFirstName() == null) throw new IllegalArgumentException("firstname : must not be null");
    if(person.getFirstName().length() < 2) throw new IllegalArgumentException("firstname : must have at least 2 characters");
    if(person.getFirstName().length() > 30) throw new IllegalArgumentException("firstname : must have less than 30 characters");
    
    if(person.getLastName() == null) throw new IllegalArgumentException("lastname : must not be null");
    if(person.getLastName().length() < 4) throw new IllegalArgumentException("lastname : must have at least 4 characters");
    if(person.getLastName().length() > 30) throw new IllegalArgumentException("lastname : must have less than 30 characters");

    if(person.getEmail() == null) throw new IllegalArgumentException("email : must not be null");
    if(person.getEmail().length() < 3) throw new IllegalArgumentException("email : must have at least 3 characters");
    if(person.getEmail().length() > 50) throw new IllegalArgumentException("email : must have less than 50 characters");
    if(!person.getEmail().contains("@")) throw new IllegalArgumentException("email : must contains @");
    
    
    if(person.getAge() < 0) throw new IllegalArgumentException("age : must be greater than 0");
    if(person.getAge() > 110) throw new IllegalArgumentException("age : must be lower than 0");
  }

}
{% endhighlight %}

So, in a first look you can see the big amount of boilerplate code we had to write. Of course we can work around to clean this code a little but the result wont vary too much, after that, validations are always a tedious thing to take care of!!

### Validations with lambda
Our main objective is to have a cleaner and understandable code. So, the first stept will be to create a `Validation` interface. This interface will be a `@FunctionalInterface` and it will have a `test` method which receives a parameter to be validated and returns the validation result. A `ValidationResult` is a class with a boolean and a message which represents how the validation ended.
The `Validation` interface also has an `and` method and an `or` method which are already implemented by default based on the `test` method.

{% highlight java linenos %}
@FunctionalInterface
public interface Validation<K> {

  ValidationResult test(K param);

  default Validation<K> and(Validation<K> other) {
    return (param) -> {
      ValidationResult firstResult = this.test(param);
      return !firstResult.isvalid() ? firstResult : other.test(param);
    };
  }

  default Validation<K> or(Validation<K> other) {
    return (param) -> {
      ValidationResult firstResult = this.test(param);
      return firstResult.isvalid() ? firstResult : other.test(param);
    };
  }

}
{% endhighlight %}

I will also create a `SimpleValidation` class that implements `Validation`, it evaluates a java `Predicate` and returns an ERROR result with the `onErrorMessage` you sent in case the `Predicate` fails.

{% highlight java linenos %}
public class SimpleValidation<K> implements Validation<K> {

  private Predicate<K> predicate;
  private String onErrorMessage;
  
  public static <K> SimpleValidation<K> from(Predicate<K> predicate, String onErrorMessage) {
     return new SimpleValidation<K>(predicate, onErrorMessage);
  }
  
  private SimpleValidation(Predicate<K> predicate, String onErrorMessage) {
    this.predicate = predicate;
    this.onErrorMessage = onErrorMessage;
  }
  
  @Override
  public ValidationResult test(K param) {
    return predicate.test(param) ? ValidationResult.ok() : ValidationResult.fail(onErrorMessage);
  }

}
{% endhighlight %}

The intent of this class is to be used as a kind of *helper* to create your custom validations, for example, you can now create a new validation simply writing a code like this one:

{% highlight java linenos %}
public class StringValidationHelpers {
  public static Validation<String> notNull = SimpleValidation.from((s) -> s != null, "must not be null.");
  
  public static Validation<String> moreThan(int size){
    return SimpleValidation.from((s) -> s.length() >= size, format("must have more than %s chars.", size));
  }
  
  public static Validation<String> lessThan(int size){
    return SimpleValidation.from((s) -> s.length() <= size, format("must have less than %s chars.", size));
  }
  
  public static Validation<String> between(int minSize, int maxSize){
    return moreThan(minSize).and(lessThan(maxSize));
  }
  
  public static Validation<String> contains(String c){
    return SimpleValidation.from((s) -> s.contains(c), format("must contain %s", c));
  }
}


StringValidationHelpers.notNull.test("not null string").isvalid()
{% endhighlight %}

So, at this point we are able to rewrite the entire `OldFashionedPersonValidator` but now using our validation classes and helpers. I will rename it as `LambdaPersonValidator` so you can differenciate from each other. I would also like to encourage you to compare both versions of the `PersonValidator` and then tell me which one do you prefer.

{% highlight java linenos %}
public class LamdaPersonValidator implements PersonValidator {

  public void validate(Person person) {
    notNull.and(between(2, 12)).test(person.getFirstName()).throwIfInvalid("firstname");
    notNull.and(between(4, 30)).test(person.getLastName()).throwIfInvalid("secondname");
    notNull.and(between(3, 50)).and(contains("@")).test(person.getEmail()).throwIfInvalid("email");
    intBetween(0, 110).test(person.getAge()).throwIfInvalid("age");
  }
  
}
{% endhighlight %}

As you can see, it took only 4 lines to validate our 4 fields. But that is not the biggest achievment, the great thing here is that our code is pretty expressive and we can read it as if it was a letter. With a few lines of code we ended up using a pretty `Sintactic sugar` validations.

### Alright … but how can I be so sure that all this works as expected?
I've prepared some special for those sceptical folks. I've written some tests in order to be sure both `PersonValidator` behave the same way.
At first, we have an `AbstractPersonValidationTest` which works as a template for all the scenarios that have to be tested:

{% highlight java linenos %}
public abstract class AbstractPersonValidationsTest {
  
  // children must provide a specific validator to be tested
  protected abstract PersonValidator getValidatorInstance();

  @Test
  public void person_isComplete_validationSucceed() {
    getValidatorInstance().validate(
      new Person("bill", "clinton", "bill@gmail.com", 60)
    );
  }
  
  @Test
  public void person_withoutFirstName_validationFail() {
    try {
      getValidatorInstance().validate(
        new Person(null, "clinton", "bill@gmail.com", 60)
      );
      fail();
    } catch (IllegalArgumentException e) {
      assertTrue(e.getMessage().contains("firstname"));
    }
  }
  
  @Test
  public void person_shortFirstName_validationFail() {
    try {
      getValidatorInstance().validate(
        new Person("b", "clinton", "bill@gmail.com", 60)
      );
      fail();
    } catch (IllegalArgumentException e) {
      assertTrue(e.getMessage().contains("firstname"));
    }
  }
  
  @Test
  public void person_wrongEmail_validationFail() {
    try {
      getValidatorInstance().validate(
        new Person("bill", "clinton", "bill_gmail.com", 60)
      );
      fail();
    } catch (IllegalArgumentException e) {
      assertTrue(e.getMessage().contains("email"));
    }
  }
  
  @Test
  public void person_didntBorn_validationFail() {
    try {
      getValidatorInstance().validate(
        new Person("bill", "clinton", "bill@gmail.com", -10)
      );
      fail();
    } catch (IllegalArgumentException e) {
      assertTrue(e.getMessage().contains("age"));
    }
  }
  
  @Test
  public void person_isDeath_validationFail() {
    try {
      getValidatorInstance().validate(
        new Person("bill", "clinton", "bill@gmail.com", 100000)
      );
      fail();
    } catch (IllegalArgumentException e) {
      assertTrue(e.getMessage().contains("age"));
    }
  }
  
}
{% endhighlight %}

After all this tests we can provide one test for each `PersonValidator` we have, we only have to extend AbstractPersonValidationsTest`:

{% highlight java linenos %}
public class OldFashionedPersonValidatorTest extends AbstractPersonValidationsTest {
  protected PersonValidator getValidatorInstance() {
    return new OldFashionedPersonValidator();
  }
}
{% endhighlight %}

{% highlight java linenos %}

public class LamdaPersonValidatorTest extends AbstractPersonValidationsTest {
  protected PersonValidator getValidatorInstance() {
    return new LamdaPersonValidator();
  }
}
{% endhighlight %}

As a final step, we can run both tests using Junit and we can verify the correctness of the code we wrote:

![junit results](/images/posts/lambda-validations-test-result.png)

### Thanks!
Thank you for taking the time to read this article, I hope it has been helpful for you and you have enjoyed reading as much as I enjoyed writing it!