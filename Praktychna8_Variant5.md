# Практична робота 8 — Варіант 5
## Фітнес-трекер тренування

> Окремий файл Markdown містить опис і повний код програми. Для запуску потрібні
> файли Flutter-проєкту з архіву `Praktychna8_Variant5_Fitness.zip`.
> Markdown сам по собі не є виконуваним застосунком. Зображення та історія Git
> входять до архіву, а не до цього текстового файла.

**Дисципліна:** Програмування для мобільних платформ.  
**Виконавець:** ____________________.

Застосунок показує план тренування, секундомір, імітований пульс і статистику.
Після завершення результат передається сервісу-імітації. Справжньої мережі та
медичного вимірювання пульсу немає.

## Запуск

Рекомендоване середовище для відтворення: Flutter 3.35.5 / Dart 3.9.2.

```bash
flutter pub get
flutter run -d chrome
```

Для Android: `flutter devices`, потім `flutter run -d DEVICE_ID`.
У репозиторії є `android/` та `web/`; перевірки виконуються для вебзбірки й Flutter Test.
Запуск на фізичному телефоні та APK окремо не перевірялися.

## Реалізовані можливості

1. План, профіль та три попередні тренування завантажуються паралельно.
2. Спільний індикатор завантаження, повідомлення про помилку, кнопка «Повторити» й порожній стан.
3. Кнопка «Почати» запускає таймер; щосекунди приходить кадр із часом, пульсом, середнім та максимальним значенням.
4. Поточна вправа визначається за часом: розминка 20 с, присідання 30 с, планка 20 с.
   Після 70 с виводиться прохання завершити тренування. Секундомір працює до натискання «Завершити».
5. «Пауза» скасовує таймер джерела. «Продовжити» відновлює його зі збереженого часу.
6. Після завершення пульс і час більше не змінюються, формується незмінний результат.
7. Збереження має тайм-аут 3 с на спробу та до 3 повторних спроб: максимум 4 запити.
8. Після невдачі результат зберігається у стані екрана, є кнопка «Повторити збереження».
9. Вихід під час активного тренування або за наявності незбереженого результату потребує підтвердження.

## Структура та вимоги

| Частина | Файл і метод |
|---|---|
| Моделі профілю, вправ, кадру і результату | `lib/models/workout.dart` |
| Мережа-імітація | `lib/services/fake_api.dart`: `request()` з `Future.delayed` |
| Частота збоїв | `FakeApi.failureRate`, змінюється в UI: 0, 50, 100 % |
| Паралельні запити | `FakeApi.load()`: `Future.wait<Object>` |
| Початковий тайм-аут | `load()`: 3 секунди на групу запитів |
| FutureBuilder | `WorkoutScreen`, Future створюється в `initState()` або `_reload()`, не в `build()` |
| Власний StreamController | `WorkoutSession._frames` |
| Потік показників | `WorkoutSession.start()`, Timer.periodic кожну секунду |
| StreamBuilder | `WorkoutPanel` |
| Помилка потоку без його закриття | `WorkoutSession`: `addError` |
| Пауза джерела | `WorkoutSession.pause()`: `Timer.cancel()` |
| Тайм-аут і повтори збереження | `FakeApi.save()`: `.timeout`, затримки 300 / 600 / 1200 мс |
| Перевірка mounted | `_save()`, `_confirmExit()`, callback оновлення спроби та post-frame callback |
| Звільнення ресурсів | `WorkoutPanel.dispose()` → `WorkoutSession.dispose()` → cancel таймера і close контролера |
| Підписка на потік | Належить StreamBuilder і скасовується ним при видаленні віджета |
| Захист виходу | `PopScope` та `showDialog` |

Пульс генерується синхронно на кожному такті; він не виконує відкладеного HTTP-запиту.
Один кадр потоку містить і секундомір, і пульс, тому пауза атомарно зупиняє обидва.
Кількість секунд обчислюється за `Timer.tick` із базою, збереженою перед продовженням.
Середнє враховує лише успішні вимірювання; за відсутності вибірки показано 0 і кількість вимірювань 0.

У StreamBuilder є очікування першого кадру, числові дані, відсутність показника та
помилка датчика з повтором. Помилка датчика не зупиняє секундомір. Наступний успішний
кадр відновлює нормальний вигляд. Під час паузи повторне читання також недоступне.

Кнопка повтору датчика повертає екран до очікування чергового такту; сам такт читає
джерело знову. Жодний додатковий таймер не створюється.

## Збереження та його межі

Результат `_pending` створюється один раз. Ручні й автоматичні повтори використовують
той самий id. Сервіс записує прийнятий результат у Map за id, тому повторний успіх
не створює дубліката. Після помилки дані не губляться **доки відкритий екран**.
У цій навчальній версії немає дискового сховища: закриття застосунку або підтверджений
вихід може видалити незбережений результат, про що попереджає діалог.

