# Prayer Tracker MVP (0–2 bosqichlar): topshiriq kartalari

> **Kim bajaradi:** Go'ni o'rganayotgan dasturchi (o'quvchi), qo'lda. Mentor review qiladi.
> Kartalarda **tayyor yechim kodi yo'q**: signaturalar, maslahatlar, hujjat havolalari va
> `curl` bilan tekshirish bor. Kodni o'zingiz yozasiz.

**Maqsad:** API orqali o'tgan kunlar va bugun uchun 5 namozning har biriga holat qo'yish,
o'zgartirish, kun va sana oralig'i bo'yicha ko'rish. Ma'lumot xotirada saqlanadi.

**Arxitektura:** so'rov `api/handlers`dan `service`ga, undan `storage`ga o'tadi. Pastki qatlam
yuqoridagini bilmaydi. `models` hamma qatlam ishlatadigan turlar.

**Texnologiyalar:** Go, Gin. Boshqa tashqi paket yo'q.

**Spec:** [`docs/superpowers/specs/2026-09-29-prayer-tracker-design.md`](../specs/2026-09-29-prayer-tracker-design.md)
Kartani boshlashdan oldin spec'ning tegishli bo'limini o'qing.

**UI mockup:** https://claude.ai/artifact/E9z7kbrY9DeaVQZgpn6JgU

## Umumiy qoidalar

- Module nomi: `prayer-tracker`. Tashqi bog'liqlik faqat Gin.
- **Kartadagi fayl, tur va funksiya nomlarini o'zgartirmang.** Mentor PR'ingizni shu nomlarga
  yozilgan testlar bilan tekshiradi. Funksiya ichini xohlaganingizcha yozishingiz mumkin.
- Kod va API'dagi nomlar inglizcha (`fajr`, `not_prayed`). Izohlar o'zbekcha bo'lishi mumkin.
- Sana formati: `YYYY-MM-DD` (Go layout: `"2006-01-02"`).
- Standart qiymatlar: `PORT=8080`, `APP_TIMEZONE=Asia/Tashkent`.
- Hozircha barcha so'rovlar `userID = 1` nomidan bajariladi.
- Ma'lumot xotirada turadi: server qayta ishga tushsa, u o'chadi. Bu hozircha normal.
- Xato javobida `code` aynan kartadagidek bo'lishi kerak, `message` matni farq qilishi mumkin.

## Ish tartibi (har bir karta uchun bir xil)

1. `main`ni yangilang va yangi branch oching.
2. Kodni qadamma-qadam yozing, har qadamni tekshiring, kichik commit qiling.
3. Kartadagi hamma `curl` tekshiruvlarini bajaring.
4. Ikkala tekshiruvni o'tkazing, ikkalasi ham hech narsa chiqarmasligi kerak:
   ```bash
   go vet ./...
   gofmt -l .
   ```
5. Push qiling va PR oching.

---

## Karta 0: Skelet. Server ko'tariladi, `/health` ishlaydi

**Branch:** `phase-0-skeleton`
**Spec:** 3 va 8-bo'limlar.

**O'rganiladi:** package va papka bog'liqligi, import yo'llari, Gin router, env sozlamalari,
`time.LoadLocation`, xatoni `%w` bilan o'rash.

**Fayllar:** `go.mod`, `.gitignore`, `cmd/main.go`, `config/config.go`, `api/api.go`,
`api/handlers/handler.go`

**Yoziladigan narsalar:**
```go
// config/config.go
type Config struct {
    Port     string
    Location *time.Location
}
func Load(getenv func(string) string) (Config, error)

// api/api.go
func SetUpRouter(h *handlers.Handler) *gin.Engine

// api/handlers/handler.go
type Handler struct{}              // 1-kartada maydon qo'shiladi
func NewHandler() *Handler         // 1-kartada signatura o'zgaradi
func (h *Handler) Health(c *gin.Context)
```

- [ ] **1. Loyihani boshlash**
  ```bash
  git clone https://github.com/Javlon2000/prayer_tracker.git
  cd prayer_tracker
  git switch -c phase-0-skeleton
  go mod init prayer-tracker
  go get github.com/gin-gonic/gin
  ```
  `.gitignore`da kamida `.DS_Store`, `.env`, `/bin/`, `*.test` bo'lsin.

- [ ] **2. `config.Load`**
  - `getenv` nega funksiya: keyinroq haqiqiy environment'ga tegmasdan soxta qiymat berish uchun.
    `main`da unga `os.Getenv` uzatiladi.
  - Bo'sh qiymatga standartni qo'ying. Timezone'ni `time.LoadLocation` bilan yuklang,
    xatoni `fmt.Errorf("...: %w", err)` bilan o'rang.
  - Hujjat: https://pkg.go.dev/time#LoadLocation

- [ ] **3. `Handler`, `Health`, `SetUpRouter`**
  - Javob: `{"status":"ok"}`, `c.JSON` bilan.
  - Router: `gin.New()` va `r.Use(gin.Logger(), gin.Recovery())`.
  - `r.HandleMethodNotAllowed = true` qo'ying. Busiz noto'g'ri method 405 emas, 404 qaytaradi.

- [ ] **4. `cmd/main.go`**
  `config.Load(os.Getenv)`, xato bo'lsa `slog.Error` va `os.Exit(1)`. Keyin
  `api.SetUpRouter(handlers.NewHandler()).Run(":" + cfg.Port)`, uning xatosini ham tekshiring.
  So'ng `go mod tidy`. `cfg.Location` hozircha ishlatilmaydi, 1-kartada kerak bo'ladi.

- [ ] **5. Tekshirish.** Bir terminalda `go run ./cmd`, boshqasida:

  | Buyruq | Kutilgan natija |
  |---|---|
  | `curl -i localhost:8080/health` | `200 OK`, `Content-Type: application/json; charset=utf-8`, `{"status":"ok"}` |
  | `curl -s -o /dev/null -w '%{http_code}\n' -X POST localhost:8080/health` | `405` |
  | `curl -s -o /dev/null -w '%{http_code}\n' localhost:8080/nomalum` | `404` |

  Serverni to'xtatib, qo'shimcha:

  | Buyruq | Kutilgan natija |
  |---|---|
  | `APP_TIMEZONE=Mars/X go run ./cmd` | `unknown time zone` xatosi, server ishga tushmaydi |
  | `PORT=9000 go run ./cmd` | Server 9000-portda ishlaydi |

- [ ] **6. Commit va PR**
  ```bash
  go vet ./...
  gofmt -l .
  git add -A
  git commit -m "phase 0: project structure and /health"
  git push -u origin phase-0-skeleton
  ```

---

## Karta 1: Bitta kun. Holat qo'yish va ko'rish

**Branch:** `phase-1-day-api`
**Spec:** 4, 5, 6 va 7-bo'limlar (`DatesBetween`, `GetDays` va uchinchi endpoint'dan tashqari).

