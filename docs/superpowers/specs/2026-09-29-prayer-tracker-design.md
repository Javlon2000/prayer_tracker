# Prayer Tracker — dizayn (spec)

Sana: 2026-09-29, yangilangan: 2026-11-04
Holat: tasdiqlangan

## 1. Maqsad va kontekst

**Nima:** har kuni 5 vaqt namozning qanday o'qilganini belgilab borish uchun ilova.
Backend Go'da yoziladi, uning ustiga keyin veb-sayt va mobil ilova quriladi.

**Kim uchun:** hozircha bitta foydalanuvchi (login yo'q). Keyinchalik ko'p foydalanuvchiga
o'tish oson bo'lishi kerak.

**Kim yozadi:** Go'ni yangi o'rganayotgan dasturchi. Mentor yo'l ko'rsatadi va review qiladi.
Shuning uchun loyiha ham mahsulot, ham o'quv kursi: har bir bosqich kichik, oxirida ishlaydigan
natija beradi va Go'dan 1–2 yangi mavzuni o'rgatadi.

**Muvaffaqiyat mezoni (MVP, 0–2 bosqichlar):** API orqali istalgan o'tgan kun yoki bugun
uchun 5 namozning har biriga holat qo'yish, uni o'zgartirish va kun hamda sana oralig'i
bo'yicha ko'rish mumkin. Kartadagi `curl` tekshiruvlari va mentor testlari o'tadi.

**Oldingi kod:** o'quvchining birinchi versiyasi (`Javirat/M_Pro`) o'z reposida qoladi. Loyiha
yangi repoda (`Javlon2000/prayer_tracker`) noldan yoziladi. `M_Pro`dagi HTML prototiplar
veb UI bosqichida g'oya sifatida ishlatiladi.

## 2. Qabul qilingan qarorlar

| Mavzu | Qaror | Sabab |
|---|---|---|
| Repo | `Javlon2000/prayer_tracker`, module nomi `prayer-tracker` | Toza boshlanadi, mentor repo egasi |
| Testlar | O'quvchi hozircha test yozmaydi va yurgizmaydi; `curl` bilan tekshiradi. Mentor testlari repodan tashqarida | Boshlanishni yengillashtiradi, sifat nazorati PR'da qoladi |
| Struktura | Rizo `admin_api_gateway` uslubi: `cmd`, `config`, `api`, `models`, `service`, `storage` | Haqiqiy loyiha tuzilishini o'rgatadi; REST servisga eng yaqin namuna |
| HTTP | Gin | O'quvchi allaqachon Gin'da yozyapti, Rizo ham Gin'da; testlar `http.Handler` orqali ishlaydi |
| Foydalanuvchilar | Bitta user, lekin `userID` hamma joyda bor (hozir doim `1`) | Auth keyin qo'shiladi, storage/DB o'zgarmaydi |
| Holatlar | `not_prayed`, `late`, `on_time`, `jamaah` + "belgilanmagan" (yozuv yo'q) | "Belgilanmagan" va "o'qilmagan" boshqa-boshqa narsa |
| Klientlar | Avval faqat REST API, keyin veb UI, keyinroq mobil ilova | Avval Go'ga e'tibor; API ikki klientga xizmat qiladi |
| Saqlash | Avval xotirada, keyin PostgreSQL, bitta interface orqali | Interface'ni amalda o'rgatish; handler va service o'zgarmaydi |
| Sanalar | O'tgan kunlar va bugun belgilanadi, kelajak yo'q. Format `YYYY-MM-DD` | Kelajakni belgilash mantiqsiz; bu format satr sifatida to'g'ri tartiblanadi |
| Statistika | Alohida bosqich (5), oddiy hisob-kitob | MVP'ni kichik saqlash |

Rizo'dan olinmaydi: gRPC, `genproto`, `vendor`, Helm, Redis, `sqlc`, `{status, description, data}`
javob shakli. `pkg/` va `migrations/` haqiqatan kerak bo'lganda qo'shiladi.

## 3. Loyiha tuzilishi

```
cmd/main.go              ishga tushirish: config, storage, service, handler'ni ulash
config/config.go         environment'dan sozlamalar
api/
  api.go                 SetUpRouter: route'lar
  handlers/
    handler.go           Handler, Health, xato yordamchilari
    prayer.go            GetDay, SetPrayerStatus, GetDays
models/
  prayer.go              Prayer, Status, Record, Day, PrayerStatus
  date.go                ParseDate, Today, DatesBetween
service/
  prayer.go              PrayerService: biznes qoidalari
storage/
  storage.go             StorageI va PrayerRepoI interface'lari
  memory/                xotiradagi implementatsiya
  postgres/              (4-bosqich)
migrations/              (4-bosqich)
docs/
```

**Asosiy qoida:** so'rov yuqoridan pastga o'tadi: `handlers`, `service`, `storage`.
Pastki qatlam yuqoridagini bilmaydi. `models` hech narsani import qilmaydi, uni hamma ishlatadi.

| Qatlam | Vazifasi | Nimani bilmaydi |
|---|---|---|
| `api/handlers` | So'rovni o'qish va tekshirish, javob yozish, HTTP kodlari | Ma'lumot qayerda saqlanishini |
| `service` | Qoidalar: kelajak taqiqi, oraliq chegarasi, 5 namozga yoyish | HTTP'ni, Gin'ni |
| `storage` | Yozish va o'qish | Qoidalarni |

## 4. Modellar (`models`)

```go
type Prayer string

const (
    Fajr    Prayer = "fajr"
    Dhuhr   Prayer = "dhuhr"
    Asr     Prayer = "asr"
    Maghrib Prayer = "maghrib"
    Isha    Prayer = "isha"
)

// Tartib muhim: API va UI shu tartibda qaytaradi.
var AllPrayers = []Prayer{Fajr, Dhuhr, Asr, Maghrib, Isha}

type Status string

const (
    NotPrayed Status = "not_prayed"
    Late      Status = "late"
    OnTime    Status = "on_time"
    Jamaah    Status = "jamaah"
)

func (p Prayer) Valid() bool
func (s Status) Valid() bool

type Record struct {
    Date      string // "YYYY-MM-DD"
    Prayer    Prayer
    Status    Status
    UpdatedAt time.Time
}

// API javobi uchun. Status nil bo'lsa JSON'da null.
type PrayerStatus struct {
    Prayer Prayer  `json:"prayer"`
    Status *Status `json:"status"`
}

type Day struct {
    Date    string         `json:"date"`
    Prayers []PrayerStatus `json:"prayers"`
}

var ErrInvalidDate error

// Format va sananing haqiqiyligini tekshiradi ("2026-02-30" xato).
// Xato ErrInvalidDate'ni o'rab oladi.
func ParseDate(s string) (string, error)

// Berilgan timezone'dagi bugungi sana, "YYYY-MM-DD".
func Today(now time.Time, loc *time.Location) string

// from..to (ikkalasi ham kiradi) oralig'idagi sanalar. from > to bo'lsa nil. (2-bosqich)
func DatesBetween(from, to string) []string
```

**Sana nega `string`?** `YYYY-MM-DD` formatidagi satrlarni oddiy satr sifatida solishtirish
to'g'ri ishlaydi (`"2026-09-01" < "2026-09-28"`). `DD-MM-YYYY` da bu ishlamaydi.

## 5. Storage (`storage`)

```go
type StorageI interface {
    Prayer() PrayerRepoI
}

type PrayerRepoI interface {
    // Upsert: yozuv bo'lmasa yaratadi, bo'lsa statusni va UpdatedAt'ni yangilaydi.
    SetStatus(ctx context.Context, userID int64, date string, p models.Prayer, s models.Status) error

    // from..to oralig'idagi (ikkala chegara ham kiradi) barcha yozuvlar.
    // Tartib: sana bo'yicha o'sish, bir kun ichida models.AllPrayers tartibida.
    ListRecords(ctx context.Context, userID int64, from, to string) ([]models.Record, error)
}
```

**Memory (`storage/memory`):** `memory.New()` `*memory.Store` qaytaradi, u `StorageI`ni bajaradi.
`Prayer()` har safar o'sha bitta repo'ni qaytaradi. Repo ichida
`map[key]models.Record` (`key` = `UserID`, `Date`, `Prayer`) va `sync.RWMutex` bor: yozishda
`Lock`, o'qishda `RLock`. Gin har so'rovni alohida goroutine'da bajaradi.

**Postgres (4-bosqich), taxminiy sxema:**

```sql
CREATE TABLE prayer_records (
    user_id    BIGINT      NOT NULL,
    date       DATE        NOT NULL,
    prayer     TEXT        NOT NULL CHECK (prayer IN ('fajr','dhuhr','asr','maghrib','isha')),
    status     TEXT        NOT NULL CHECK (status IN ('not_prayed','late','on_time','jamaah')),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, date, prayer)
);
```

`users` jadvali va `user_id` uchun foreign key auth loyihasida qo'shiladi.

## 6. Service (`service`)

```go
var (
    ErrDateInFuture error
    ErrInvalidRange error
)

type PrayerService struct { /* storage, loc, now */ }

func NewPrayerService(st storage.StorageI, loc *time.Location, now func() time.Time) *PrayerService

func (s *PrayerService) SetStatus(ctx context.Context, userID int64, date string, p models.Prayer, st models.Status) error
func (s *PrayerService) GetDay(ctx context.Context, userID int64, date string) (models.Day, error)
func (s *PrayerService) GetDays(ctx context.Context, userID int64, from, to string) ([]models.Day, error) // 2-bosqich
```

- `SetStatus`: `date > Today(...)` bo'lsa `ErrDateInFuture`, hech narsa saqlanmaydi.
- `GetDay`: har doim 5 ta namoz, `AllPrayers` tartibida. Yozuv yo'q bo'lsa `Status: nil`.
- `GetDays`: oraliqdagi har bir kun, bo'shlari ham. `from > to` yoki 31 kundan uzun bo'lsa
  `ErrInvalidRange`.
- O'qishda kelajakka ruxsat bor, kunlar shunchaki bo'sh qaytadi. Bu haftalik lenta uchun kerak.
- Storage xatosi o'zgartirilmasdan yuqoriga qaytariladi.

## 7. HTTP API (`api`)

Barcha javoblar `Content-Type: application/json; charset=utf-8`.

```go
func SetUpRouter(h *handlers.Handler) *gin.Engine   // HandleMethodNotAllowed = true
func NewHandler(prayer *service.PrayerService) *handlers.Handler
```

Hozircha handler'lar `const defaultUserID int64 = 1` dan foydalanadi. Auth qo'shilganda
`userID` token'dan olinadi.

### 7.1 Endpoint'lar

**`GET /health`** natijasi `200 {"status":"ok"}`

**`GET /api/v1/days/{date}`**: bitta kun

```json
{
  "date": "2026-09-28",
  "prayers": [
    {"prayer": "fajr",    "status": "not_prayed"},
    {"prayer": "dhuhr",   "status": "on_time"},
    {"prayer": "asr",     "status": "on_time"},
    {"prayer": "maghrib", "status": "jamaah"},
    {"prayer": "isha",    "status": null}
  ]
}
```

**`PUT /api/v1/days/{date}/prayers/{prayer}`**: holat qo'yish yoki o'zgartirish

So'rov `{"status": "jamaah"}`, javob `200 {"prayer": "isha", "status": "jamaah"}`.

Tekshiruv tartibi (birinchi topilgan xato qaytadi):
`invalid_date`, `invalid_prayer`, `invalid_json`, `invalid_status`, `date_in_future`.
Birinchi to'rttasi so'rov shaklini handler'da tekshiradi, oxirgisi service'dagi qoida.

- So'rov tanasi ko'pi bilan 1 KB (`http.MaxBytesReader`).
- Noma'lum maydonlar rad etiladi (`DisallowUnknownFields`), natija `invalid_json`.
  Gin'ning `ShouldBindJSON`i buni qilmaydi, shuning uchun `json.NewDecoder` ishlatiladi.
- `Content-Type` sarlavhasi talab qilinmaydi.

**`GET /api/v1/days?from=YYYY-MM-DD&to=YYYY-MM-DD`**: sana oralig'i

```json
{"days": [ {"date": "2026-09-22", "prayers": [...]}, ... ]}
```

Tekshiruv tartibi: `from` yoki `to` yo'q bo'lsa `invalid_range`; format noto'g'ri bo'lsa
`invalid_date`; `from > to` yoki 31 kundan uzun bo'lsa `invalid_range`.

### 7.2 Xatolar

```json
{"error": {"code": "date_in_future", "message": "cannot mark future dates"}}
```

| Holat | HTTP | `code` |
|---|---|---|
| Sana formati yoki qiymati noto'g'ri | 400 | `invalid_date` |
| Kelajakdagi sanani belgilash | 400 | `date_in_future` |
| Noma'lum namoz nomi | 400 | `invalid_prayer` |
| Noma'lum status | 400 | `invalid_status` |
| JSON buzuq, bo'sh, ortiqcha maydon yoki juda katta | 400 | `invalid_json` |
| Oraliq yo'q, teskari yoki 31 kundan uzun | 400 | `invalid_range` |
| Kutilmagan server xatosi | 500 | `internal` |

`code` klientlar (sayt, mobil) uchun barqaror identifikator, `message` dasturchi uchun.
500 xatoning tafsiloti klientga yuborilmaydi, faqat `log/slog` orqali log'ga yoziladi.

Noma'lum yo'l `404`, noto'g'ri method `405` (Gin'ning standart javobi yetarli).

## 8. Konfiguratsiya

```go
type Config struct {
    Port     string
    Location *time.Location
}

func Load(getenv func(string) string) (Config, error)
```

| Env | Standart | Ma'nosi |
|---|---|---|
| `PORT` | `8080` | Server porti |
| `APP_TIMEZONE` | `Asia/Tashkent` | "Bugun"ni aniqlash uchun (`time.LoadLocation`) |
| `DATABASE_URL` | yo'q | (4-bosqich) bo'lsa Postgres, bo'lmasa memory |

Noto'g'ri `APP_TIMEZONE` bo'lsa server ishga tushmaydi va aniq xato chiqaradi.

## 9. Bosqichlar

| # | Bosqich | Natija | Go'dan o'rganiladi |
|---|---|---|---|
| 0 | Skelet | Papkalar, `config`, `GET /health` | package'lar, import yo'llari, Gin router, `time.LoadLocation` |
| 1 | Bitta kun | `models`, `storage`, `service`, `GET /days/{date}`, `PUT .../prayers/{prayer}` | turlar, metodlar, interface, map, `sync.RWMutex`, sentinel xato, dependency injection, JSON decoder, status kodlar |
| 2 | Sana oralig'i | `GET /days?from=&to=` | query parametrlar, sanalar ustida sikl, refactor |
| 3 | Veb UI | `web/`: mobilga mos HTML+JS, Go serveri beradi | `embed`, statik fayllar, `fetch` |
| 4 | PostgreSQL | Docker Compose, SQL migratsiya, `pgx`, `storage/postgres` | SQL, `ON CONFLICT`, integratsion testlar |
| 5 | Statistika | `GET /api/v1/stats?from=&to=`: holatlar soni, umumiy va har namoz bo'yicha | agregatsiya, `GROUP BY` |

Batafsil topshiriq kartalari avval faqat **0–2** uchun yoziladi. Testlar mavzusi o'quvchi kodga
o'rgangach alohida bosqich sifatida qo'shiladi. 3–5 uchun reja o'sha bosqichga yetilganda yoziladi.

## 10. Tekshirish

- **O'quvchi:** kartadagi `curl` buyruqlari va kutilgan javoblar, `go vet ./...`, `gofmt -l .`.
- **Mentor:** har PR'da repodan tashqaridagi testlarni yurgizadi (`mentor_tests/check.sh`).
  Yiqilgan test review izohiga aylanadi. Bu testlar butun karta bo'yicha yoziladi, shuning uchun
  o'quvchi kartadagi nom va signaturalarni o'zgartirmasligi kerak.
- `curl` bilan topish qiyin xatolarni (mutex yo'qligi, yarim tundagi timezone, yozuvlar tartibi)
  aynan mentor testlari ushlaydi.

## 11. Ish jarayoni

- Har bir bosqich alohida git branch'da qilinadi (`phase-1-day-api` va hokazo) va PR orqali
  mentor review qiladi.
- Topshiriq kartasida: maqsad, o'rganiladigan mavzular, `curl` tekshiruvlari, maslahatlar va
  hujjat havolalari. **Tayyor yechim kodi berilmaydi.**

## 12. Hozircha doiradan tashqarida

Auth va ko'p foydalanuvchi, mobil ilova uchun CORS va token, mobil ilova, belgini o'chirish
(`DELETE`), streak'lar va foizlar, hijriy sana, namoz vaqtlari, bildirishnomalar,
graceful shutdown, deploy.

## 13. Ilovalar

- UI mockup (interaktiv, backend'siz): https://claude.ai/artifact/E9z7kbrY9DeaVQZgpn6JgU
  - "Kun ekrani + API so'rovlari": telefon ekrani va har bir bosishda yuboriladigan
    so'rov va javob (spec'dagi formatda simulyatsiya).
  - "Holatlar va API xaritasi": holatlar, namoz nomlari, UI amali va endpoint jadvali.
  - Mockup'dagi UI matnlari o'zbekcha (Bomdod, Peshin, ...). API qiymatlari spec'dagidek.
