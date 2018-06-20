---
layout: post
title: "Safe-Dereference / Inverse Null Coalescing Support for C#"
date: 2012-08-23 20:19
published: true
categories: C# groovy expressionstrees
---
Groovy has a very handy safe-dereference operator `?.` which allows for nested dereferencing without having to worry about a null reference exception.

``` groovy
people << new Person(name:'Ian')
highestZipCode = people.collect{ p -> p.Address?.ZipCode }.max()
println highestZipCode
```

While we cannot add custom operators to C#, we can use extension methods and expressions to simulate `p -> p.Address?.ZipCode`. To follow the naming convention in LINQ, I am going to create an extension method on `object` called `ValueOrDefault`. Since operators are not an option, expression trees are the primary option. Assuming we have a an object, our goal is to support `target.ValueOrDefault( item => item.Foo.Bar().Baz )` evaluating the expression layer by layer until we finish or encounter a `default(T)` value. To evaluate the expression, we have to parse from the outside in, but evaluate from the inside out, and wrap every subexpression evaluation in the same safe call to `ValueOrDefault`. The recursion is rather fun:

<!--
Digraph G {
"Start"              -> "ValueOrDefault"
"ValueOrDefault"     -> "TerminalValue"
"ValueOrDefault"     -> "EvaluateExpression"
"EvaluateExpression" -> "ValueOrDefault"
"EvaluateExpression" -> "EvaluateMember"
"EvaluateExpression" -> "TerminalValue"
"EvaluateMember"     -> "EvaluateMember"
"EvaluateMember"     -> "ValueOrDefault"
}-->

<svg width="370pt" height="260pt" viewBox="0.00 0.00 370.00 260.00" xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink">
  <g id="evaluation_graph" class="graph" transform="scale(1 1) rotate(0) translate(4 256)">
    <title>graph</title>
    <!-- background -->
    <!--<polygon fill="white" stroke="white" points="-4,5 -4,-256 367,-256 367,5 -4,5"></polygon>-->
    <!-- Start -->
    <g id="node1" class="node"><title>Start</title>
      <ellipse fill="none" stroke="black" cx="168" cy="-234" rx="28.9133" ry="18"></ellipse>
      <text text-anchor="middle" x="168" y="-229.8" font-family="Times,serif" font-size="14.00">Start</text>
    </g>
    <!-- ValueOrDefault -->
    <g id="node3" class="node"><title>ValueOrDefault</title>
      <ellipse fill="none" stroke="black" cx="168" cy="-162" rx="73.3528" ry="18"></ellipse>
      <text text-anchor="middle" x="168" y="-157.8" font-family="Times,serif" font-size="14.00">ValueOrDefault</text>
    </g>
    <!-- Start&#45;&gt;ValueOrDefault -->
    <g id="edge2" class="edge"><title>Start-&gt;ValueOrDefault</title>
      <path fill="none" stroke="black" d="M168,-215.697C168,-207.983 168,-198.712 168,-190.112"></path>
      <polygon fill="black" stroke="black" points="171.5,-190.104 168,-180.104 164.5,-190.104 171.5,-190.104"></polygon>
    </g>
    <!-- TerminalValue -->
    <g id="node5" class="node"><title>TerminalValue</title>
      <ellipse fill="none" stroke="black" cx="69" cy="-18" rx="69.2336" ry="18"></ellipse>
      <text text-anchor="middle" x="69" y="-13.8" font-family="Times,serif" font-size="14.00">TerminalValue</text>
    </g>
    <!-- ValueOrDefault&#45;&gt;TerminalValue -->
    <g id="edge4" class="edge"><title>ValueOrDefault-&gt;TerminalValue</title>
      <path fill="none" stroke="black" d="M122.829,-147.741C103.391,-139.573 82.5208,-126.935 71,-108 59.8003,-89.5926 60.2059,-64.8389 62.8981,-46.1103"></path>
      <polygon fill="black" stroke="black" points="66.3671,-46.5892 64.6231,-36.1389 59.4696,-45.3959 66.3671,-46.5892"></polygon>
    </g>
    <!-- EvaluateExpression -->
    <g id="node7" class="node"><title>EvaluateExpression</title>
      <ellipse fill="none" stroke="black" cx="168" cy="-90" rx="87.5947" ry="18"></ellipse>
      <text text-anchor="middle" x="168" y="-85.8" font-family="Times,serif" font-size="14.00">EvaluateExpression</text>
    </g>
    <!-- ValueOrDefault&#45;&gt;EvaluateExpression -->
    <g id="edge6" class="edge"><title>ValueOrDefault-&gt;EvaluateExpression</title>
      <path fill="none" stroke="black" d="M162.122,-144.055C161.304,-136.346 161.061,-127.027 161.395,-118.364"></path>
      <polygon fill="black" stroke="black" points="164.894,-118.491 162.087,-108.275 157.911,-118.012 164.894,-118.491"></polygon>
    </g>
    <!-- EvaluateExpression&#45;&gt;ValueOrDefault -->
    <g id="edge8" class="edge"><title>EvaluateExpression-&gt;ValueOrDefault</title>
      <path fill="none" stroke="black" d="M173.913,-108.275C174.715,-116.03 174.94,-125.362 174.591,-134.005"></path>
      <polygon fill="black" stroke="black" points="171.094,-133.832 173.878,-144.055 178.077,-134.327 171.094,-133.832"></polygon>
    </g>
    <!-- EvaluateExpression&#45;&gt;TerminalValue -->
    <g id="edge12" class="edge"><title>EvaluateExpression-&gt;TerminalValue</title>
      <path fill="none" stroke="black" d="M144.789,-72.5881C131.412,-63.1297 114.427,-51.1198 99.9276,-40.868"></path>
      <polygon fill="black" stroke="black" points="101.91,-37.9832 91.7242,-35.0676 97.8687,-43.6988 101.91,-37.9832"></polygon>
    </g>
    <!-- EvaluateMember -->
    <g id="node10" class="node"><title>EvaluateMember</title>
      <ellipse fill="none" stroke="black" cx="267" cy="-18" rx="77.2998" ry="18"></ellipse>
      <text text-anchor="middle" x="267" y="-13.8" font-family="Times,serif" font-size="14.00">EvaluateMember</text>
    </g>
    <!-- EvaluateExpression&#45;&gt;EvaluateMember -->
    <g id="edge10" class="edge"><title>EvaluateExpression-&gt;EvaluateMember</title>
      <path fill="none" stroke="black" d="M191.211,-72.5881C204.493,-63.1972 221.331,-51.2912 235.761,-41.088"></path>
      <polygon fill="black" stroke="black" points="237.79,-43.9402 243.934,-35.3091 233.749,-38.2246 237.79,-43.9402"></polygon>
    </g>
    <!-- EvaluateMember&#45;&gt;ValueOrDefault -->
    <g id="edge16" class="edge"><title>EvaluateMember-&gt;ValueOrDefault</title>
      <path fill="none" stroke="black" d="M271.122,-36.1534C274.694,-55.1946 277.415,-86.1213 264,-108 254.26,-123.885 237.935,-135.371 221.485,-143.499"></path>
      <polygon fill="black" stroke="black" points="220.005,-140.327 212.373,-147.676 222.922,-146.69 220.005,-140.327"></polygon>
    </g>
    <!-- EvaluateMember&#45;&gt;EvaluateMember -->
    <g id="edge14" class="edge"><title>EvaluateMember-&gt;EvaluateMember</title>
      <path fill="none" stroke="black" d="M319.273,-31.3146C342.238,-32.4929 362,-28.0547 362,-18 362,-9.47708 347.801,-4.98973 329.487,-4.53793"></path>
      <polygon fill="black" stroke="black" points="329.221,-1.04131 319.273,-4.68539 329.323,-8.04058 329.221,-1.04131"></polygon>
    </g>
  </g>