`.timeout` не скасовує сам Future. Імітований запит повертає результат без запису;
запис у Map виконується лише після успішного завершення обгорнутого Future.
Пізні відповіді після тайм-ауту не створюють записів. Після dispose сервіс перевіряє
прапорець закриття й не виконує подальших спроб або записів.

## Демонстрація

1. Відкрити тренування. Показати індикатор і завантажений план, профіль та історію.
2. До початку вибрати «Помилка завантаження», потім «Порожній план» і «Тайм-аут».
   Для успішного повтору повернути «Звичайний режим» й установити 0 % збоїв.
3. Натиснути «Почати». Спостерігати за часом, пульсом, середнім і максимумом.
4. Натиснути «Пауза», почекати 5 с: час і пульс незмінні. Натиснути «Продовжити».
5. У «Демонстрації збоїв» перевірити помилку датчика й відсутність показників.
6. Установити 50 % збоїв сервісу та завершити тренування. Можливі автоматичні повтори.
   Якщо всі 4 спроби невдалі, натиснути «Повторити збереження». За незалежних збоїв
   50 % імовірність чотирьох невдач поспіль становить 6,25 %, тому помилка не гарантована.
7. Для гарантованої демонстрації невдачі ввімкнути «Примусова помилка збереження»
   **перед завершенням**. Після помилки вимкнути її та повторити збереження при 0 %.
8. Для перевірки тайм-ауту ввімкнути «Повільне збереження»: відповідь через 4 с,
   але кожна спроба обмежена 3 с.
9. Під час тренування натиснути назад. Перевірити «Залишитися», потім «Вийти».
   Після успішного збереження вихід можливий без попередження.

## Знімки екранів

Зображення нижче отримано через рендеринг справжніх віджетів у Flutter Test,
а не на фізичному телефоні. Для українського тексту під час рендерингу використано
DejaVu Sans; ліцензія у `test/assets/FONT_LICENSE.txt`. Пульс є випадковим.

| Стан | Зображення |
|---|---|
| Завантаження | `docs/loading.png` — Завантаження |
| Помилка | `docs/error.png` — Помилка |
| Порожній план | `docs/empty.png` — Порожній план |
| План і профіль | `docs/data.png` — Дані |
| Оновлення наживо | `docs/live.png` — Тренування |
| Пауза | `docs/paused.png` — Пауза |
| Невдале збереження | `docs/save_error.png` — Помилка збереження |
| Збережений результат | `docs/saved.png` — Успішне збереження |

Відтворення знімків:

```bash
flutter test --update-goldens tool/capture_screens.dart
```

## Перевірки

```bash
flutter analyze
flutter test
flutter build web --release
```

Перевірено 28.10.2026: **flutter analyze — без зауважень; 12 тестів пройшли;
8 сценаріїв рендерингу пройшли; release-вебзбірка успішна.**
Окремий тест при failureRate = 0.5 використовує керовану послідовність випадкових
чисел, щоб відтворювано показати чотири невдачі й успішний ручний повтор.

Результати фактичних перевірок: див. `docs/analyze.txt`, `docs/tests.txt`,
`docs/build_web.txt` та `docs/screenshots_test.txt`.
Перелік тестів: паралельність і тайм-аут завантаження; час, статистика та пауза;
відновлення потоку після помилки; повторне збереження без дублювання;
відсутність пізніх записів після тайм-аутів; порожній стан; діалог виходу;
збереження результату після збою та dispose під час запиту.

## Здача

Репозиторій має містити весь проєкт, цей README, docs/ та щонайменше 4 коміти.
`build/`, `.dart_tool/` та локальні параметри SDK не додаються до Git.
Архів надано зі збереженою папкою `.git`. Після розпакування перевірте `git log --oneline`.
Створіть порожній репозиторій GitHub і виконайте:

```bash
git remote add origin https://github.com/YOUR_USERNAME/fitness-async.git
git push -u origin main
```

Замість YOUR_USERNAME вкажіть власний логін. У Classroom здається посилання на репозиторій.
Файл `.md` є описом і лістингом: сам по собі він не запускає застосунок.

## Джерела

- https://dart.dev/libraries/async/async-await
- https://dart.dev/libraries/async/creating-streams
- https://api.flutter.dev/flutter/widgets/FutureBuilder-class.html
- https://api.flutter.dev/flutter/widgets/StreamBuilder-class.html
- https://api.flutter.dev/flutter/widgets/PopScope-class.html


## Повні лістинги

Кожний блок відповідає окремому файлу в проєкті. Для відновлення з тексту
створіть основу `flutter create --platforms=android,web --project-name fitness_async fitness_async`,
замініть pubspec.yaml і lib/main.dart, додайте решту файлів за вказаними шляхами.
Видаліть стандартний test/widget_test.dart із тестом лічильника.
Після цього виконайте flutter pub get, flutter analyze, flutter test та flutter run -d chrome.
Для здачі зі скриншотами й історією комітів використовуйте готовий архів.

