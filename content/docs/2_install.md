
# Подключение библиотеки

## Установите библиотеку

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




## Импортируйте библиотеку

Теперь в вашем коде Dart вы можете использовать:

```dart
import 'package:cherrypick/cherrypick.dart';
```