</svg>

In order to evaluate `target.ValueOrDefault( item => item.Foo.Bar().Baz )` we need to

1.  Determine if `target` is null, if so, return null
2.  Pass `target` as the instance and `item => item.Foo.Bar().Baz`
3.  Pull off the `.Baz` expression and evaluate `item.Foo.Bar()` with `target` as the instance
4.  Pull off the `.Bar()` and evaluate `item.Foo` with `target` as the instance
5.  Pull off the `.Foo` and evaluate `item` with `target` as the instance
6.  There are no more member or method expressions, so `target` is passed back up the stack
7.  `.Foo` is evaluated on the `target` instance. If `Foo` evaluates to `default(T)` we cascade back up the recursion tree and stop evaluating. If not, we proceed to #8 passing the value returned by `.Foo` as the `target` instance.
8.  `.Bar()` is executed on the `target` instance. Same behavior as #7, pass the return `value` back up the stack
9.  `.Baz` is evaluated on the `target` instance returned from `.Bar()`

Well, that's enough theory, let's look at the implementation.

``` csharp
using System;
using System.Linq.Expressions;
using System.Reflection;

public static class ObjectExtensions
{
  public static TValue ValueOrDefault<TSource, TValue>
                         ( this TSource instance,
                           Expression<Func<TSource, TValue>> expression )
  {
    return ValueOrDefault( instance, expression, true );
  }

  private static TValue ValueOrDefault<TSource, TValue>
                          ( this TSource instance,
                            Expression<Func<TSource, TValue>> expression,
                            bool nested )
  {
      return ReferenceEquals( instance, default( TSource ) )
                 ? default( TValue )
                 : nested ? EvaluateExpression( instance, expression )
                          : expression.Compile()( instance );
  }

  internal static TProperty EvaluateExpression<TSource, TProperty>
                              ( TSource source,
                                Expression<Func<TSource, TProperty>> expression )
  {
    var method = expression.Body as MethodCallExpression;
    if ( method != null ) {
      return ValueOrDefault( source, expression, false );
    }

    var body = expression.Body as MemberExpression;
    if ( body == null ) {
      const string format = "Expression '{0}' must refer to a property.";
      string message = string.Format( format, expression );
      throw new ArgumentException( message );
    }

    object value = EvaluateMemberExpression( source, body );
    if ( ReferenceEquals( value, null ) ) {
      return default( TProperty );
    }
    return (TProperty) value;
  }

  private static object EvaluateMemberExpression
                          ( object instance,
                            MemberExpression memberExpression )
  {
    if ( memberExpression == null ) {
      return instance;
    }
    instance = EvaluateMemberExpression( instance, memberExpression.Expression as MemberExpression );
    var propertyInfo = memberExpression.Member as PropertyInfo;
    instance = ValueOrDefault( instance, item => propertyInfo.GetValue( item, null ), false );
    return instance;
  }
}
```