### `pubspec.yaml`

```yaml
name: fitness_async
description: 'Практична 8. Варіант 5: фітнес-трекер тренування.'
publish_to: none
version: 1.0.0+1
environment:
  sdk: ^3.9.0
dependencies:
  flutter:
    sdk: flutter
  cupertino_icons: ^1.0.8
dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^5.0.0
  fake_async: ^1.3.3
flutter:
  uses-material-design: true
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

### `lib/models/workout.dart`

```dart
enum LoadMode { normal, error, empty, timeout }

class Exercise {
  const Exercise(this.name, this.seconds);
  final String name;
  final int seconds;
}

class Profile {
  const Profile(this.name, this.goal);
  final String name, goal;
}

class WorkoutResult {
  const WorkoutResult({
    required this.id,
    required this.seconds,
    required this.average,
    required this.maximum,
    required this.samples,
  });
  final String id;
  final int seconds, maximum, samples;
  final double average;
}

class WorkoutData {
  const WorkoutData(this.plan, this.profile, this.history);
  final List<Exercise> plan;
  final Profile profile;
  final List<WorkoutResult> history;
}

class WorkoutFrame {
  const WorkoutFrame({
    this.seconds = 0,
    this.pulse,
    this.average = 0,
    this.maximum = 0,
    this.samples = 0,
    this.paused = false,
    this.finished = false,
  });
  final int seconds, maximum, samples;
  final int? pulse;
  final double average;
  final bool paused, finished;
}
```

### `lib/services/fake_api.dart`

```dart
import 'dart:async';
import 'dart:math';
import '../models/workout.dart';

class FakeApi {
  FakeApi({
    this.failureRate = 0,
    this.loadMode = LoadMode.normal,
    Random? random,
  }) : _random = random ?? Random();
  double failureRate;
  LoadMode loadMode;
  bool forceSaveFailure = false;
  bool forcePulseFailure = false;
  bool emptyPulse = false;
  bool slowSave = false;
  bool _disposed = false;
  int saveAttempts = 0;
  final Random _random;
  final saved = <String, WorkoutResult>{};

  Future<T> request<T>(
    T Function() respond, {
    int delayMs = 700,
    bool fail = false,
  }) async {
    final rate = failureRate;
    await Future<void>.delayed(Duration(milliseconds: delayMs));
    if (_disposed) throw StateError('Сервіс закрито');
    if (fail || _random.nextDouble() < rate) {
      throw StateError('Сервер недоступний');
    }
    return respond();
  }

  Future<WorkoutData> load() async {
    final delay = loadMode == LoadMode.timeout ? 4000 : 700;
    final fail = loadMode == LoadMode.error;
    final empty = loadMode == LoadMode.empty;
    final values = await Future.wait<Object>([
      request(
        () => empty
            ? <Exercise>[]
            : const [
                Exercise('Розминка', 20),
                Exercise('Присідання', 30),
                Exercise('Планка', 20),
              ],
        delayMs: delay,
        fail: fail,
      ),
      request(
        () => const Profile('Спортсмен', 'Підтримувати регулярну активність'),
        delayMs: delay,
        fail: fail,
      ),
      request(
        () => const [
          WorkoutResult(
            id: 'previous-1',
            seconds: 600,
            average: 112,
            maximum: 138,
            samples: 600,
          ),
          WorkoutResult(
            id: 'previous-2',
            seconds: 720,
            average: 115,
            maximum: 141,
            samples: 720,
          ),
          WorkoutResult(
            id: 'previous-3',
            seconds: 540,
            average: 109,
            maximum: 130,
            samples: 540,
          ),
        ],
        delayMs: delay,
        fail: fail,
      ),
    ]).timeout(const Duration(seconds: 3));
    return WorkoutData(
      values[0] as List<Exercise>,
      values[1] as Profile,
      values[2] as List<WorkoutResult>,
    );
  }

  int? pulse() {
    if (forcePulseFailure) throw StateError('Датчик пульсу недоступний');
    if (emptyPulse) return null;
    return 85 + _random.nextInt(66);
  }

  Future<void> save(
    WorkoutResult result, {
    void Function(int)? onAttempt,
  }) async {
    // One initial attempt + at most three retries. Stable result.id avoids duplicates.
    for (var attempt = 0; attempt <= 3; attempt++) {
      if (_disposed) throw StateError('Сервіс закрито');
      onAttempt?.call(attempt + 1);
      saveAttempts++;
      try {
        final accepted = await request(
          () => result,
          delayMs: slowSave ? 4000 : 700,
          fail: forceSaveFailure,
        ).timeout(const Duration(seconds: 3));
        if (_disposed) throw StateError('Сервіс закрито');
        saved[accepted.id] = accepted;
        return;
      } catch (_) {
        if (_disposed || attempt == 3) rethrow;
        await Future<void>.delayed(
          Duration(milliseconds: 300 * (1 << attempt)),
        );
      }
    }
  }

