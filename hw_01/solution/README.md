# ДЗ 1. Анализ референсной архитектуры — решение

> Условие: [../Задание.md](../Задание.md) · Чек-лист: [../Чеклист.md](../Чеклист.md)
> Схемы лежат в [diagrams/](diagrams/) (PNG для просмотра, PlantUML-исходники для правок).

## Выбранная архитектура

Сервис такси (Uber / Яндекс Go).

Система сводит в одной сделке трёх акторов: пассажира, водителя и платформу. Отдельно стоит оператор поддержки, у него свой админский интерфейс. Пассажир заказывает поездку, водитель её принимает и везёт, платформа отвечает за подбор, расчёт цены, оплату и уведомления.

**Оценка масштаба (порядок величин).**

| Параметр | Значение | Как посчитано |
|---|---|---|
| Активные пользователи (DAU) | ~12 млн | оценка по аудитории крупного сервиса |
| Одновременных поездок в пик | ~250k | доля активных пользователей в час пик |
| Read от пассажиров | ~83k RPS | 250k × 0.33 (запрос раз в ~3 сек) |
| Водителей онлайн в пик | ~1 млн | соотношение водитель/пассажир ~1:10 |
| Write геолокации водителей | ~250k RPS | 1 млн × 0.25 (запись раз в ~3–4 сек) |
| **Суммарно в пик** | **~340k RPS** | 83k (read) + 250k (write) |

Главный вывод из оценки: доминирует поток записи геолокации (~250k RPS). Это сразу диктует выбор хранилища для гео (in-memory слой вместо построчной записи в реляционную БД).

### Схема архитектуры (Container)

![Архитектура сервиса такси](diagrams/taxi-architecture.png)

<details>
<summary>PlantUML-исходник</summary>

```plantuml
@startuml TaxiArchitecture
title Сервис такси — архитектура (Container)

skinparam shadowing false
skinparam defaultFontName Helvetica
skinparam rectangle {
  BackgroundColor #4C84C4
  FontColor white
  BorderColor #2C5A8C
}
skinparam database {
  BackgroundColor #2E7D5B
  FontColor white
  BorderColor #1B5E3A
}
skinparam queue {
  BackgroundColor #B8762E
  FontColor white
  BorderColor #8C5A1F
}
skinparam actor {
  BackgroundColor #1B3A5C
  BorderColor #1B3A5C
}
skinparam node {
  BackgroundColor #9E9E9E
  FontColor white
  BorderColor #707070
}
skinparam ArrowColor #555555
skinparam ArrowFontColor #333333

actor "Пассажир" as passenger
actor "Водитель" as driver

rectangle "Client App\n<size:11><i>Android / iOS / Web</i></size>" as client
rectangle "API Gateway\n<size:11><i>stateless</i></size>" as gateway

package "Платформа такси" {
  rectangle "User Service" as userSvc
  database "User DB\n<size:11><i>PostgreSQL</i></size>" as userDb

  rectangle "Driver Service" as driverSvc
  database "Driver DB\n<size:11><i>PostgreSQL</i></size>" as driverDb

  rectangle "Verification Service" as verifySvc

  rectangle "Ride Service" as rideSvc
  database "Ride DB\n<size:11><i>PostgreSQL</i></size>" as rideDb

  rectangle "Matching Service" as matchSvc
  database "Geo Store\n<size:11><i>Redis Geo</i></size>" as geoStore
  database "Matching DB\n<size:11><i>PostgreSQL</i></size>" as matchDb

  rectangle "Payment Service\n<size:11><i>Saga</i></size>" as paySvc
  database "Payment DB\n<size:11><i>PostgreSQL</i></size>" as payDb

  rectangle "Support Service" as support

  rectangle "Notification Service\n<size:11><i>stateless</i></size>" as notifySvc
  database "Notification DB\n<size:11><i>PostgreSQL</i></size>" as notifyDb

  queue "Message Broker\n<size:11><i>Kafka</i></size>" as kafka
}

node "Гос. сервисы" as govApi
node "Платёжные провайдеры" as payProvider
node "Каналы доставки\n<size:11><i>Push / SMS / Email</i></size>" as channels
node "LDAP\n<size:11><i>каталог учётных записей</i></size>" as ldap

passenger --> client : использует
driver --> client : использует
client --> gateway : HTTPS / REST
gateway --> ldap : аутентификация (sync)

gateway --> userSvc : sync
gateway --> driverSvc : sync
gateway --> rideSvc : sync
gateway --> paySvc : sync
gateway --> support : sync
gateway ..> matchSvc : геолокация (async)

userSvc --> userDb
userSvc ..> kafka : события

driverSvc --> driverDb
driverSvc ..> kafka : события
kafka ..> verifySvc : запрос верификации
verifySvc ..> kafka : результат верификации
verifySvc --> govApi : REST / SOAP

rideSvc --> rideDb
rideSvc ..> kafka : события поездки
kafka ..> matchSvc : запрос на матчинг

matchSvc --> geoStore
matchSvc --> matchDb
matchSvc ..> kafka : результат матчинга

paySvc --> payDb
paySvc --> payProvider : sync
paySvc ..> kafka : события оплаты

support ..> kafka : уведомления

kafka ..> notifySvc : события
notifySvc --> notifyDb
notifySvc --> channels
@enduml
```
</details>

