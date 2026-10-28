# Практична робота 8 — Варіант 1
## Моніторинг розумної теплиці

> Цей Markdown-файл містить опис роботи та повні текстові лістинги програми.
> Його можна додати до репозиторію як `README.md`. Для запуску потрібні окремі
> файли Flutter-проєкту: сам файл `.md` не є виконуваною програмою.
> Готовий проєкт із платформами, зображеннями та 4 комітами є в архіві
> `Praktychna8_Variant1_Greenhouse.zip`, наданому разом із роботою.

Дисципліна: **Програмування для мобільних платформ**.
Виконавець: ____________________.

Flutter-застосунок імітує панель теплиці з температурою, вологістю та освітленістю.
Справжні HTTP-запити й фізичні датчики не використовуються.

## Запуск

Перевірено на **Flutter 3.35.5 / Dart 3.9.2**. Потрібні Flutter SDK і браузер Chrome
або Android SDK з емулятором/телефоном.

```bash
flutter pub get
flutter run -d chrome
```

Для Android:

```bash
flutter devices
flutter run -d DEVICE_ID
```

Проєкт містить платформи `android/` та `web/`. У цьому середовищі підтверджено
вебкомпіляцію й запуск віджетів у Flutter Test; APK та фізичний телефон не перевірялися.

## Можливості

- Список сенсорів, пороги та підсумок доби завантажуються паралельно з одним індикатором.
- Граничний час початкового завантаження — 3 секунди.
- Три потоки показників працюють із періодом 1,5 секунди.
- Поза діапазоном картка змінює колір, з’являється SnackBar.
- Повторне сповіщення для того самого датчика можливе не частіше ніж раз на 10 секунд.
- «Пауза» скасовує таймери джерел. Показники залишаються на екрані.
- «Продовжити» створює таймери знову; новий показник надходить через 1,5 секунди.
- Збій одного сенсора показується тільки на його картці. Потік не закривається.
- Для помилок і порожніх даних доступна кнопка «Повторити».
- Є окремий початковий екран, щоб перевірити навігацію назад під час оновлень.

## Архітектура й відповідність вимогам

| Вимога | Реалізація |
|---|---|
| Моделі в окремому файлі | `lib/models/greenhouse.dart` |
| Окремий сервіс імітації | `lib/services/fake_api.dart`, `request()` з `Future.delayed` |
| Параметр частоти збоїв | `FakeApi.failureRate`, значення 0 / 20 / 50 / 100 % в UI |
| Паралельні запити | `FakeApi.loadDashboard()`: `Future.wait<Object>` |
| Спільний тайм-аут | `.timeout(const Duration(seconds: 3))` у `loadDashboard()` |
| FutureBuilder | `DashboardScreen` |
| Future не створюється в build | `_prepare()` викликається з `initState` та обробників дій |
| StreamBuilder | `SensorCard`, окремий для кожного датчика |
| Власний контролер | `SensorFeed`: `StreamController<double?>.broadcast()` |
| Подія помилки | `SensorFeed.sample()`: `_controller.addError(error, stack)` |
| Незалежні датчики | Окремий `SensorFeed` на кожний сенсор |
| Реальна пауза | `SensorFeed.pause()`: `Timer.cancel()` |
| Звільнення ресурсів | `_LiveDashboardState.dispose()`: cancel підписок і `feed.dispose()` |
| Закриття контролера | `SensorFeed.dispose()`: `close()` після скасування таймера |
| Обмеження сповіщень | `AlertThrottle`, окрема позначка часу для кожного sensorId |

`FutureBuilder` сам керує доставкою результату асинхронного завантаження віджету.
Методи State у цій реалізації **не містять await**, тому немає переходу через await,
після якого необхідна перевірка mounted. У callback власної підписки перед зверненням
до контексту перевіряється `mounted`. Методи сервісу не мають BuildContext.

На один broadcast-потік підписані `StreamBuilder` картки й слухач для SnackBar.
У слухачі є `onError`, а `cancelOnError` залишається false. Помилка не розриває
потік; наступний успішний показник автоматично відновлює картку.

`StreamBuilder` обробляє чотири ситуації: очікування першої події, помилку,
`null` як відсутність показника та числові дані. Порожній список сенсорів
обробляється окремо в `FutureBuilder`.

