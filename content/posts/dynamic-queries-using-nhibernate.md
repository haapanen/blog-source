---
title: "Dynamic queries using NHibernate with the help of C# expression trees"
date: 2017-02-05T00:00:00+03:00
---

I had to recently figure out a way to query database objects based on user input without knowing beforehand which classes user wants to query. Basically user would input a type and the system would find it’s child types from a database table. The children would then be fetched based on a set of rules (we do not want all items on the table, just the ones linked to the parent).

There was a “no raw SQL” limitation on the project and I had the data model mapped as NHibernate classes so my initial idea was to use reflection to match the inputted class name with the actual NHibernate class (e.g. user input of MyClass would be mapped to Foo.Bar.MyClass) and then use NHibernate session to simply query for the objects. This would also let me easily verify that the database actually contains the specific model as I could just find the mapping class for the object type based on the NHibernate mapping class convention (e.g. MyClassMap, YourClassMap). If there was no map, there was no table on the database meaning the user input was invalid.

It was a fairly simple process to do the initial input to object type mapping and simply querying the objects. I was quickly able to fetch a list of objects in the database based on just the type. Basically NHibernate let’s you query objects by type name string and specifying the return type as an “object”.

```csharp
// NHibernate entities are stored in a single assembly
// Get all classes from the assembly that MyClass is in
var entities = Assembly.GetAssembly(typeof(MyClass))
  .GetTypes()
  .Where(x => x.IsClass)
  .ToList();
// Input from user
var objectTypeName = "MyClass";
// See if the type actually exists
var objectType = entities
  .SingleOrDefault(x => x.FullName.Split('.').Last() == objectTypeName);
// Check if object type was actually found
if (objectType == null) {
  // Input isn't valid
  throw ...
}
// We need to fetch as "object" as we do not know the type
// compile time.
// We'll pass the object type name as a parameter to tell NHibernate
// what the type actually is.
// NHibernate session should be opened before
var objects = session.Query<object>(objectType.FullName);
```

I could then use reflection to access properties of the actual type:

```csharp
// An object taken from the result set from a session.Query call
var someObject = objects.FirstOrDefault();
// Read the value of someObject.SomeProperty
var value = objectType
  .GetProperty("SomeProperty")
  .GetValue(someObject, null);
Console.WriteLine(value);
```

While it was simple to query all objects in the database the next issue was a bit more puzzling for me. Previously I’ve used Fluent NHibernate to do the queries with LINQ but I couldn’t do that anymore in the same exact way as I didn’t know which fields I would have to use for filtering. The fields weren’t same for each class either. I couldn’t use reflection for the filtering as that’d mean I’d have to read all objects from a single table into memory and do the filtering there.

After some searching I figured out C# expression trees are probably the feature I’m looking for. LINQ queries are based on them after all. I was quickly able to find some examples about how other’s have used them to implement dynamic queries. I wanted to include objects where SomeProperty contained the text I was looking for. I ended up with code like this:

```csharp
var item = Expression.Parameter(typeof(object), "item");
// Where(item => item.SomeProperty)
var property = Expression.Property(item, "SomeProperty");
// Where(item => item.SomeProperty.Contains)
var containsMethod = typeof(string)
  .GetMethod("Contains", new[]{typeof(string));
// What we're searching for (e.g. SomeProperty.Contains("foo"))
var searchExpression = Expression.Constant("foo", typeof(string));
// Call the "Contains" method for the "SomeProperty" with
// searchExpression as the constant to compare with
var methodExpression = Expression
  .Call(property, containsMethod, searchExpression);
// Create a lambda to use inside the where call
var lambda = Expression.Lambda<Func<object, bool>>(methodExpression, item);
// Do the query with the expression
var objects = session
  .Query<object>(objectType.FullName).Where(lambda);
```

I ran into a problem with this. “object” doesn’t have a property of “SomeProperty”. I tried to switch the type in the Expression.Parameter call to typeof(objectType) but then the lambda would fail as the types didn’t match. I couldn’t change the lambda type (e.g. Func<someRuntimeVariable, bool>) as generics require a compile time type definition. All of the examples I was able to find always knew the object type on compile time. Took me some time to figure it out but the fix was fairly simple in the end. All I had to do was convert the type of “item” to the correct type in the Expression.Property call.

```csharp
// Where(item => item.SomeProperty)
var property = Expression
  .Property(Expression.Convert(item, objectType),
    "SomeProperty");
```

So the code would end up looking like this:

```csharp
// Declare the parameter we're accessing
var item = Expression.Parameter(typeof(object), "item");
// Where(item => item)
var property = Expression
  .Property(
    Expression.Convert(item, objectType),
    "SomeProperty"
);
// Where(item => item.Contains)
var containsMethod = typeof(string)
  .GetMethod("Contains", new[] {typeof(string)});
// Where(item => item.Contains("foo"))
var searchExpression = Expression.Constant("foo", typeof(string));
// Call the "Contains" method for the "SomeProperty"
var methodExpression = Expression
  .Call(property, method, searchExpression);
// Wrap the method into a lambda
var lambda = Expression
  .Lambda<Func<object, bool>>(methodExpression, item);
// Do the NHibernate search with the lambda
var result = session.Query<object>(objectType.FullName)
  .Where(lambda);
```

I could now replace the constants “SomeProperty” and “foo” with a variable and make it possible to do dynamic queries based on user input.

This was a rather short but very interesting peek into the internals of LINQ queries. C# expression trees are a powerful feature that let you do complex things that just aren’t possible using LINQ.