## 1. Компоненты

| Компонент | Тип | Зона ответственности                                                                                                                                                                       | State                  |
|---|---|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------|
| Client App | Клиент (Android / iOS / Web) | Отображение, отправка геолокации и данных поездки. Вне контура платформы.                                                                                                                  | —                      |
| API Gateway | Шлюз | Роутинг запросов по path, балансировка, аутентификация (проверка auth-заголовков, связь с LDAP).                                                                                           | stateless              |
| User Service | Сервис + PostgreSQL | Данные пассажиров (ФИО, телефон, почта), регистрация.                                                                                                                                      | stateful (через БД)    |
| Driver Service | Сервис + PostgreSQL | Данные водителей и их регистрация. Выделен из User для отдельной обработки и верификации.                                                                                                  | stateful               |
| Verification Service | Сервис | Верификация водителей через гос. API (права, ИП, самозанятость). REST / SOAP.                                                                                                              | stateless              |
| Ride Service | Сервис + PostgreSQL | Поездки: создание, статусы, история. Событийная связь с Matching и Notification.                                                                                                           | stateful               |
| Matching Service | Сервис + Redis Geo + PostgreSQL | Ядро системы: подбор водителя, обработка геолокации, учёт час-пиков, рейтингов, форс-мажоров. Расчет поездки. Сам по себе сервис получается большой, поэтому требует более мелкой нарезки. | stateful (гео в Redis) |
| Payment Service | Сервис + PostgreSQL | Оплата, история, интеграции с провайдерами. Транзакционность через Saga.                                                                                                                   | stateful               |
| Notification Service | Сервис + PostgreSQL | Уведомления по каналам (push / SMS / email), шаблоны.                                                                                                                                      | stateful               |
| Support Service | Сервис | Техподдержка: доступ к сервисам, чтение всех событий из брокера, админ-интерфейс оператора.                                                                                                | stateless              |
| Message Broker | Kafka | Асинхронный обмен событиями между сервисами, буфер под нагрузку.                                                                                                                           | stateful (инфра)       |

Внешние системы: LDAP (каталог учёток), гос. сервисы (верификация), платёжные провайдеры (эквайринг), каналы доставки уведомлений.

## 2. Потоки данных

Ниже три ключевых сценария со схемами. Они покрывают разные типы взаимодействия: синхронный запрос-ответ, асинхронную обработку через брокер, обработку сбоев и горячий путь геолокации. Остальные сценарии (история поездок, отмена, изменение профиля, добавление способа оплаты) проще и сводятся к синхронному запросу в соответствующий сервис, для них схемы не нужны.

### 2.1 Создание поездки

Самый богатый сценарий. Поездка создаётся синхронно в Ride Service, дальше подбор водителя идёт асинхронно через брокер, статусы прилетают пассажиру через Notification.

![Создание поездки](diagrams/ride-creation.png)

- При сбое на этапе создания поездка помечается `FAILED`, пассажир получает уведомление. По дефолту, если retry, то будет использоваться изначальная поездка, только с новым статусом, но пассажир сможет создать новую, если будет необходимо.
- При сбое матчинга (нет водителя) процесс поиска перезапускается, статус возвращается в `SEARCHING`.

<details>
<summary>PlantUML-исходник</summary>

