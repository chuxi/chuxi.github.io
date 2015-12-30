---
layout: post
title: markdown style test cases
date: 2015-07-03
published: false
categories: markdown
---

this post is used for personal test of markdown reference to the [standard markdown!](https://help.github.com/articles/markdown-basics/) format and [github flavored markdown!](https://help.github.com/articles/github-flavored-markdown/)



code style and highlight format display

make sure there is a line before and after the content code

```java
import java.util.Thread
import java.collections.Arrays

class HelloWorld {
  System.out.println("hello world!");
}
```



```

sealed trait Message
case class HelloMessage(msg: String) extends Message

class Actor{

}

```

{% highlight ruby %}
sealed trait Message
case class HelloMessage(msg: String) extends Message

class Actor{

}
{% endhighlight %}

---
```scala
    sealed trait Message
    case class HelloMessage(msg: String) extends Message
    
    class Actor{
    
    }
```
---


### No BlankSpace

### Normal Header