`.timeout` не скасовує вихідні Future. Запізнілі результати сервісу не мають
побічних ефектів і не викликають setState. Під час повторного завантаження старий
LiveDashboard видаляється разом із потоками; новий FutureBuilder має новий ключ.

## Сценарій демонстрації

1. Відкрити теплицю. Показати спільний індикатор трьох запитів, потім показники.
2. Вибрати «Помилка сервера». Показати «Повторити». Повернути «Звичайний режим»
   для успішного повторного завантаження.
3. Вибрати «Тайм-аут 3 с»: сервіс відповідає через 4 с, екран повідомляє про тайм-аут.
4. «Порожній список» показує заглушку без сенсорів; «Немає показників» показує
   заглушки потоків на трьох картках.
5. «Збій температури»: температура недоступна, вологість і освітленість оновлюються.
   Повтор у примусово аварійному режимі також завершиться помилкою — це очікувано.
6. «Перевищення порога»: температура дорівнює 36 °C за максимуму 30 °C.
   Показати підсвічування та обмеження частоти сповіщень.
7. Натиснути «Пауза», почекати 5 с: показники незмінні. Відновити оновлення.
8. Вийти назад під час оновлень, відкрити екран повторно. Перевірити консоль.
9. Установити 50 % збоїв. Частота застосовується незалежно до кожного початкового
   запиту та показника. Успіх усіх трьох початкових запитів становить близько 12,5 %,
   тому можуть знадобитися повтори. Для надійної демонстрації локального збою
   використовуйте режим «Збій температури» з частотою 0 %.

## Скриншоти станів

Зображення зберігаються в каталозі `docs/` повного проєкту. Цей окремий текстовий файл їх не містить.

Це знімки **реальних Flutter-віджетів**, отримані через golden-рендеринг Flutter Test
на поверхні 900×1000. Вони не є макетами й не видаються за скриншоти фізичного телефона.
Для читабельного українського тексту тест завантажує DejaVu Sans; ліцензія в
`test/assets/FONT_LICENSE.txt`. У звичайному застосунку використовується системний шрифт.

### Завантаження
Файл знімка в повному проєкті: `docs/loading.png` (Завантаження).

### Помилка з повтором
Файл знімка в повному проєкті: `docs/error.png` (Помилка).

### Порожній список
Файл знімка в повному проєкті: `docs/empty.png` (Порожні дані).

### Отримані показники
Файл знімка в повному проєкті: `docs/data.png` (Дані).

### Оновлення та перевищення порога
Файл знімка в повному проєкті: `docs/live.png` (Оновлення).

### Збій одного сенсора
Файл знімка в повному проєкті: `docs/sensor_error.png` (Збій датчика).

### Відсутність показників у потоках
Файл знімка в повному проєкті: `docs/no_readings.png` (Порожні потоки).

Відтворення знімків:

```bash
flutter test --update-goldens tool/capture_screens.dart
```

Числові значення в режимі нормальної роботи випадкові й можуть відрізнятися.
Статичне зображення показує кадр оновлення; безперервність роботи підтверджується
запуском застосунку та тестами таймерів.

## Перевірки

```bash
flutter analyze
flutter test
flutter build web --release
```

Фактично виконано 27.10.2026:

- `flutter analyze`: **No issues found**.
- `flutter test`: **9 тестів пройшли**.
- Генерація скриншотів: **7 сценаріїв пройшли**.
- `flutter build web --release`: **успішно**.

Логи: `docs/analyze.txt`, `docs/tests.txt`, `docs/screenshots_test.txt`, `docs/build_web.txt`.
Тести перевіряють паралельність (1100 мс віртуального часу замість 2700 мс послідовно),
тайм-аут, відновлення після addError, зупинення джерела на паузі, закриття потоку,
10-секундний інтервал сповіщень, стани екрана, незалежність сенсорів і вихід назад.
Віртуальний час тесту не є вимірюванням швидкості реальної мережі.

У `.github/workflows/flutter.yml` є автоматична перевірка аналізатора, тестів і вебзбірки.
Каталог `build/` не входить до репозиторію.

## Здача в GitHub

Архів містить `.git` із осмисленою історією комітів. Розпакуйте його повністю,
відкрийте термінал у каталозі проєкту й перевірте:

```bash
git log --oneline
```

Створіть порожній репозиторій GitHub без README, ліцензії та .gitignore. Потім:

```bash
git remote add origin https://github.com/YOUR_USERNAME/greenhouse-async.git
git push -u origin main
```

