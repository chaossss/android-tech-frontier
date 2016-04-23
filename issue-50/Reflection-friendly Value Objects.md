AutoValue简介
---

> * 原文链接 : [Reflection-friendly Value Objects](https://publicobject.com/2016/04/10/reflection-friendly-value-objects/)
* 原文作者 : [Jesse Wilson](https://github.com/swankjesse)
* 译文出自 : [开发技术前线 www.devtf.cn](http://www.devtf.cn)
* 转载声明: 本译文已授权[开发者头条](http://toutiao.io/download)享有独家转载权，未经允许，不得转载!
* 译者 : [chaossss](https://github.com/chaossss) 
* 校对者: 
* 状态 :  完成 



There’s plenty of good advice on how to do value objects well in Java. Ryan Harter’s AutoValue intro is particularly handy.

When I’m not using AutoValue, I like to do my builders as recommended in Item 2 of Effective Java Second Edition. In it Josh Bloch recommends a private constructor that accepts the Builder as a constructor parameter:

```java
public final class Pizza {  
  public final Size size;
  public final List<Topping> toppings;
  public final Cheese cheese;

  private Pizza(Builder builder) {
    this.size = builder.size;
    this.toppings = immutableList(builder.toppings);
    this.cheese = builder.cheese;
  }

  public static final class Builder {
    private Size size = Size.MEDIUM;
    private List<Topping> toppings = new ArrayList<>();
    private Cheese cheese = Cheese.MOZZA;

    // …setters…

    public Pizza build() {
      return new Pizza(this);
    }
  }
}
```

This example also assigns default values in the builder. But these defaults are lost when Moshi or Gson use reflection to decode a pizza from JSON. This problem is explained in Moshi’s readme:

If the class doesn’t have a no-arguments constructor, Moshi can’t assign the field’s default value, even if it’s specified in the field declaration. Instead, the field’s default is always 0 for numbers, false for booleans, and null for references.

This is a problem for our pizza example. Suppose the JSON doesn’t specify the pizza’s size:

```java
{
  "toppings": ["ham", "pineapple"],
  "cheese": "MOZZA"
}
```

When we decode this with Moshi, it’s size will be null, not MEDIUM as we’d prefer.

A no-args constructor

Reflective libraries will use the no-arguments constructor if you have one. This is true for all the libraries that I’ve tried including Moshi, Gson, SnakeYAML, Hibernate, and JAXB. To make our immutable Pizza class work with these frameworks we just need to add a no-arguments constructor:

```java
  public Pizza() {
    this.size = Size.MEDIUM;
    this.toppings = immutableList(new ArrayList<>());
    this.cheese = Cheese.MOZZA;
  }

  private Pizza(Builder builder) {
    ...
  }
```

But this is ugly because now we’ve duplicated our defaults. If we ever change the default size to LARGE we’ve got to remember to make that fix in both places.

The builder has the defaults!

The fix is fun. Just get the defaults from a new builder in the no-args constructor:

```java
  public Pizza() {
    this(new Builder());
  }

  private Pizza(Builder builder) {
    ...
  }
```

I discovered this trick a few months ago and I’ve really enjoyed it. And it’s not just for reflection. We’re even using it when you call new OkHttpClient().