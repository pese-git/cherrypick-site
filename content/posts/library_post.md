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

DI-контейнер – это библиотека, которая обеспечивает функциональность механизма внедрения зависимостей.


## Содержание

 1. Предисловие
 2. Возможности библиотеки
 3. Компоненты библиотеки
    - 3.1. Scope
    - 3.2. Module
    - 3.3. Binding
 4. Пример использования
 5. Заключение


## 1. Предисловие

Первые попытки разработать свой DI для пет проектов написанных на Flutter SDK были начаты в конце 2019 года.

Сподвигло меня на этот шаг несколько причин: 

1. На тот момент я не нашел DI в pub.dev с возможностью делить контейнер на scope (возможно плохо искал)
2. Упростить работу с зависимостями в проекте
3. Желание написать собственный  DI
4. Иметь в арсенале простой DI  (надеюсь с простым API)

В июне 2020 был принято решение вести разработку в публичном репозитории.

В марте 2021 было добавлена поддержка null-safety.

В апреле 2021 было переработано api библиотеки.

С апреля 2021 было принято решение использовать библиотеку в разработке коммерческого проекта.


## 2. Возможности библиотеки

Основные возможности DI контейнера:
 - Инициализация экземпляра с именем
 - Инициализация экземпляра как singleton
 - Разделение контейнера на области видимости (scopes)




## 3. Основные компоненты DI

Библиотека состоит из трех основных компонентов:
 - Scope
 - Module
 - Binding


### 3.1. Scope

**Scope** - это контейнер, который хранит все дерево зависимостей (scope,modules,instances).
Через scope можно получить доступ к `instance`, для этого нужно вызвать метод `resolve<T>()` и указать тип объекта, а так же можно передать дополнительные параметры.

**Scope** определяет область видимости и время жизни зависимостей. 

**Scope** в приложении образуют древовидную иерархическую структуру. Например, у вас может быть **Scope** для всего приложения, и дочерний **Scope** для конкретного экрана или группы экранов.

Чтобы получить объект **Scope**, его нужно “открыть”. Для простоты сделаем один **Scope** на всё приложение:

```dart
    final rootScope = CherryPick.openScope(named: 'appScope');
```

Если повторно открыть **Scope** с тем же самым именем, мы получим уже существующий экземпляр **Scope**.

Когда **Scope** перестанет быть нужным, его (и всё дерево “дочерних” **Scope**) можно будет закрыть с помощью метода `CherryPick.closeScope(name)`


### 3.2. Module

**Module** - это набор правил, по которым CherryPick будет разрешать зависимости в конкретном `Scope`. Пользователь в своем модуле должен реализовать метод `void builder(Scope currentScope)`. Модули добавляются в `Scope` с помощью метода `scope.installModules(…)`, после чего `Scope` может разрешать зависимости по правилам, определённым в его модулях.


Пример:

```dart
class AppModule extends Module {
  @override
  void builder(Scope currentScope) {
    bind<ApiClient>().toInstance(ApiClientMock());
  }
}
```


### 3.3. Binding

**Binding** - по сути это конфигуратор  для  пользовательского instance, который содержит методы для конфигурирования зависимости.

Есть два основных метода для инициализации пользовательского instance `toInstance()` и `toProvide()` и вспомогательных `withName()` и `singleton()`.

`toInstance()` - принимает готовый экземпляр

`toProvide()` -  принимает функцию `provider` (конструктор экземпляра)

`withName()` - принимает строку для именования экземпляра. По этому имени можно будет извлечь instance из  DI контейнера

`singleton()` -  устанавливает флаг в Binding, который говорит DI контейнеру, что зависимость одна.

Пример:

```dart
 // инициализация экземпляра текстовой строки через метод toInstance()
 bind<String>().toInstance("hello world");

 // или

 // инициализация экземпляра текстовой строки
 bind<String>().toProvide(() => "hello world");

 // инициализация экземпляра строки с именем
 bind<String>().withName("my_string").toInstance("hello world");
 // или
 bind<String>().withName("my_string").toProvide(() => "hello world");

 // инициализация экземпляра, как singleton
 bind<String>().toInstance("hello world");
 // или
 bind<String>().toProvide(() => "hello world").singleton();

```

## 4. Пример приложения


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


## 5. Заключение

На текущий момент библиотека используется в трех коммерческих проектах и собственных пет проектах.