Замість YOUR_USERNAME вкажіть свій логін. Не виконуйте git init повторно й не
обмежуйтеся завантаженням файлів через вебформу: важливо зберегти історію комітів.
У Classroom здається **посилання на створений репозиторій**, а не ZIP.

## Джерела

- https://dart.dev/libraries/async/async-await
- https://dart.dev/libraries/async/creating-streams
- https://api.flutter.dev/flutter/widgets/FutureBuilder-class.html
- https://api.flutter.dev/flutter/widgets/StreamBuilder-class.html
- https://api.dart.dev/dart-async/StreamController-class.html


## Повний код і файли налаштувань

Нижче наведено вміст файлів без скорочень. Назва над кожним блоком указує,
де цей файл має лежати у Flutter-проєкті. Код у Markdown призначений для
читання та копіювання; завантаження лише README не створює файли автоматично.

Якщо відновлюєте програму з цих лістингів замість готового архіву:

1. Створіть основу командою `flutter create --platforms=android,web --org ua.edu.iot --project-name greenhouse_async greenhouse_async`.
2. Перейдіть у `greenhouse_async` та замініть `pubspec.yaml` і `lib/main.dart` наведеним вмістом.
3. Створіть інші файли за вказаними шляхами. Видаліть стандартний тест лічильника `test/widget_test.dart`, оскільки він не стосується цієї програми.
4. Виконайте `flutter pub get`, `flutter analyze`, `flutter test` та `flutter run -d chrome`.

Цей спосіб не відновлює попередні коміти та бінарні файли скриншотів.
Для здачі з готовою історією Git скористайтеся повним архівом.

### `pubspec.yaml`

```yaml
name: greenhouse_async
description: 'Практична 8. Варіант 1: моніторинг розумної теплиці.'
publish_to: none
version: 1.0.0+1
environment:
  sdk: ^3.9.0
dependencies:
  cupertino_icons: ^1.0.8
  flutter:
    sdk: flutter
dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^5.0.0
  fake_async: ^1.3.3
flutter:
  uses-material-design: true
```

### `.gitignore`

```gitignore
# Miscellaneous
*.class
*.log
*.pyc
*.swp
.DS_Store
.atom/
.build/
.buildlog/
.history
.svn/
.swiftpm/
migrate_working_dir/

# IntelliJ related
*.iml
*.ipr
*.iws
.idea/

# The .vscode folder contains launch configuration and tasks you configure in
# VS Code which you may wish to be included in version control, so this line
# is commented out by default.
#.vscode/

# Flutter/Dart/Pub related
**/doc/api/
**/ios/Flutter/.last_build_id
.dart_tool/
.flutter-plugins-dependencies
.pub-cache/
.pub/
/build/
/coverage/

# Symbolication related
app.*.symbols

# Obfuscation related
app.*.map.json

# Android Studio will place build artifacts here
/android/app/debug
/android/app/profile
/android/app/release
```

### `analysis_options.yaml`

```yaml
# This file configures the analyzer, which statically analyzes Dart code to
# check for errors, warnings, and lints.
#
# The issues identified by the analyzer are surfaced in the UI of Dart-enabled
# IDEs (https://dart.dev/tools#ides-and-editors). The analyzer can also be
# invoked from the command line by running `flutter analyze`.

# The following line activates a set of recommended lints for Flutter apps,
# packages, and plugins designed to encourage good coding practices.
include: package:flutter_lints/flutter.yaml

linter:
  # The lint rules applied to this project can be customized in the
  # section below to disable rules from the `package:flutter_lints/flutter.yaml`
  # included above or to enable additional rules. A list of all available lints
  # and their documentation is published at https://dart.dev/lints.
  #
  # Instead of disabling a lint rule for the entire project in the
  # section below, it can also be suppressed for a single line of code
  # or a specific dart file by using the `// ignore: name_of_lint` and
  # `// ignore_for_file: name_of_lint` syntax on the line or in the file
  # producing the lint.
  rules:
    # avoid_print: false  # Uncomment to disable the `avoid_print` rule
    # prefer_single_quotes: true  # Uncomment to enable the `prefer_single_quotes` rule

# Additional information about this file can be found at
# https://dart.dev/guides/language/analysis-options
```

### `lib/models/greenhouse.dart`

```dart
enum DemoMode {
  normal,
  loadError,
  timeout,
  empty,
  sensorError,
  threshold,
  noReadings,
}