  void dispose() {
    _disposed = true;
  }
}
```

### `lib/services/workout_session.dart`

```dart
import 'dart:async';
import '../models/workout.dart';
import 'fake_api.dart';

class WorkoutSession {
  WorkoutSession(this.api);
  final FakeApi api;
  final _frames = StreamController<WorkoutFrame>.broadcast();
  Timer? _timer;
  bool _disposed = false;
  int _seconds = 0, _samples = 0, _sum = 0, _maximum = 0;
  int? _pulse;
  bool _paused = false, _finished = false;
  final String id = DateTime.now().microsecondsSinceEpoch.toString();
  Stream<WorkoutFrame> get stream => _frames.stream;
  bool get running => _timer != null;
  WorkoutFrame get current => WorkoutFrame(
    seconds: _seconds,
    pulse: _pulse,
    average: _samples == 0 ? 0 : _sum / _samples,
    maximum: _maximum,
    samples: _samples,
    paused: _paused,
    finished: _finished,
  );
  void start() {
    if (_disposed || _finished || _timer != null) return;
    _paused = false;
    _frames.add(current);
    final before = _seconds;
    _timer = Timer.periodic(const Duration(seconds: 1), (timer) {
      _seconds = before + timer.tick;
      try {
        _pulse = api.pulse();
        if (_pulse != null) {
          _samples++;
          _sum += _pulse!;
          if (_pulse! > _maximum) _maximum = _pulse!;
        }
        _frames.add(current);
      } catch (error, stack) {
        _pulse = null;
        _frames.addError(error, stack);
      }
    });
  }

  void pause() {
    if (_disposed || _finished) return;
    _timer?.cancel();
    _timer = null;
    _paused = true;
    _frames.add(current);
  }

  void retryPulse() {
    if (_disposed || _paused || _finished) return;
    // The next timer tick re-reads the source; no second timer is created.
    _frames.add(current);
  }

  WorkoutResult finish() {
    _timer?.cancel();
    _timer = null;
    _finished = true;
    if (!_disposed) _frames.add(current);
    return WorkoutResult(
      id: id,
      seconds: _seconds,
      average: current.average,
      maximum: _maximum,
      samples: _samples,
    );
  }

  void dispose() {
    if (_disposed) return;
    _disposed = true;
    _timer?.cancel();
    _timer = null;
    unawaited(_frames.close());
  }
}
```

### `lib/main.dart`

```dart
import 'package:flutter/material.dart';
import 'screens/workout_screen.dart';

void main() => runApp(const FitnessApp());

class FitnessApp extends StatelessWidget {
  const FitnessApp({super.key});
  @override
  Widget build(BuildContext context) => MaterialApp(
    debugShowCheckedModeBanner: false,
    title: 'Моє тренування',
    theme: ThemeData(
      useMaterial3: true,
      colorScheme: ColorScheme.fromSeed(seedColor: const Color(0xFF5C48AC)),
      scaffoldBackgroundColor: const Color(0xFFF5F3FA),
    ),
    home: Scaffold(
      appBar: AppBar(title: const Text('Практична 8 • Варіант 5')),
      body: Builder(
        builder: (context) => Center(
          child: FilledButton.icon(
            icon: const Icon(Icons.fitness_center),
            label: const Text('Відкрити тренування'),
            onPressed: () => Navigator.of(context).push(
              MaterialPageRoute<void>(builder: (_) => const WorkoutScreen()),
            ),
          ),
        ),
      ),
    ),
  );
}
```

### `lib/screens/workout_screen.dart`

```dart
import 'dart:async';
import 'package:flutter/material.dart';
import '../models/workout.dart';
import '../services/fake_api.dart';
import '../services/workout_session.dart';

class WorkoutScreen extends StatefulWidget {
  const WorkoutScreen({super.key, this.api});
  final FakeApi? api;
  @override
  State<WorkoutScreen> createState() => _WorkoutScreenState();
}

class _WorkoutScreenState extends State<WorkoutScreen> {
  late final FakeApi _api;
  late Future<WorkoutData> _future;
  int _generation = 0;
  bool _active = false;
  @override
  void initState() {
    super.initState();
    _api = widget.api ?? FakeApi();
    _future = _api.load();
  }

