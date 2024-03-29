# Создайте  первое приложение с применением CherryPick DI

## Step 1. Create application

Use to command `flutter create` for create new project:

```sh
flutter create sample_app
cd sample_app
```

## Step 2. Added dependencies

```yaml
dependencies:

  cherrypick: ^1.0.2 # DI

  shared_preferences: ^2.0.13

  flutter_bloc: ^8.0.1
  equatable: ^2.0.3

  flutter_form_builder: ^7.1.1
  form_builder_validators: ^8.1.1
```



## Step 3. Write simple app for user editable.


### Added user model to project directory `lib/user.dart`:

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


### Added  protocol for `UserRepository` to project directory `lib/user_repository.dart`:


```dart
import './user.dart';

abstract class UserRepository {
  User getUser();

  void saveUser(User user);
}
```

### Added implementation user repository to project directory `lib/pref_user_repository.dart`:

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


### Added events for `UserBloc`  `lib/bloc/user_bloc_event.dart`:

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


### Added states for `UserBloc`  `lib/bloc/user_bloc_state.dart`:

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

### Added bloc for used to repository  `lib/bloc/user_bloc.dart`:

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

### Added user edit screen  `lib/user_page.dart`:

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


### Added main widget  `lib/app.dart`:

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


### Added DI module  `lib/di/user_module.dart`:


Write all dependencies in module `UserModule`. 

We can see when `UserBloc` depend by `UserRepository`.

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

### Added scope name  `lib/di/scopes.dart`:


```dart
class Scopes {
  static const String appScope = 'appScope';
}
```


### Added function `main`  to  `lib/main.dart`:


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

We can see  to function `main()` when DI initialization in two stage.


Stage 1. Open `scope` 

```dart
 final appScope = CherryPick.openScope(scopeName: Scopes.appScope);
```

Stage 2. Module initialization `UserModule` in scope  `appScope`

```dart
appScope.installModules(
    [
      UserModule(sharedPreferences: sharedPreferences),
    ],
  );
```


## Step 4. Run application

Use to command `flutter run` for run application.

```sh
flutter run
```