class Sensor {
  const Sensor(this.id, this.name, this.unit, this.baseline, this.spread);
  final String id, name, unit;
  final double baseline, spread;
}

class Limits {
  const Limits(this.min, this.max);
  final double min, max;
  bool outside(double value) => value < min || value > max;
}

class DailySummary {
  const DailySummary(this.averageTemperature, this.waterLiters);
  final double averageTemperature, waterLiters;
}

class DashboardData {
  const DashboardData(this.sensors, this.limits, this.summary);
  final List<Sensor> sensors;
  final Map<String, Limits> limits;
  final DailySummary summary;
}

class SensorFailure implements Exception {
  const SensorFailure(this.sensorId);
  final String sensorId;
  @override
  String toString() => 'Датчик тимчасово недоступний';
}
```

### `lib/services/fake_api.dart`

```dart
import 'dart:async';
import 'dart:math';
import '../models/greenhouse.dart';

class FakeApi {
  FakeApi({this.failureRate = 0, this.mode = DemoMode.normal, Random? random})
    : assert(failureRate >= 0 && failureRate <= 1),
      _random = random ?? Random();
  final double failureRate;
  final DemoMode mode;
  final Random _random;
  Future<T> request<T>(T Function() respond, {int delayMs = 800}) async {
    await Future<void>.delayed(
      Duration(milliseconds: mode == DemoMode.timeout ? 4000 : delayMs),
    );
    if (mode == DemoMode.loadError || _random.nextDouble() < failureRate) {
      throw StateError('Сервер недоступний. Спробуйте ще раз.');
    }
    return respond();
  }

  Future<List<Sensor>> sensors() => request(
    () => mode == DemoMode.empty
        ? <Sensor>[]
        : const [
            Sensor('temperature', 'Температура', '°C', 25, 6),
            Sensor('humidity', 'Вологість', '%', 60, 20),
            Sensor('light', 'Освітленість', 'лк', 14000, 6500),
          ],
    delayMs: 700,
  );
  Future<Map<String, Limits>> thresholds() => request(
    () => const {
      'temperature': Limits(18, 30),
      'humidity': Limits(40, 75),
      'light': Limits(8000, 20000),
    },
    delayMs: 900,
  );
  Future<DailySummary> dailySummary() =>
      request(() => const DailySummary(24.3, 12.5), delayMs: 1100);
  Future<DashboardData> loadDashboard() async {
    // The timeout covers all three parallel requests.
    final values = await Future.wait<Object>([
      sensors(),
      thresholds(),
      dailySummary(),
    ]).timeout(const Duration(seconds: 3));
    return DashboardData(
      values[0] as List<Sensor>,
      values[1] as Map<String, Limits>,
      values[2] as DailySummary,
    );
  }

  double? reading(Sensor sensor) {
    if (mode == DemoMode.noReadings) return null;
    if ((mode == DemoMode.sensorError && sensor.id == 'temperature') ||
        _random.nextDouble() < failureRate) {
      throw SensorFailure(sensor.id);
    }
    if (mode == DemoMode.threshold && sensor.id == 'temperature') return 36;
    return sensor.baseline + (_random.nextDouble() * 2 - 1) * sensor.spread;
  }
}
```

### `lib/services/sensor_feed.dart`

```dart
import 'dart:async';
import '../models/greenhouse.dart';
import 'fake_api.dart';

class SensorFeed {
  SensorFeed(this.sensor, this.api);
  final Sensor sensor;
  final FakeApi api;
  final _controller = StreamController<double?>.broadcast();
  Timer? _timer;
  bool _disposed = false;
  Stream<double?> get stream => _controller.stream;
  bool get running => _timer != null;
  void start() {
    if (_disposed || _timer != null) return;
    _timer = Timer.periodic(
      const Duration(milliseconds: 1500),
      (_) => sample(),
    );
  }

  void sample() {
    if (_disposed) return;
    try {
      _controller.add(api.reading(sensor));
    } catch (error, stack) {
      _controller.addError(error, stack);
    }
  }

  void pause() {
    _timer?.cancel();
    _timer = null;
  }

  void dispose() {
    if (_disposed) return;
    _disposed = true;
    pause();
    unawaited(_controller.close());
  }
}