**O'rganiladi:** o'z turlari va metodlar, sentinel xato, `time.Parse`, interface, map va struct
kalit, `sync.RWMutex`, dependency injection, pointer orqali JSON `null`, JSON'ni qat'iy o'qish,
HTTP status kodlar, `log/slog`.

Bu eng katta karta. Qadamlar pastdan yuqoriga boradi: avval turlar, oxirida HTTP. Natijani
`curl` bilan 5-qadamdan boshlab ko'rasiz, undan oldin `go build ./...` xato bermasligi kifoya.
Har qadamni alohida commit qiling.

**Yoziladigan narsalar:**
```go
// models/prayer.go
type Prayer string   // Fajr, Dhuhr, Asr, Maghrib, Isha: "fajr", "dhuhr", "asr", "maghrib", "isha"
var AllPrayers = []Prayer{Fajr, Dhuhr, Asr, Maghrib, Isha}
func (p Prayer) Valid() bool

type Status string   // NotPrayed, Late, OnTime, Jamaah: "not_prayed", "late", "on_time", "jamaah"
func (s Status) Valid() bool

type Record struct {
    Date      string
    Prayer    Prayer
    Status    Status
    UpdatedAt time.Time
}
type PrayerStatus struct {
    Prayer Prayer  `json:"prayer"`
    Status *Status `json:"status"`
}
type Day struct {
    Date    string         `json:"date"`
    Prayers []PrayerStatus `json:"prayers"`
}

// models/date.go
var ErrInvalidDate error
func ParseDate(s string) (string, error)
func Today(now time.Time, loc *time.Location) string

// storage/storage.go
type StorageI interface {
    Prayer() PrayerRepoI
}
type PrayerRepoI interface {
    SetStatus(ctx context.Context, userID int64, date string, p models.Prayer, s models.Status) error
    ListRecords(ctx context.Context, userID int64, from, to string) ([]models.Record, error)
}

// storage/memory/memory.go va storage/memory/prayer.go
type Store struct { /* repo */ }
func New() *Store
func (s *Store) Prayer() storage.PrayerRepoI

// service/prayer.go
var ErrDateInFuture error
type PrayerService struct { /* storage, loc, now */ }
func NewPrayerService(st storage.StorageI, loc *time.Location, now func() time.Time) *PrayerService
func (s *PrayerService) SetStatus(ctx context.Context, userID int64, date string, p models.Prayer, st models.Status) error
func (s *PrayerService) GetDay(ctx context.Context, userID int64, date string) (models.Day, error)

// api/handlers/handler.go
const defaultUserID int64 = 1
type Handler struct { /* prayer *service.PrayerService */ }
func NewHandler(prayer *service.PrayerService) *Handler
func (h *Handler) handleError(c *gin.Context, status int, code, message string)
func (h *Handler) handleInternal(c *gin.Context, err error)

// api/handlers/prayer.go
func (h *Handler) GetDay(c *gin.Context)
func (h *Handler) SetPrayerStatus(c *gin.Context)
```