The key thing to grok is the inside out evaluation. If you want to play around with this code, I have included a number of tests below to make sure that everything is in working order. Having `?.` added to the C# language is high on my list of desired vNext features. Hopefully the C# team agrees.

## Tests and Example Usage
``` csharp
using Xunit;

public class Person
{
  public string Name { get; set; }
  public Address Address { get; set; }
}

public class Address
{
  public string StreetName { get; set; }
  public ZipCode ZipCode { get; set; }
}

public class ZipCode
{
  public int Body { get; set; }
  public int? Suffix { get; set; }
  public Zone Zone { get; set; }
  public Zone GetZone() { return Zone; }
}

public class Zone
{
  public string Name { get; set; }
}

public class ObjectExtensionTests
{
  [Fact]
  public void when_the_accessed_object_is_null_then_null_is_returned()
  {
    Person person = null;
    string result = person.ValueOrDefault( p => p.Name );
    Assert.Equal( null, result );
  }

  [Fact]
  public void when_the_accessed_property_is_null_then_null_is_returned()
  {
    var person = new Person { Name = null };
    string value = person.ValueOrDefault( p => p.Name );
    Assert.Equal( null, value );
  }

  [Fact]
  public void when_the_accessed_property_is_not_null_then_its_value_is_returned()
  {
    const string name = "Ian";
    var person = new Person { Name = name };
    string value = person.ValueOrDefault( p => p.Name );
    Assert.Equal( name, value );
  }

  [Fact]
  public void when_a_complex_property_is_null_then_null_is_returned()
  {
    var person = new Person { Address = null };
    Address value = person.ValueOrDefault( p => p.Address );
    Assert.Equal( null, value );
  }

  [Fact]
  public void EvaluateExpression_non_nested_properties_are_evaluated()
  {
     const string name = "Ian";
     var person = new Person { Name = name };
     string value = ObjectExtensions.EvaluateExpression( person, p => p.Name );
     Assert.Equal( name, value );
  }

  [Fact]
  public void EvaluateExpression_nested_properties_are_evaluated()
  {
    const string name = "Ian";
    var person = new Person { Address = new Address { StreetName = name } };
    string value = ObjectExtensions.EvaluateExpression( person, p => p.Address.StreetName );
    Assert.Equal( name, value );
  }

  [Fact]
  public void when_a_nested_accessed_property_is_null_then_null_is_returned()
  {
    var person = new Person { Address = null };
    string value = person.ValueOrDefault( p => p.Address.StreetName );
    Assert.Equal( null, value );
  }

  [Fact]
  public void when_a_nested_accessed_property_is_not_null_then_its_value_is_returned()
  {
    const string name = "Ian";
    var person = new Person { Address = new Address { StreetName = name } };
    string value = person.ValueOrDefault( p => p.Address.StreetName );
    Assert.Equal( name, value );
  }

  [Fact]
  public void when_a_double_nested_accessed_propertys_parent_is_null_then_defaultT_is_returned()
  {
    var person = new Person { Address = null };
    int value = person.ValueOrDefault( p => p.Address.ZipCode.Body );
    Assert.Equal( 0, value );
  }

  [Fact]
  public void when_a_double_nested_accessed_propertys_parent_is_null_then_defaultT_is_returned_for_nullable_types()
  {
    var person = new Person { Address = null };
    int? value = person.ValueOrDefault( p => p.Address.ZipCode.Suffix );
    Assert.Equal( null, value );
  }

  [Fact]
  public void when_a_double_nested_accessed_property_is_not_null_then_its_value_is_returned()
  {
    const int body = 99016;
    var person = new Person { Address = new Address { ZipCode = new ZipCode { Body = body } } };
    int value = person.ValueOrDefault( p => p.Address.ZipCode.Body );
    Assert.Equal( body, value );
  }

  [Fact]
  public void when_a_triple_nested_accessed_propertys_parent_is_null_then_null_is_returned()
  {
    var person = new Person { Address = null };
    string value = person.ValueOrDefault( p => p.Address.ZipCode.Zone.Name );
    Assert.Equal( (object) null, value );
  }

  [Fact]
  public void when_a_triple_nested_accessed_property_is_not_null_then_its_value_is_returned()
  {
    const string name = "Yay";
    var person = new Person
                   { Address = new Address { ZipCode = new ZipCode { Zone = new Zone { Name = name } } } };
    string value = person.ValueOrDefault( p => p.Address.ZipCode.Zone.Name );
    Assert.Equal( name, value );
  }
}
```
