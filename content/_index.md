---
title: Introduction
type: docs
---

# CherryPick DI

{{< columns >}}
## DI-контейнер

DI-контейнер – это библиотека, которая обеспечивает функциональность механизма внедрения зависимостей.

<--->

## Возможности библиотеки

Основные возможности DI контейнера:
 - Инициализация экземпляра с именем
 - Инициализация экземпляра как singleton
 - Разделение контейнера на области видимости (scopes)
{{< /columns >}}


## Подключение библиотеки

### Установите библиотеку

Запустите команду:

Для Dart проекта:

```sh
dart pub add cherrypick
```


Для Flutter проекта:

```sh
flutter pub add cherrypick
```

Команда добавит  строку  пакета  `cherrypick` (и запустит в фоне неявно dart pub get):

```yaml
dependencies:
  cherrypick: <version>
```




### Импортируйте библиотеку

Теперь в вашем коде Dart вы можете использовать:

```dart
import 'package:cherrypick/cherrypick.dart';
```
