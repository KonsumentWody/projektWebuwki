# F1 Results Browser

Aplikacja webowa do przeglądania wyników wyścigów Formuły 1, czasów kwalifikacji oraz klasyfikacji mistrzostw dla każdego sezonu od 1950 roku.

## Spis treści

- [Stos technologiczny](#stos-technologiczny)
- [Architektura projektu](#architektura-projektu)
- [Wymagania wstępne](#wymagania-wstępne)
- [Uruchomienie projektu](#uruchomienie-projektu)
- [Zmienne środowiskowe](#zmienne-środowiskowe)
- [API – endpointy](#api--endpointy)
- [Schemat bazy danych](#schemat-bazy-danych)
- [System cache'owania](#system-cachowania)
- [Frontend](#frontend)
- [Testy](#testy)
- [Struktura plików](#struktura-plików)

---

## Stos technologiczny

**Backend**
- Node.js + Express 4
- TypeScript 5.3
- SQLite (better-sqlite3) – lokalna baza danych z cache'em
- Axios – klient HTTP do Jolpica F1 API
- ts-node-dev – hot reload podczas developmentu
- Jest + Supertest – testy jednostkowe i integracyjne

**Frontend**
- React 18 + TypeScript
- Vite 5 – bundler i dev server
- React Router 6 – routing po stronie klienta
- Axios – klient HTTP
- Vitest – testy jednostkowe
- Testing Library – testy komponentów

**Źródło danych**
- [Jolpica F1 API](https://api.jolpi.ca/ergast/f1) – darmowe API (następca Ergast), bez klucza

---

## Architektura projektu

```
projektWebuwki-main/
├── backend/          # Express API + SQLite cache
│   ├── src/
│   │   ├── app.ts            # Konfiguracja Express (CORS, middleware, routing)
│   │   ├── server.ts         # Punkt wejścia – nasłuchiwanie na porcie
│   │   ├── database.ts       # Inicjalizacja SQLite, schemat tabel
│   │   ├── routes/           # Handlery endpointów REST
│   │   │   ├── seasons.ts
│   │   │   ├── races.ts
│   │   │   ├── results.ts
│   │   │   └── standings.ts
│   │   ├── services/
│   │   │   ├── f1ApiService.ts   # Klient Jolpica API + typy odpowiedzi
│   │   │   └── cacheService.ts   # Logika cache (SQLite + TTL)
│   │   └── middleware/
│   │       └── errorHandler.ts   # Globalny handler błędów + 404
│   └── tests/        # Testy jednostkowe i integracyjne (Jest)
│
└── frontend/         # React SPA
    ├── index.html
    ├── vite.config.ts        # Konfiguracja Vite + proxy /api → localhost:3001
    └── src/
        ├── components/       # Komponenty UI
        ├── hooks/            # Custom hooks (useRaces, useResults itp.)
        ├── services/
        │   └── api.ts        # Klient HTTP dla backendu
        └── types/
            └── f1.ts         # Typy TypeScript dla wszystkich danych F1
```

---

## Wymagania wstępne

- **Node.js** w wersji 18 lub wyższej
- **npm** (dołączony do Node.js)

Sprawdź wersję Node.js:
```bash
node -v
```

---

## Uruchomienie projektu

### 1. Backend

```bash
cd backend

# Opcjonalnie – stwórz plik .env (nie jest wymagany, działa bez niego)
touch .env

# Zainstaluj zależności
npm install

# Uruchom w trybie developerskim (hot reload)
npm run dev
```

Backend działa pod adresem **http://localhost:3001**

Inne dostępne komendy:
```bash
npm run build       # Kompilacja TypeScript do dist/
npm start           # Uruchomienie z dist/ (po build)
npm test            # Uruchomienie testów
npm run test:coverage  # Testy z raportem pokrycia
```

### 2. Frontend

Otwórz **osobny terminal**:

```bash
cd frontend

npm install

npm run dev
```

Frontend działa pod adresem **http://localhost:5173**

Inne dostępne komendy:
```bash
npm run build       # Build produkcyjny do dist/
npm run preview     # Podgląd builda produkcyjnego
npm test            # Testy jednostkowe (Vitest)
npm run test:watch  # Testy w trybie watch
npm run test:coverage  # Testy z pokryciem kodu
```

> **Uwaga:** Vite automatycznie proxy'uje żądania `/api` na `http://localhost:3001`, więc frontend i backend komunikują się bez konieczności ręcznej konfiguracji CORS podczas developmentu.

---

## Zmienne środowiskowe

Plik `.env` w katalogu `backend/` (wszystkie są opcjonalne):

| Zmienna | Domyślna wartość | Opis |
|---|---|---|
| `PORT` | `3001` | Port nasłuchiwania serwera |
| `CORS_ORIGIN` | `http://localhost:5173` | Dozwolone źródło CORS (adres frontendu) |
| `DB_DIR` | `backend/data/` | Katalog z plikiem bazy SQLite |
| `NODE_ENV` | – | Ustaw `test` aby wyciszyć logi błędów |

Przykład `.env`:
```env
PORT=3001
CORS_ORIGIN=http://localhost:5173
DB_DIR=./data
```

---

## API – endpointy

Wszystkie endpointy zwracają JSON. Prefix: `/api`.

### `GET /health`

Sprawdzenie stanu serwera.

**Odpowiedź:**
```json
{
  "status": "ok",
  "timestamp": "2024-03-15T12:00:00.000Z"
}
```

---

### `GET /api/seasons`

Lista wszystkich dostępnych sezonów F1 (od 1950), posortowana malejąco.

**Odpowiedź:**
```json
{
  "seasons": [2024, 2023, 2022, ..., 1950]
}
```

---

### `GET /api/races/:season`

Lista wyścigów w danym sezonie.

**Parametry URL:**
- `season` – rok sezonu (1950 – rok bieżący)

**Przykład:** `GET /api/races/2023`

**Odpowiedź:**
```json
{
  "season": 2023,
  "races": [
    {
      "id": 1,
      "season": 2023,
      "round": 1,
      "race_name": "Bahrain Grand Prix",
      "circuit_id": "bahrain",
      "circuit_name": "Bahrain International Circuit",
      "locality": "Sakhir",
      "country": "Bahrain",
      "date": "2023-03-05",
      "time": "15:00:00Z",
      "lat": 26.0325,
      "lng": 50.5106
    }
  ]
}
```

**Błędy:** `400` – nieprawidłowy rok sezonu

---

### `GET /api/results/:season/:round`

Wyniki wyścigu i kwalifikacji dla danej rundy sezonu.

**Parametry URL:**
- `season` – rok sezonu
- `round` – numer rundy (zaczyna się od 1)

**Przykład:** `GET /api/results/2023/1`

**Odpowiedź:**
```json
{
  "season": 2023,
  "round": 1,
  "results": [
    {
      "position": 1,
      "position_text": "1",
      "grid": 1,
      "laps": 57,
      "status": "Finished",
      "points": 25,
      "time_text": "1:33:56.736",
      "fastest_lap_rank": 1,
      "fastest_lap_time": "1:33.996",
      "fastest_lap_speed": "206.018",
      "driver_id": "max_verstappen",
      "code": "VER",
      "permanent_number": "1",
      "forename": "Max",
      "surname": "Verstappen",
      "nationality": "Dutch",
      "constructor_id": "red_bull",
      "constructor_name": "Red Bull",
      "constructor_nationality": "Austrian"
    }
  ],
  "qualifying": [
    {
      "position": 1,
      "q1": "1:30.000",
      "q2": "1:29.500",
      "q3": "1:29.050",
      "driver_id": "max_verstappen",
      "code": "VER",
      "permanent_number": "1",
      "forename": "Max",
      "surname": "Verstappen",
      "nationality": "Dutch",
      "constructor_id": "red_bull",
      "constructor_name": "Red Bull",
      "constructor_nationality": "Austrian"
    }
  ]
}
```

**Błędy:** `400` – nieprawidłowy sezon lub numer rundy

---

### `GET /api/standings/drivers/:season`

Klasyfikacja kierowców dla danego sezonu.

**Przykład:** `GET /api/standings/drivers/2023`

**Odpowiedź:**
```json
{
  "season": 2023,
  "standings": [
    {
      "position": 1,
      "points": 575,
      "wins": 19,
      "driver_id": "max_verstappen",
      "code": "VER",
      "permanent_number": "1",
      "forename": "Max",
      "surname": "Verstappen",
      "nationality": "Dutch"
    }
  ]
}
```

---

### `GET /api/standings/constructors/:season`

Klasyfikacja konstruktorów dla danego sezonu.

**Przykład:** `GET /api/standings/constructors/2023`

**Odpowiedź:**
```json
{
  "season": 2023,
  "standings": [
    {
      "position": 1,
      "points": 860,
      "wins": 21,
      "constructor_id": "red_bull",
      "constructor_name": "Red Bull",
      "nationality": "Austrian"
    }
  ]
}
```

---

## Schemat bazy danych

Backend używa SQLite (plik `backend/data/f1.db`, tworzony automatycznie). Baza inicjalizowana jest przy pierwszym uruchomieniu.

```
seasons            – lista sezonów z datą pobrania
circuits           – tory wyścigowe (id, nazwa, lokalizacja, współrzędne GPS)
races              – wyścigi (sezon, runda, tor, data)
drivers            – kierowcy (id, kod, numer, imię, nazwisko, narodowość)
constructors       – zespoły (id, nazwa, narodowość)
race_results       – wyniki wyścigów (pozycja, punkty, czas, najszybsze okrążenie)
qualifying_results – wyniki kwalifikacji (Q1, Q2, Q3)
driver_standings   – klasyfikacja kierowców (sezon, pozycja, punkty, wygrane)
constructor_standings – klasyfikacja konstruktorów
api_cache          – klucze cache z timestampem pobrania
```

Baza korzysta z trybu WAL (`journal_mode = WAL`) i wymusza klucze obce (`foreign_keys = ON`).

---

## System cache'owania

Backend nie pobiera danych z Jolpica API przy każdym żądaniu – persystuje je lokalnie w SQLite i odświeża według TTL (time-to-live):

| Typ danych | TTL |
|---|---|
| Sezony | 24 godziny |
| Lista wyścigów | 6 godzin |
| Wyniki wyścigów i kwalifikacji | 30 dni (wyniki nie zmieniają się po zakończeniu) |
| Klasyfikacje | 1 godzina |

Przy kolejnym żądaniu: jeśli TTL nie wygasł, dane zwracane są z SQLite bez wywołania zewnętrznego API.

---

## Frontend

### Komponenty (`frontend/src/components/`)

| Komponent | Opis |
|---|---|
| `Navbar` | Górna nawigacja aplikacji |
| `RaceCard` | Karta pojedynczego wyścigu (nazwa, tor, data) |
| `ResultsTable` | Tabela wyników wyścigu |
| `QualifyingTable` | Tabela wyników kwalifikacji (Q1/Q2/Q3) |
| `StandingsTable` | Tabela klasyfikacji (kierowcy / konstruktorzy) |
| `LoadingSpinner` | Wskaźnik ładowania |
| `ErrorMessage` | Komunikat błędu |

### Custom Hooks (`frontend/src/hooks/`)

| Hook | Opis |
|---|---|
| `useSeasons` | Pobiera listę sezonów |
| `useRaces` | Pobiera wyścigi dla wybranego sezonu |
| `useResults` | Pobiera wyniki wyścigu i kwalifikacji |
| `useStandings` | Pobiera klasyfikacje kierowców i konstruktorów |

### Proxy Vite

Vite w trybie dev przekierowuje żądania `/api/*` na `http://localhost:3001`, dzięki czemu frontend i backend mogą działać na różnych portach bez problemów z CORS.

```ts
// vite.config.ts
proxy: {
  '/api': {
    target: 'http://localhost:3001',
    changeOrigin: true,
  },
},
```

---

## Testy

### Backend (Jest)

Testy znajdują się w `backend/tests/`:

```bash
cd backend
npm test                  # Uruchom wszystkie testy
npm run test:coverage     # Testy z raportem pokrycia kodu
```

| Plik testu | Zakres |
|---|---|
| `cacheService.test.ts` | Logika cache: TTL, upsert, pobieranie z in-memory SQLite |
| `f1ApiService.test.ts` | Klient Jolpica API: mockowanie axios, parsowanie odpowiedzi |
| `routes.test.ts` | Testy integracyjne endpointów HTTP (Supertest) |

Testy backendu używają in-memory SQLite (`:memory:`), więc nie zapisują danych na dysku.

### Frontend (Vitest)

```bash
cd frontend
npm test                  # Uruchom raz
npm run test:watch        # Tryb watch
npm run test:coverage     # Pokrycie kodu (lcov + tekst)
```

---

## Struktura plików

```
projektWebuwki-main/
│
├── README.md
│
├── backend/
│   ├── package.json
│   ├── tsconfig.json
│   ├── jest.config.ts
│   ├── src/
│   │   ├── app.ts
│   │   ├── server.ts
│   │   ├── database.ts
│   │   ├── middleware/
│   │   │   └── errorHandler.ts
│   │   ├── routes/
│   │   │   ├── index.ts
│   │   │   ├── seasons.ts
│   │   │   ├── races.ts
│   │   │   ├── results.ts
│   │   │   └── standings.ts
│   │   └── services/
│   │       ├── f1ApiService.ts
│   │       └── cacheService.ts
│   └── tests/
│       ├── cacheService.test.ts
│       ├── f1ApiService.test.ts
│       └── routes.test.ts
│
└── frontend/
    ├── package.json
    ├── tsconfig.json
    ├── vite.config.ts
    ├── index.html
    └── src/
        ├── components/
        │   ├── Navbar.tsx
        │   ├── RaceCard.tsx
        │   ├── ResultsTable.tsx
        │   ├── QualifyingTable.tsx
        │   ├── StandingsTable.tsx
        │   ├── LoadingSpinner.tsx
        │   └── ErrorMessage.tsx
        ├── hooks/
        │   ├── useSeasons.ts
        │   ├── useRaces.ts
        │   ├── useResults.ts
        │   └── useStandings.ts
        ├── services/
        │   └── api.ts
        └── types/
            └── f1.ts
```
