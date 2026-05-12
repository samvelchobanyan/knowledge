Нам нуэно реализовать фичу подписки на напоминяне о программе

Пользователь видит расписание эфира радио (RadioScheduleSlot) и может подписаться на программу. За 10 минут до начала программы ему должно прийти уведомление. При тапе на уведомление — открывается экран расписания радио.

Состояние подписки (hasReminder) не хранится на бэкенде — только локально на устройстве.

что думаешь?
Show less
Grep "RadioScheduleSlot"
3 lines of output
Grep "flutter_local_notifications|awesome_notifications|notification" (in c:\flutter-projects\go_sport\pubspec.yaml)
1 line of output
Grep "RadioScheduleSlot" (in c:\flutter-projects\go_sport\docs\DOMAIN_SPEC.md)
23 lines of output
Read c:\flutter-projects\go_sport\pubspec.yaml
Grep "RadioScheduleSlot|hasReminder" (in c:\flutter-projects\go_sport\lib)
1 line of output
Grep "radio.*schedule|RadioSchedule" (in c:\flutter-projects\go_sport\lib)
4 lines of output
Подход рабочий — локального состояния достаточно, т.к. и подписка, и уведомление живут на устройстве. Предлагаю стэк: flutter_local_notifications + timezone для планирования zonedSchedule за 10 мин до startsAt, shared_preferences для Set<slotId> подписок, deep-link через go_router по payload при тапе.

Главный трейдоф: сервер может переносить/отменять слоты, а локальные уведомления уже запланированы — можем показать юзеру напоминание о программе, которой уже нет. Варианты: (1) при каждом открытии расписания ресинкать — отменять запланированные уведомления для слотов, чьи startsAt изменились, и пере-планировать; (2) игнорировать — считая что редкий кейс.

Ещё важные мелочи, которые стоит обсудить до кода:

permissions: Android 13+ POST_NOTIFICATIONS, SCHEDULE_EXACT_ALARM; iOS — запрос при первой подписке
слоты в прошлом — кнопка подписки скрыта/дизейблится
слот стартует через <10 мин — сразу show() или не подписываем
Как хочешь — опишу пошагово весь план (deps → permissions → service → state → UI), или сразу уточняем детали и начинаем с первого слоя?

ну ресинк при открытии расписания вопрос не решит полностью, пользователь может на зати на страницу расписания
Справедливо. Тогда ресинк нужно триггерить шире, чем только на экране расписания:

На старте приложения и на resume (AppLifecycleState.resumed) — тихо дёргаем расписание и пере-планируем. Покрывает большинство кейсов, т.к. радио-юзер заходит в апп регулярно.
Background fetch через workmanager — раз в N часов подтягивать расписание даже без открытия апы. Тяжелее по инфраструктуре (Android/iOS особенности), но честно решает проблему «юзер неделю не заходил».
Честный компромисс в UI — в уведомлении писать «Через 10 минут: название. Проверьте расписание» и в тексте оставить зацеп, что время могло измениться. Дёшево и снимает ответственность.
Если фича не критичная — я бы пошёл путём 1 + 3: resync на старте/resume + аккуратная формулировка в уведомлении. Workmanager добавим, если реально начнут жаловаться.

Ещё альтернатива, которая решает радикально: напоминания через FCM с сервера (сервер сам знает актуальное расписание и шлёт push в нужный момент). Но это ломает твоё требование «только локально» и тянет бэкенд-работу.

Какой вариант ближе — 1+3, добавлять workmanager, или всё-таки рассмотреть серверный push?

пока будем игрорировать это редкий кейс. если что пойдем по варианто 3
ресинк не нужен

напомни что еще хотел обсудить?
Show less
Оставались три момента:

Permissions — Android 13+ требует POST_NOTIFICATIONS (runtime), для точного времени — SCHEDULE_EXACT_ALARM (Android 12+). iOS — запрос при первой подписке. Когда спрашиваем: лениво при первом тапе на «подписаться», или заранее на онбординге?

Слоты в прошлом — кнопку подписки прячем/дизейблим, или просто игнорим тап?

Слот стартует через <10 минут — что делаем: (а) сразу show() уведомление, (б) планируем на ближайший возможный момент, (в) вообще не даём подписаться?

Плюс один новый вопрос по ходу: персистентность подписок между переустановками — локально же, значит при переустановке всё теряется. Ок или нужно как-то мигрировать? (обычно ок, просто фиксируем.)

при переустаноке норм если теряется, тоже редкий кейс
прмищн для iOS наверное на старте
а Андроид прблемы или просто нужно что-то установить?
Для Android в основном конфиг + один рантайм-запрос:

AndroidManifest.xml — добавить:

POST_NOTIFICATIONS (Android 13+)
SCHEDULE_EXACT_ALARM или USE_EXACT_ALARM (для точного времени)
RECEIVE_BOOT_COMPLETED (чтобы уведомления переживали ребут)
Runtime — на Android 13+ нужно запросить POST_NOTIFICATIONS у пользователя (пакет flutter_local_notifications это делает через свой API). Логичнее спросить так же на старте, как и iOS — единый флоу.

Потенциальные проблемы, не блокеры:

USE_EXACT_ALARM vs SCHEDULE_EXACT_ALARM: первое разрешено только для будильников/напоминаний по политике Google Play. Для нашего кейса подходит, но при публикации нужно указать это назначение. Безопаснее SCHEDULE_EXACT_ALARM — на Android 14+ юзер разрешает через настройки, если не разрешил — fallback на неточное время (±несколько минут, что для «за 10 мин» нормально).
Агрессивные вендоры (Xiaomi, Huawei, Oppo) могут убивать запланированные уведомления в целях экономии батареи — это общая боль Android, нечего с этим не сделать на уровне кода.
Итого: ставим flutter_local_notifications + timezone, добавляем permissions в манифест, на старте приложения один раз запрашиваем разрешение на уведомления для обеих платформ.

Ок двигаемся к плану реализации?

в приципе ок, но обясни ка про SCHEDULE_EXACT_ALARM
Коротко, как Android эволюционировал по этой теме:

До Android 12 (API <31) — любое приложение могло планировать точные будильники без спроса. Никаких разрешений.

Android 12 (API 31) — Google заметил, что точные алармы жрут батарею (будят устройство из Doze). Ввели разрешение SCHEDULE_EXACT_ALARM. Оно автоматически выдаётся при установке, но юзер может отозвать его в настройках.

Android 13 (API 33) — добавили альтернативу USE_EXACT_ALARM. Тоже авто-выдача, но нельзя отозвать. Однако Google Play разрешает его только приложениям, у которых точный тайминг — основная функциональность (будильники, календари, напоминания). Если использовать не по назначению — риск бана в сторе.

Android 14 (API 34) — ужесточили SCHEDULE_EXACT_ALARM: теперь при установке не выдаётся автоматически. Приложение должно отправить юзера в системные настройки через интент ACTION_REQUEST_SCHEDULE_EXACT_ALARM, где он вручную переключает тумблер. USE_EXACT_ALARM остался как был.

Что это значит для нас:

Если не получили точный аларм — система не кидает ошибку, а использует неточный (setInexactRepeating). Разброс — несколько минут, в Doze режиме может быть и до 15. Для «за 10 минут до программы» это не критично: придёт примерно тогда же.
flutter_local_notifications даёт выбор: AndroidScheduleMode.exact / exactAllowWhileIdle / inexactAllowWhileIdle. Можно попробовать exact и при отсутствии разрешения упасть на inexact без вмешательства юзера.
Что я рекомендую: использовать SCHEDULE_EXACT_ALARM + inexactAllowWhileIdle как fallback. Не дёргать юзера в системные настройки — это плохой UX ради пары минут точности. Радио-напоминание ≠ будильник на утро, ±5 минут никого не убьёт.

