---
layout: post
title: "Modules and Mixins for .NET: Taking Delegation to a New Level"
date: 2012-10-31 20:58
published: true
categories: dynamic c# dlr expressiontrees
---
In the last article on [prototypal inheritance][] using [Archetype][], I showed how to construct a delegation chain and simulate JavaScript's prototypal inheritance for .NET languages. If you haven't looked at that post, I highly recommend reading it through as I am going to skip over a lot of the theory I covered previously. Today I am taking delegation in .NET a step further. If you aren't familiar with mixins and ruby modules, I would also recommend looking into them as what I am showing here is directly inspired by them.

When looking at prototypal inheritance for .NET, we had a very simple delegation chain. With module support we need to ammend how we evaluate and interpret expressions being called on our objects. Does the current object support the operation? Can it respond? If not, we need to loop through the prototypes (modules/mixins) that have been attached to this instance; however, as with most things in life, there is a catch. We need to process them last to first and at each level we need to start the evaluation all over again. Why last first?

When we declare a module/mixin for our class, it is taking precededence over the modules that have already been imported. This mimics the way that Ruby works. Defining a property or method that already exists redefines that member.

Revisiting the `DelegatingObject` (formerly `PrototypalObject`), I have added a new backing list to replace the single prototype from the earlier version. All existing tests and code continue to work just fine with these minimal changes. The big change is the call to creating the `DynamicMetaObject` with the `ModuleMetaObject` `ctor`. We are passing a collection of prototypes/modules which will be evaluated by the `ModuleMetaObject` which may contain many more sets of modules and delegation chains.

``` csharp
public class DelegatingObject : DynamicObject {
    public DelegatingObject( params object[] modules ) {
        Modules = new List<object>( modules ?? new object[]{} );
    }

    public IList<object> Modules { get; protected set; }

    public override DynamicMetaObject GetMetaObject( Expression parameter ) {
        DynamicMetaObject baseMetaObject = base.GetMetaObject( parameter );

        if ( Modules == null || Modules.Count == 0 ) {
            return baseMetaObject;
        }

        return new ModuleMetaObject( parameter, this, baseMetaObject, Modules );
    }
}
```

Instead of executing a preorder traversal of the module chains, we are routing all binding calls to a new method `ApplyBinding` which isolates all of the resolution logic to a single method. We have an additional constraint this time in that we are binding the expression to delegation chains that may fail (whereas before we had only a single chain that could fail). When this binding failure occurs, an expression with a `NodeType` of `ExpressionType.Throw` is returned and we need to ignore these failed chains. I am also able to leverage `Func<,>`, `Func<,,>`, and closures to capture the binding calls and context arguments which makes every override an easy call to `ApplyBinding`.

There is one operation that has a custom implementation. The `BindConvert` call passes a lamdba expression which calls the `Convert` utility method. Casting is .NET has some interesting nuances that have to be taken into account that would break when doing the module resolution if it were done just like everything else. 

