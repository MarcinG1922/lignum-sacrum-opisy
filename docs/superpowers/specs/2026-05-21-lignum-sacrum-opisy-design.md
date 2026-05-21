# Lignum Sacrum — Generator opisów produktów (CSV + Gemini)

Data: 2026-05-21
Status: Zaakceptowany

## Cel

Aplikacja webowa, która pobiera plik CSV z BaseLinker (eksport produktów Lignum Sacrum) i za pomocą Google Gemini API generuje nowe, zoptymalizowane pod SEO i konwersję opisy produktów (HTML). Wszystkie kolumny CSV pozostają niezmienione — nadpisywana jest tylko kolumna `opis` (J). Aplikacja musi być prosta, hostowana na Vercel jako strona statyczna.

## Wymagania funkcjonalne

1. Użytkownik wkleja swój klucz API Gemini (BYOK — Bring Your Own Key). Klucz może być zapamiętany w `localStorage`.
2. Użytkownik wgrywa plik CSV (separator `;`, UTF-8, format BaseLinker — kolumny `produkt_nazwa` w B, `opis` w J).
3. Po wgraniu UI pokazuje liczbę wykrytych produktów i opcje:
   - Delay między requestami (default 1000ms)
   - Czy nadpisywać też puste opisy (default: tak)
4. Po kliknięciu **Start**: progress bar + log na żywo (`[N/total] Nazwa produktu... OK / błąd`).
5. Możliwość anulowania w trakcie. Po anulowaniu — można pobrać częściowy wynik.
6. Po zakończeniu — przycisk pobrania nowego CSV (`<oryginalna-nazwa>_AI_<timestamp>.csv`).

## Wymagania niefunkcjonalne

- **Bezpieczeństwo:** klucz API nigdy nie opuszcza przeglądarki użytkownika. Brak backendu = brak ryzyka wycieku.
- **Prostota:** zero `npm install`, zero build steps. Jedno repo, dwa pliki kodu, gotowe do `vercel deploy`.
- **Stabilność:** rate limit + retry, częściowe pobranie wyniku w razie błędu.

## Architektura

Statyczna strona — czysty HTML + vanilla JS. Hostowana na Vercel jako static site.

```
lignum-sacrum-opisy/
├── index.html       # UI + cała logika
├── prompt.js        # prompt master (osobny plik)
├── README.md        # instrukcja
├── .gitignore
└── docs/superpowers/specs/2026-05-21-lignum-sacrum-opisy-design.md
```

Zależności (z CDN, bez bundlera):
- **PapaParse 5.x** — parsowanie/budowanie CSV z separatorem `;`

## Komponenty

### `prompt.js`
Eksportuje stałą `PROMPT_MASTER` — pełny prompt copywriterski Lignum Sacrum. Funkcja `buildPrompt(name, currentDesc)` zwraca tekst do wysłania do Gemini (prompt master + sekcja DANE PRODUKTU).

### `index.html`
Cała aplikacja: stan UI, parser CSV, kolejka requestów, downloader.

#### Funkcje główne:
- `parseCsv(file): {rows, headers, nameIdx, descIdx}` — wczytuje plik, waliduje obecność kolumn.
- `callGemini(apiKey, model, prompt): Promise<string>` — POST do `https://generativelanguage.googleapis.com/v1beta/models/${model}:generateContent?key=${apiKey}` z retry (max 2× exponential backoff przy 429/5xx).
- `extractBodyHtml(geminiResponse): string` — z odpowiedzi modelu wyciąga pierwszy blok ```html ... ``` (lub fallback: pierwszy `<ul>` do końca tekstu / cały tekst po `Body (HTML)`).
- `processAll(state)` — pętla po wierszach, aktualizuje progres, respektuje delay i flag anulowania.
- `downloadCsv(rows, headers, originalName)` — buduje CSV (PapaParse `unparse` z `quotes: true`, `delimiter: ";"`), tworzy Blob z prefiksem BOM `﻿` (żeby Excel poprawnie pokazał polskie znaki), wymusza download.

## Flow przetwarzania (krok po kroku)

1. Użytkownik wgrywa CSV → `parseCsv` waliduje. Jeśli brak `produkt_nazwa` lub `opis` → komunikat.
2. Klik Start → pętla:
   - Pobierz wiersz `i`
   - `prompt = buildPrompt(row.produkt_nazwa, row.opis)`
   - `geminiOutput = await callGemini(...)`
   - `newDesc = extractBodyHtml(geminiOutput)`
   - `row.opis = newDesc`
   - `log.append("[i/total] " + row.produkt_nazwa + " OK")`
   - `await sleep(delayMs)`
   - Jeśli `state.cancelled` → przerwij pętle.
3. Po pętli — aktywuj przycisk "Pobierz CSV".

## Obsługa błędów

| Sytuacja | Obsługa |
|---|---|
| Brak klucza API | Disable Start, komunikat |
| Pusty/błędny CSV | Komunikat, brak Start |
| Brak kol. `opis` / `produkt_nazwa` | Komunikat, brak Start |
| HTTP 429 / 5xx z Gemini | Retry 2× (1s, 3s), potem pomiń wiersz (zostaw stary opis) i loguj |
| Network error | Retry j.w. |
| Anulowanie | Przerwij, ale częściowy wynik dostępny do pobrania |
| Brak ```html``` block w odpowiedzi | Wstaw cały tekst odpowiedzi do `opis` + zaloguj ostrzeżenie |

## Deployment

1. Lokalnie: `git init` + commit
2. `gh repo create lignum-sacrum-opisy --public --source=. --push` (konto MarcinG1922)
3. Użytkownik klika "Import" w panelu Vercel → wybiera repo → Deploy (zero konfiguracji, Vercel sam wykryje statyczny site)
4. Output: `https://lignum-sacrum-opisy.vercel.app`

## Co NIE jest w zakresie (YAGNI)

- Autoryzacja / multi-user (BYOK rozwiązuje problem)
- Backend / baza danych
- Edycja opisów w UI przed zatwierdzeniem (user wybrał "wszystkie na raz")
- Tagi / SEO Title / SEO Description (BaseLinker CSV nie ma tych kolumn — user wybrał tylko Body HTML)
- Wsparcie dla innych formatów (Shopify CSV, XLSX) — w razie potrzeby później
- Wybór modelu w UI (default Gemini 2.5 Flash; w prawdzie zostawiamy stałą w kodzie do łatwej zmiany)