```plantuml
@startuml RideCreation
title Поток: создание поездки

skinparam shadowing false
skinparam defaultFontName Helvetica
skinparam sequence {
  ArrowColor #555555
  LifeLineBorderColor #888888
  LifeLineBackgroundColor #F0F0F0
  ParticipantBorderColor #2C5A8C
  ParticipantBackgroundColor #4C84C4
  ParticipantFontColor white
  ActorBorderColor #1B3A5C
  ActorBackgroundColor #1B3A5C
}
skinparam database {
  BorderColor #1B5E3A
  BackgroundColor #2E7D5B
  FontColor white
}
skinparam note {
  BackgroundColor #FFF6D8
  BorderColor #C9A227
}

actor "Пассажир" as P
participant "API Gateway" as GW
participant "Ride Service" as RIDE
database "Ride DB" as RDB
queue "Kafka" as K #B8762E
participant "Matching Service" as MATCH
database "Geo Store\n(Redis)" as GEO
participant "Notification\nService" as NOTIF

P -> GW : POST /rides (sync)
GW -> RIDE : создать поездку (sync)
RIDE -> RDB : save (status = SEARCHING)

alt поездка создана
  RIDE -> K : RideCreated (outbox, async)
  RIDE --> GW : 201 (rideId, SEARCHING)
  GW --> P : поездка создана
else сбой при создании
  RIDE -> RDB : status = FAILED
  RIDE --> GW : ошибка
  GW --> P : не удалось создать
  RIDE ->> K : RideFailed
  K ->> NOTIF : уведомить пассажира
end

== подбор водителя (async) ==

K ->> MATCH : RideCreated (inbox)
MATCH -> GEO : поиск ближайших водителей
GEO --> MATCH : кандидаты

alt водитель найден
  MATCH -> MATCH : выбор лучшего
  MATCH ->> K : DriverMatched
  K ->> RIDE : обновить статус (DRIVER_ASSIGNED)
  RIDE -> RDB : update
  K ->> NOTIF : статусы поездки
  NOTIF ->> P : машина найдена / в пути / подана
else водитель не найден
  MATCH ->> K : MatchRetry
  K ->> RIDE : status = SEARCHING
  note right of MATCH : процесс поиска\nперезапускается
end
@enduml
```
</details>

### 2.2 Регистрация водителя

Похож на регистрацию пассажира, но с дополнительной верификацией через гос. сервисы. Заявка принимается синхронно, верификация идёт асинхронно, результат прилетает водителю уведомлением.

![Регистрация водителя](diagrams/driver-registration.png)

<details>
<summary>PlantUML-исходник</summary>

```plantuml
@startuml DriverRegistration
title Поток: регистрация водителя

skinparam shadowing false
skinparam defaultFontName Helvetica
skinparam sequence {
  ArrowColor #555555
  LifeLineBorderColor #888888
  LifeLineBackgroundColor #F0F0F0
  ParticipantBorderColor #2C5A8C
  ParticipantBackgroundColor #4C84C4
  ParticipantFontColor white
  ActorBorderColor #1B3A5C
  ActorBackgroundColor #1B3A5C
}
skinparam database {
  BorderColor #1B5E3A
  BackgroundColor #2E7D5B
  FontColor white
}

actor "Водитель" as D
participant "API Gateway" as GW
participant "Driver Service" as DRV
database "Driver DB" as DDB
participant "Verification\nService" as VER
participant "Гос. сервисы" as GOV #9E9E9E
queue "Kafka" as K #B8762E
participant "Notification\nService" as NOTIF

D -> GW : POST /drivers/register (sync)
GW -> DRV : регистрация (sync)
DRV -> DDB : save (status = PENDING)
DRV --> GW : 202 Accepted
GW --> D : заявка принята

== верификация (через брокер, async) ==

DRV ->> K : VerificationRequested
K ->> VER : VerificationRequested
VER -> GOV : проверка прав, ИП, СЗ (REST / SOAP)
GOV --> VER : результат
VER ->> K : VerificationCompleted
K ->> DRV : VerificationCompleted

alt проверка пройдена
  DRV -> DDB : status = ACTIVE
  DRV ->> K : DriverActivated
else проверка не пройдена
  DRV -> DDB : status = REJECTED
  DRV ->> K : DriverRejected
end

K ->> NOTIF : событие статуса
NOTIF ->> D : результат верификации
@enduml
```
</details>

### 2.3 Обновление геолокации

Горячий путь. Водитель шлёт координаты каждые несколько секунд, запись идёт асинхронно в Geo Store. Отдельная обработка сбоя не нужна: следующий тик перезапишет. Вторая половина схемы показывает обратную сторону, как пассажир читает позицию машины во время поездки.

![Обновление геолокации](diagrams/geo-update.png)

<details>
<summary>PlantUML-исходник</summary>

```plantuml
@startuml GeoUpdate
title Поток: обновление геолокации

skinparam shadowing false
skinparam defaultFontName Helvetica
skinparam sequence {
  ArrowColor #555555
  LifeLineBorderColor #888888
  LifeLineBackgroundColor #F0F0F0
  ParticipantBorderColor #2C5A8C
  ParticipantBackgroundColor #4C84C4
  ParticipantFontColor white
  ActorBorderColor #1B3A5C
  ActorBackgroundColor #1B3A5C
}
skinparam database {
  BorderColor #1B5E3A
  BackgroundColor #2E7D5B
  FontColor white
}
skinparam note {
  BackgroundColor #FFF6D8
  BorderColor #C9A227
}

actor "Водитель" as D
participant "API Gateway" as GW
queue "Kafka" as K #B8762E
participant "Matching\nService" as MATCH
database "Geo Store\n(Redis)" as GEO
actor "Пассажир" as P

loop каждые 3-4 сек (во время смены)
  D ->> GW : координаты (async)
  GW ->> K : LocationUpdated
  K ->> MATCH : LocationUpdated
  MATCH -> GEO : запись координат (GEOADD)
end

note over D, GEO
  Запись fire-and-forget. Если апдейт потерян,
  следующий тик перезапишет. Отдельная обработка
  сбоя не нужна.
end note

== чтение во время поездки ==

P ->> GW : где машина? (sync)
GW -> MATCH : позиция водителя
MATCH -> GEO : GEOPOS / GEOSEARCH
GEO --> MATCH : координаты
MATCH --> GW : позиция
GW --> P : машина на карте
@enduml
```
</details>

