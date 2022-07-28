---
author: "Sergey Penkovsky"
date: 2022-05-07
linktitle: CherryPick DI
menu:
  main:
    parent: tutorials
#next: /tutorials/github-pages-blog
#prev: /tutorials/automated-deployments
title: CherryPick DI
weight: 10
---


# CherryPick DI

DI-container – this is library, when provide mechanism dependency injection.


## Content

1. Preface
  2. Library features
  3. Library components
     - 3.1. scope
     - 3.2. module
     - 3.3. Binding
  4. Usage example
  5. Conclusion


## 1. Preface

The first attempts to develop our own DI for pet projects written on the Flutter SDK were started at the  start of 2020.

Several reasons prompted me to take this step:

1. At that time, I did not find DI in pub.dev with the ability to divide the container into scope (perhaps I was looking badly)
2. Simplify working with dependencies in the project
3. Desire to write your own DI
4. Have a simple DI in your arsenal (hopefully with a simple API)


## 2. Library's features:

DI container's main features:
 - Initialization instance with name
 - Initialization instance when singleton
 - Separate container to scopes


## 3. Main components DI

Library contain three main components:
 - Scope
 - Module
 - Binding


### 3.1. Scope

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

### 3.2. Module

**Module** is a container of user instances, and on the basis of which the user can create their modules. The user in his module must implement the `void builder (Scope currentScope)` method.


Example:

```dart
class AppModule extends Module {
  @override
  void builder(Scope currentScope) {
    bind<ApiClient>().toInstance(ApiClientMock());
  }
}
```


### 3.3. Binding

Binding is a custom instance configurator that contains methods for configuring a dependency.

There are two main methods for initializing a custom instance `toInstance ()` and `toProvide ()` and auxiliary `withName ()` and `singleton ()`.

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

## 4. Example


```dart
import 'dart:async';
import 'package:meta/meta.dart';
import 'package:cherrypick/cherrypick.dart';

class AppModule extends Module {
  @override
  void builder(Scope currentScope) {
    bind<ApiClient>().withName("apiClientMock").toInstance(ApiClientMock());
    bind<ApiClient>().withName("apiClientImpl").toInstance(ApiClientImpl());
  }
}

class FeatureModule extends Module {
  bool isMock;

  FeatureModule({required this.isMock});

  @override
  void builder(Scope currentScope) {
    bind<DataRepository>()
        .withName("networkRepo")
        .toProvide(
          () => NetworkDataRepository(
            currentScope.resolve<ApiClient>(
              named: isMock ? "apiClientMock" : "apiClientImpl",
            ),
          ),
        )
        .singleton();
    bind<DataBloc>().toProvide(
      () => DataBloc(
        currentScope.resolve<DataRepository>(named: "networkRepo"),
      ),
    );
  }
}

void main() async {
  final scope = openRootScope().installModules([
    AppModule(),
  ]);

  final subScope = scope
      .openSubScope("featureScope")
      .installModules([FeatureModule(isMock: true)]);

  final dataBloc = subScope.resolve<DataBloc>();
  dataBloc.data.listen((d) => print('Received data: $d'),
      onError: (e) => print('Error: $e'), onDone: () => print('DONE'));

  await dataBloc.fetchData();
}

class DataBloc {
  final DataRepository _dataRepository;

  Stream<String> get data => _dataController.stream;
  StreamController<String> _dataController = new StreamController.broadcast();

  DataBloc(this._dataRepository);

  Future<void> fetchData() async {
    try {
      _dataController.sink.add(await _dataRepository.getData());
    } catch (e) {
      _dataController.sink.addError(e);
    }
  }

  void dispose() {
    _dataController.close();
  }
}

abstract class DataRepository {
  Future<String> getData();
}

class NetworkDataRepository implements DataRepository {
  final ApiClient _apiClient;
  final _token = 'token';

  NetworkDataRepository(this._apiClient);

  @override
  Future<String> getData() async => await _apiClient.sendRequest(
      url: 'www.google.com', token: _token, requestBody: {'type': 'data'});
}

abstract class ApiClient {
  Future sendRequest({@required String url, String token, Map requestBody});
}

class ApiClientMock implements ApiClient {
  @override
  Future sendRequest(
      {@required String? url, String? token, Map? requestBody}) async {
    return 'Local Data';
  }
}

class ApiClientImpl implements ApiClient {
  @override
  Future sendRequest(
      {@required String? url, String? token, Map? requestBody}) async {
    return 'Network data';
  }
}
```


## 5. Conclusion

Now this library used to in three commercial projects and own pet projects.