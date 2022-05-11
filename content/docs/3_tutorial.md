# Создайте  первое приложение с применением CherryPick DI

## Шаг 1. Создайте приложение

Используйте команду `flutter create` для создания нового проекта:

```sh
flutter create sample_app
cd sample_app
```

## Шаг 2. Добавьте зависимости

```yaml
dependencies:

  cherrypick: ^1.0.2 # DI

  shared_preferences: ^2.0.13

  flutter_bloc: ^8.0.1
  equatable: ^2.0.3

  flutter_form_builder: ^7.1.1
  form_builder_validators: ^8.1.1
```



## Шаг 3. Напишите простое приложение для редактирования пользователя


### Добавьте модель пользователя в каталог проекта  `lib/user.dart`:

```dart
import 'package:equatable/equatable.dart';

class User extends Equatable {
  final String name;
  final String email;

  const User({required this.name, required this.email});

  const User.empty()
      : name = '',
        email = '';

  @override
  List<Object?> get props => [name, email];

  @override
  bool? get stringify => true;
}
```


### Добавьте описание взаимодействия с репозиторием пользователя в каталог проекта  `lib/user_repository.dart`:

```dart
import './user.dart';

abstract class UserRepository {
  User getUser();

  void saveUser(User user);
}
```

### Добавьте реализацию репозитория пользователя в каталог проекта  `lib/pref_user_repository.dart`:

```dart
import 'package:shared_preferences/shared_preferences.dart';

import 'user.dart';
import 'user_repository.dart';

class PrefUserRepository extends UserRepository {
  final SharedPreferences sharedPreferences;

  static const String keyName = "KEY_NAME";
  static const String keyEmail = "KEY_EMAIL";

  PrefUserRepository({required this.sharedPreferences});

  @override
  User getUser() {
    return User(
      name: sharedPreferences.getString(keyName) ?? '',
      email: sharedPreferences.getString(keyEmail) ?? '',
    );
  }

  @override
  void saveUser(User user) async {
    await sharedPreferences.setString(keyName, user.name);
    await sharedPreferences.setString(keyEmail, user.email);
  }
}
```


### Добавьте event-ы для `UserBloc`  `lib/bloc/user_bloc_event.dart`:

```dart
import 'package:equatable/equatable.dart';
import 'package:sample_app/user.dart';

abstract class UserBlocEvent extends Equatable {
  const UserBlocEvent();

  factory UserBlocEvent.getUser() = GetUserEvent;

  const factory UserBlocEvent.saveUser(User user) = SaveUserEvent;
}

class GetUserEvent extends UserBlocEvent {
  @override
  List<Object?> get props => [];
}

class SaveUserEvent extends UserBlocEvent {
  final User user;

  const SaveUserEvent(this.user);

  @override
  List<Object?> get props => [user];
}
```


### Добавьте state-ы для `UserBloc`  `lib/bloc/user_bloc_state.dart`:

```dart
import 'package:equatable/equatable.dart';

import '../user.dart';

abstract class UserBlocState extends Equatable {
  final User user;

  const UserBlocState(this.user);

  const factory UserBlocState.init(User user) = InitState;

  const factory UserBlocState.successGetUser(User user) = SuccessGetUserState;

  const factory UserBlocState.successSaveUser(User user) = SuccessSaveUserState;

  @override
  List<Object?> get props => [user];
}

class InitState extends UserBlocState {
  const InitState(User user) : super(user);
}

class LoadingGetUserState extends UserBlocState {
  const LoadingGetUserState(User user) : super(user);
}

class SuccessGetUserState extends UserBlocState {
  const SuccessGetUserState(User user) : super(user);
}

class SuccessSaveUserState extends UserBlocState {
  const SuccessSaveUserState(User user) : super(user);
}
```


### Добавьте блок для управления репозиторием  `lib/bloc/user_bloc.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_bloc/flutter_bloc.dart';
import 'package:sample_app/bloc/user_bloc_event.dart';
import 'package:sample_app/bloc/user_bloc_state.dart';
import 'package:sample_app/user_repository.dart';

import '../user.dart';

class UserBloc extends Bloc<UserBlocEvent, UserBlocState> {
  final UserRepository userRepository;

  UserBloc({
    required this.userRepository,
  }) : super(const UserBlocState.init(User.empty())) {
    on<GetUserEvent>(_onGetUserEvent);
    on<SaveUserEvent>(_onSaveUserEvent);
  }

  void _onGetUserEvent(GetUserEvent event, emit) {
    final user = userRepository.getUser();
    emit(UserBlocState.successGetUser(user));
  }

  void _onSaveUserEvent(SaveUserEvent event, emit) {
    userRepository.saveUser(event.user);
    emit(UserBlocState.successSaveUser(event.user));
  }
}
```

### Добавьте  экран редактирования пользователя   `lib/user_page.dart`:

```dart
import 'package:cherrypick/cherrypick.dart';
import 'package:flutter/material.dart';
import 'package:flutter_bloc/flutter_bloc.dart';
import 'package:flutter_form_builder/flutter_form_builder.dart';
import 'package:form_builder_validators/form_builder_validators.dart';
import 'package:sample_app/bloc/user_bloc.dart';
import 'package:sample_app/bloc/user_bloc_event.dart';
import 'package:sample_app/bloc/user_bloc_state.dart';
import 'package:sample_app/di/scopes.dart';
import 'package:sample_app/user.dart';

