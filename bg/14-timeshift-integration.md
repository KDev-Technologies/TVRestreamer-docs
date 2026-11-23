# 14 · Интеграция с TimeShift (catch-up)

[← Индекс](00-README.md) · [English](../en/14-timeshift-integration.md)

TvRestream няма собствен архив (DVR) — виж [01 · Визия](01-vision-prd.md). Catch-up / timeshift идва от отделния
**TimeShift Server**: той записва каналите от TvRestream и сервира архива, а TvRestream слага адреса му в плейлистите
на абонатите. Този документ описва връзката от страната на TvRestream (tvr-core). Страната на TimeShift е в неговата
документация: `docs/bg/TVRESTREAM.md` в repo-то TimeshiftServer (порталът `index.html` → 05 „Интеграция с
TvRestream“).

| | |
|---|---|
| Тикети | TVR-132 (catch-up в плейлистите, API за TimeShift), TVR-133 (служебен абонат за запис), BUG-059…068 |
| В TimeShift | TSS-064…071 |
| Настройки | Настройки → **TimeShift (catch-up)**, група `TIMESHIFT` |
| Пакет | `io.tvr.core.timeshift` |
| Миграция | V30 `package.catchup` |

## 1. Кой какво прави

| | TvRestream | TimeShift |
|---|---|---|
| Live (HLS / HTTP-TS) | ✓ edge `/live/<slug>/`, `/ts/<slug>.ts` | – |
| Плейлист на абоната (`/pl/{token}.m3u`, `get.php`) | ✓ с `catchup`, `catchup-source`, `catchup-days` | – |
| Абонати, пакети, IP / държава, MAC lock, срок | ✓ | пита TvRestream (`/api/timeshift/v1/access`) |
| Запис на каналите | дава потоците на служебен абонат | ✓ ffmpeg на канал |
| Архив (catch-up плейлист и сегменти) | – | ✓ плейърът ги тегли директно |
| Кои канали имат архив и колко дни | чете го от TimeShift | ✓ `GET /api/integration/v1/channels` |

Видеото на архива **никога** не минава през TvRestream — нито през tvr-core, нито през edge-а.

## 2. Схема

```
 Плейър ──① /pl/{token}.m3u, get.php ─────────────▶ tvr-core :8080 ──(кеш от ②)── catchup-source към TimeShift
 Плейър ──  live /live/<slug>/ ───────────────────▶ edge :8081
 Плейър ──③ catch-up /?ch=<id>&utc=…&token=…&mac=… ─────────────────────────────▶ TimeShift :13100

 tvr-core ──② GET  /api/integration/v1/channels (Bearer, If-None-Match) ─────────▶ TimeShift Java :18080
 tvr-core ◀─④ GET  /api/timeshift/v1/access?token|username+password&ip&mac (Bearer) ── TimeShift
 tvr-core ◀─⑤ POST /api/timeshift/v1/events {"type":"channels.changed"} (Bearer) ─── TimeShift
 edge     ◀─⑥ запис: /pl/<токен на служебния абонат>.m3u → /live/<slug>/ на 127.0.0.1 ─ ffmpeg на TimeShift
```

1. Абонатът тегли плейлиста си както досега. Каналите, които TimeShift записва и чийто пакет позволява catch-up,
   получават трите атрибута. Плейлистът **никога не чака** TimeShift — ползва кеширания списък.
2. tvr-core чете списъка на TimeShift при старт, на `timeshift.refresh` (5 мин) и след събитие ⑤, с ETag.
3. Плейърът тегли архива директно от TimeShift с данните на абоната.
4. TimeShift пита tvr-core дали абонатът може да гледа и кои канали.
5. TimeShift съобщава, че списъкът е сменен — tvr-core го чете веднага във фонов режим.
6. TimeShift записва през плейлиста на служебен абонат (TVR-133) — вътрешни адреси, без ограничения за устройства.

## 3. Процедура по свързване (от страната на TvRestream)

Пълната процедура има стъпки и в двата панела. Тук са стъпките на TvRestream подробно; тези на TimeShift — накратко
(подробно в документацията на TimeShift).

### 3.1 Предварително

* Портове на една машина: tvr-core `127.0.0.1:8080`, edge `:8081`, Java на TimeShift `127.0.0.1:18080`, nginx на
  TimeShift `:13100`. Java на TimeShift **не може** да е на 8080 (`install.sh --app-port 18080`).
* На различни машини: tvr-core трябва да стига Java на TimeShift, TimeShift — tvr-core и edge-а. Между машини — HTTPS
  или частна мрежа.
* NTP и на двете машини.

### 3.2 Служебен абонат за запис (TVR-133)

