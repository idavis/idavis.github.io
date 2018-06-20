---
layout: post
title: "Prototypal Inheritance in .NET: Delegation at Last"
date: 2012-10-08 17:19
published: true
categories: dynamic c# dlr
---
In .NET 4.0, we have access to the [Dynamic Language Runtime (DLR)][] giving us [dynamic dispatch][] via the [IDynamicMetaObjectProvider][] interface coupled with the [dynamic][] keyword. When a variable is declared as `dynamic` and it implements the ```IDynamicMetaObjectProvider``` interface, we are given the opportunity to control the delegation of calls on that object by returning a ```DynamicMetaObject``` containing an expression which will be evaluated by the runtime. We only get this opportunity if the direct target was unable to directly handle the expression.

The ```DynamicObject``` and ```ExpandoObject``` classes both implement ```IDynamicMetaObjectProvider```, but their implementations are explicitly implemented and the nested classes used to return the ```DynamicMetaObject``` are private and sealed. I understand that the classes may not have been tested enough to make them inheritable, but not having access to these classes really hurts our ability to easily modify the behavior of the underlying ```DynamicMetaObject```. The key word here is easily; implementing the needed expression building is a great deal of work and the internal workings of the Microsoft implementations leverage many internal framework calls.

A key thing to consider is that we don't want to replicate classical inheritance; instead, we are going to focus on prototypal inheritance that mostly replicates JavaScript's prototypal inheritance. Rather than trying to replicate all of the hard work that the DRL team put into writing their implementation, we can add our own implementation on top of theirs. It is simple to hook in, but we need to save off a method to access the base ```DynamicMetaObject``` implementation. This will allow us to attempt to interpret the expression on the object itself or pass it along.

``` csharp
public class DelegatingPrototype : DynamicObject {
    public DelegatingPrototype( object prototype = null ) {
        Prototype = prototype;
    }

    public virtual object Prototype { get; set; }

    public override DynamicMetaObject GetMetaObject( Expression parameter ) {
        if ( Prototype == null ) {
            return GetBaseMetaObject( parameter );
        }
        return new PrototypalMetaObject( parameter, this, Prototype );
    }

    public virtual DynamicMetaObject GetBaseMetaObject( Expression parameter ) {
        return base.GetMetaObject( parameter );
    }
}
```

This small amount of code just sets the hook. Now we need set up the delegation expression. 

