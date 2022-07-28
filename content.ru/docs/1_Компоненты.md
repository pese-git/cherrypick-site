## Основные компоненты DI

Библиотека состоит из трех основных компонентов:
 - Scope
 - Module
 - Binding


### Scope

**Scope** - это контейнер, который хранит все дерево зависимостей (scope,modules,instances).
Через scope можно получить доступ к `instance`, для этого нужно вызвать метод `resolve<T>()` и указать тип объекта, а так же можно передать дополнительные параметры.

**Scope** определяет область видимости и время жизни зависимостей. 

**Scope** в приложении образуют древовидную иерархическую структуру. Например, у вас может быть **Scope** для всего приложения, и дочерний **Scope** для конкретного экрана или группы экранов.

Чтобы получить объект **Scope**, его нужно “открыть”. Для простоты сделаем один **Scope** на всё приложение:

```dart
    final rootScope = CherryPick.openScope(named: 'appScope');
```

Если повторно открыть **Scope** с тем же самым именем, мы получим уже существующий экземпляр **Scope**.

Когда **Scope** перестанет быть нужным, его (и всё дерево “дочерних” **Scope**) можно будет закрыть с помощью метода `CHerryPick.closeScope(name)`


### Module

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


### Binding

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