  void _reload() => setState(() {
    _future = _api.load();
    _generation++;
  });
  @override
  void dispose() {
    _api.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) => Scaffold(
    appBar: AppBar(title: const Text('Моє тренування')),
    body: Column(
      children: [
        Padding(
          padding: const EdgeInsets.symmetric(horizontal: 16),
          child: Wrap(
            spacing: 20,
            children: [
              DropdownButton<LoadMode>(
                value: _api.loadMode,
                items: const [
                  DropdownMenuItem(
                    value: LoadMode.normal,
                    child: Text('Звичайний режим'),
                  ),
                  DropdownMenuItem(
                    value: LoadMode.error,
                    child: Text('Помилка завантаження'),
                  ),
                  DropdownMenuItem(
                    value: LoadMode.empty,
                    child: Text('Порожній план'),
                  ),
                  DropdownMenuItem(
                    value: LoadMode.timeout,
                    child: Text('Тайм-аут'),
                  ),
                ],
                onChanged: _active
                    ? null
                    : (value) {
                        if (value == null) return;
                        _api.loadMode = value;
                        _reload();
                      },
              ),
              Row(
                mainAxisSize: MainAxisSize.min,
                children: [
                  const Text('Збої сервісу: '),
                  DropdownButton<double>(
                    value: _api.failureRate,
                    items: [0.0, 0.5, 1.0]
                        .map(
                          (rate) => DropdownMenuItem(
                            value: rate,
                            child: Text('${(rate * 100).round()} %'),
                          ),
                        )
                        .toList(),
                    onChanged: (value) {
                      if (value != null) {
                        setState(() => _api.failureRate = value);
                      }
                    },
                  ),
                ],
              ),
            ],
          ),
        ),
        Expanded(
          child: FutureBuilder<WorkoutData>(
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
                      Text('Завантаження плану, профілю та історії…'),
                    ],
                  ),
                );
              }
              if (snapshot.hasError) {
                return _message(
                  snapshot.error is TimeoutException
                      ? 'Час очікування вичерпано.'
                      : 'Не вдалося завантажити тренування.',
                );
              }
              if (snapshot.data == null || snapshot.data!.plan.isEmpty) {
                return _message('План тренування порожній.');
              }
              return WorkoutPanel(
                data: snapshot.data!,
                api: _api,
                onStarted: () => setState(() => _active = true),
              );
            },
          ),
        ),
      ],
    ),
  );
  Widget _message(String message) => Center(
    child: Column(
      mainAxisSize: MainAxisSize.min,
      children: [
        const Icon(Icons.info_outline, size: 48),
        Text(message),
        FilledButton(onPressed: _reload, child: const Text('Повторити')),
      ],
    ),
  );
}

class WorkoutPanel extends StatefulWidget {
  const WorkoutPanel({
    super.key,
    required this.data,
    required this.api,
    required this.onStarted,
  });
  final WorkoutData data;
  final FakeApi api;
  final VoidCallback onStarted;
  @override
  State<WorkoutPanel> createState() => _WorkoutPanelState();
}

class _WorkoutPanelState extends State<WorkoutPanel> {
  late final WorkoutSession _session;
  WorkoutResult? _pending;
  bool _started = false,
      _saving = false,
      _saved = false,
      _allowExit = false,
      _dialogOpen = false;
  String? _saveError;
  int _attempt = 0;
  @override
  void initState() {
    super.initState();
    _session = WorkoutSession(widget.api);
  }

  @override
  void dispose() {
    _session.dispose();
    super.dispose();
  }

  Future<void> _save() async {
    if (_saving || _pending == null || _saved) return;
    setState(() {
      _saving = true;
      _saveError = null;
    });
    try {
      await widget.api.save(
        _pending!,
        onAttempt: (attempt) {
          if (!mounted) return;
          setState(() => _attempt = attempt);
        },
      );
      if (!mounted) return;
      setState(() {
        _saving = false;
        _saved = true;
      });
    } catch (error) {
      if (!mounted) return;
      setState(() {
        _saving = false;
        _saveError = 'Збереження не вдалося. Результат залишився на екрані.';
      });
    }
  }

  void _finish() {
    if (_pending != null) return;
    setState(() => _pending = _session.finish());
    unawaited(_save());
  }

  Future<void> _confirmExit() async {
    if (_dialogOpen) return;
    _dialogOpen = true;
    final leave = await showDialog<bool>(
      context: context,
      builder: (dialogContext) => AlertDialog(
        title: const Text('Вийти з тренування?'),
        content: const Text(
          'Поточне тренування або незбережений результат буде втрачено.',
        ),
        actions: [
          TextButton(
            onPressed: () => Navigator.pop(dialogContext, false),
            child: const Text('Залишитися'),
          ),
          FilledButton(
            onPressed: () => Navigator.pop(dialogContext, true),
            child: const Text('Вийти'),
          ),
        ],
      ),
    );
    if (!mounted) return;
    _dialogOpen = false;
    if (leave == true) {
      setState(() => _allowExit = true);
      WidgetsBinding.instance.addPostFrameCallback((_) {
        if (!mounted) return;
        Navigator.of(context).pop();
      });
    }
  }