``` csharp
public class ModuleMetaObject : DynamicMetaObject
{
    private readonly DynamicMetaObject _BaseMetaObject;
    private readonly IList<object> _Modules;

    public ModuleMetaObject( Expression expression,
                             object value,
                             DynamicMetaObject baseMetaObject,
                             IList<object> modules )
            : base( expression, BindingRestrictions.Empty, value ) {
        _Modules = modules;
        _BaseMetaObject = baseMetaObject;
    }

    protected DynamicMetaObject BaseMetaObject { get { return _BaseMetaObject; } }

    protected IList<object> Modules { get { return _Modules; } }

    protected virtual DynamicMetaObject AddTypeRestrictions( DynamicMetaObject result, object value ) {
        BindingRestrictions typeRestrictions =
                GetTypeRestriction().Merge( result.Restrictions );
        var metaObject = new DynamicMetaObject( result.Expression, typeRestrictions, value );
        return metaObject;
    }

    protected virtual DynamicMetaObject CreateModuleMetaObject( object module ) {
        DynamicMetaObject moduleMetaObject = Create( module, Expression.Constant( module ) );
        return moduleMetaObject;
    }

    protected virtual BindingRestrictions GetTypeRestriction() {
        if ( Value == null && HasValue ) {
            return BindingRestrictions.GetInstanceRestriction( Expression, null );
        }
        return BindingRestrictions.GetTypeRestriction( Expression, LimitType );
    }

    protected Expression GetLimitedSelf() {
        return AreEquivalent( Expression.Type, LimitType )
                       ? Expression
                       : Expression.Convert( Expression, LimitType );
    }

    protected bool AreEquivalent( Type lhs, Type rhs ) {
        return lhs == rhs || lhs.IsEquivalentTo( rhs );
    }

    public override DynamicMetaObject BindBinaryOperation( BinaryOperationBinder binder, DynamicMetaObject arg ) {
        return ApplyBinding( meta => meta.BindBinaryOperation( binder, arg ),
                             ( target, errorSuggestion ) =>
                             binder.FallbackBinaryOperation( target, arg, errorSuggestion ) );
    }

    public override DynamicMetaObject BindConvert( ConvertBinder binder ) {
        return ApplyBinding( meta => Convert( binder, meta ), binder.FallbackConvert );
    }

    public override DynamicMetaObject BindCreateInstance( CreateInstanceBinder binder, DynamicMetaObject[] args ) {
        return ApplyBinding( meta => meta.BindCreateInstance( binder, args ),
                             ( target, errorSuggestion ) =>
                             binder.FallbackCreateInstance( target, args, errorSuggestion ) );
    }

    public override DynamicMetaObject BindDeleteIndex( DeleteIndexBinder binder, DynamicMetaObject[] indexes ) {
        return ApplyBinding( meta => meta.BindDeleteIndex( binder, indexes ),
                             ( target, errorSuggestion ) =>
                             binder.FallbackDeleteIndex( target, indexes, errorSuggestion ) );
    }

    public override DynamicMetaObject BindDeleteMember( DeleteMemberBinder binder ) {
        return ApplyBinding( meta => meta.BindDeleteMember( binder ), binder.FallbackDeleteMember );
    }

    public override DynamicMetaObject BindGetMember( GetMemberBinder binder ) {
        return ApplyBinding( meta => meta.BindGetMember( binder ), binder.FallbackGetMember );
    }

    public override DynamicMetaObject BindGetIndex( GetIndexBinder binder, DynamicMetaObject[] indexes ) {
        return ApplyBinding( meta => meta.BindGetIndex( binder, indexes ),
                             ( target, errorSuggestion ) =>
                             binder.FallbackGetIndex( target, indexes, errorSuggestion ) );
    }

    public override DynamicMetaObject BindInvokeMember( InvokeMemberBinder binder, DynamicMetaObject[] args ) {
        return ApplyBinding( meta => meta.BindInvokeMember( binder, args ),
                             ( target, errorSuggestion ) =>
                             binder.FallbackInvokeMember( target, args, errorSuggestion ) );
    }

    public override DynamicMetaObject BindInvoke( InvokeBinder binder, DynamicMetaObject[] args ) {
        return ApplyBinding( meta => meta.BindInvoke( binder, args ),
                             ( target, errorSuggestion ) => binder.FallbackInvoke( target, args, errorSuggestion ) );
    }

    public override DynamicMetaObject BindSetIndex( SetIndexBinder binder,
                                                    DynamicMetaObject[] indexes,
                                                    DynamicMetaObject value ) {
        return ApplyBinding( meta => meta.BindSetIndex( binder, indexes, value ),
                             ( target, errorSuggestion ) =>
                             binder.FallbackSetIndex( target, indexes, value, errorSuggestion ) );
    }

    public override DynamicMetaObject BindSetMember( SetMemberBinder binder, DynamicMetaObject value ) {
        return ApplyBinding( meta => meta.BindSetMember( binder, value ),
                             ( target, errorSuggestion ) =>
                             binder.FallbackSetMember( target, value, errorSuggestion ) );
    }

    public override DynamicMetaObject BindUnaryOperation( UnaryOperationBinder binder ) {
        return ApplyBinding( meta => meta.BindUnaryOperation( binder ), binder.FallbackUnaryOperation );
    }

    protected virtual DynamicMetaObject ApplyBinding( Func<DynamicMetaObject, DynamicMetaObject> bindTarget,
                                                      Func<DynamicMetaObject, DynamicMetaObject, DynamicMetaObject>
                                                              bindFallback ) {
        DynamicMetaObject errorSuggestion = ResolveModuleChain( bindTarget );
        return ( errorSuggestion == null ) 
            ? bindTarget( BaseMetaObject )
            : bindFallback( BaseMetaObject, errorSuggestion );
    }

    private DynamicMetaObject ResolveModuleChain( Func<DynamicMetaObject, DynamicMetaObject> bindTarget ) {
        for ( int index = Modules.Count - 1; index >= 0; index-- ) {
            object module = Modules[index];
            DynamicMetaObject metaObject = GetDynamicMetaObjectFromModule( bindTarget, module );

            if ( metaObject == null ||
                 metaObject.Expression.NodeType == ExpressionType.Throw ) {
                continue;
            }

            return metaObject;
        }
        return null;
    }

    private DynamicMetaObject GetDynamicMetaObjectFromModule( Func<DynamicMetaObject, DynamicMetaObject> bindTarget,
                                                              object module ) {
        DynamicMetaObject moduleMetaObject = CreateModuleMetaObject( module );
        DynamicMetaObject boundMetaObject = bindTarget( moduleMetaObject );
        DynamicMetaObject result = AddTypeRestrictions( boundMetaObject, boundMetaObject.Value );
        return result;
    }

    private static bool TryConvert( ConvertBinder binder, DynamicMetaObject instance, out DynamicMetaObject result ) {
        if ( instance.HasValue && instance.RuntimeType.IsValueType ) {
            result = instance.BindConvert( binder );
            return true;
        }

        if ( binder.Type.IsInterface ) {
            result = new DynamicMetaObject( Convert( instance.Expression, binder.Type ),
                                            BindingRestrictions.Empty,
                                            instance.Value );
            result = result.BindConvert( binder );
            return true;
        }

        if ( typeof (IDynamicMetaObjectProvider).IsAssignableFrom( instance.RuntimeType ) ) {
            result = instance.BindConvert( binder );
            return true;
        }

        result = null;
        return false;
    }

    private static DynamicMetaObject Convert( ConvertBinder binder, DynamicMetaObject instance ) {
        DynamicMetaObject result;
        return TryConvert( binder, instance, out result ) ? result : instance;
    }

    private static Expression Convert( Expression expression, Type type ) {
        return expression.Type == type ? expression : Expression.Convert( expression, type );
    }
}
```