class AlertThrottle {
  final _last = <String, DateTime>{};
  bool allow(String sensorId, DateTime now) {
    final previous = _last[sensorId];
    if (previous != null &&
        now.difference(previous) < const Duration(seconds: 10)) {
      return false;
    }
    _last[sensorId] = now;
    return true;
  }
}
```

### `lib/main.dart`

```dart
import 'package:flutter/material.dart';
import 'screens/dashboard_screen.dart';

void main() => runApp(const GreenhouseApp());

class GreenhouseApp extends StatelessWidget {
  const GreenhouseApp({super.key});
  @override
  Widget build(BuildContext context) => MaterialApp(
    title: 'Розумна теплиця',
    debugShowCheckedModeBanner: false,
    theme: ThemeData(
      colorScheme: ColorScheme.fromSeed(seedColor: const Color(0xFF20664C)),
      scaffoldBackgroundColor: const Color(0xFFF3F6F2),
      useMaterial3: true,
    ),
    home: const HomeScreen(),
  );
}

class HomeScreen extends StatelessWidget {
  const HomeScreen({super.key});
  @override
  Widget build(BuildContext context) => Scaffold(
    appBar: AppBar(title: const Text('Практична 8 • Варіант 1')),
    body: Center(
      child: FilledButton.icon(
        icon: const Icon(Icons.eco_outlined),
        label: const Text('Відкрити теплицю'),
        onPressed: () => Navigator.of(context).push(
          MaterialPageRoute<void>(builder: (_) => const DashboardScreen()),
        ),
      ),
    ),
  );
}
```

### `lib/screens/dashboard_screen.dart`

```dart
import 'dart:async';
import 'package:flutter/material.dart';
import '../models/greenhouse.dart';
import '../services/fake_api.dart';
import '../services/sensor_feed.dart';

class DashboardScreen extends StatefulWidget {
  const DashboardScreen({super.key, this.initialMode = DemoMode.normal});
  final DemoMode initialMode;
  @override
  State<DashboardScreen> createState() => _DashboardScreenState();
}

class _DashboardScreenState extends State<DashboardScreen> {
  late DemoMode _mode;
  double _failureRate = 0;
  late FakeApi _api;
  late Future<DashboardData> _future;
  int _generation = 0;
  @override
  void initState() {
    super.initState();
    _mode = widget.initialMode;
    _prepare();
  }

  void _prepare() {
    _api = FakeApi(failureRate: _failureRate, mode: _mode);
    _future = _api.loadDashboard();
    _generation++;
  }

  void _reload() => setState(_prepare);
  String _label(DemoMode mode) => switch (mode) {
    DemoMode.normal => 'Звичайний режим',
    DemoMode.loadError => 'Помилка сервера',
    DemoMode.timeout => 'Тайм-аут 3 с',
    DemoMode.empty => 'Порожній список',
    DemoMode.sensorError => 'Збій температури',
    DemoMode.threshold => 'Перевищення порога',
    DemoMode.noReadings => 'Немає показників',
  };
  @override
  Widget build(BuildContext context) => Scaffold(
    appBar: AppBar(
      title: const Text('Розумна теплиця'),
      actions: [
        IconButton(
          onPressed: _reload,
          tooltip: 'Оновити',
          icon: const Icon(Icons.refresh),
        ),
      ],
    ),
    body: Column(
      children: [
        Padding(
          padding: const EdgeInsets.all(12),
          child: Wrap(
            spacing: 20,
            runSpacing: 8,
            crossAxisAlignment: WrapCrossAlignment.center,
            children: [
              DropdownButton<DemoMode>(
                value: _mode,
                items: DemoMode.values
                    .map(
                      (mode) => DropdownMenuItem(
                        value: mode,
                        child: Text(_label(mode)),
                      ),
                    )
                    .toList(),
                onChanged: (mode) {
                  if (mode == null) return;
                  setState(() {
                    _mode = mode;
                    _prepare();
                  });
                },
              ),
              Row(
                mainAxisSize: MainAxisSize.min,
                children: [
                  const Text('Частота збоїв: '),
                  DropdownButton<double>(
                    value: _failureRate,
                    items: [0.0, 0.2, 0.5, 1.0]
                        .map(
                          (value) => DropdownMenuItem(
                            value: value,
                            child: Text('${(value * 100).round()} %'),
                          ),
                        )
                        .toList(),
                    onChanged: (value) {
                      if (value == null) return;
                      setState(() {
                        _failureRate = value;
                        _prepare();
                      });
                    },
                  ),
                ],
              ),
            ],
          ),
        ),
        Expanded(
          child: FutureBuilder<DashboardData>(
            key: ValueKey(_generation),
            future: _future,
            builder: (context, snapshot) {
              if (snapshot.connectionState != ConnectionState.done) {
                return const Center(
                  child: Column(
                    mainAxisSize: MainAxisSize.min,
                    children: [
                      CircularProgressIndicator(),
                      SizedBox(height: 16),
                      Text('Завантаження датчиків, порогів і підсумку…'),
                    ],
                  ),
                );
              }
              if (snapshot.hasError) {
                final message = snapshot.error is TimeoutException
                    ? 'Сервер не відповів за 3 секунди.'
                    : 'Не вдалося завантажити дані.';
                return EmptyPanel(
                  icon: Icons.cloud_off,
                  message: message,
                  onRetry: _reload,
                );
              }
              final data = snapshot.data;
              if (data == null || data.sensors.isEmpty) {
                return EmptyPanel(
                  icon: Icons.eco_outlined,
                  message: 'Датчиків поки немає.',
                  onRetry: _reload,
                );
              }
              return LiveDashboard(
                key: ValueKey(_generation),
                data: data,
                api: _api,
              );
            },
          ),
        ),
      ],
    ),
  );
}