class UserPage extends StatelessWidget {
  UserPage({Key? key, required this.title}) : super(key: key);

  final String title;

  final _formKey = GlobalKey<FormBuilderState>();

  @override
  Widget build(BuildContext context) {
    return BlocProvider<UserBloc>(
      create: (context) {
        return CherryPick.openScope(scopeName: Scopes.APP_SCOPE)
            .resolve<UserBloc>();
      },
      child: BlocBuilder<UserBloc, UserBlocState>(
        builder: ((context, state) {
          return Scaffold(
            appBar: AppBar(
              title: Text(title),
            ),
            body: Column(
              children: [
                Text(state.user.toString()),
                const SizedBox(width: 20),
                FormBuilder(
                  key: _formKey,
                  child: Column(
                    children: [
                      FormBuilderTextField(
                        name: 'name',
                        decoration: const InputDecoration(labelText: 'Name'),
                        keyboardType: TextInputType.text,
                        validator: FormBuilderValidators.compose([
                          FormBuilderValidators.required<String>(),
                        ]),
                      ),
                      FormBuilderTextField(
                        name: 'email',
                        decoration: const InputDecoration(labelText: 'Email'),
                        keyboardType: TextInputType.text,
                        validator: FormBuilderValidators.compose([
                          FormBuilderValidators.required<String>(),
                        ]),
                      ),
                    ],
                  ),
                ),
                const SizedBox(width: 20),
                MaterialButton(
                  color: Theme.of(context).colorScheme.secondary,
                  child: const Text(
                    "Save",
                    style: TextStyle(color: Colors.white),
                  ),
                  onPressed: () {
                    _formKey.currentState?.saveAndValidate();
                    BlocProvider.of<UserBloc>(context).add(
                      UserBlocEvent.saveUser(
                        User(
                          name: _formKey.currentState?.fields['name']?.value ??
                              '',
                          email:
                              _formKey.currentState?.fields['email']?.value ??
                                  '',
                        ),
                      ),
                    );
                  },
                ),
              ],
            ),
          );
        }),
      ),
    );
  }
}
```


### Добавьте  главный виджет   `lib/app.dart`:

```dart
import 'package:flutter/material.dart';

import 'user_page.dart';

class App extends StatelessWidget {
  const App({Key? key}) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Sample App',
      theme: ThemeData(
        primarySwatch: Colors.blue,
      ),
      home: UserPage(title: 'Sample App'),
    );
  }
}
```


### Добавьте di модуль  `lib/di/user_module.dart`:

В модуле `UserModule` описываем все зависимости. Из кода видно что `UserBloc` зависит от `UserRepository`.

```dart
import 'package:cherrypick/cherrypick.dart';
import 'package:sample_app/bloc/user_bloc.dart';
import 'package:sample_app/pref_user_repository.dart';
import 'package:sample_app/user_repository.dart';
import 'package:shared_preferences/shared_preferences.dart';

class UserModule extends Module {
  final SharedPreferences sharedPreferences;

  UserModule({required this.sharedPreferences});

  @override
  void builder(Scope currentScope) {
    bind<UserRepository>()
        .toProvide(
            () => PrefUserRepository(sharedPreferences: sharedPreferences))
        .singleton();

    bind<UserBloc>().toProvide(
      () => UserBloc(
        userRepository: currentScope.resolve<UserRepository>(),
      ),
    );
  }
}
```

### Добавьте имя scope  `lib/di/scopes.dart`:


```dart
class Scopes {
  static const String appScope = 'appScope';
}
```


### Добавьте  функцию `main` в  `lib/main.dart`:


```dart
import 'package:cherrypick/cherrypick.dart';
import 'package:flutter/material.dart';
import 'package:sample_app/di/user_module.dart';
import 'package:sample_app/di/scopes.dart';
import 'package:shared_preferences/shared_preferences.dart';

import './app.dart';

void main() async {
  final sharedPreferences = await SharedPreferences.getInstance();

  final appScope = CherryPick.openScope(scopeName: Scopes.appScope);

  appScope.installModules(
    [
      UserModule(sharedPreferences: sharedPreferences),
    ],
  );

  runApp(const App());
}
```

Если посмотреть на тело функции `main()`, то можно увидеть, что инициализация di происходит в два этапа:

1 Открытие `scope` 

```dart
 final appScope = CherryPick.openScope(scopeName: Scopes.appScope);
```

1 Инициализация модуля `UserModule` в scope  `appScope`

```dart
appScope.installModules(
    [
      UserModule(sharedPreferences: sharedPreferences),
    ],
  );
```


## Шаг 4. Запустите приложение

Используйте команду `flutter run` для запуска приложения:

```sh
flutter run
```