Ок так?

тоесть SCHEDULE_EXACT_ALARM работает точно но пользователь должен сам включить
USE_EXACT_ALARM разрешать не надо, но работает не точно?
Show less
Нет, я неясно объяснил. Оба разрешения дают точный тайминг. Разница в том, как их выдают:

Точность	Как выдаётся	Ограничение
SCHEDULE_EXACT_ALARM	Точно	Android 14+ — юзер сам включает в настройках	—
USE_EXACT_ALARM	Точно	Авто, юзер не участвует	Google Play: только для апп, где это core feature (будильники, календари, напоминания)
Неточный режим (inexactAllowWhileIdle) — это третья опция, которая вообще не требует ни одного из этих двух разрешений. Точность — ±несколько минут, система сама решает когда разбудить устройство экономичнее. Его и используют как fallback.

Варианты для нашей фичи:

USE_EXACT_ALARM — точно, без согласия юзера. Но при публикации в Play нужно обосновать, что напоминания о радиопрограммах = reminder-функциональность. Риск есть, но наш кейс в серую зону попадает нормально.

SCHEDULE_EXACT_ALARM + fallback на inexact — на Android <14 точно, на 14+ неточно (если не вести юзера в настройки). Без риска с Play.

Сразу inexactAllowWhileIdle — самый простой путь. Всегда ±5 мин. Без разрешений, без политик Play.

Для «напомнить за 10 минут до радиопрограммы» я бы взял вариант 3. Но если хочешь именно минуту-в-минуту — тогда USE_EXACT_ALARM.

нет не обязательно 
вариант 3 вполне подходит

и так давай поймем структуру 
как организуем?
Show less
Globpattern: "lib/**/*.dart"
Found 2 files
Globpattern: "lib/features/radio/**/*.dart"
Found 5 files
Globpattern: "lib/core/**/*.dart"
Found 15 files
Globpattern: "lib/domain/**/*.dart"
Found 50 files
По структуре проекта (core для инфраструктуры, domain для сущностей/state/repos, features для UI) предлагаю так:


lib/
├── core/
│   └── notifications/
│       ├── notification_service.dart        # обёртка над flutter_local_notifications:
│       │                                    #   init(), requestPermissions(),
│       │                                    #   scheduleAt(id, title, body, dateTime, payload),
│       │                                    #   cancel(id), cancelAll()
│       └── notification_payload.dart        # типизированный payload (deep-link routing)
│
├── domain/
│   ├── repositories/
│   │   └── program_reminder_repository.dart # интерфейс: Set<String> getSubscribed(),
│   │                                        #   add(slotId), remove(slotId), has(slotId)
│   └── state/
│       └── program_reminders_state.dart     # StateNotifier<Set<String>>, orchestrates:
│                                            #   repo (persist) + NotificationService (schedule/cancel)
│
├── data/
│   └── repositories/
│       └── program_reminder_repository_impl.dart  # SharedPreferences, ключ 'program_reminders'
│
└── features/radio/presentation/
    └── widgets/
        └── reminder_button.dart             # consumer, слушает provider, тапает toggle
Роли компонентов:

NotificationService — знает только про flutter_local_notifications/timezone. Чистая инфраструктура, про слоты ничего не знает. Генерит notificationId из slotId.hashCode.
ProgramReminderRepository — только персист Set<slotId>. Не трогает уведомления.
ProgramRemindersNotifier (state) — оркестратор: при subscribe зовёт repo.add + service.scheduleAt, при unsubscribe — repo.remove + service.cancel. Публикует Set<String> для UI.
ReminderButton — smart widget в radio-фиче: ref.watch(remindersProvider).contains(slotId) → иконка, onTap → notifier.toggle(slot).
Init-флоу: в main.dart (или в корневом виджете) до runApp — NotificationService.init() + запрос пермишнов. Там же обрабатываем getLaunchDetails (тап по уведомлению, когда апп был killed) — пушим нужный роут в go_router.

Deep-link: payload = slotId или {'type': 'radio_schedule', 'slotId': ...}. onDidReceiveNotificationResponse → router.push('/radio/schedule').

Пара вопросов до начала:

Где лежит экран расписания радио — уже есть отдельный роут, или это часть radio_page_screen? От этого зависит deep-link.
SharedPreferences уже в проекте или добавляем? Вижу только flutter_secure_storage. Можно и через secure_storage, но для не-секретных данных это оверкилл.
Как тебе структура?

что делает program_reminder_repository?
ProgramReminderRepository — это тонкая обёртка над локальным хранилищем (SharedPreferences). Его единственная задача — хранить множество slotId, на которые юзер подписан, и уметь его читать/писать.

Примерно такой интерфейс:


abstract class ProgramReminderRepository {
  Future<Set<String>> getAll();        // прочитать все подписки
  Future<void> add(String slotId);     // добавить одну
  Future<void> remove(String slotId);  // убрать одну
}
Реализация — просто сериализация Set<String> в SharedPreferences под ключом типа 'program_reminders' (через setStringList / getStringList).

Зачем он вообще нужен, если есть уведомления?

Уведомления в системе — это побочный эффект подписки, а не источник правды. У flutter_local_notifications есть pendingNotificationRequests(), но полагаться на него как на источник состояния — плохо:

Нужно знать подписки до инициализации сервиса уведомлений (для отрисовки кнопок в UI сразу при загрузке экрана).
Уведомление уже сработало → оно исчезло из pending, но подписка юзера логически осталась (чтобы на следующей неделе та же программа опять напомнила — если будем делать повторяющиеся; для разовых это не важно).
Юзер мог отключить пермишн на уведомления — система их не запланирует, но подписка как намерение должна сохраняться.
Чистое разделение ответственности: repo знает про состояние, service — про механизм доставки.
Как это работает вместе (в ProgramRemindersNotifier):


subscribe(slot):
  repo.add(slot.id)          // сохранили намерение
  service.scheduleAt(...)    // запланировали доставку

unsubscribe(slot):
  repo.remove(slot.id)
  service.cancel(slot.id.hashCode)

// на старте:
final ids = await repo.getAll();  // восстановили для UI
В общем: repo = «на что подписан», service = «как доставить». Два разных concern'а.

а стоит обертку класть в репощиторий? мы же на запрашиваем доменный данные не дергаем апи
или все равно стоит клать в репозиторий?
Честно — не стоит. Ты прав: репозиторий в этом проекте дёргает API и возвращает доменные сущности (треки, плейлисты, программы). Здесь же — просто Set<String> в SharedPreferences. Это не domain data, это локальный флаг UI-состояния.

Пара вариантов, как сделать проще:

Вариант А — встроить в Notifier. Нотифаер сам работает с SharedPreferences, без промежуточного слоя:


class ProgramRemindersNotifier extends StateNotifier<Set<String>> {
  final SharedPreferences _prefs;
  final NotificationService _notifications;
  static const _key = 'program_reminders';
  // load/save прямо тут
}
Минус — смешивается логика (подписка + персист + планирование), но файл остаётся небольшой.

Вариант Б — маленький storage-хелпер без интерфейса. Один класс, конкретный, без абстракции:


// core/storage/reminder_storage.dart
class ReminderStorage {
  Future<Set<String>> load();
  Future<void> save(Set<String> ids);
}
Без abstract class, без domain/repositories/. Просто утилита в core/. Notifier зовёт её, и она инкапсулирует формат хранения (ключ, сериализация).

Что бы я выбрал: вариант Б. Три причины:

Notifier не раздувается персист-логикой.
Если завтра понадобится поменять storage (secure_storage, hive, файл) — одно место.
Не создаём «репозиторий», который не репозиторий — семантика проекта остаётся чистой.
Итоговая структура тогда:


lib/
├── core/
│   ├── notifications/
│   │   ├── notification_service.dart
│   │   └── notification_payload.dart
│   └── storage/
│       └── reminder_storage.dart
├── domain/state/
│   └── program_reminders_state.dart
└── features/radio/presentation/widgets/
    └── reminder_button.dart
Так лучше?

давай подумаем
а как у нас релазовано secure_storage? там есть обертка ?
Grep "secure_storage|FlutterSecureStorage|SecureStorage" (in c:\flutter-projects\go_sport\lib)
2 lines of output
Read c:\flutter-projects\go_sport\lib\core\auth\token_storage.dart
Да, есть — lib/core/auth/token_storage.dart. Смотри паттерн:

Конкретный класс, без интерфейса — TokenStorage, никакого abstract.
Domain-aware, не generic — знает про accessToken, refreshToken, chose_guest. Не просто «KV-хранилище».
Лежит рядом с фичей, которую обслуживает — core/auth/, не core/storage/.
Владеет FlutterSecureStorage внутри, наружу утекает только доменный API.
Регистрируется как Provider с override в ProviderScope (инит асинхронный, поэтому создаётся на старте и прокидывается).
init() предзагружает значения в in-memory кеш → наружу синхронные геттеры.
Для напоминаний логично повторить тот же паттерн, только с SharedPreferences:


lib/core/notifications/
├── notification_service.dart       # обёртка над flutter_local_notifications
├── notification_payload.dart
└── reminder_storage.dart           # ← по аналогии с token_storage.dart
ReminderStorage — конкретный класс:

Future<void> init() — читает Set<String> из SharedPreferences в поле.
Синхронный геттер Set<String> get subscribedSlotIds.
Future<void> add(String slotId) / remove(String slotId) — обновляет кеш + пишет.
Провайдер с override в ProviderScope, инит на старте приложения вместе с TokenStorage.
Notifier тогда зовёт storage.add(id) + notifications.scheduleAt(...) и публикует в state.

Почему лучше такого паттерна, чем вариант А (встроить в Notifier):

Согласованно с тем, как уже устроен проект.
Инит можно сделать на старте (как у TokenStorage), и Notifier стартует синхронно, без AsyncValue.
Если понадобится то же хранилище в другом месте (например, на экране настроек «очистить все напоминания») — просто зовёшь провайдер.
Ок с таким подходом?

согласен
а что такое notification_payload?
payload — это строка, которую flutter_local_notifications позволяет прикрепить к каждому уведомлению. Когда юзер тапает по нему, колбэк onDidReceiveNotificationResponse отдаёт эту строку обратно — и по ней мы решаем, куда перейти в приложении.

То есть:


// при планировании:
flutterLocalNotifications.zonedSchedule(
  id, title, body, scheduledDate,
  notificationDetails,
  payload: 'какая-то-строка',   // ← это
);

// при тапе:
onDidReceiveNotificationResponse: (response) {
  final payload = response.payload;  // ← получаем обратно
  // роутим на основе payload
}
Зачем отдельный файл notification_payload.dart:

Payload — просто String. Если писать его как голую строку, получается хрупко:


// плохо
service.scheduleAt(..., payload: 'radio_schedule');
if (response.payload == 'radio_schedule') router.push(...);
Магические строки, опечатки не ловятся, при добавлении новых типов уведомлений (например, напоминания о новом эпизоде программы) — ад. Типизированная обёртка это чинит:


// core/notifications/notification_payload.dart
sealed class NotificationPayload {
  String encode();
  static NotificationPayload? decode(String? raw) { ... }
}

class RadioSchedulePayload extends NotificationPayload {
  final String slotId;
  @override String encode() => jsonEncode({'type': 'radio_schedule', 'slotId': slotId});
}

// потом добавится:
class NewEpisodePayload extends NotificationPayload { ... }
И роутинг становится чистым switch по типу:


onDidReceiveNotificationResponse: (response) {
  final payload = NotificationPayload.decode(response.payload);
  switch (payload) {
    case RadioSchedulePayload(): router.push('/radio/schedule');
    case NewEpisodePayload(:final programId): router.push('/programs/$programId');
    case null: break;  // невалидный payload
  }
}
Нужно ли это прямо сейчас? Если единственный тип уведомлений на ближайшее время — радио-расписание, можно начать с простой строки-константы 'radio_schedule'. Но т.к. фича-флоу с уведомлениями в приложении почти всегда растёт (push-и из FCM, напоминания об эпизодах, промо), я бы сразу заложил типизированный payload — это 30 строк кода, а потом в одном месте расширяется.

Нужен сразу типизированный или начнём с константы?

оставим пока константу, потом если будут другие нотификешн сделаем рефактор
заранее усложнять не хочу
Ок, тогда финальная структура:


lib/core/notifications/
├── notification_service.dart    # init, requestPermissions,
│                                # scheduleAt(id, title, body, at, payload),
│                                # cancel(id), + колбэк тапа с константой payload
└── reminder_storage.dart        # по паттерну TokenStorage: init + in-memory Set + add/remove

lib/domain/state/
└── program_reminders_state.dart # StateNotifier<Set<String>>, оркестрирует storage + service

lib/features/radio/presentation/widgets/
└── reminder_button.dart         # consumer-виджет, toggle
Payload = константа 'radio_schedule' прямо в notification_service.dart. При тапе — роутим на экран расписания.

Готов расписать пошаговый план реализации (deps → permissions/манифест → NotificationService → ReminderStorage → Notifier → UI → deep-link), чтобы потом идти по шагам с твоим подтверждением на каждом?

а что за reminder_button.dart?
Read c:\flutter-projects\go_sport\lib\features\radio\presentation\radio\radio_page_screen.dart
reminder_button.dart — это UI-кнопка «подписаться/отписаться», которая будет рендериться на каждом тайле слота расписания (RadioScheduleSlot). Иконка колокольчика (выключенный/включённый), по тапу — toggle подписки.

Внутри примерно такое:


class ReminderButton extends ConsumerWidget {
  final RadioScheduleSlot slot;
  const ReminderButton({required this.slot, super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final reminders = ref.watch(programRemindersProvider);
    final isSubscribed = reminders.contains(slot.id);

    return IconButton(
      icon: Icon(isSubscribed ? Icons.notifications_active : Icons.notifications_none),
      onPressed: () => ref.read(programRemindersProvider.notifier).toggle(slot),
    );
  }
}
Вся логика (persist + schedule/cancel) — в нотифаере, кнопка тупая.

НО — я сейчас заметил момент, который меняет картину. В коде вижу, что в проекте экрана расписания ещё нет. В radio_page_screen.dart:69 роут /radio/schedule настроен, но самого экрана и виджета RadioScheduleSlot в lib/ пока нет (только в docs/DOMAIN_SPEC.md).

Два вопроса:

Экран расписания мы делаем в рамках этой же задачи (тогда reminder сразу встраиваем туда), или он будет отдельной задачей (тогда reminder-фича делается впрок без UI-точки монтирования)?

Как плитка слота будет выглядеть — кнопка напоминания вписывается как маленькая иконка справа, или как отдельный элемент на всю ширину? От этого зависит, нужен ли ReminderButton как отдельный виджет или достаточно вставить Consumer прямо внутрь SlotTile.

По твоему feedback_prefer_separate_widgets я бы всё-таки держал ReminderButton отдельным — per-feature виджет, не заваливая SlotTile флагами. Но если он используется только в одном месте и это реально одна иконка — можно обойтись inline-Consumer'ом внутри SlotTile.

Что с экраном расписания — делаем вместе или фича-напоминания живёт без UI, пока не придёт время?

экран расписаний есть
на ветрке шел радио есть экран расписание открывается ботом шит
тот же виджет использруется на жкране радиоплеера
проверь пожалуйста
Show less
Bash List all branches
IN
git branch -a

OUT
  audio
  carusel_changes
* dev
  fix
  login
  main
  merge
  origin
  playlist
  radio
  remotes/origin/HEAD -> origin/main
  remotes/origin/audio
  remotes/origin/carusel_changes
  remotes/origin/dev
  remotes/origin/login
  remotes/origin/main
  remotes/origin/merge
  remotes/origin/merge1
  remotes/origin/music_screen
  remotes/origin/playlist
  remotes/origin/radio
  remotes/origin/refactor
  remotes/origin/registration
  remotes/origin/user_profile
Globpattern: "lib/**/schedule*.dart"
Found 12 files
Globpattern: "lib/**/*schedule*.dart"
Found 12 files
Bash Find commits touching schedule files
IN
git log --all --oneline --source -- "lib/**/schedule*" 2>&1 | head -30

OUT
25b0f38	refs/heads/dev radio screen
3ab49ab	refs/remotes/origin/playlist some markup fix
d0eae97	refs/remotes/origin/music_screen radio schedule via selected date is done
afd08e7	refs/remotes/origin/music_screen radio schedule pop up
Read c:\flutter-projects\go_sport\lib\features\shared_widgets\schedule_tile.dart
Read c:\flutter-projects\go_sport\lib\features\shared_widgets\schedule_list.dart
Read c:\flutter-projects\go_sport\lib\domain\entities\scheduled_program.dart
Проверил — нашёл ключевое:

Сущность в коде называется ScheduledProgram (не RadioScheduleSlot, как в доках).
Тайл уже есть — lib/features/shared_widgets/schedule_tile.dart, и в нём уже нарисована кнопка подписки с DSNotificationIcon. Но это плейсхолдер: StatefulWidget с локальным _isSubscribed, который сбрасывается при перестроении и никуда не сохраняется (schedule_tile.dart:24, schedule_tile.dart:94-104).
Тайл лежит в shared_widgets/, значит одним местом используется и в bottom sheet расписания, и в плеере — твои слова подтвердились.
Это сильно упрощает план:

ReminderButton отдельным виджетом не нужен — кнопка уже внутри ScheduleTile. Достаточно переделать тайл из StatefulWidget в ConsumerWidget и привязать _isSubscribed к провайдеру.
Точка интеграции одна — тайл. Ни в bottom sheet, ни в плеере ничего менять не нужно, оба используют тот же ScheduleTile.
Финальная структура:


lib/core/notifications/
├── notification_service.dart
└── reminder_storage.dart

lib/domain/state/
└── program_reminders_state.dart       # StateNotifier<Set<String>>

lib/features/shared_widgets/
└── schedule_tile.dart                 # ← правим: Stateful → ConsumerWidget,
                                       #   _isSubscribed → watch(провайдер).contains(id)
Ещё один нюанс: у ScheduledProgram нет поля endDate — оно вычисляется из startDate + duration (scheduled_program.dart:18). Для планирования напоминания важен только startDate — всё ок, будем использовать startDate.subtract(Duration(minutes: 10)).

Готов раскладывать пошаговый план реализации?

почти давай еще обсудим как будем показывать что програма выбранна для напоминания?
из бэка это не приходит , нужно будет сферятся с локальными данными
Show less
Read c:\flutter-projects\go_sport\lib\features\shared_widgets\episode_tile.dart
Смотри — в проекте уже есть устоявшийся паттерн для похожего случая. EpisodeTile — тупой StatelessWidget, получает isPlaying как проп. Родитель (radio_page_screen.dart:148-178) сам читает playerStateProvider, сравнивает episode.id == playingTrackId и пробрасывает флаг вниз.

По этой логике для подписок есть два подхода:

А) Пробрасывать проп (по конвенции проекта)

ScheduleTile остаётся тупым, получает isSubscribed и onSubscribeToggle как пропсы. Родитель (ScheduleList) — ConsumerWidget, читает programRemindersProvider один раз, склеивает с каждой программой:


// в ScheduleList
final subscribed = ref.watch(programRemindersProvider);
...
ScheduleTile(
  program: program,
  isLive: ...,
  isSubscribed: subscribed.contains(program.id),
  onSubscribeToggle: () =>
      ref.read(programRemindersProvider.notifier).toggle(program),
)
Склейка = Set.contains(id) — O(1), проблем с производительностью нет даже на сотне тайлов.

Б) Тайл сам читает провайдер (ConsumerWidget с .select)

Каждый ScheduleTile через ref.watch(programRemindersProvider.select((s) => s.contains(program.id))) перестраивается только если именно его подписка переключилась. Меньше ребилдов, но ломает конвенцию «shared-тайл тупой».

Рекомендую А — по трём причинам:

Матчит существующий паттерн с isPlaying в EpisodeTile — код проекта остаётся однородным.
ScheduleTile живёт в shared_widgets/ и по смыслу не должен знать про конкретный домен «подписки на программы». Флаг + колбэк — чистый контракт.
Склейка в ScheduleList — одна точка, легко потом поменять источник (например, если в будущем подписки синкаются с бэком, тайл вообще не трогаем).
Перфоманс: да, ребилдится весь список при тапе. Но Set.contains дешёвый, а список обычно 10-30 элементов. Если когда-нибудь станет узким местом — перейдём на Б.

Ок с вариантом А?

isPlaying  только в EpisodeTile  или треках тоже?
Grep "isPlaying" (in c:\flutter-projects\go_sport\lib\features\shared_widgets)
10 lines of output
Globpattern: "lib/features/shared_widgets/*tile*.dart"
Found 7 files
И там, и там — одинаковый паттерн:

episode_tile.dart:14 — final bool? isPlaying
track_tile.dart:12 — final bool? isPlaying
Оба — StatelessWidget, флаг приходит пропсом от родителя. Конвенция последовательна: все тайлы в shared_widgets/ тупые, state-зависимые флаги пробрасываются сверху.

Значит вариант А (проп + колбэк) — прямое продолжение этой конвенции. Делаем так?

давай еще чуток подумаем чтобы быть уверенными что обсудили все нюансы
тут есть разница
трек не может управдыть своим состоянием, отобрадается из мрлеинг емли в играет этот трек

для прорам есть ыункция отписки
это не меняет ситуацию?
Show less
Хорошее замечание, но ситуацию не меняет — вот почему:

EpisodeTile и TrackTile тоже не чисто «только отображают». У них есть onTap (сыграть трек) и onMenuTap (открыть меню трека) — это такие же mutation-действия, как тап по колокольчику. Смотри episode_tile.dart:12-13:


final VoidCallback onTap;      // мутация: запуск трека в плеере
final VoidCallback onMenuTap;  // мутация: открытие меню
Паттерн в проекте: тайл не знает про провайдер, он получает и state-флаг, и action-колбэк сверху. Родитель оркестрирует оба: читает провайдер для state, дёргает нотифаер в колбэке. Для подписки это ровно та же форма:


ScheduleTile(
  program: program,
  isLive: ...,
  isSubscribed: subscribed.contains(program.id),  // ← как isPlaying
  onSubscribeToggle: () =>                        // ← как onTap
      ref.read(programRemindersProvider.notifier).toggle(program),
)
Тем не менее, один нюанс в пользу варианта Б стоит упомянуть:

При подписке колбэк всегда один и тот же — toggle(program). Это не индивидуальный per-tile onTap (где в radio_page_screen колбэк закрывается над index и списком). Это делает аргумент «родитель владеет действием» чуть слабее: тут действие обобщённое, его можно смело колокейтить с иконкой внутри тайла.