To set up the prototypal hierarchy, we are going to need to do a lot of recursion. Unfortunately, it is well hidden (I'll explain shortly). When a call is made on the root object, we are given the expression being interpreted. Using the ```IDynamicMetaObjectProvider``` overload, we will hand off the expression to the ```PrototypalMetaObject``` to construct delegation expression (or the ```DynamicObject``` implementation if our prototype is ```null``` thus trying to interpret the expression on the current object). We want to make a preorder traversal of the prototype hierarchy; at each step, the current object's evaluation will take precedence over its prototype tree. Consider the following prototypal class hierarchy:

<svg width="275pt" height="314pt" viewBox="0.00 0.00 274.66 314.00" xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink">
    <g id="hierarchy_graph" class="graph" transform="scale(1 1) rotate(0) translate(4 310)">
        <title>hierarchy</title>
        <!-- Square -->
        <g id="node2" class="node"><title>Square</title>
            <ellipse fill="none" stroke="black" cx="62" cy="-288" rx="37.2772" ry="18"></ellipse>
            <text text-anchor="middle" x="62" y="-283.8" font-family="Times,serif" font-size="14.00">Square</text>
        </g>
        <!-- s1 -->
        <g id="node4" class="node"><title>s1</title>
            <ellipse fill="none" stroke="black" cx="27" cy="-198" rx="27" ry="18"></ellipse>
            <text text-anchor="middle" x="27" y="-193.8" font-family="Times,serif" font-size="14.00">self</text>
        </g>
        <!-- Square&#45;&gt;s1 -->
        <g id="edge3" class="edge"><title>Square-&gt;s1</title>
            <path fill="none" stroke="black" d="M55.2516,-270.033C50.1837,-257.291 43.1517,-239.61 37.3713,-225.076"></path>
            <polygon fill="black" stroke="black" points="40.5569,-223.615 33.6089,-215.617 34.0525,-226.202 40.5569,-223.615"></polygon>
        </g>
        <!-- Quadrilateral -->
        <g id="node6" class="node"><title>Quadrilateral</title>
            <ellipse fill="none" stroke="black" cx="134" cy="-198" rx="61.4641" ry="18"></ellipse>
            <text text-anchor="middle" x="134" y="-193.8" font-family="Times,serif" font-size="14.00">Quadrilateral</text>
        </g>
        <!-- Square&#45;&gt;Quadrilateral -->
        <g id="edge5" class="edge"><title>Square-&gt;Quadrilateral</title>
            <path fill="none" stroke="black" d="M75.2059,-270.859C86.0302,-257.63 101.525,-238.692 113.89,-223.578"></path>
            <polygon fill="black" stroke="black" points="116.765,-225.592 120.389,-215.636 111.347,-221.159 116.765,-225.592"></polygon>
            <text text-anchor="middle" x="130.828" y="-238.8" font-family="Times,serif" font-size="14.00">prototype</text>
        </g>
        <!-- s2 -->
        <g id="node9" class="node"><title>s2</title>
            <ellipse fill="none" stroke="black" cx="104" cy="-108" rx="27" ry="18"></ellipse>
            <text text-anchor="middle" x="104" y="-103.8" font-family="Times,serif" font-size="14.00">self</text>
        </g>
        <!-- Quadrilateral&#45;&gt;s2 -->
        <g id="edge8" class="edge"><title>Quadrilateral-&gt;s2</title>
            <path fill="none" stroke="black" d="M128.216,-180.033C123.914,-167.413 117.96,-149.95 113.033,-135.496"></path>
            <polygon fill="black" stroke="black" points="116.204,-133.952 109.665,-125.617 109.579,-136.211 116.204,-133.952"></polygon>
        </g>
        <!-- Polygon -->
        <g id="node11" class="node"><title>Polygon</title>
            <ellipse fill="none" stroke="black" cx="192" cy="-108" rx="43.3469" ry="18"></ellipse>
            <text text-anchor="middle" x="192" y="-103.8" font-family="Times,serif" font-size="14.00">Polygon</text>
        </g>
        <!-- Quadrilateral&#45;&gt;Polygon -->
        <g id="edge10" class="edge"><title>Quadrilateral-&gt;Polygon</title>
            <path fill="none" stroke="black" d="M145.183,-180.033C153.743,-167.045 165.684,-148.928 175.364,-134.241"></path>
            <polygon fill="black" stroke="black" points="178.467,-135.892 181.048,-125.617 172.623,-132.04 178.467,-135.892"></polygon>
            <text text-anchor="middle" x="194.828" y="-148.8" font-family="Times,serif" font-size="14.00">prototype</text>
        </g>
        <!-- s3 -->
        <g id="node14" class="node"><title>s3</title>
            <ellipse fill="none" stroke="black" cx="156" cy="-18" rx="27" ry="18"></ellipse>
            <text text-anchor="middle" x="156" y="-13.8" font-family="Times,serif" font-size="14.00">self</text>
        </g>
        <!-- Polygon&#45;&gt;s3 -->
        <g id="edge13" class="edge"><title>Polygon-&gt;s3</title>
            <path fill="none" stroke="black" d="M185.059,-90.0327C179.846,-77.2905 172.613,-59.61 166.668,-45.0764"></path>
            <polygon fill="black" stroke="black" points="169.824,-43.547 162.798,-35.6168 163.345,-46.1975 169.824,-43.547"></polygon>
        </g>
        <!-- null -->
        <g id="node16" class="node"><title>null</title>
            <ellipse fill="none" stroke="black" cx="228" cy="-18" rx="27" ry="18"></ellipse>
            <text text-anchor="middle" x="228" y="-13.8" font-family="Times,serif" font-size="14.00">null</text>
        </g>
        <!-- Polygon&#45;&gt;null -->
        <g id="edge15" class="edge"><title>Polygon-&gt;null</title>
            <path fill="none" stroke="black" d="M198.941,-90.0327C204.154,-77.2905 211.387,-59.61 217.332,-45.0764"></path>
            <polygon fill="black" stroke="black" points="220.655,-46.1975 221.202,-35.6168 214.176,-43.547 220.655,-46.1975"></polygon>
            <text text-anchor="middle" x="239.828" y="-58.8" font-family="Times,serif" font-size="14.00">prototype</text>
        </g>
    </g>
</svg>

Since we never know which level of the hierarchy will be handling the expression, we need to build an expression for the entire tree every time. We want to get the ```DynamicMetaObject``` representing the current object's tree first. Once done, we get the  ```DynamicMetaObject``` for evaluating the expression on the current instance. With these two, we can create a new  ```DynamicMetaObject``` which try to bind the expression to the current instance first, and then fallback to the prototype. At the root level, the prototype ```DynamicMetaObject``` contains the same fallback for the next two layers.

There is another caveat that we need to address. When we try to invoke and expression on an object, the expression is bound to that type. When accessing the prototype, if we don't do anything, the system will throw a binding exception because the matching object won't match the ```DynamicMetaObject```'s type restrictions. To fix this, we need to relax the type restrictions for each prototype.

Remember the recursion I mentions earlier? In the code sample below, I have pulled out all binding code except for the ```BindInvokeMember``` method. The ```_metaObject.Bind[...]``` will actually call into ```DelegatingPrototype::GetMetaObject``` which will try to call back into ```_metaObject.Bind[...]```, which will...well you get the idea. At each call, the prototype becomes the target and we get a new prototype.

``` csharp
public class PrototypalMetaObject : DynamicMetaObject {
    private readonly DynamicMetaObject _baseMetaObject;
    private readonly DynamicMetaObject _metaObject;
    private readonly DelegatingPrototype _prototypalObject;
    private readonly object _prototype;

    public PrototypalMetaObject( Expression expression, DelegatingPrototype value, object prototype )
            : base( expression, BindingRestrictions.Empty, value ) {
        _prototypalObject = value;
        _prototype = prototype;
        _metaObject = CreatePrototypeMetaObject();
        _baseMetaObject = CreateBaseMetaObject();
    }

    protected virtual DynamicMetaObject CreateBaseMetaObject() {
        return _prototypalObject.GetBaseMetaObject( Expression );
    }
	
    protected virtual DynamicMetaObject CreatePrototypeMetaObject() {
        Expression castExpression = GetLimitedSelf();
        MemberExpression memberExpression = Expression.Property( castExpression, "Prototype" );
        return Create( _prototype, memberExpression );
    }

    protected Expression GetLimitedSelf() {
        return AreEquivalent( Expression.Type, LimitType ) ? Expression : Expression.Convert( Expression, LimitType );
    }

    protected bool AreEquivalent( Type lhs, Type rhs ) {
        return lhs == rhs || lhs.IsEquivalentTo( rhs );
    }
		
    protected virtual BindingRestrictions GetTypeRestriction() {
        if ( Value == null && HasValue ) {
            return BindingRestrictions.GetInstanceRestriction( Expression, null );
        }
        return BindingRestrictions.GetTypeRestriction( Expression, LimitType );
    }

    protected virtual DynamicMetaObject AddTypeRestrictions( DynamicMetaObject result ) {
        BindingRestrictions typeRestrictions = GetTypeRestriction().Merge( result.Restrictions );
        return new DynamicMetaObject( result.Expression, typeRestrictions, _metaObject.Value );
    }
	
    public override DynamicMetaObject BindInvokeMember( InvokeMemberBinder binder, DynamicMetaObject[] args ) {
        DynamicMetaObject errorSuggestion = AddTypeRestrictions( _metaObject.BindInvokeMember( binder, args ) );
        return binder.FallbackInvokeMember( _baseMetaObject, args, errorSuggestion );
    }
}
```

You may be thinking, ok, this is cool, but what use it is it? What is the use case? First, it's cool. Second, it sets the foundation for .NET [mixins][]. Third, it gives us a second form of inheritance (after parasitic) for PowerShell. 

What if we take the prototype and make it a collection of prototypes? What if instead of inheriting from ```DelegatingPrototype``` we reuse the internal prototypal skeleton? If this sounds familiar, it should. I am describing ruby classes with modules and a base class, but with C#...

If you want to see more or play around with the code, you can find full implementations in the [Archetype][] project.

  [Dynamic Language Runtime (DLR)]: http://en.wikipedia.org/wiki/Dynamic_Language_Runtime
  [dynamic dispatch]: http://en.wikipedia.org/wiki/Dynamic_dispatch
  [IDynamicMetaObjectProvider]: http://msdn.microsoft.com/en-us/library/system.dynamic.idynamicmetaobjectprovider(v=vs.100).aspx
  [dynamic]: http://msdn.microsoft.com/en-us/library/dd264741(v=vs.100).aspx
  [mixins]: http://en.wikipedia.org/wiki/Mixin
  [Archetype]: https://github.com/idavis/Archetype