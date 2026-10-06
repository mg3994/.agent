---
name: flutter-basic-setup
description: "Provides standardized boilerplate and configuration templates for a new Flutter project including standard dependencies, dev_dependencies, l10n.yaml, build.yaml (Drift and worker compilation), analysis_options.yaml, and Drift WASM web worker tools and sqlite3.wasm asset."
---

# Flutter Basic Project Setup Skill

Use this skill when setting up a new project with the project's standard architecture, dependencies, localization settings, build-runner configurations, and Drift WASM web worker tools and `sqlite3.wasm`.

## 1. Dependencies & Dev Dependencies (`pubspec.yaml`)

Add the following to `pubspec.yaml`:

```yaml
dependencies:
  flutter:
    sdk: flutter
  flutter_localizations:
    sdk: flutter
  intl: ^0.20.3
  bloc_signals_flutter: ^1.3.2
  drift: ^2.35.1
  drift_flutter: ^0.3.1
  kaisel: ^1.1.0
  path_provider: ^2.1.6

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^6.0.0
  build: ^4.0.7
  build_runner: ^2.15.1
  build_web_compilers: ^4.8.5
  drift_dev: ^2.35.1
  kaisel_lint: ^0.5.1
```

## 2. Localization Configuration (`l10n.yaml`)

Create `l10n.yaml` in the project root:

```yaml
arb-dir: lib/l10n
output-dir: lib/l10n
template-arb-file: app_en.arb
output-localization-file: app_localizations.dart
output-class: AppLocalizations
nullable-getters: false
untranslated-messages-file: lib/l10n/untranslated.json
use-escaping: false
use-deferred-loading: true
relax-syntax: true
required-resource-attributes: false
preferred-supported-locales:
  - en
```

And create `lib/l10n/app_en.arb`:
```json
{
    "@@locale":"en"
}
```

## 3. Build & Drift Configuration (`build.yaml`)

Create `build.yaml` in the project root:

```yaml
targets:
  $default:
    sources:
      - lib/**
      - web/**
      - tools/**
      - $package$
      - lib/$lib$
      - pubspec.yaml
      - "!build/**"
      - "!**/build/**"
      - "!**/generated/**"
    builders:
      drift_dev:
        options:
          databases:
            default: lib/src/storage/drift/app_database.dart
          sql:
            dialect: sqlite
            options:
              version: "3.38"
              modules:
                - fts5
      build_web_compilers:entrypoint:
        generate_for:
          - tools/drift_worker.dart
        options:
          compiler: dart2js
      ":copy_compiled_worker_js":
        enabled: true

builders:
  copy_compiled_worker_js:
    import: "tools/builder.dart"
    builder_factories: ["CopyCompiledJs.new"]
    required_inputs:
      - .js
    build_to: source
    build_extensions:
      "tools/drift_worker.dart": ["web/drift_worker.js"]
```

## 4. Linting Configuration (`analysis_options.yaml`)

Configure `analysis_options.yaml`:

```yaml
extensions:
  - drift: true

include:
  - package:flutter_lints/flutter.yaml
  - package:kaisel_lint/recommended.yaml

plugins:
  kaisel_lint:
    version: ^0.5.1
    diagnostics:
      prefer_const_route_constructors: true
      prefer_pattern_match_over_is_check: true
      unused_guard_redirect: true
      prefer_push_or_replace_top_in_adaptive: false
```

## 5. Drift WASM Web Worker, Custom Builder (`tools/`) & `sqlite3.wasm` (`web/`)

- Ensure `web/sqlite3.wasm` is present in the `web/` directory for SQLite WebAssembly support in Drift.
- `tools/builder.dart`:
```dart
import 'package:build/build.dart';

class CopyCompiledJs extends Builder {
  // ignore: unnecessary_type_name_in_constructor, avoid_unused_constructor_parameters
  CopyCompiledJs([BuilderOptions? options]);

  @override
  Future<void> build(BuildStep buildStep) async {
    final inputId = buildStep.inputId;
    final outputId = buildStep.allowedOutputs.single;
    final compiledId = AssetId(inputId.package, '${inputId.path}.js');

    final compiledWorker = await buildStep.readAsBytes(compiledId);
    await buildStep.writeAsBytes(outputId, compiledWorker);
  }

  @override
  Map<String, List<String>> get buildExtensions => {
    'tools/drift_worker.dart': ['web/drift_worker.js'],
  };
}
```

- `tools/drift_worker.dart`:
```dart
import 'package:drift/wasm.dart';

/// This Dart program is the entrypoint of a web worker that will be compiled to
/// JavaScript by running `build_runner build`.
void main() {
  return WasmDatabase.workerMainForOpen();
}
```