Поэтому если хочется строгой когезии («всё, что связано с иконкой-колокольчиком, — в одном файле»), вариант Б тоже честный выбор. Но тогда появляется исключение из конвенции и дальше придётся объяснять, почему isSubscribed делается иначе, чем isPlaying.

Моё мнение остаётся: вариант А по конвенции. Функциональная разница (мутация vs чтение) — важная мысль, но проект уже показал, что мутации пробрасываются колбэками и это норма. Пусть тайл остаётся тупым, ScheduleList оркестрирует.

Ок двигаемся с А?

да двигаемся с А
Отлично. Тогда вот пошаговый план — иду по одному шагу, на каждом жду твоё «ок» перед следующим.

Шаг 1 — зависимости
Добавить в pubspec.yaml:

flutter_local_notifications — планирование/показ уведомлений
timezone — зависимость для zonedSchedule
shared_preferences — для ReminderStorage
Шаг 2 — Android манифест
В android/app/src/main/AndroidManifest.xml:

POST_NOTIFICATIONS
SCHEDULE_EXACT_ALARM (под inexact fallback нам формально не нужен, но проще оставить; можно и без него)
RECEIVE_BOOT_COMPLETED Плюс <receiver> из доки flutter_local_notifications для восстановления уведомлений после ребута.
Шаг 3 — NotificationService
lib/core/notifications/notification_service.dart:

init() — инициализация плагина, timezone, регистрация колбэка тапа
requestPermissions() — iOS + Android 13+
scheduleAt({id, title, body, at, payload}) — zonedSchedule с inexactAllowWhileIdle
cancel(id) / cancelAll()
Внутри колбэка тапа: если payload == 'radio_schedule' → пуш роута через go_router
Провайдер с override в ProviderScope
Шаг 4 — ReminderStorage
lib/core/notifications/reminder_storage.dart по паттерну TokenStorage:

init() — читает Set<String> из SharedPreferences в in-memory кеш
Синхронный геттер Set<String> get subscribedIds
add(id) / remove(id) — обновляют кеш + пишут
Провайдер с override
Шаг 5 — ProgramRemindersNotifier
lib/domain/state/program_reminders_state.dart:

StateNotifier<Set<String>>, state = копия storage.subscribedIds
toggle(ScheduledProgram program) — если подписан: storage.remove + service.cancel; если нет: storage.add + service.scheduleAt на startDate - 10 мин
Провайдер
Шаг 6 — инициализация на старте
В main.dart до runApp:

NotificationService.init()
ReminderStorage.init()
requestPermissions()
Проброс override'ов в ProviderScope
Обработка getNotificationAppLaunchDetails (если апп открыли с killed-состояния тапом по уведомлению)
Шаг 7 — ScheduleTile

StatefulWidget → StatelessWidget
Добавить пропсы isSubscribed: bool и onSubscribeToggle: VoidCallback
Убрать _isSubscribed / setState
Шаг 8 — ScheduleList

ref.watch(programRemindersProvider)
Пробросить isSubscribed и onSubscribeToggle в каждый ScheduleTile
Шаг 9 — граничные кейсы

Если startDate - 10 мин уже в прошлом → в toggle не планируем уведомление, но подписку сохраняем (чтобы иконка отображалась). Или сразу отказываем — обсудим.
Deep-link из killed-состояния.
Начинаем с Шага 1?

а нам нужен timezone ?
Да, нужен — это не наша прихоть, это требование flutter_local_notifications.

Плагин с версии 9.x депрекейтнул schedule(DateTime) и оставил только zonedSchedule(TZDateTime, ...) — принимает tz.TZDateTime, а не обычный DateTime. Причины — корректная обработка DST и смены временной зоны устройством. То есть без timezone просто нечем передать момент времени в API.

Минимум, что с ним делаем:


import 'package:timezone/data/latest_all.dart' as tz_data;
import 'package:timezone/timezone.dart' as tz;

// один раз на старте (в NotificationService.init):
tz_data.initializeTimeZones();

// при планировании:
final scheduledAt = tz.TZDateTime.from(
  program.startDate.subtract(const Duration(minutes: 10)),
  tz.local,
);
Один нюанс: tz.local по умолчанию = UTC, пока явно не установишь локальную зону. Чтобы получить её имя с устройства, обычно ставят ещё пакет flutter_timezone и:


final name = await FlutterTimezone.getLocalTimezone();
tz.setLocalLocation(tz.getLocation(name));
Но для нашего кейса можно обойтись без него: мы планируем на близкий момент (часы, не недели), сравниваем с DateTime.now(), и используем TZDateTime.from(dateTime, tz.local). Даже если tz.local = UTC, .from() корректно конвертирует — итоговый момент в абсолютном времени тот же. DST переходы в ±часовом окне тоже не страшны.

Итог: timezone — да, обязательно. flutter_timezone — нет, пропускаем.

Ок, остаёмся с двумя пакетами в шаге 1? Или дополнительно добавить flutter_timezone для перестраховки?

почему с двумя? 3 эе пакета
Моя опечатка — конечно три: flutter_local_notifications, timezone, shared_preferences. flutter_timezone не берём.

Добавляем все три — идём на Шаг 1?

да пошли
Edit c:\flutter-projects\go_sport\pubspec.yaml
Added 7 lines
Bash Resolve new dependencies
IN
flutter pub get 2>&1 | tail -20

OUT
/usr/bin/bash: flutter: command not found
PowerShell
IN
{
  "command": "flutter pub get 2>&1 | Select-Object -Last 20",
  "description": "Resolve new dependencies via PowerShell",
  "timeout": 180000
}
OUT
Exit code 1
flutter : The term 'flutter' is not recognized as the name of a cmdlet, function, script file, or operable program. 
Check the spelling of the name, or if a path was included, verify that the path is correct and try again.
At line:1 char:1
+ flutter pub get 2>&1 | Select-Object -Last 20
+ ~~~~~~~
    + CategoryInfo          : ObjectNotFound: (flutter:String) [], CommandNotFoundException
    + FullyQualifiedErrorId : CommandNotFoundException
Добавил три пакета в pubspec.yaml:33-39:


# Notifications
flutter_local_notifications: ^17.2.3
timezone: ^0.9.4

# Local storage
shared_preferences: ^2.3.2
flutter не в PATH у моего шелла — запусти локально сам:


flutter pub get
Как получишь успешный резолв — двигаемся к Шагу 2 (Android-манифест).

а что добавляем в манифест?
Read c:\flutter-projects\go_sport\android\app\src\main\AndroidManifest.xml
Смотри, что уже есть и что надо добавить.

Уже есть (AndroidManifest.xml:5):

POST_NOTIFICATIONS — дублировать не нужно
WAKE_LOCK — тоже пригодится плагину
Нужно добавить:

RECEIVE_BOOT_COMPLETED — чтобы запланированные уведомления пережили перезагрузку устройства. Плагин при бутe сам перерегистрирует всё, что было запланировано, но только если ему разрешён приём этого бродкаста.

Два <receiver> внутри <application>, которые поставляет flutter_local_notifications:


<receiver android:exported="false"
    android:name="com.dexterous.flutterlocalnotifications.ScheduledNotificationReceiver" />
<receiver android:exported="false"
    android:name="com.dexterous.flutterlocalnotifications.ScheduledNotificationBootReceiver">
    <intent-filter>
        <action android:name="android.intent.action.BOOT_COMPLETED" />
        <action android:name="android.intent.action.MY_PACKAGE_REPLACED" />
    </intent-filter>
</receiver>
Первый — принимает alarm от системы в запланированный момент и показывает уведомление.
Второй — после ребута/обновления апы перерегистрирует всё, что было запланировано.
Что НЕ добавляем (договорились):