- [ ] **1. Branch**
  ```bash
  git switch main
  git pull
  git switch -c phase-1-day-api
  ```

- [ ] **2. `models`**
  - Const blokida `Dhuhr = "dhuhr"` deb yozsangiz, turi `Prayer` emas, oddiy string bo'lib qoladi.
    Har qatorga turini yozing.
  - `Valid()` uchun bir nechta qiymatli `case` bilan `switch` eng soddasi.
  - `ParseDate`: Go'da format namuna sana bilan beriladi (`"2006-01-02"`). Nega aynan shu sana:
    https://pkg.go.dev/time#pkg-constants. `time.Parse` `"2026-02-30"` kabi sanani o'zi rad etadi.
    Xatoni o'rang: `fmt.Errorf("%w: %q", ErrInvalidDate, s)`. Hujjat: https://go.dev/blog/go1.13-errors
  - `Today`: bitta qator. Vaqtni timezone'ga o'tkazing (`In`), keyin formatlang.
  - `Status` nega `*Status`: oddiy `Status` bo'lsa, bo'sh qiymat JSON'da `""` bo'ladi, `null` emas.
  - Savol: eski kodingizdagi `DD-MM-YYYY` format nega oraliq uchun yaramaydi? `"14-10-2026"` va
    `"02-11-2026"` satrlarini solishtirib ko'ring.