  String _time(int seconds) =>
      '${(seconds ~/ 60).toString().padLeft(2, '0')}:${(seconds % 60).toString().padLeft(2, '0')}';
  String _exercise(int seconds) {
    var end = 0;
    for (final exercise in widget.data.plan) {
      end += exercise.seconds;
      if (seconds < end) return exercise.name;
    }
    return 'План виконано — завершіть тренування';
  }

  @override
  Widget build(BuildContext context) => PopScope<void>(
    canPop: _allowExit || !_started || _saved,
    onPopInvokedWithResult: (didPop, result) {
      if (!didPop) unawaited(_confirmExit());
    },
    child: StreamBuilder<WorkoutFrame>(
      stream: _session.stream,
      builder: (context, snapshot) {
        final frame = snapshot.data ?? _session.current;
        return ListView(
          padding: const EdgeInsets.all(16),
          children: [
            Text(
              'Вітаємо, ${widget.data.profile.name}',
              style: Theme.of(context).textTheme.headlineSmall,
            ),
            Text(widget.data.profile.goal),
            const SizedBox(height: 12),
            Card(
              child: Padding(
                padding: const EdgeInsets.all(20),
                child: Column(
                  children: [
                    const Text('ЧАС ТРЕНУВАННЯ'),
                    Text(
                      _time(_session.current.seconds),
                      key: const Key('time'),
                      style: Theme.of(context).textTheme.displayMedium,
                    ),
                    Text(_exercise(_session.current.seconds)),
                    const SizedBox(height: 12),
                    if (snapshot.hasError) ...[
                      const Text('Не вдалося отримати пульс.'),
                      TextButton(
                        onPressed: frame.paused || _pending != null
                            ? null
                            : _session.retryPulse,
                        child: const Text('Повторити'),
                      ),
                    ] else if (snapshot.connectionState ==
                        ConnectionState.waiting) ...[
                      Text(
                        _started
                            ? 'Очікування першого показника…'
                            : 'Натисніть «Почати»',
                      ),
                    ] else if (frame.pulse == null) ...[
                      const Text('Даних пульсу поки немає.'),
                    ] else ...[
                      Text(
                        '${frame.pulse} уд/хв',
                        key: const Key('pulse'),
                        style: Theme.of(context).textTheme.headlineMedium,
                      ),
                    ],
                    const SizedBox(height: 10),
                    Text(
                      'Середній: ${frame.average.toStringAsFixed(1)} • Максимальний: ${frame.maximum}',
                    ),
                    Text('Вимірювань: ${frame.samples}'),
                  ],
                ),
              ),
            ),
            Wrap(
              spacing: 10,
              runSpacing: 8,
              children: [
                if (!_started)
                  FilledButton(
                    onPressed: () {
                      setState(() => _started = true);
                      widget.onStarted();
                      _session.start();
                    },
                    child: const Text('Почати'),
                  ),
                if (_started && _pending == null) ...[
                  FilledButton.tonal(
                    onPressed: () {
                      if (_session.current.paused) {
                        _session.start();
                      } else {
                        _session.pause();
                      }
                    },
                    child: Text(
                      _session.current.paused ? 'Продовжити' : 'Пауза',
                    ),
                  ),
                  FilledButton(
                    onPressed: _finish,
                    child: const Text('Завершити'),
                  ),
                ],
              ],
            ),
            if (_pending != null)
              Card(
                child: Padding(
                  padding: const EdgeInsets.all(16),
                  child: Column(
                    crossAxisAlignment: CrossAxisAlignment.start,
                    children: [
                      Text(
                        'Результат: ${_time(_pending!.seconds)}; середній пульс ${_pending!.average.toStringAsFixed(1)}',
                      ),
                      if (_saving) ...[
                        const LinearProgressIndicator(),
                        Text('Збереження: спроба $_attempt із 4'),
                      ],
                      if (_saveError != null) ...[
                        Text(_saveError!),
                        TextButton(
                          onPressed: _save,
                          child: const Text('Повторити збереження'),
                        ),
                      ],
                      if (_saved) const Text('Результат успішно збережено.'),
                    ],
                  ),
                ),
              ),
            const SizedBox(height: 16),
            Text('План', style: Theme.of(context).textTheme.titleLarge),
            for (final exercise in widget.data.plan)
              Text('${exercise.name} — ${exercise.seconds} с'),
            const SizedBox(height: 16),
            Text(
              'Останні три тренування',
              style: Theme.of(context).textTheme.titleLarge,
            ),
            if (widget.data.history.isEmpty) const Text('Історія порожня.'),
            for (final result in widget.data.history)
              Text(
                '${_time(result.seconds)} • середній ${result.average.toStringAsFixed(0)} уд/хв',
              ),
            const SizedBox(height: 16),
            ExpansionTile(
              title: const Text('Демонстрація збоїв'),
              children: [
                SwitchListTile(
                  title: const Text('Примусова помилка збереження'),
                  value: widget.api.forceSaveFailure,
                  onChanged: _saving
                      ? null
                      : (value) =>
                            setState(() => widget.api.forceSaveFailure = value),
                ),
                SwitchListTile(
                  title: const Text('Повільне збереження (тайм-аут)'),
                  value: widget.api.slowSave,
                  onChanged: _saving
                      ? null
                      : (value) => setState(() => widget.api.slowSave = value),
                ),
                SwitchListTile(
                  title: const Text('Помилка датчика пульсу'),
                  value: widget.api.forcePulseFailure,
                  onChanged: (value) =>
                      setState(() => widget.api.forcePulseFailure = value),
                ),
                SwitchListTile(
                  title: const Text('Немає показників пульсу'),
                  value: widget.api.emptyPulse,
                  onChanged: (value) =>
                      setState(() => widget.api.emptyPulse = value),
                ),
              ],
            ),
          ],
        );
      },
    ),
  );
}
```

### `test/workout_test.dart`

```dart
import 'dart:async';
import 'dart:math';
import 'package:fake_async/fake_async.dart';
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:fitness_async/main.dart';
import 'package:fitness_async/models/workout.dart';
import 'package:fitness_async/services/fake_api.dart';
import 'package:fitness_async/services/workout_session.dart';
import 'package:fitness_async/screens/workout_screen.dart';