SCHEDULE_EXACT_ALARM / USE_EXACT_ALARM — используем inexact, точный аларм не нужен.
Итого правка — одна новая пермишшн и один <receiver>-блок из двух ресиверов. Подтверждаешь — применяю?

для os ничего не нужно?
Globpattern: "ios/Runner/AppDelegate.*"
Found 1 file
Read c:\flutter-projects\go_sport\ios\Runner\AppDelegate.swift
Немного, но кое-что есть.

Текущий AppDelegate.swift минимальный. Для flutter_local_notifications на iOS стандартная рекомендация — зарегистрировать делегат UNUserNotificationCenter. Без этого, если уведомление срабатывает пока апп на переднем плане, iOS его по дефолту скрывает (только иконка-бейдж). С делегатом плагин сам решает показывать или нет.

Диффф такой:


import Flutter
import UIKit
import UserNotifications   // ← добавить

@main
@objc class AppDelegate: FlutterAppDelegate {
  override func application(
    _ application: UIApplication,
    didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?
  ) -> Bool {
    // ↓ добавить эти две строки
    if #available(iOS 10.0, *) {
      UNUserNotificationCenter.current().delegate = self as? UNUserNotificationCenterDelegate
    }
    GeneratedPluginRegistrant.register(with: self)
    return super.application(application, didFinishLaunchingWithOptions: launchOptions)
  }
}
Что НЕ нужно на iOS:

Ничего в Info.plist под разрешения — iOS-диалог про уведомления показывается автоматом при вызове requestPermissions() из Dart.
Background modes — не используем, уведомления локальные и запланированные, iOS сам их доставит без работающего приложения.
Ещё один момент — для iOS-симулятора доставка локальных уведомлений работает с некоторыми оговорками в разных версиях Xcode. На реальном девайсе всё предсказуемо. Отмечаю, чтобы потом не удивляться при тестировании.

Итого Шаг 2 разрастается на две правки: Android-манифест + AppDelegate.swift. Применяю?

да давй
Edit c:\flutter-projects\go_sport\android\app\src\main\AndroidManifest.xml
Added 1 line
Edit c:\flutter-projects\go_sport\android\app\src\main\AndroidManifest.xml
Added 10 lines
Edit c:\flutter-projects\go_sport\ios\Runner\AppDelegate.swift
Added 4 lines
Готово:

AndroidManifest.xml:6 — RECEIVE_BOOT_COMPLETED
AndroidManifest.xml:49-58 — два ресивера плагина
AppDelegate.swift — import UserNotifications + регистрация делегата
Идём на Шаг 3 — NotificationService?

да давай
Прежде чем писать — один точечный вопрос, чтобы не пришлось переделывать. Как сервис сообщает про тап по уведомлению наружу?

Вариант 1 — Stream<String> onTap (мой выбор)
Сервис эмитит payload в стрим. main.dart или корневой виджет подписывается и пушит роут через go_router. Сервис ничего не знает про навигацию.

Вариант 2 — колбэк в init(onTap: ...)
Проще, но ProviderScope на момент init ещё не готов, а нам в колбэке нужен доступ к роутеру. Придётся через ProviderContainer или глобалку — грязновато.

Stream чище — идём с ним?

Если ок, напишу файл со следующим API:

init() — initializeTimeZones, инициализация плагина с Android + iOS settings, колбэк onDidReceiveNotificationResponse → стрим
requestPermissions() — iOS + Android 13+
scheduleAt({id, title, body, at, payload}) — zonedSchedule с inexactAllowWhileIdle
cancel(id)
getInitialPayload() — возвращает payload, если апп запустили тапом из killed-состояния
Константа kRadioSchedulePayload = 'radio_schedule'
Провайдер notificationServiceProvider с throw UnimplementedError (override в ProviderScope)
Ок?

а через  через Riverpod? что думаешь?
Отличная идея, даже лучше получится. Riverpod-native — обернуть внутренний стрим сервиса в StreamProvider, и дальше всё через ref.listen. Плюсы:

Сервис остаётся чистым Dart-классом, ничего не знает про Riverpod/роутер.
Потребитель (главный виджет или оболочка навигации) использует ref.listen — без ручной подписки/отписки на StreamSubscription.
Тестируется через override провайдера.
Выглядит так:


// внутри notification_service.dart
class NotificationService {
  final _tapController = StreamController<String>.broadcast();
  Stream<String> get onTap => _tapController.stream;
  // ...
}

final notificationServiceProvider = Provider<NotificationService>(
  (_) => throw UnimplementedError('Override in ProviderScope'),
);

final notificationTapProvider = StreamProvider<String>((ref) {
  return ref.watch(notificationServiceProvider).onTap;
});
Потом в корневом виджете (позже, в Шаге 6):


ref.listen(notificationTapProvider, (prev, next) {
  next.whenData((payload) {
    if (payload == kRadioSchedulePayload) {
      router.push('/radio/schedule');
    }
  });
});
Для killed-state launch (getNotificationAppLaunchDetails) — логика отдельная, в main.dart перед runApp, это тоже будет в Шаге 6.

Итого меняется: в сервисе есть и Stream<String> get onTap, и дополнительный StreamProvider<String> рядом с основным провайдером. Две строчки, зато единообразно с Riverpod.

Пишу?

не знаю не уверен
вариант со стрим выбрать или с риверпод
Объясню проще — это не два противопоставленных варианта, а один базовый механизм (Stream) и опциональная обёртка над ним.

Без обёртки — потребитель подписывается на сырой стрим руками:


late StreamSubscription _sub;

@override
void initState() {
  super.initState();
  _sub = ref.read(notificationServiceProvider).onTap.listen((payload) {
    if (payload == kRadioSchedulePayload) router.push('/radio/schedule');
  });
}

@override
void dispose() {
  _sub.cancel();
  super.dispose();
}
С обёрткой в StreamProvider — потребитель использует ref.listen как везде в проекте:


ref.listen(notificationTapProvider, (prev, next) {
  next.whenData((payload) {
    if (payload == kRadioSchedulePayload) router.push('/radio/schedule');
  });
});
Что даёт обёртка:

Не нужно заводить StreamSubscription-поле и вручную звать cancel() в dispose — Riverpod управляет жизненным циклом.
Единообразно со всем остальным кодом: везде ref.watch / ref.listen, нигде вручную со StreamSubscription не возятся.
Цена — одна доп. строка в notification_service.dart.
Моя рекомендация — с Riverpod-обёрткой. Цена одна строка, выгода — нет ручного управления подпиской и нет исключения из общей модели работы с состоянием в проекте. В коде проекта нигде не видел подписок Stream.listen — везде Riverpod, и это удобно держать единообразным.

Идём с Riverpod-вариантом?

когда я говорил про ривераод я думал о таком варианте
через Riverpod (pendingNotificationProvider)
NotificationService при тапе пишет payload в простой StateProvider. App виджет его слушает и реагирует когда роутер уже готов.
Show less
А это интересный паттерн — и для нашего случая он лучше, чем стрим. Объясню почему.

Разница в семантике:

StreamProvider	StateProvider<String?> (pending)
Если слушатель ещё не готов	событие теряется	значение остаётся в state, ждёт консюмера
Killed-state launch	нужен отдельный метод getInitialPayload()	тоже пишет в этот же провайдер — единый поток
Обработка	реактивная	консюмер сам решает когда сбросить
Почему для нас state лучше:

Unified flow. И тап во время работы апы, и старт из killed-состояния через getNotificationAppLaunchDetails() — оба случая просто пишут payload в один провайдер. Не нужен отдельный публичный метод для initial payload.

Нет гонки «тап vs router-ready». Если железо успевает тапнуть по уведомлению до монтирования MaterialApp с роутером (редко, но возможно на iOS из killed), событие не теряется — лежит в state, консюмер подхватит когда будет готов.