- [ ] **3. `storage` va `memory`**
  - Kalit: `UserID`, `Date`, `Prayer` maydonli struct. Map'ga shu kalit bilan yozish o'zi upsert.
    Eski kodingizdagi `for` + `found` siklidan qanchalik soddaroq ekaniga e'tibor bering.
  - Map'ni `make` bilan yarating: `nil` map'ga yozish panic beradi.
  - `Prayer()` har safar yangi repo yaratmasligi kerak, aks holda ma'lumot yo'qoladi.
  - `ListRecords`: `userID` va sana oralig'i bo'yicha filtrlang (`>=`, `<=` satrlarda ishlaydi).
    **Tartib shart:** sana bo'yicha o'sish, bir kun ichida `AllPrayers` tartibida. Map'da tartib
    yo'q, natijani tartiblang. `slices.SortFunc` va `slices.Index(models.AllPrayers, p)` yordam beradi.
  - **Mutex shart:** Gin har so'rovni alohida goroutine'da bajaradi. `sync.RWMutex` qo'ying:
    `SetStatus`da `Lock`, `ListRecords`da `RLock`, `defer` bilan qo'yib yuboring. Eski kodingizda
    mutex yo'q edi: 40 ta parallel so'rovdan 31 tasi saqlangan.
    Hujjat: https://go.dev/doc/articles/race_detector
  - Savol: nega `Store` aniq turni emas, `storage.PrayerRepoI`ni qaytaradi?

- [ ] **4. `service`**
  - `SetStatus`: `date > models.Today(s.now(), s.loc)` bo'lsa `ErrDateInFuture`, storage'ga tegmang.
    Boshqa xatoni o'zgartirmasdan qaytaring.
  - `GetDay`: `ListRecords(ctx, userID, date, date)` natijasini `AllPrayers` bo'yicha 5 ta elementga
    yoying. Yozuvi yo'q namozda `Status: nil` qoladi. Eski kodingizdagi GET sana bo'yicha
    filtrlamasdi, bu yerda u hal bo'ladi.
  - `service` Gin'ni import qilmaydi. `time.Now()`ni to'g'ridan-to'g'ri chaqirmang, `s.now()` ishlating.

- [ ] **5. `GET /api/v1/days/:date`**
  - `NewHandler`ni yangilang. `main.go`da `service.NewPrayerService(memory.New(), cfg.Location, time.Now)`
    yaratib handler'ga bering. `main` hamma qatlamni ulaydigan yagona joy.
  - Xato shakli: `{"error": {"code": "...", "message": "..."}}`. `handleInternal` haqiqiy xatoni
    `slog.Error` bilan log'ga yozadi, klientga faqat `"internal"` va umumiy matn yuboradi.
    Eski kodingizdagi `err.Error()` fayl yo'lini klientga ko'rsatib qo'yardi.
  - `models.ParseDate` xato bersa `invalid_date`, aks holda
    `h.prayer.GetDay(c.Request.Context(), defaultUserID, date)`.

  **Tekshirish:** `go run ./cmd`, keyin:
  ```bash
  curl -s localhost:8080/api/v1/days/2026-09-28
  ```
  Kutilgan natija (5 ta namoz, hammasi `null`):
  ```json
  {"date":"2026-09-28","prayers":[{"prayer":"fajr","status":null},{"prayer":"dhuhr","status":null},{"prayer":"asr","status":null},{"prayer":"maghrib","status":null},{"prayer":"isha","status":null}]}
  ```

- [ ] **6. `PUT /api/v1/days/:date/prayers/:prayer`**
  - Tekshiruv tartibi **qat'iy**: `invalid_date`, `invalid_prayer`, `invalid_json`, `invalid_status`,
    keyin service chaqiriladi va `date_in_future`.
  - Tanani o'qish: `http.MaxBytesReader(c.Writer, c.Request.Body, 1024)`, `json.NewDecoder(...)`,
    `dec.DisallowUnknownFields()`, `dec.Decode(&req)`. Istalgan xato `invalid_json`. `ShouldBindJSON`
    ishlatmang: u ortiqcha maydonni rad etmaydi.
    Hujjat: https://pkg.go.dev/encoding/json#Decoder.DisallowUnknownFields
  - Service xatosini `errors.Is(err, service.ErrDateInFuture)` bilan ajrating, qolgani `handleInternal`.
  - Har bir xato javobidan keyin `return` qiling.