1. **Пакети → Нов пакет**, напр. „TimeShift запис“: всички канали, които ще се записват.
2. **Абонати → Нов абонат**, напр. потребител `timeshift`, без краен срок, с този пакет. Никога истински клиент.
3. Копирай токена от личния му плейлист (`/pl/<токен>.m3u`) — той отива в TimeShift.
4. **Настройки → TimeShift (catch-up):**

| Поле | Стойност |
|---|---|
| Абонат за запис в TimeShift (`timeshift.recorder-subscriber`) | `timeshift` |
| Адрес за запис (`timeshift.recorder-url`) | празно = адресът на edge-а с хост `127.0.0.1` (същата схема и порт, напр. `http://127.0.0.1:8081`); иначе вътрешният адрес на edge-а, ако TimeShift е в LAN |

Какво променя това за този абонат (`TimeshiftRecorder`):

* адресите в плейлиста му сочат към адреса за запис, не към публичния адрес на edge-а — без hairpin NAT;
* при гледане **не се проверяват** разрешените IP, държави, MAC lock и лимитът на връзките (TimeShift държи по една
  връзка на записван канал); статусът, срокът и пакетите важат — пакетите му са това, което TimeShift може да записва;
* плейлистът му е **без** catch-up атрибути — той записва, не гледа;
* потребителското име се превежда до id веднъж на 30 s, защото проверката при гледане е на всяка заявка на edge-а.

### 3.3 В TimeShift (накратко)

1. Страница **TvRestream:** Включено, адрес на tvr-core `http://127.0.0.1:8080`, токенът от 3.2, „Абонатите влизат
   с TvRestream“ вкл. → Запази → **Провери връзката**.
2. **Използвай TvRestream за източник на каналите** → **Канали → Синхронизирай** → включи **Запис** за каналите.
   Новите канали получават id = `tvrSlug` = slug на потока.
3. **Ключ на интеграцията → Нов ключ** — копирай го (показва се веднъж).

### 3.4 Настройки → TimeShift (catch-up)

| Поле | Ключ / env | По подразб. | Стойност |
|---|---|---|---|
| Интеграция с TimeShift | `timeshift.enabled` / `TVR_TIMESHIFT_ENABLED` | false | вкл. |
| Вътрешен адрес на TimeShift | `timeshift.api-url` / `TVR_TIMESHIFT_API_URL` | – | `http://127.0.0.1:18080` (Java, не nginx) |
| Ключ на интеграцията | `timeshift.key` / `TVR_TIMESHIFT_KEY` | – | ключът от 3.3 (16–256 знака) |
| Публичен адрес на TimeShift | `timeshift.public-url` / `TVR_TIMESHIFT_PUBLIC_URL` | – | празно = `publicBaseUrl` от TimeShift; иначе напр. `http://tv.example.com:13100` |
| Опресняване | `timeshift.refresh` | 5m | 30 s … 1 h |
| Данни в catch-up за абонатите с име и парола | `timeshift.xtream-auth` | `TOKEN` | `TOKEN` (препоръчително) или `USERNAME_PASSWORD` |
| Пази последния списък (експертна) | `timeshift.stale-after` | 6h | 5 мин … 7 дни |
| Абонат за запис / Адрес за запис | `timeshift.recorder-subscriber` / `timeshift.recorder-url` | – | виж 3.2 |

Всички стойности важат веднага, без рестарт. Кешът на списъка е във файла `tvr.timeshift.cache-file`
(`TVR_TIMESHIFT_CACHE_FILE`, по подразбиране `timeshift-cache.json` в работната папка на tvr-core).

**Провери връзката** (`POST /api/v1/timeshift/actions/check`, ADMIN) чете списъка наново без `If-None-Match` и
показва: „TimeShift отговаря: N канала, M с catch-up, K съпоставени с потоци“. Каналите на TimeShift без поток в
TvRestream са изброени отдолу („без tvrSlug“ — каналът е добавен в TimeShift преди интеграцията).

### 3.5 Пакети и потоци

* **Пакети:** отметка **Catch-up** (V30 `package.catchup`, по подразбиране вкл. за всички, и съществуващите).
  Изключи я за пакет, който не трябва да има архив. Канал получава catch-up, ако поне един от активните пакети на
  абоната, в който е каналът, има отметката.
* **Потоци:** колона **Catch-up** — дни архив и състояние от TimeShift (`RUNNING`, `RESTARTING`, `STOPPED`).
  Данните идват от `GET /api/v1/timeshift` (всяка роля).

### 3.6 Проверка

