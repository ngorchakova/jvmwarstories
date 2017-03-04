---
layout: post
title: ETA expansion causes additional serialization
date: 2016-12-18 00:00:00 +00:00
tags: [compiler, eta-expansion, scala, serialization]
---
From first glance Client1 and Client2 are differ only by 'code refactoring'. However, if the same invocations are done in map (or any other) function on RDD, this small refactoring cause additional serialization of Service object.

{% highlight java %}
trait Service {  
  def process(s: String)  
 }  
   
 object ServiceImpl extends Service{  
  override def process(s: String): Unit = {  
   println(s)  
  }  
 }  
   
 object Register {  
  var serviceInst : Service = ServiceImpl  
 }  
   
 object Client1 {    
  def process(l: List[String]): Unit ={  
   l.foreach(x => Register.serviceInst.process(x))  
  }     
 }  
   
 object Client2 {    
  def process(l: List[String]): Unit ={  
   l.foreach(Register.serviceInst.process)  
  }     
 }  
{% endhighlight %}

Result of decompiling each client:
{% highlight java %}
 public final class Client1$$anonfun$process$1$$anonfun$apply$1 extends AbstractFunction1<String, BoxedUnit> implements Serializable {  
   public static final long serialVersionUID = 0L;  
   public final void apply(final String x$1) {  
     Register$.MODULE$.serviceInst().process(x$1);  
   }  
 }  
   
 public static final class Client2$$anonfun$process$1 extends AbstractFunction1<String, BoxedUnit> implements Serializable {  
   public static final long serialVersionUID = 0L;  
   private final Service eta$0$1$1;  
   
   public final void apply(final String s) {  
     this.eta$0$1$1.process(s);  
   }  
 }  
{% endhighlight %}

So, first client doesn't require Service object to be serialiazed, while the second does. It is happening as Scala compiler performs eta-expantion on method given in Client2. Compiler generates new Function that calls process directly on concreate Service instance.

There is great page about ETA expanstion [**read**](http://blog.jaceklaskowski.pl/2013/11/23/how-much-one-ought-to-know-eta-expansion.html)