class EmptyPanel extends StatelessWidget {
  const EmptyPanel({
    super.key,
    required this.icon,
    required this.message,
    required this.onRetry,
  });
  final IconData icon;
  final String message;
  final VoidCallback onRetry;
  @override
  Widget build(BuildContext context) => Center(
    child: Column(
      mainAxisSize: MainAxisSize.min,
      children: [
        Icon(icon, size: 48),
        const SizedBox(height: 16),
        Text(message, textAlign: TextAlign.center),
        const SizedBox(height: 16),
        FilledButton(onPressed: onRetry, child: const Text('Повторити')),
      ],
    ),
  );
}

class LiveDashboard extends StatefulWidget {
  const LiveDashboard({super.key, required this.data, required this.api});
  final DashboardData data;
  final FakeApi api;
  @override
  State<LiveDashboard> createState() => _LiveDashboardState();
}

class _LiveDashboardState extends State<LiveDashboard> {
  final _feeds = <SensorFeed>[];
  final _subscriptions = <StreamSubscription<double?>>[];
  final _throttle = AlertThrottle();
  bool _paused = false;
  @override
  void initState() {
    super.initState();
    for (final sensor in widget.data.sensors) {
      final feed = SensorFeed(sensor, widget.api);
      _feeds.add(feed);
      _subscriptions.add(
        feed.stream.listen(
          (value) {
            if (!mounted || value == null) return;
            final limits = widget.data.limits[sensor.id]!;
            if (limits.outside(value) &&
                _throttle.allow(sensor.id, DateTime.now())) {
              ScaffoldMessenger.of(context)
                ..hideCurrentSnackBar()
                ..showSnackBar(
                  SnackBar(
                    content: Text(
                      '${sensor.name}: ${value.toStringAsFixed(1)} ${sensor.unit} — поза нормою',
                    ),
                    duration: const Duration(seconds: 2),
                  ),
                );
            }
          },
          onError: (Object error) {
            /* Card shows the error; keep listening. */
          },
        ),
      );
      feed.start();
    }
  }

  void _togglePause() => setState(() {
    _paused = !_paused;
    for (final feed in _feeds) {
      if (_paused) {
        feed.pause();
      } else {
        feed.start();
      }
    }
  });
  @override
  void dispose() {
    for (final subscription in _subscriptions) {
      unawaited(subscription.cancel());
    }
    for (final feed in _feeds) {
      feed.dispose();
    }
    super.dispose();
  }

  @override
  Widget build(BuildContext context) => ListView(
    padding: const EdgeInsets.all(16),
    children: [
      Text(
        'Зона 01 • Мікроклімат',
        style: Theme.of(context).textTheme.headlineSmall,
      ),
      const SizedBox(height: 8),
      Text(
        'За добу: середня температура ${widget.data.summary.averageTemperature} °C; полив ${widget.data.summary.waterLiters} л.',
      ),
      const SizedBox(height: 12),
      Align(
        alignment: Alignment.centerLeft,
        child: FilledButton.icon(
          key: const Key('pause'),
          onPressed: _togglePause,
          icon: Icon(_paused ? Icons.play_arrow : Icons.pause),
          label: Text(_paused ? 'Продовжити' : 'Пауза'),
        ),
      ),
      const SizedBox(height: 8),
      Text(
        _paused
            ? 'Оновлення зупинено. Таймери скасовано.'
            : 'Оновлення кожні 1,5 с',
      ),
      const SizedBox(height: 12),
      for (final feed in _feeds)
        SensorCard(
          feed: feed,
          limits: widget.data.limits[feed.sensor.id]!,
          paused: _paused,
        ),
    ],
  );
}