```bash
# TimeShift отговаря на ключа
curl -s -H "Authorization: Bearer <ключ>" http://127.0.0.1:18080/api/integration/v1/ping

# плейлистът на абонат има catch-up
curl -s "http://127.0.0.1:8080/pl/<токен на абонат>.m3u" | grep -m3 catchup-source

# get.php (m3u_plus) — токен или име/парола според timeshift.xtream-auth
curl -s "http://127.0.0.1:8080/get.php?username=<u>&password=<p>&type=m3u_plus" | grep -m1 catchup-source

# това, което TimeShift пита за абоната
curl -s -H "Authorization: Bearer <ключ>" \
     "http://127.0.0.1:8080/api/timeshift/v1/access?token=<токен на абонат>&ip=203.0.113.5"

# служебният абонат: вътрешни адреси, без catchup-source
curl -s "http://127.0.0.1:8080/pl/<токен на timeshift>.m3u" | head -4
```

Накрая: TiviMate (или друг плейър) със стария адрес на плейлиста пуска предаване отпреди 2 часа от EPG.

## 4. Какво получава абонатът

`/pl/{token}.m3u` (и `type=ts`):

```
#EXTM3U url-tvg="http://tv.example.com:8081/pl/<token>.xml" x-tvg-url="…"
#EXTINF:-1 catchup="default" catchup-source="http://tv.example.com:13100?ch=btv&utc=${start}&lutc=${timestamp}&token=<token>" catchup-days="7" tvg-id="btv" tvg-name="bTV HD" group-title="Национални",bTV HD
http://tv.example.com:8081/live/btv/index.m3u8?token=<token>
```

| Случай | `catchup-source` завършва с |
|---|---|
| `/pl/{token}.m3u` | `&token=<token>` |
| `/pl/{token}.m3u?mac=00:1A:79:…` (MAC lock) | `&token=<token>&mac=00:1A:79:…` |
| `get.php`, `timeshift.xtream-auth = TOKEN` | `&token=<токенът на абоната>` |
| `get.php`, `USERNAME_PASSWORD` | `&username=<u>&password=<p>` (паролата е видима в плейъра и в логовете — затова не е по подразбиране) |

* `${start}` и `${timestamp}` остават буквално — замества ги плейърът.
* Адресът е `catchupSource` от TimeShift (с `ch=<id на канала в TimeShift>`); `timeshift.public-url` заменя само
  началото му (`publicBaseUrl`).
* `catchup-days` е от TimeShift: настроените дни, но не повече от реалния архив в цели дни.
* EPG: ако TvRestream няма собствен източник на програма, в заглавието влиза програмата на TimeShift
  (`/epg.xml.gz` с данните на абоната).
* Xtream API (`player_api.php`) не обявява архив (`tv_archive` = 0): catch-up е в M3U плейлистите (`/pl`, `get.php`).

## 5. Как работи кешът

`TimeshiftService` държи списъка на TimeShift (`TimeshiftCatalog`):

| Кога | Какво |
|---|---|
| старт на tvr-core | чете `timeshift-cache.json` — catch-up не изчезва при рестарт |
| на 10 s проверка, четене след `timeshift.refresh` | `GET /api/integration/v1/channels` с `If-None-Match` → 304 само потвърждава списъка |
| събитие `channels.changed` | 204 веднага; ако `etag` е различен от кеширания — четене във фонов режим; повторено събитие не прави нищо |
| „Провери връзката“ | четене без `If-None-Match`; изчаква до 30 s вече вървящо четене, за да покаже свой отговор (BUG-066) |
| 304 | потвърждението се записва и във файла (BUG-067) |
| TimeShift не отговаря | последният списък важи `timeshift.stale-after` (6 ч) от последния успешен контакт; после плейлистите са без catch-up |

Съпоставяне: канал на TimeShift отговаря на поток със `slug` = `tvrSlug`; без `tvrSlug` — `slug` = `id` (TVR-133).
Каналите с `tvrSlug` имат предимство пред съвпадение по id.

## 6. API на TvRestream за TimeShift

Само с `Authorization: Bearer <timeshift.key>`; 404, докато интеграцията е изключена или няма ключ; 401 при липсващ
или грешен ключ. Ключът и паролите не се логват. Сигурност в OpenAPI: `timeshiftKey`.

### `GET /api/timeshift/v1/access`

| Параметър | Значение |
|---|---|
| `token` | токенът на личния плейлист (32 знака) |
| `username`, `password` | Xtream данните — същото заключване и Argon2 лимит като `get.php` |
| `ip` | IP на плейъра: грешните пароли се броят срещу него, не срещу TimeShift; проверяват се разрешените IP и държави — без него абонат с такова ограничение е отказан (BUG-061) |
| `mac` | MAC на приемника от catch-up адреса, за MAC lock (BUG-061) |

