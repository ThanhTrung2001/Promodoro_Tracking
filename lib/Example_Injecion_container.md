In a **Clean Architecture** Flutter project, the `injection_container.dart` file is responsible for managing dependency injection (DI). This file typically uses a service locator like `get_it` to register the dependencies used across the app, such as repositories, data sources, use cases, and Riverpod providers. It allows you to keep the app loosely coupled and testable.

Here’s an example of what `injection_container.dart` might look like using **get_it** for dependency injection in combination with **Riverpod** for state management.

### Example of `injection_container.dart`

```dart
import 'package:get_it/get_it.dart';
import 'package:myapp/core/network/network_info.dart';
import 'package:myapp/core/repositories/user_repository.dart';
import 'package:myapp/features/data/datasources/user_local_data_source.dart';
import 'package:myapp/features/data/datasources/user_remote_data_source.dart';
import 'package:myapp/features/data/repositories/user_repository_impl.dart';
import 'package:myapp/features/domain/usecases/get_user.dart';
import 'package:http/http.dart' as http;

final sl = GetIt.instance;  // sl = service locator

Future<void> init() async {
  //! Features - User
  // Providers are created using Riverpod in a separate provider file
  // We will inject use cases and repositories here
  
  // Use cases
  sl.registerLazySingleton(() => GetUser(sl()));

  // Repositories
  sl.registerLazySingleton<UserRepository>(() => UserRepositoryImpl(
    remoteDataSource: sl(),
    localDataSource: sl(),
    networkInfo: sl(),
  ));

  // Data sources
  sl.registerLazySingleton<UserRemoteDataSource>(() => UserRemoteDataSourceImpl(client: sl()));
  sl.registerLazySingleton<UserLocalDataSource>(() => UserLocalDataSourceImpl());

  //! Core
  sl.registerLazySingleton<NetworkInfo>(() => NetworkInfoImpl());

  //! External
  sl.registerLazySingleton(() => http.Client());
}
```

### Breakdown of the `injection_container.dart` File:
1. **Service Locator Initialization**:
   - `GetIt` is used as the service locator, accessed via `sl` (short for service locator). This instance will manage the registration of dependencies throughout the app.

2. **Feature-Specific Registrations**:
   - **Use Cases**: For example, the `GetUser` use case is registered with `sl()`. This allows you to inject the use case wherever needed in the app.
   - **Repositories**: The `UserRepository` is registered, which points to `UserRepositoryImpl`. This repository requires a remote data source, local data source, and network info, all of which are injected using `sl()` as dependencies.
   - **Data Sources**: Both remote and local data sources (e.g., `UserRemoteDataSource` and `UserLocalDataSource`) are registered here. They are used inside the repository implementation.
   
3. **Core and External Dependencies**:
   - **NetworkInfo**: This is a service that checks network connectivity and is injected where necessary.
   - **External Libraries**: For example, the `http.Client` is registered as a dependency for making HTTP calls.

### Integrating `injection_container.dart` with Riverpod Providers

Riverpod providers usually depend on other services such as repositories or use cases. You can still use `GetIt` to register dependencies but inject those into the Riverpod providers.

Here’s how you might access `sl()` in a **Riverpod provider**.

```dart
// features/presentation/providers/user_provider.dart

import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:myapp/features/domain/usecases/get_user.dart';
import 'package:myapp/features/presentation/state/user_state.dart';
import 'package:myapp/injection_container.dart';

// UserNotifier using the GetUser use case
class UserNotifier extends StateNotifier<UserState> {
  final GetUser getUserUseCase;

  UserNotifier({required this.getUserUseCase}) : super(UserState.initial());

  Future<void> getUser(int id) async {
    state = state.loading();
    try {
      final user = await getUserUseCase.call(id);
      state = state.success(user);
    } catch (error) {
      state = state.failure(error.toString());
    }
  }
}

// Injecting the use case via the service locator (GetIt)
final userProvider = StateNotifierProvider<UserNotifier, UserState>((ref) {
  return UserNotifier(getUserUseCase: sl<GetUser>());
});
```

### Calling `init()` in `main.dart`

You need to initialize your dependency injection setup early in the app lifecycle, typically in the `main()` function.

```dart
// main.dart
import 'package:flutter/material.dart';
import 'package:myapp/injection_container.dart' as di;
import 'package:flutter_riverpod/flutter_riverpod.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  
  // Initialize dependency injection
  await di.init();
  
  runApp(ProviderScope(child: MyApp()));
}

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Flutter Clean Architecture',
      theme: ThemeData(
        primarySwatch: Colors.blue,
      ),
      home: UserPage(),
    );
  }
}
```

### Summary:
- **`injection_container.dart`** manages dependency injection using **GetIt**.
- It registers **use cases**, **repositories**, **data sources**, and other core services such as **NetworkInfo** and external dependencies like **http.Client**.
- You can inject these dependencies into your **Riverpod providers** by accessing them via `sl()` (service locator).
- Ensure that `di.init()` is called in `main.dart` before the app starts to ensure that all dependencies are registered.

---------------

## Example of Repository
Here's an example of a `UserRepository` implementation in the context of **Clean Architecture** for Flutter, where the repository interacts with both **remote** and **local** data sources to fetch user data. This example will also show how `UserRepository` connects with use cases and Riverpod providers, as discussed earlier.

### User Repository Interface

The repository interface typically resides in the **domain** layer, and it defines the contract that the implementation must follow. It also returns **Entities** instead of data models to keep the domain layer pure and independent of external dependencies.

```dart
// lib/core/repositories/user_repository.dart

import 'package:myapp/features/domain/entities/user.dart';

abstract class UserRepository {
  Future<User> getUser(int id);
}
```