Проще тестировать — state-провайдеры переопределяются и читаются тривиально.

Как это меняет сервис:

Единственный компромисс — NotificationService теперь должен уметь записать в провайдер, то есть принять Ref в конструктор:


class NotificationService {
  final Ref _ref;
  NotificationService(this._ref);

  Future<void> init() async {
    tz_data.initializeTimeZones();
    // ...
    await _plugin.initialize(
      settings,
      onDidReceiveNotificationResponse: (response) {
        final payload = response.payload;
        if (payload != null) {
          _ref.read(pendingNotificationProvider.notifier).state = payload;
        }
      },
    );

    // killed-state launch — тот же путь
    final details = await _plugin.getNotificationAppLaunchDetails();
    if (details?.didNotificationLaunchApp ?? false) {
      final payload = details?.notificationResponse?.payload;
      if (payload != null) {
        _ref.read(pendingNotificationProvider.notifier).state = payload;
      }
    }
  }
  // ...
}

final pendingNotificationProvider = StateProvider<String?>((ref) => null);
И на стороне консюмера (оболочка навигации или корневой виджет) один ref.listen:


ref.listen(pendingNotificationProvider, (prev, next) {
  if (next == null) return;
  if (next == kRadioSchedulePayload) {
    router.push('/radio/schedule');
  }
  ref.read(pendingNotificationProvider.notifier).state = null;  // сбросили
});
Это делает ненужным отдельный метод getInitialPayload() — init сам всё в провайдер кладёт. Код сервиса чище.

Согласен, что это лучший вариант. Идём так?

вто что советует клод
через Riverpod тоже рабочий, но избыточный для этой задачи.
StateProvider<String?> — это по сути тот же "стрим" но через Riverpod. Проблема в том что это состояние, а не событие. После обработки нужно вручную сбрасывать в null, иначе при пересоздании виджета роутер снова среагирует на старый payload.
Stream семантически точнее — тап это событие, не состояние.
Вариант C имел бы смысл если бы NotificationService уже был Riverpod провайдером и нужно было шарить между несколькими нотифайерами. В нашем случае это лишнее.
Show less
Согласен с этим Клодом — он прав по существу. Мои аргументы за StateProvider были слабее.

Главный его аргумент бьёт в точку: тап — это событие, а не состояние. StateProvider заставляет нас моделировать дискретное событие через долгоживущую переменную, из-за чего появляются искусственные проблемы — ручной сброс в null, потенциальный дедуп при одинаковых payload'ах. Это лечится дисциплиной, но зачем, если семантика стрима бесплатно делает правильно.

Одна мелочь, в которой тот Клод немного преувеличил: ref.listen не срабатывает при маунте для текущего значения — только на переходы. Так что «при пересоздании виджета роутер снова среагирует на старый payload» — нет, не среагирует (если экземпляр листенера новый). Реальная же боль — дедуп одинаковых значений: если не сбросить state, второй тап с тем же payload не триггернёт листенер (Riverpod не уведомляет при state = sameValue).

По итогу — возвращаемся к варианту со стримом.

Про killed-launch (единственное, что удобнее было у state-подхода): делаем просто — NotificationService выставляет публичное поле String? initialPayload после init, и main.dart однократно проверяет его после сборки роутера:


class NotificationService {
  String? _initialPayload;
  String? get initialPayload => _initialPayload;

  final _tapController = StreamController<String>.broadcast();
  Stream<String> get onTap => _tapController.stream;

  Future<void> init() async {
    tz_data.initializeTimeZones();
    // ... initialize plugin ...
    final details = await _plugin.getNotificationAppLaunchDetails();
    if (details?.didNotificationLaunchApp ?? false) {
      _initialPayload = details?.notificationResponse?.payload;
    }
  }
}

final notificationTapProvider = StreamProvider<String>((ref) {
  return ref.watch(notificationServiceProvider).onTap;
});
И корневой виджет:

на ref.listen(notificationTapProvider, ...) ловит рантайм-тапы
один раз в initState / после сборки роутера читает service.initialPayload и обрабатывает, если не null
Ок, идём с этим?

да идем со стрим
Write c:\flutter-projects\go_sport\lib\core\notifications\notification_service.dart
115 lines
Edit c:\flutter-projects\go_sport\lib\core\notifications\notification_service.dart
Added 2 lines
Добавил uiLocalNotificationDateInterpretation: absoluteTime — значит, установилась более старая версия плагина, где этот параметр ещё обязательный. Работает корректно в обоих случаях.

Сервис готов: lib/core/notifications/notification_service.dart.

Краткая сводка содержимого:

Константа kRadioSchedulePayload = 'radio_schedule'
notificationServiceProvider — override в ProviderScope
notificationTapProvider — StreamProvider<String> обёртка над onTap
NotificationService.init() — timezone, плагин, колбэк тапа → стрим, captures killed-state launch payload
requestPermissions() — iOS + Android 13+
scheduleAt(...) — zonedSchedule с inexact
cancel(id) / cancelAll()
Запусти flutter analyze у себя — проверь, что других диагностик нет. Идём на Шаг 4 — ReminderStorage?

да
Write c:\flutter-projects\go_sport\lib\core\notifications\reminder_storage.dart
47 lines
Готово: lib/core/notifications/reminder_storage.dart.

По паттерну TokenStorage:

Конкретный класс, без интерфейса
init() подгружает Set<String> из SharedPreferences в in-memory кеш
Синхронный геттер subscribedIds возвращает unmodifiable-view (защита от случайной мутации снаружи)
add(id) / remove(id) — обновляют кеш и пишут на диск только если множество реально изменилось
Провайдер с throw UnimplementedError — override в ProviderScope на Шаге 6
Идём на Шаг 5 — ProgramRemindersNotifier?

да
Один момент уточнить перед написанием — граничный кейс «программа стартует через <10 минут»: что делаем?

Варианты:

A) Сохраняем подписку (иконка активна), но уведомление не планируем — значит оно не придёт
B) Планируем на ближайший момент (почти сразу стрельнёт)
C) Показываем уведомление сразу через show()
По умолчанию возьму A — самое простое и честное: если юзер подписался поздно, уведомления не будет, но состояние корректное. Ок?

Также про текст уведомления — UI в проекте английский ("Radio", "Featured programs", "Today"), так что возьму:

title: program.title
body: "Starts in 10 minutes"
Если ок — пишу файл.

да ок
Write c:\flutter-projects\go_sport\lib\domain\state\program_reminders_state.dart
58 lines
Готово: lib/domain/state/program_reminders_state.dart.

programRemindersProvider — StateNotifierProvider<ProgramRemindersNotifier, Set<String>>
toggle(program) — единственный публичный метод
Нотифайер сам решает subscribe/unsubscribe по текущему state
Subscribe = storage.add + scheduleAt (только если startDate - 10 мин ещё в будущем)
Unsubscribe = storage.remove + cancel (cancel безопасен — не падает, если уведомления нет)
notificationId = slotId.hashCode — стабильно для одного slotId
Идём на Шаг 6 — инициализация на старте в main.dart + override провайдеров + обработка initialPayload?

что за  override провайдеров ?
Read c:\flutter-projects\go_sport\lib\main.dart
Это Riverpod-паттерн, уже вовсю используется в main.dart:38-43:


runApp(
  ProviderScope(
    overrides: [
      audioHandlerProvider.overrideWithValue(audioHandler),
      apiClientProvider.overrideWithValue(apiClient),
      tokenStorageProvider.overrideWithValue(tokenStorage),
    ],
    child: ...
  ),
);
Зачем это вообще:

Наши notificationServiceProvider и reminderStorageProvider объявлены так:


final notificationServiceProvider = Provider<NotificationService>(
  (_) => throw UnimplementedError('must be overridden in ProviderScope'),
);
Они не умеют сами себя создать — потому что для создания нужен асинхронный init (SharedPreferences.getInstance(), tz_data.initializeTimeZones(), плагин), а фабрика Provider синхронная.

Поэтому схема такая:

В main() до runApp создаём инстансы руками и вызываем await init().
Передаём их в ProviderScope.overrides — тогда при первом ref.watch(provider) Riverpod отдаст наш уже готовый инстанс вместо throw.
Ровно так же уже делается с TokenStorage:

main.dart:20-21 — final tokenStorage = TokenStorage(); await tokenStorage.init();
main.dart:42 — tokenStorageProvider.overrideWithValue(tokenStorage)
Для нас это будет:


// до runApp:
final container = ProviderContainer();  // нужен, чтобы NotificationService мог писать в другие провайдеры — хотя нам сейчас не нужен, можно обойтись
final notificationService = NotificationService();
await notificationService.init();
await notificationService.requestPermissions();

final reminderStorage = ReminderStorage();
await reminderStorage.init();

// в overrides:
notificationServiceProvider.overrideWithValue(notificationService),
reminderStorageProvider.overrideWithValue(reminderStorage),
Плюс в корневом виджете — обработка рантайм-тапов через ref.listen(notificationTapProvider, ...) и однократная проверка notificationService.initialPayload после монтирования роутера.

Это и есть весь Шаг 6. Пишу?

да
Read c:\flutter-projects\go_sport\lib\core\navigation\app_router.dart
Read c:\flutter-projects\go_sport\lib\core\navigation\routes.dart
Изменения будут такие:

В main() — создать и инициализировать NotificationService и ReminderStorage до runApp, добавить два override'а.
Преобразовать MainApp из StatelessWidget в ConsumerStatefulWidget:
Хранить инстанс GoRouter в поле (раньше создавался в build — в этом же заходе это исправим, получится чище).
В initState через addPostFrameCallback прочитать initialPayload и, если там kRadioSchedulePayload, запушить расписание.
В build через ref.listen(notificationTapProvider, ...) ловить рантайм-тапы и тоже пушить.
Путь возьмём из константы AppRoutes.radioSchedule.

Пишу.

Write c:\flutter-projects\go_sport\lib\main.dart
127 lines
Готово: main.dart.

Что сделал:

Создание + init NotificationService и ReminderStorage до runApp (main.dart:28-34)
Два новых override'а в ProviderScope (main.dart:55-56)
MainApp → ConsumerStatefulWidget, роутер теперь создаётся один раз в initState (побочный бонус — раньше пересоздавался на каждом билде)
В initState через addPostFrameCallback читаем initialPayload → пушим роут при killed-state launch
В build через ref.listen(notificationTapProvider, ...) ловим рантайм-тапы
После flutter pub get проверь, что проект собирается. Идём на Шаг 7 — переделать ScheduleTile?

c:\flutter-projects\go_sport\lib\core\audio\app_audio_handler.dart
что будем переделывать в ScheduleTile?
В schedule_tile.dart три правки:

StatefulWidget → StatelessWidget. Локальный state здесь не нужен — теперь приходит сверху.

Убрать _isSubscribed и setState (schedule_tile.dart:24, schedule_tile.dart:95).

Добавить два required пропса:

isSubscribed: bool — идёт в DSNotificationIcon.isFilled
onSubscribeToggle: VoidCallback — дёргается на тап по иконке
До:


class ScheduleTile extends StatefulWidget {
  final ScheduledProgram program;
  final bool isLive;
  ...
}

// State:
bool _isSubscribed = false;
onTap: () => setState(() => _isSubscribed = !_isSubscribed),
DSNotificationIcon(isFilled: _isSubscribed)
После:


class ScheduleTile extends StatelessWidget {
  final ScheduledProgram program;
  final bool isLive;
  final bool isSubscribed;
  final VoidCallback onSubscribeToggle;
  
  const ScheduleTile({
    super.key,
    required this.program,
    this.isLive = false,
    required this.isSubscribed,
    required this.onSubscribeToggle,
  });
  ...
}

// build:
onTap: onSubscribeToggle,
DSNotificationIcon(isFilled: isSubscribed)
Контракт тайла матчит конвенцию проекта (EpisodeTile.isPlaying + EpisodeTile.onTap). Применяю?

FAILURE: Build failed with an exception.

* What went wrong:
Execution failed for task ':app:checkDebugAarMetadata'.
> A failure occurred while executing com.android.build.gradle.internal.tasks.CheckAarMetadataWorkAction
   > An issue was found when checking AAR metadata:

       1.  Dependency ':flutter_local_notifications' requires core library desugaring to be enabled
           for :app.

           See https://developer.android.com/studio/write/java8-support.html for more
           details.

* Try:
> Run with --stacktrace option to get the stack trace.
> Run with --info or --debug option to get more log output.
> Run with --scan to get full insights.
> Get more help at https://help.gradle.org.

BUILD FAILED in 1m 43s
Error: Gradle task assembleDebug failed with exit code 1
Globpattern: "android/app/build.gradle*"
Found 1 file
Read c:\flutter-projects\go_sport\android\app\build.gradle.kts
Стандартное требование flutter_local_notifications 17+ — нужно включить core library desugaring. Две правки в android/app/build.gradle.kts: флаг в compileOptions + блок dependencies. Применяю.

Edit c:\flutter-projects\go_sport\android\app\build.gradle.kts
Added 1 line
Edit c:\flutter-projects\go_sport\android\app\build.gradle.kts
Added 4 lines
Готово — две правки в android/app/build.gradle.kts:

isCoreLibraryDesugaringEnabled = true в compileOptions
coreLibraryDesugaring("com.android.tools:desugar_jdk_libs:2.0.4") в dependencies
Попробуй снова:


flutter clean && flutter pub get && flutter run
flutter clean нужен, чтобы gradle подхватил новые опции. Отпишись, если соберётся — продолжим с Шагом 7 (ScheduleTile).

c#11-13
собралось
Отлично. Тогда возвращаюсь к Шагу 7 — правка ScheduleTile. Напомню, описывал её выше: StatefulWidget → StatelessWidget, убираем _isSubscribed/setState, добавляем пропсы isSubscribed и onSubscribeToggle. Применяю?

да
Write c:\flutter-projects\go_sport\lib\features\shared_widgets\schedule_tile.dart
106 lines
Grep "ScheduleTile\(" (in c:\flutter-projects\go_sport\lib)
15 lines of output
Единственный кол-сайт — schedule_list.dart:40. Чтобы проект собирался, правку ScheduleList (Шаг 8) делаю сразу следом:

Edit c:\flutter-projects\go_sport\lib\features\shared_widgets\schedule_list.dart
Added 1 line
Edit c:\flutter-projects\go_sport\lib\features\shared_widgets\schedule_list.dart
Added 6 lines
Готово. Правки:

schedule_tile.dart — Stateful → Stateless, новые пропсы isSubscribed и onSubscribeToggle, без локального state
schedule_list.dart — ref.watch(programRemindersProvider) + проброс флага и колбэка в тайл
Основная реализация фичи готова. Запускай — тапай колокольчик, должна сохраняться подписка и планироваться уведомление за 10 мин до старта программы.

Что осталось из плана (Шаг 9, граничные кейсы):

Слоты в прошлом — отдельно не обрабатывали; кнопка сейчас реагирует на любой слот. Если тапнуть уже прошедший — подписка сохранится, но уведомление не запланируется (защита есть в нотифайере). Визуально никак не отличается. Стоит скрывать/дизейблить колокольчик для прошлых программ?
Проверка deep-link из killed-state — протестируй на девайсе (эмулятор iOS может глючить).
Как тебе результат?