| Отговор | Кога |
|---|---|
| 200 | ACTIVE абонат с непреминал срок; тяло по-долу |
| 400 | нито токен, нито име и парола |
| 403 `{"reason": "disabled"}` | спрян (SUSPENDED) или забранен (BANNED) |
| 403 `expired` | изтекъл или не е ACTIVE |
| 403 `ip` / `country` / `mac` | не е от разрешен адрес, държава или приемник |
| 404 `{"reason": "unknown"}` | няма такъв абонат (и грешна парола) |
| 503 + `Retry-After: 5` | твърде много проверки на пароли едновременно |

```json
{"subscriber": "ivan", "enabled": true, "expiresAt": "2026-12-31T21:59:59Z", "catchup": true,
 "channels": ["btv", "nova", "bnt1"], "catchupChannels": ["btv", "nova"]}
```

* `channels` — включените канали от активните пакети с публичен изход (HLS или TS), същите като в плейлиста.
* `catchupChannels` — тези от тях, чиито пакети позволяват catch-up; TimeShift пуска архив само за тях.
* `expiresAt` — краят на последния ден (часова зона на абонатите); `null` = без край.

### `POST /api/timeshift/v1/events`

```json
{"type": "channels.changed", "etag": "\"7f3a\"", "at": "2026-11-18T12:00:00Z"}
```

204 веднага. Всеки тип се приема като „списъкът може да е сменен“ — непознат бъдещ тип струва една условна заявка.

### За панела

| Адрес | Роля | Описание |
|---|---|---|
| `GET /api/v1/timeshift` | всяка | състояние (`enabled`, `configured`, `usable`, последно четене, грешка) и каналите на TimeShift със съпоставянето |
| `POST /api/v1/timeshift/actions/check` | ADMIN | „Провери връзката“ |

## 7. Отказоустойчивост

| Ситуация | Резултат |
|---|---|
| TimeShift спрян | плейлистите пазят catch-up 6 ч от последния списък, после се дават без него; live не се засяга |
| tvr-core рестартиран | списъкът се чете от `timeshift-cache.json` |
| tvr-core не отговаря на `/access` | TimeShift пуска познатите абонати още 30 мин (`graceMinutes`), новите получават 503 |
| edge-ът спрян | записите в TimeShift са `RESTARTING`, в архива остава дупка |
| Изгубено събитие | хваща го опресняването на 5 мин |
| Спрян абонат / нова забрана | TimeShift кешира отговора ≤ 10 s (`tvrestream.authCacheSeconds`), после е отказ |

## 8. Сигурност

* Един ключ за двете посоки, генериран в TimeShift („Нов ключ“); смяната спира стария веднага — въведи новия тук.
* `timeshift.api-url` е вътрешен адрес; не публикувай Java порта на TimeShift.
* Паролите на Xtream абонатите не влизат в catch-up адресите по подразбиране (`TOKEN`, BUG-062).
* Служебният абонат няма ограничения за устройства — не давай токена му и не го ползвай за гледане.
* `/api/timeshift/v1/*` не е зад вход в панела, а само зад ключа; при грешен ключ tvr-core връща 401.

## 9. Чести проблеми

| Симптом | Причина / проверка |
|---|---|
| „TimeShift не отговаря: … 404“ | интеграцията в TimeShift е изключена или там няма ключ |
| „… 401“ | ключът тук не е последният „Нов ключ“ от TimeShift |
| „… Connection refused“ | `timeshift.api-url` сочи грешен порт (nginx :13100 вместо Java :18080) или TimeShift не работи |
| 0 съпоставени канала | каналите в TimeShift нямат `tvrSlug` и id-то им не е slug — синхронизирай TimeShift от служебния плейлист |
| Плейлистът е без `catchup-source` | пакетът е без отметка Catch-up; TimeShift не записва канала (`catchupDays` 0); списъкът е по-стар от `stale-after`; абонатът е служебният |
| Плейърът: 403 за catch-up | причината е в лога на TimeShift: `mac` (плейърът не праща MAC — плейлистът трябва да е зареден с `?mac=`), `ip` / `country`, `expired` / `disabled` |
| Всички абонати с ограничение по IP получават 403 | TimeShift не подава `ip` (стара версия) или IP-то на плейъра не стига до него зад nginx |
| Записът в TimeShift спира за всички канали | служебният абонат е спрян / изтекъл, пакетът му е сменен или токенът му е подновен |
| Записът минава през рутера | „Адрес за запис“ сочи публичния адрес; празно = edge на `127.0.0.1` |
| 502 при запис на плейлиста на канала | стар edge без поправката на BUG-069 (`Range` от ffmpeg) |
