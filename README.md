# promodoro_tracking_app

A new Flutter project.

## Getting Started

This project is a starting point for a Flutter application.

A few resources to get you started if this is your first Flutter project:

- [Lab: Write your first Flutter app](https://docs.flutter.dev/get-started/codelab)
- [Cookbook: Useful Flutter samples](https://docs.flutter.dev/cookbook)

For help getting started with Flutter development, view the
[online documentation](https://docs.flutter.dev/), which offers tutorials,
samples, guidance on mobile development, and a full API reference.
--------
Project Strcuture (Update)
lib/
├── core/
│   ├── error/
│   │   ├── exceptions.dart
│   │   └── failures.dart
│   ├── repositories/
│   │   └── repository_name.dart
│   ├── usecases/
│   │   └── usecase.dart
│   ├── utils/
│   │   └── input_converter.dart
│   └── network/
│       ├── network_info.dart
│       └── api_client.dart
├── features/
│   ├── data/
│   │   ├── datasources/
│   │   │   ├── local_data_source.dart
│   │   │   └── remote_data_source.dart
│   │   ├── models/
│   │   │   └── model_name.dart
│   ├── domain/
│   │   ├── entities/
│   │   │   └── entity_name.dart
│   │   ├── usecases/
│   │       └── usecase_name.dart
│   ├── presentation/
│       ├── providers/
│       │   └── user_provider.dart
│       ├── state/                # New folder for state management
│       │   └── user_state.dart   # State class or models
│       ├── pages/
│       │   └── feature_page.dart
│       └── widgets/
│           └── feature_widget.dart
├── injection_container.dart
└── main.dart
