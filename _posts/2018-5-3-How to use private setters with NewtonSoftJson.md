---
layout: post
title: How to use private setters with NewtonSoftJson
---

Writing software with OOP language like c# requires good encapsulation in order to have healthy code base and extensible system. To achieve this you have to use the `internal`, `private`, `protected`, `public` access modifiers properly. However, when you want to deserialize to an object which has `private set` properties using the default NewtonSoft.Json settings you are out of luck. It is pretty easy to achieve this.

This is our class which we want to deserialize:

```
public class UserBook
{
    UserBook()
    {
        Notes = new List<string>();
    }

    public UserBook(string username) : this()
    {
        if (string.IsNullOrEmpty(username)) throw new System.ArgumentNullException(nameof(username));

        Username = username;
    }

    public string Username { get; private set; }

    public IEnumerable<string> Notes { get; private set; }

    public void TakeNote(string note) { ... }
}
```

And this is the json:

```
{
	"username": "mynkow ",
	"notes": [
		"Today was sunny day.",
		"Today was rainy day."
	]
}
```

Now we need a `JsonSerializer` instance so we could deserialize. To configure the serializer you need to set the proper `JsonSerializerSettings` and pass them to the static factory `JsonSerializer.Create(settings)`. In previous versions of Newtonsoft.Json this was done like this:

```
public JsonSerializer GetSerializer()
{
    var settings = new JsonSerializerSettings();
    settings.ConstructorHandling = ConstructorHandling.AllowNonPublicDefaultConstructor;
    var contractResolver = new Newtonsoft.Json.Serialization.CamelCasePropertyNamesContractResolver();
    contractResolver.DefaultMembersSearchFlags = contractResolver.DefaultMembersSearchFlags | System.Reflection.BindingFlags.NonPublic;
    settings.ContractResolver = contractResolver;

    serializer = JsonSerializer.Create(settings);
}
```

There are two things happening here. The first is `.ConstructorHandling` which allows the serializer to use non-public constructors and instantiate the `Notes` collection. The second is `.ContractResolver` which hints the serializer to consider also non-public properties.

With the introduction of DotNet Core the Newtonsoft.Json library got some updates. One of which was the `DefaultMembersSearchFlags` being marked as obsolete. The good part is that you could continue using them with the full dot net framework and you will just see the following warning: 

>DefaultMembersSearchFlags is obsolete. To modify the members serialized inherit from DefaultContractResolver and override the GetSerializableMembers method instead.

It is easy to get rid of this warning. Here is the code:

```
public JsonSerializer GetSerializer()
{
    var settings = new JsonSerializerSettings();
    settings.ConstructorHandling = ConstructorHandling.AllowNonPublicDefaultConstructor;
    settings.ContractResolver = new ContractResolverWithPrivates();

    serializer = JsonSerializer.Create(settings);
}

public class ContractResolverWithPrivates : Newtonsoft.Json.Serialization.CamelCasePropertyNamesContractResolver
{
    protected override Newtonsoft.Json.Serialization.JsonProperty CreateProperty(System.Reflection.MemberInfo member, MemberSerialization memberSerialization)
    {
        var prop = base.CreateProperty(member, memberSerialization);

        if (!prop.Writable)
        {
            var property = member as System.Reflection.PropertyInfo;
            if (property != null)
            {
                var hasPrivateSetter = property.GetSetMethod(true) != null;
                prop.Writable = hasPrivateSetter;
            }
        }

        return prop;
    }
}

```

```
UserBook userBook = GetSerializer().Deserialize<UserBook>(json);
```
------------------------------

Software is fun! Happy coding!

------------------------------

[1]: https://github.com/Elders/RedLock