- [ ] **7. Tekshirish.** Serverni qayta ishga tushiring va **shu tartibda** bajaring:

  ```bash
  curl -s -X PUT localhost:8080/api/v1/days/2026-09-28/prayers/isha -d '{"status":"jamaah"}'
  ```
  Kutilgan natija: `{"prayer":"isha","status":"jamaah"}`

  ```bash
  curl -s -X PUT localhost:8080/api/v1/days/2026-09-28/prayers/isha -d '{"status":"late"}'
  ```
  Kutilgan natija: `{"prayer":"isha","status":"late"}`

  ```bash
  curl -s localhost:8080/api/v1/days/2026-09-28
  ```
  Kutilgan natija: `isha` `"late"`, qolgan 4 tasi `null`. Yozuv bitta bo'lib qoladi.

  ```bash
  curl -s localhost:8080/api/v1/days/2026-09-27
  ```
  Kutilgan natija: 5 tasi ham `null`. Boshqa kunning yozuvi aralashmaydi.

  ```bash
  curl -s -X PUT localhost:8080/api/v1/days/$(date +%F)/prayers/fajr -d '{"status":"on_time"}'
  ```
  Kutilgan natija: `{"prayer":"fajr","status":"on_time"}`. Bugunni belgilash mumkin.

  ```bash
  curl -s localhost:8080/api/v1/days/2099-01-01
  ```
  Kutilgan natija: `200`, 5 tasi ham `null`. Kelajakni ko'rish mumkin.

  **Xatolar.** Har biri HTTP `400` va shu `code`ni qaytarishi kerak. Kodni ko'rish uchun buyruq
  oxiriga `-w ' %{http_code}\n'` qo'shing.

  | So'rov | `code` |
  |---|---|
  | `curl -s localhost:8080/api/v1/days/28-09-2026` | `invalid_date` |
  | `curl -s localhost:8080/api/v1/days/2026-02-30` | `invalid_date` |
  | `curl -s -X PUT localhost:8080/api/v1/days/2026-09-28/prayers/witr -d '{"status":"jamaah"}'` | `invalid_prayer` |
  | `curl -s -X PUT localhost:8080/api/v1/days/2026-09-28/prayers/isha -d '{"status":'` | `invalid_json` |
  | `curl -s -X PUT localhost:8080/api/v1/days/2026-09-28/prayers/isha -d '{"status":"jamaah","note":"x"}'` | `invalid_json` |
  | `curl -s -X PUT localhost:8080/api/v1/days/2026-09-28/prayers/isha -d '{"status":"prayed"}'` | `invalid_status` |
  | `curl -s -X PUT localhost:8080/api/v1/days/2026-09-28/prayers/isha -d 'null'` | `invalid_status` |
  | `curl -s -X PUT localhost:8080/api/v1/days/2099-01-01/prayers/isha -d '{"status":"jamaah"}'` | `date_in_future` |

  ```bash
  curl -s -o /dev/null -w '%{http_code}\n' -X DELETE localhost:8080/api/v1/days/2026-09-28/prayers/isha
  ```
  Kutilgan natija: `405`

  Oxirida mockup'ni oching va o'ng paneldagi so'rovlarni o'z serveringiz javoblari bilan solishtiring.

- [ ] **8. Commit va PR**
  ```bash
  go vet ./...
  gofmt -l .
  git push -u origin phase-1-day-api
  ```

---

## Karta 2: Sana oralig'i

**Branch:** `phase-2-range-api`
**Spec:** 4 (`DatesBetween`), 6 (`GetDays`), 7.1 (uchinchi endpoint).