class SensorCard extends StatelessWidget {
  const SensorCard({
    super.key,
    required this.feed,
    required this.limits,
    required this.paused,
  });
  final SensorFeed feed;
  final Limits limits;
  final bool paused;
  @override
  Widget build(BuildContext context) => StreamBuilder<double?>(
    stream: feed.stream,
    builder: (context, snapshot) {
      final value = snapshot.data;
      final alert =
          !snapshot.hasError && value != null && limits.outside(value);
      return Card(
        key: ValueKey(feed.sensor.id),
        color: alert ? const Color(0xFFFFE1DC) : null,
        child: Padding(
          padding: const EdgeInsets.all(16),
          child: Column(
            crossAxisAlignment: CrossAxisAlignment.start,
            children: [
              Text(
                feed.sensor.name,
                style: Theme.of(context).textTheme.titleLarge,
              ),
              Text(
                'Норма: ${limits.min.toStringAsFixed(0)}–${limits.max.toStringAsFixed(0)} ${feed.sensor.unit}',
              ),
              const SizedBox(height: 10),
              if (snapshot.hasError) ...[
                Text(
                  '${snapshot.error}',
                  style: TextStyle(color: Theme.of(context).colorScheme.error),
                ),
                TextButton(
                  onPressed: paused ? null : feed.sample,
                  child: const Text('Повторити'),
                ),
              ] else if (snapshot.connectionState ==
                  ConnectionState.waiting) ...[
                const LinearProgressIndicator(),
                const Text('Очікування першого показника…'),
              ] else if (value == null) ...[
                const Text('Показників поки немає.'),
                TextButton(
                  onPressed: paused ? null : feed.sample,
                  child: const Text('Повторити'),
                ),
              ] else ...[
                Text(
                  '${value.toStringAsFixed(feed.sensor.id == 'light' ? 0 : 1)} ${feed.sensor.unit}',
                  key: ValueKey('reading-${feed.sensor.id}'),
                  style: Theme.of(context).textTheme.headlineMedium,
                ),
                Text(alert ? 'Увага: вихід за поріг' : 'У межах норми'),
              ],
            ],
          ),
        ),
      );
    },
  );
}
```

### `test/async_test.dart`

```dart
import 'dart:async';
import 'dart:math';
import 'package:fake_async/fake_async.dart';
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:greenhouse_async/main.dart';
import 'package:greenhouse_async/models/greenhouse.dart';
import 'package:greenhouse_async/screens/dashboard_screen.dart';
import 'package:greenhouse_async/services/fake_api.dart';
import 'package:greenhouse_async/services/sensor_feed.dart';

class RecoveringApi extends FakeApi {
  bool fail = true;
  @override
  double? reading(Sensor sensor) {
    if (fail) throw SensorFailure(sensor.id);
    return 25;
  }
}