### User Repository Implementation

The actual implementation of the repository resides in the **data** layer. It will use both **local** and **remote** data sources to fetch data, along with error handling.

```dart
// lib/features/data/repositories/user_repository_impl.dart

import 'package:myapp/core/error/exceptions.dart';
import 'package:myapp/core/network/network_info.dart';
import 'package:myapp/core/repositories/user_repository.dart';
import 'package:myapp/features/data/datasources/user_local_data_source.dart';
import 'package:myapp/features/data/datasources/user_remote_data_source.dart';
import 'package:myapp/features/domain/entities/user.dart';

class UserRepositoryImpl implements UserRepository {
  final UserRemoteDataSource remoteDataSource;
  final UserLocalDataSource localDataSource;
  final NetworkInfo networkInfo;

  UserRepositoryImpl({
    required this.remoteDataSource,
    required this.localDataSource,
    required this.networkInfo,
  });

  @override
  Future<User> getUser(int id) async {
    // Check if device has internet connection
    if (await networkInfo.isConnected) {
      try {
        // Fetch user from remote data source (API)
        final remoteUser = await remoteDataSource.getUser(id);
        // Cache the user locally for offline use
        localDataSource.cacheUser(remoteUser);
        return remoteUser;
      } on ServerException {
        throw ServerException(); // Handle remote errors
      }
    } else {
      try {
        // Fetch user from local data source if offline
        return await localDataSource.getLastUser();
      } on CacheException {
        throw CacheException(); // Handle cache errors
      }
    }
  }
}
```

### Data Sources

The repository depends on **Remote** and **Local** data sources to fetch data.

#### Remote Data Source

This data source interacts with an external API to fetch user data.

```dart
// lib/features/data/datasources/user_remote_data_source.dart

import 'dart:convert';
import 'package:myapp/core/error/exceptions.dart';
import 'package:myapp/features/data/models/user_model.dart';
import 'package:http/http.dart' as http;

abstract class UserRemoteDataSource {
  Future<UserModel> getUser(int id);
}

class UserRemoteDataSourceImpl implements UserRemoteDataSource {
  final http.Client client;

  UserRemoteDataSourceImpl({required this.client});

  @override
  Future<UserModel> getUser(int id) async {
    final response = await client.get(Uri.parse('https://jsonplaceholder.typicode.com/users/$id'));

    if (response.statusCode == 200) {
      return UserModel.fromJson(json.decode(response.body));
    } else {
      throw ServerException();
    }
  }
}
```

#### Local Data Source

This data source interacts with local storage (e.g., SQLite, shared preferences, or a file) to cache the user data.

```dart
// lib/features/data/datasources/user_local_data_source.dart

import 'dart:convert';
import 'package:shared_preferences/shared_preferences.dart';
import 'package:myapp/core/error/exceptions.dart';
import 'package:myapp/features/data/models/user_model.dart';

abstract class UserLocalDataSource {
  Future<UserModel> getLastUser();
  Future<void> cacheUser(UserModel userToCache);
}

const CACHED_USER = 'CACHED_USER';

class UserLocalDataSourceImpl implements UserLocalDataSource {
  final SharedPreferences sharedPreferences;

  UserLocalDataSourceImpl({required this.sharedPreferences});

  @override
  Future<UserModel> getLastUser() {
    final jsonString = sharedPreferences.getString(CACHED_USER);
    if (jsonString != null) {
      return Future.value(UserModel.fromJson(json.decode(jsonString)));
    } else {
      throw CacheException();
    }
  }

  @override
  Future<void> cacheUser(UserModel userToCache) {
    return sharedPreferences.setString(
      CACHED_USER,
      json.encode(userToCache.toJson()),
    );
  }
}
```

### User Model

In the data layer, the **UserModel** represents the data structure that maps JSON to Dart objects. It extends the **User** entity from the domain layer.

```dart
// lib/features/data/models/user_model.dart

import 'package:myapp/features/domain/entities/user.dart';

class UserModel extends User {
  UserModel({
    required int id,
    required String name,
    required String email,
  }) : super(id: id, name: name, email: email);

  // Mapping from JSON
  factory UserModel.fromJson(Map<String, dynamic> json) {
    return UserModel(
      id: json['id'],
      name: json['name'],
      email: json['email'],
    );
  }

  // Mapping to JSON
  Map<String, dynamic> toJson() {
    return {
      'id': id,
      'name': name,
      'email': email,
    };
  }
}
```

### Domain Entity (User)

This is the **User** entity in the **domain** layer that represents the core business logic data. It is decoupled from data layer structures like JSON or HTTP responses.

```dart
// lib/features/domain/entities/user.dart

import 'package:equatable/equatable.dart';

class User extends Equatable {
  final int id;
  final String name;
  final String email;

  const User({
    required this.id,
    required this.name,
    required this.email,
  });

  @override
  List<Object> get props => [id, name, email];
}
```

### Summary of Key Points:
1. **Repository Interface**: `UserRepository` defines a contract for fetching a user by ID.
2. **Repository Implementation**: `UserRepositoryImpl` decides whether to fetch the data from the remote or local data source based on network availability.
3. **Remote Data Source**: Handles API calls to get user data and throws exceptions on errors.
4. **Local Data Source**: Caches user data for offline use and retrieves cached data if the network is unavailable.
5. **UserModel**: Handles the mapping between JSON data and the `User` entity.
6. **User Entity**: Represents the user in the domain layer, independent of the data source specifics.

This structure allows you to decouple the domain layer from the details of the data sources, making the code more modular, testable, and maintainable.