**O'rganiladi:** query parametrlar (`c.Query`), `time.AddDate` bilan sikl, takrorlangan kodni
umumiy funksiyaga ajratish.

**Yoziladigan narsalar:**
```go
// models/date.go. Kirish qiymatlari ParseDate'dan o'tgan deb hisoblanadi. from > to bo'lsa nil.
func DatesBetween(from, to string) []string

// service/prayer.go
var ErrInvalidRange error
func (s *PrayerService) GetDays(ctx context.Context, userID int64, from, to string) ([]models.Day, error)

// api/handlers/prayer.go
func (h *Handler) GetDays(c *gin.Context)
```

- [ ] **1. Branch**
  ```bash
  git switch main
  git pull
  git switch -c phase-2-range-api
  ```

- [ ] **2. `DatesBetween`.** Satrni `time.Time`ga aylantiring va `d.AddDate(0, 0, 1)` bilan
  aylantiring. Oy, yil va kabisa chegaralarini `AddDate` o'zi hisoblaydi.

- [ ] **3. "Kunlarni yoyish"ni umumiy funksiyaga ajratish (refactor).**
  `GetDay`dagi mantiqni `buildDays(dates []string, recs []models.Record) []models.Day`ga oling.
  Keyin 1-kartadagi `curl` tekshiruvlarini qayta yurgizing: natija o'zgarmasligi kerak.

- [ ] **4. `GetDays`.** `from > to` yoki 31 kundan uzun bo'lsa `ErrInvalidRange`. `31`ni nomlangan
  const qiling. `ListRecords`ni bitta marta butun oraliq uchun chaqiring, har kun uchun alohida emas.

- [ ] **5. `GET /api/v1/days`.** Route'ni qo'shing.
  Tartib: `from` yoki `to` yo'q bo'lsa `invalid_range`; format noto'g'ri bo'lsa `invalid_date`;
  keyin service, `ErrInvalidRange` bo'lsa `invalid_range`. Javob: `{"days": [...]}`.

- [ ] **6. Tekshirish.** Serverni qayta ishga tushiring va shu tartibda bajaring:

  ```bash
  curl -s -X PUT localhost:8080/api/v1/days/2026-09-28/prayers/isha -d '{"status":"late"}'
  ```
  ```bash
  curl -s 'localhost:8080/api/v1/days?from=2026-09-27&to=2026-09-28'
  ```
  Kutilgan natija: `{"days":[...]}` ichida ikkita kun. 27-sentabrda hammasi `null`,
  28-sentabrda `isha` `"late"`.

  ```bash
  curl -s 'localhost:8080/api/v1/days?from=2026-09-30&to=2026-10-02'
  ```
  Kutilgan natija: oy chegarasidan o'tadi: 3 ta kun, 30-sentabr, 1 va 2-oktabr.

  Xatolar, har biri HTTP `400`:

  | So'rov | `code` |
  |---|---|
  | `curl -s 'localhost:8080/api/v1/days?to=2026-09-28'` | `invalid_range` |
  | `curl -s 'localhost:8080/api/v1/days?from=2026-09-29&to=2026-09-28'` | `invalid_range` |
  | `curl -s 'localhost:8080/api/v1/days?from=2026-09-01&to=2026-10-02'` | `invalid_range` (32 kun) |
  | `curl -s 'localhost:8080/api/v1/days?from=28-09-2026&to=2026-09-28'` | `invalid_date` |

  `from=2026-09-01&to=2026-10-01` esa 31 kun, u `200` qaytarishi kerak.

- [ ] **7. Commit va PR**
  ```bash
  go vet ./...
  gofmt -l .
  git push -u origin phase-2-range-api
  ```

---

## Keyingi rejalar

3-bosqich (veb UI), 4 (PostgreSQL), 5 (statistika) va testlar mavzusi. Ularning rejasi 2-bosqich
tugagach yoziladi, chunki o'shanda nima qiyin bo'lgani ma'lum bo'ladi.