class ScriptedRandom implements Random {
  int index = 0;
  @override
  double nextDouble() => index++ < 4 ? 0.2 : 0.8;
  @override
  int nextInt(int max) => 0;
  @override
  bool nextBool() => false;
}

void main() {
  test('parallel load finishes at 700ms', () {
    fakeAsync((time) {
      WorkoutData? result;
      FakeApi().load().then((value) => result = value);
      time.elapse(const Duration(milliseconds: 699));
      expect(result, isNull);
      time.elapse(const Duration(milliseconds: 1));
      expect(result!.history.length, 3);
      expect(result!.plan.length, 3);
    });
  });
  test('load timeout is three seconds', () {
    fakeAsync((time) {
      Object? error;
      FakeApi(loadMode: LoadMode.timeout).load().then<void>(
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
  test('pause stops time and pulse; finish and dispose stop the source', () {
    fakeAsync((time) {
      final session = WorkoutSession(FakeApi(random: Random(10)));
      var events = 0;
      var done = false;
      final sub = session.stream.listen(
        (_) => events++,
        onDone: () => done = true,
      );
      session.start();
      time.elapse(const Duration(seconds: 3));
      expect(session.current.seconds, 3);
      expect(session.current.samples, 3);
      expect(session.current.average, inInclusiveRange(85, 150));
      expect(
        session.current.maximum,
        greaterThanOrEqualTo(session.current.average),
      );
      session.pause();
      time.flushMicrotasks();
      final count = events;
      final pulse = session.current.pulse;
      time.elapse(const Duration(seconds: 10));
      expect(events, count);
      expect(session.current.seconds, 3);
      expect(session.current.pulse, pulse);
      session.start();
      time.elapse(const Duration(seconds: 2));
      final result = session.finish();
      expect(result.seconds, 5);
      expect(result.samples, 5);
      time.elapse(const Duration(seconds: 10));
      expect(session.current.seconds, 5);
      session.dispose();
      time.flushMicrotasks();
      expect(done, isTrue);
      unawaited(sub.cancel());
    });
  });
  test('pulse error does not close stream; recovery works', () {
    fakeAsync((time) {
      final api = FakeApi()..forcePulseFailure = true;
      final session = WorkoutSession(api);
      var errors = 0;
      final sub = session.stream.listen(
        (_) {},
        onError: (Object _) => errors++,
      );
      session.start();
      time.elapse(const Duration(seconds: 1));
      expect(errors, 1);
      api.forcePulseFailure = false;
      time.elapse(const Duration(seconds: 1));
      expect(session.current.samples, 1);
      expect(session.current.seconds, 2);
      session.dispose();
      unawaited(sub.cancel());
    });
  });
  const result = WorkoutResult(
    id: 'one',
    seconds: 10,
    average: 100,
    maximum: 120,
    samples: 10,
  );
  test(
    'four failed attempts retain result; manual retry saves same id once',
    () {
      fakeAsync((time) {
        final api = FakeApi(failureRate: 1);
        Object? error;
        api
            .save(result)
            .then<void>(
              (_) {},
              onError: (Object e) {
                error = e;
              },
            );
        time.elapse(const Duration(seconds: 6));
        expect(error, isNotNull);
        expect(api.saveAttempts, 4);
        expect(api.saved, isEmpty);
        api.failureRate = 0;
        api.save(result);
        time.elapse(const Duration(seconds: 1));
        expect(api.saved['one'], same(result));
        api.save(result);
        time.elapse(const Duration(seconds: 1));
        expect(api.saved.length, 1);
      });
    },
  );
  test(
    '50 percent setting: four failures followed by successful manual retry',
    () {
      fakeAsync((time) {
        final api = FakeApi(failureRate: 0.5, random: ScriptedRandom());
        Object? error;
        api
            .save(result)
            .then<void>(
              (_) {},
              onError: (Object e) {
                error = e;
              },
            );
        time.elapse(const Duration(seconds: 6));
        expect(error, isNotNull);
        expect(api.saveAttempts, 4);
        api.save(result);
        time.elapse(const Duration(seconds: 1));
        expect(api.saved['one'], same(result));
      });
    },
  );
  test('save timeouts do not produce late writes', () {
    fakeAsync((time) {
      final api = FakeApi()..slowSave = true;
      Object? error;
      api
          .save(result)
          .then<void>(
            (_) {},
            onError: (Object e) {
              error = e;
            },
          );
      time.elapse(const Duration(seconds: 20));
      expect(error, isA<TimeoutException>());
      expect(api.saveAttempts, 4);
      expect(api.saved, isEmpty);
    });
  });
  for (final entry in {
    LoadMode.error: 'Не вдалося завантажити тренування.',
    LoadMode.empty: 'План тренування порожній.',
  }.entries) {
    testWidgets('loading then ${entry.key}', (tester) async {
      await tester.pumpWidget(
        MaterialApp(
          home: WorkoutScreen(api: FakeApi(loadMode: entry.key)),
        ),
      );
      expect(find.byType(CircularProgressIndicator), findsOneWidget);
      await tester.pump(const Duration(milliseconds: 700));
      await tester.pump();
      expect(find.text(entry.value), findsOneWidget);
      expect(find.text('Повторити'), findsOneWidget);
      await tester.pumpWidget(const SizedBox());
    });
  }
  testWidgets('exit confirmation and safe disposal while running', (
    tester,
  ) async {
    await tester.pumpWidget(const FitnessApp());
    await tester.tap(find.text('Відкрити тренування'));
    await tester.pumpAndSettle();
    await tester.tap(find.text('Почати'));
    await tester.pump();
    await tester.pump(const Duration(seconds: 2));
    await tester.pageBack();
    await tester.pumpAndSettle();
    expect(find.text('Вийти з тренування?'), findsOneWidget);
    await tester.tap(find.text('Залишитися'));
    await tester.pumpAndSettle();
    await tester.tap(find.text('Пауза'));
    await tester.pump();
    final time = tester.widget<Text>(find.byKey(const Key('time'))).data;
    final pulse = tester.widget<Text>(find.byKey(const Key('pulse'))).data;
    await tester.pump(const Duration(seconds: 10));
    expect(tester.widget<Text>(find.byKey(const Key('time'))).data, time);
    expect(tester.widget<Text>(find.byKey(const Key('pulse'))).data, pulse);
    await tester.pageBack();
    await tester.pumpAndSettle();
    await tester.tap(find.text('Вийти'));
    await tester.pumpAndSettle();
    await tester.pump(const Duration(seconds: 10));
    expect(tester.takeException(), isNull);
    expect(find.text('Відкрити тренування'), findsOneWidget);
  });
  testWidgets('unsaved result remains and manual retry succeeds', (
    tester,
  ) async {
    final api = FakeApi()..forceSaveFailure = true;
    await tester.pumpWidget(MaterialApp(home: WorkoutScreen(api: api)));
    await tester.pump(const Duration(milliseconds: 700));
    await tester.pump();
    await tester.tap(find.text('Почати'));
    await tester.pump();
    await tester.pump(const Duration(seconds: 2));
    await tester.tap(find.text('Завершити'));
    await tester.pump();
    for (var i = 0; i < 12; i++) {
      await tester.pump(const Duration(seconds: 1));
    }
    expect(find.text('Повторити збереження'), findsOneWidget);
    expect(api.saved, isEmpty);
    api.forceSaveFailure = false;
    await tester.tap(find.text('Повторити збереження'));
    await tester.pump();
    await tester.pump(const Duration(seconds: 1));
    await tester.pump();
    expect(find.text('Результат успішно збережено.'), findsOneWidget);
    expect(api.saved.length, 1);
    await tester.pumpWidget(const SizedBox());
  });
  testWidgets('disposing during save does not call setState after dispose', (
    tester,
  ) async {
    final api = FakeApi();
    await tester.pumpWidget(MaterialApp(home: WorkoutScreen(api: api)));
    await tester.pump(const Duration(milliseconds: 700));
    await tester.pump();
    await tester.tap(find.text('Почати'));
    await tester.pump();
    await tester.tap(find.text('Завершити'));
    await tester.pump();
    await tester.pumpWidget(const SizedBox());
    await tester.pump(const Duration(seconds: 10));
    expect(tester.takeException(), isNull);
    expect(api.saved, isEmpty);
  });
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