One great thing about this implementation is that we can replicate prototypal inheritance completely. We just need to make sure that each module we create only has a single base module all the way through the chain.

With these two classes now in place, it is time to start having some fun. While there are many fun ways we can leverage the `DelegatingObject` to mix in behavior, today I am going to focus on one very specific usage.

Invariably, when talking about module support in .NET, one of the most common requests you will here people talk about is to have a `INotifyPropertyChanged` module. While this can now be easily done, we have an added benefit from the way that `ModuleMetaObject` handles casting. We can pass around our model object and it can be cast as `INotifyPropertyChanged` without the need to generate a runtime proxy. The cast will actually dig through the module hierarchy, find that we have a module which provides the interface, and return that object. 

First, let's create a utility interface and module that will help us out and abstract some boilerplate code.

``` csharp
public interface INotifyPropertyChanges : INotifyPropertyChanged, INotifyPropertyChanging {
    void OnPropertyChanged( string propertyName = "" );
    void OnPropertyChanging( string propertyName = "" );
}

public class NotifyPropertyChangesModule : INotifyPropertyChanges {
    public event PropertyChangedEventHandler PropertyChanged;
    public event PropertyChangingEventHandler PropertyChanging;

    public virtual void OnPropertyChanged( string propertyName = "" ) {
        PropertyChangedEventHandler handler = PropertyChanged;
        if ( handler != null ) {
            handler( this, new PropertyChangedEventArgs( propertyName ) );
        }
    }

    public virtual void OnPropertyChanging( string propertyName = "" ) {
        PropertyChangingEventHandler handler = PropertyChanging;
        if ( handler != null ) {
            handler( this, new PropertyChangingEventArgs( propertyName ) );
        }
    }
}
```

One thing you may notice is that I did not use the `CallerMemberName` attribute available in .NET 4.5. When using modules, the caller name will not be the originating caller you expect. Since we are building expression trees to call the members at runtime, you cannot rely on this new feature.

With a module in place, we can now create a model which can have the functionality weaved in:

``` csharp
public class Person : DelegatingObject {
    private string _Name;

    public Person()
            : this( new NotifyPropertyChangesModule() ) { }

    public Person( params object[] modules )
            : base( modules ) { }

    public string Name {
        get { return _Name; }
        set {
            This.OnPropertyChanging( "Name" );
            if ( _Name != value ) {
                _Name = value;
                This.OnPropertyChanged( "Name" );
            }
        }
    }

    private dynamic This { get { return this; } }
}
```
The implementation is pretty simple. The thing to remember is that in order for the module chain to trigger, whether you are in the object or acting on the model object, the invocation must take place on a `dynamic` object. I have provided a simple property `This` which casts the model object to `dynamic` allowing us to access the latebound method calls.