void main() {
  test('three requests finish in 1100ms rather than 2700ms', () {
    fakeAsync((time) {
      DashboardData? data;
      FakeApi(random: Random(1)).loadDashboard().then((value) => data = value);
      time.elapse(const Duration(milliseconds: 1099));
      expect(data, isNull);
      time.elapse(const Duration(milliseconds: 1));
      expect(data!.sensors.length, 3);
    });
  });
  test('combined load times out after three seconds', () {
    fakeAsync((time) {
      Object? error;
      FakeApi(mode: DemoMode.timeout).loadDashboard().then<void>(
        (_) {},
        onError: (Object e) {
          error = e;
        },
      );
      time.elapse(const Duration(seconds: 3));
      expect(error, isA<TimeoutException>());
      time.elapse(const Duration(seconds: 2));
    });
  });
  test(
    'error event does not close stream; pause stops source; dispose stops it',
    () {
      fakeAsync((time) {
        final api = RecoveringApi();
        final feed = SensorFeed(
          const Sensor('temperature', 'T', 'C', 25, 1),
          api,
        );
        var errors = 0;
        var readings = 0;
        var done = false;
        final sub = feed.stream.listen(
          (_) => readings++,
          onError: (Object _) => errors++,
          onDone: () => done = true,
        );
        feed.start();
        time.elapse(const Duration(milliseconds: 1500));
        expect(errors, 1);
        api.fail = false;
        time.elapse(const Duration(milliseconds: 1500));
        expect(readings, 1);
        feed.pause();
        expect(feed.running, isFalse);
        time.elapse(const Duration(seconds: 10));
        expect(readings, 1);
        feed.start();
        time.elapse(const Duration(milliseconds: 1500));
        expect(readings, 2);
        feed.dispose();
        time.flushMicrotasks();
        expect(done, isTrue);
        time.elapse(const Duration(seconds: 10));
        expect(readings, 2);
        unawaited(sub.cancel());
      });
    },
  );
  test('alert limit is independent per sensor and allows again after 10s', () {
    final throttle = AlertThrottle();
    final now = DateTime(2026);
    expect(throttle.allow('t', now), isTrue);
    expect(throttle.allow('t', now.add(const Duration(seconds: 9))), isFalse);
    expect(throttle.allow('h', now), isTrue);
    expect(throttle.allow('t', now.add(const Duration(seconds: 10))), isTrue);
  });
  for (final entry in {
    DemoMode.loadError: 'Не вдалося завантажити дані.',
    DemoMode.empty: 'Датчиків поки немає.',
    DemoMode.timeout: 'Сервер не відповів за 3 секунди.',
  }.entries) {
    testWidgets('load state ${entry.key}', (tester) async {
      await tester.pumpWidget(
        MaterialApp(home: DashboardScreen(initialMode: entry.key)),
      );
      expect(find.byType(CircularProgressIndicator), findsOneWidget);
      await tester.pump(const Duration(seconds: 3));
      await tester.pump();
      expect(find.text(entry.value), findsOneWidget);
      expect(find.text('Повторити'), findsOneWidget);
      await tester.tap(find.text('Повторити'));
      await tester.pump();
      expect(find.byType(CircularProgressIndicator), findsOneWidget);
      await tester.pumpWidget(const SizedBox());
      await tester.pump(const Duration(seconds: 5));
      expect(tester.takeException(), isNull);
    });
  }
  testWidgets(
    'one sensor fails while other cards have readings; exit is safe',
    (tester) async {
      await tester.pumpWidget(
        const MaterialApp(
          home: DashboardScreen(initialMode: DemoMode.sensorError),
        ),
      );
      await tester.pump(const Duration(milliseconds: 1100));
      await tester.pump();
      await tester.pump(const Duration(milliseconds: 1500));
      await tester.pump();
      expect(find.text('Датчик тимчасово недоступний'), findsOneWidget);
      expect(find.byKey(const ValueKey('reading-humidity')), findsOneWidget);
      await tester.pumpWidget(const SizedBox());
      await tester.pump(const Duration(seconds: 20));
      expect(tester.takeException(), isNull);
    },
  );
  testWidgets(
    'pause preserves displayed reading and navigation disposes timers',
    (tester) async {
      await tester.pumpWidget(const GreenhouseApp());
      await tester.tap(find.text('Відкрити теплицю'));
      await tester.pumpAndSettle();
      await tester.pump(const Duration(seconds: 2));
      await tester.pump();
      await tester.pump(const Duration(milliseconds: 1500));
      await tester.pump();
      await tester.tap(find.byKey(const Key('pause')));
      await tester.pump();
      final reading = tester
          .widget<Text>(find.byKey(const ValueKey('reading-temperature')))
          .data;
      await tester.pump(const Duration(seconds: 15));
      expect(
        tester
            .widget<Text>(find.byKey(const ValueKey('reading-temperature')))
            .data,
        reading,
      );
      await tester.tap(find.text('Продовжити'));
      await tester.pump();
      await tester.pageBack();
      await tester.pumpAndSettle();
      await tester.pump(const Duration(seconds: 10));
      expect(tester.takeException(), isNull);
      expect(find.text('Відкрити теплицю'), findsOneWidget);
    },
  );
}
```

### `.github/workflows/flutter.yml`

```yaml
name: Flutter checks
on: [push, pull_request]
jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.35.5'
          channel: stable
          cache: true
      - run: flutter pub get
      - run: flutter analyze
      - run: flutter test
      - run: flutter build web --release
```
