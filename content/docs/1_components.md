## Main components DI

Library contain three main components:
 - Scope
 - Module
 - Binding


### Scope

**Scope** is a container that stores the entire dependency tree (scope, modules, instances).
Through the scope, you can access the custom `instance`, for this you need to call the `resolve<T>()` method and specify the type of the object, and you can also pass additional parameters.

Example:

```dart
    // open main scope
    final rootScope =  Cherrypick.openRootScope();

    // initializing scope with a custom module
    rootScope.installModules([AppModule()]);

    // takes custom instance
    final str = rootScope.resolve<String>();
    // or
    final str = rootScope.tryResolve<String>();

    // close main scope
    Cherrypick.closeRootScope();
```

### Module

**Module** is a container of user instances, and on the basis of which the user can create their modules. The user in his module must implement the `void builder (Scope currentScope)` method. Модули добавляются в `Scope` с помощью метода `scope.installModules(…)`, после чего `Scope` может разрешать зависимости по правилам, определённым в его модулях.


Example:

```dart
class AppModule extends Module {
  @override
  void builder(Scope currentScope) {
    bind<ApiClient>().toInstance(ApiClientMock());
  }
}
```


### Binding

Binding is a custom instance configurator that contains methods for configuring a dependency.

There are two main methods for initializing a custom instance `toInstance()` and `toProvide()` and auxiliary `withName()` and `singleton()`.

`toInstance()` - takes a initialized instance

`toProvide()` -  takes a `provider` function (instance constructor)

`withName()` - takes a string to name the instance. By this name, it will be possible to extract instance from the DI container

`singleton()` -  sets a flag in the Binding that tells the DI container that there is only one dependency.

Example:

```dart
 // initializing a text string instance through a method toInstance()
 Binding<String>().toInstance("hello world");

 // or

 // initializing a text string instance
 Binding<String>().toProvide(() => "hello world");

 // initializing an instance of a string named
 Binding<String>().withName("my_string").toInstance("hello world");
 // or
 Binding<String>().withName("my_string").toProvide(() => "hello world");

 // instance initialization like singleton
 Binding<String>().toInstance("hello world");
 // or
 Binding<String>().toProvide(() => "hello world").singleton();

```