There are a few different ways we can work with this object now:

``` csharp
public void UsingModelObjectAsDynamic() {
    dynamic person = new Person();
    // The cast to the interface will work returning the inner module
    INotifyPropertyChanges inpc = person;
    inpc.PropertyChanged +=
            ( sender, args ) => Console.WriteLine( "The field {0} has changed.", args.PropertyName );
    inpc.PropertyChanging +=
            ( sender, args ) => Console.WriteLine( "The field {0} is changing.", args.PropertyName );
    // We have full IntelliSense when working with inpc,
    // but now accessing person.Name looses IntelliSense
    person.Name = "Inigo Montoya"; // trigger the events
}

public void UsingModelObjectAsStronglyTyped() {
    Person person = new Person();
    // Casting first to dynamic triggers the DelegatingObject's casting system
    INotifyPropertyChanges inpc = (dynamic) person;
    inpc.PropertyChanged +=
            ( sender, args ) => Console.WriteLine( "The field {0} has changed.", args.PropertyName );
    inpc.PropertyChanging +=
            ( sender, args ) => Console.WriteLine( "The field {0} is changing.", args.PropertyName );
    // We have full IntelliSense when working with person.
    person.Name = "Inigo Montoya"; // trigger the events
}
```

Sometimes, we may want to handle accessing the interface provided by a module. Again leveraging the `dynamic` casting, we can create a new property which will provide us access to the interface. I have also created a new `Age` property which takes advantage of this feature and now gives us IntelliSense when acting on the `INotifyPropertyChanges` feature.

``` csharp
private int _Age;

public int Age {
    get { return _Age; }
    set {
        Inpc.OnPropertyChanging( "Age" );
        if ( _Age != value ) {
            _Age = value;
            Inpc.OnPropertyChanged( "Age" );
        }
    }
}

internal INotifyPropertyChanges Inpc { get { return This; } }
```

We can also leverage this property to provide external access to our mixed in behavior:

``` csharp
public void UsingModelWithProxyCastingProperty() {
    Person person = new Person();
    // The cast property to give us IntelliSense
    person.Inpc.PropertyChanged +=
            ( sender, args ) => Console.WriteLine( "The field {0} has changed.", args.PropertyName );
    person.Inpc.PropertyChanging +=
            ( sender, args ) => Console.WriteLine( "The field {0} is changing.", args.PropertyName );
    // We also still have IntelliSense on Name
    person.Name = "Inigo Montoya"; // trigger the events
}
```

So far, each model has received its own instance of a module, but that is by no means the only usage pattern. Object's can share modules and behavior:

``` csharp
public void ShareModulesToShareBehavior() {
    var module = new NotifyPropertyChangesModule();
    module.PropertyChanged +=
            ( sender, args ) => Console.WriteLine( "The field {0} has changed.", args.PropertyName );
    module.PropertyChanging +=
            ( sender, args ) => Console.WriteLine( "The field {0} is changing.", args.PropertyName );
    Person inigo = new Person( module ) { Name = "Inigo" };
    Person ian = new Person( module ) { Name = "Ian" };
    // Four events triggered by setting the names, 2x Changed, 2x Changing
    inigo.Age = 35;
    ian.Age = 30;
    // The module is now acting somewhat like a message channel in a message broker
}
```

This barely scratches the surface of what is possible, but hopefully it gives you a taste.

Another simple pattern you can apply is adding an `ExpandoObject` to the beginning of your module chain (thus it is the last resolved). This will give you a fallback behavior of the `Expando` should all else fail. You can also create a system to handle an analog to `method_missing` by overriding the `TryXyz` members you get when deriving from `DelegatingObject` (via `DynamicObject`). I playing around with a [sample implementation][] right now which will also delegate to static members.

If you find this interesting or have ideas for other modules, I'd love to hear about them.

  [prototypal inheritance]: /2012/10/prototypal-inheritance-in-net-delegation-at-last/
  [Archetype]: https://github.com/idavis/Archetype
  [sample implementation]: https://github.com/idavis/Archetype/blob/2add5d68bef178f6a76e75124dbff948474bfefe/src/Archetype/Sandbox/PrototypalObject.cs