### Прочие сценарии (без схем)

- **История поездок** — синхронный запрос в Ride Service.
- **Отмена поездки** — синхронно в Ride (смена статуса), асинхронно в Matching (остановка поиска) и Notification (уведомление).
- **Изменение данных профиля** — синхронный запрос в User Service.
- **Добавление способа оплаты** — синхронный запрос в Payment Service.
- **Обновление геолокации пассажира** — асинхронно в Matching, без обработки сбоя (следующий апдейт достаточен).

## 3. Проблемы и решения

**API Gateway.** Решает маршрутизацию: пользователи дёргают разные фичи, gateway по path подбирает нужный сервис (например, история поездок идёт в Ride Service). Без него каждый сервис должен был бы иметь публичное API, что увеличивает поверхность атаки и требует ресурсов на поддержку публичности. Здесь же балансировка нагрузки по инстансам (при больших объёмах балансировщик выносится отдельно) и аутентификация через проверку auth-заголовков фильтрами. 
Trade-off: единая точка входа упрощает безопасность и роутинг, но становится узким местом и точкой отказа.

**User Service.** Классический CRUD с БД плюс регистрация. Хранит данные пассажиров.

**Driver Service.** Выделен из User Service, чтобы отдельно вести данные водителей и их жизненный цикл (PENDING → ACTIVE / REJECTED). Разделение упрощает верификацию и снижает связанность: логика по водителям не раздувает пользовательский сервис.

**Verification Service.** Интеграции с гос. сервисами через SOAP / REST. Без отдельного сервиса эту логику пришлось бы держать в User/Driver, что увеличило бы его и усложнило поддержку и масштабирование.

**Ride Service.** В основе тот же CRUD, но с событийной архитектурой поверх: публикует события поездки для Matching, Notification и других сервисов. Trade-off: событийная архитектура добавляет асинхронности, скорости, но усложняет логику обработки различных статусов поездки. 

**Matching Service.** Ядро и самый сложный сервис. Главный вопрос — где хранить гео-данные; хороший вариант Redis Geo, он эффективно тянет большие потоки. Сервис учитывает час-пики, форс-мажоры, рейтинги, расчет стоимости. Без него нет подбора водителей либо появляются ненужные усложнения. Масштабируется хорошо: под ним Redis и брокеры (и то, и другое хорошо реплицируется), плюс сама логика это допускает. Брокер используется как буфер: сервис разбирает поездки по мере готовности. Получается довольно большой сервис, поэтому в реальном приложении, каждая часть логики будет вынесена в отдельный сервис.
Trade-offs: Redis Geo - это inmemory хранилише, поэтому в случае его падения, придется ждать некоторого ощутимого времени на восстановления'

**Payment Service.** Интеграции с платёжными системами, обязательная транзакционность. Поэтому Saga: распределённая транзакция через несколько шагов с компенсациями при сбое. Trade-offs: Saga - про надежность, но добавляет сложности на логику компенсаций

**Notification Service.** Отправка уведомлений по каналам (SMS, email, push), хранение шаблонов. Без отдельного сервиса логику отправки и шаблоны пришлось бы дублировать в каждом сервисе, который шлёт уведомления. Легко масштабируется, брокер работает буфером.

**Support Service.** Техподдержка: доступ к сервисам и чтение всех событий из брокера, отдельный админ-интерфейс оператора для поиска заказов и пользователей. Вынесен отдельно, чтобы не смешивать инструменты оператора с клиентскими API и не давать поддержке прямой доступ в чужие БД.

## 4. Вопросы к авторам архитектуры

1. Какие решения используются при нестабильном GPS-сигнале (центр Москвы или глухая местность без покрытия)?
2. Как обрабатывается всплеск нагрузки в форс-мажоры (много незапланированных заказов одновременно), какие здесь особенности?
3. В каком месте подключение кэша дало самый большой вклад?
4. Какая неочевидная технология порадовала больше всего и каким образом ?
5. От какой популярной технологии было больше всего разочарования и почемк?
6. На что уходит больше всего ресурсов и денег?