# Lignum Sacrum — Generator opisów produktów

Webowa aplikacja, która przepisuje opisy produktów z eksportu CSV BaseLinker za pomocą Google Gemini API. Wszystkie kolumny CSV pozostają niezmienione — nadpisywana jest tylko kolumna `opis` (kolumna J).

## Jak działa

1. Wgrywasz plik CSV (eksport produktów z BaseLinker, separator `;`, UTF-8).
2. Aplikacja czyta `produkt_nazwa` (kolumna B) i `opis` (kolumna J) z każdego wiersza.
3. Wysyła do Gemini API prompt copywriterski Lignum Sacrum.
4. Z odpowiedzi wyciąga **Body HTML** i wstawia w miejsce starego opisu.
5. Pobierasz nowy CSV z wszystkimi kolumnami nietkniętymi (oprócz `opis`).

## Funkcje

- **BYOK** — klucz API Gemini wpisujesz w przeglądarce, nie opuszcza Twojego komputera.
- **Wybór modelu** — Gemini 2.5 Flash (default), 2.5 Pro lub 2.0 Flash.
- **Rate limit handling** — automatyczny retry przy 429/5xx z exponential backoff.
- **Skip / overwrite** — opcje pominięcia wierszy bez opisu lub już z opisem.
- **Anulowanie** — przerwij w trakcie, pobierz częściowy wynik.
- **UTF-8 BOM** — wynikowy CSV otwiera się poprawnie w Excelu.

## Użycie lokalne

To statyczna strona — bez build, bez npm. Otwórz `index.html` bezpośrednio w przeglądarce **albo** uruchom prosty serwer:

```bash
npx serve .
# lub
python -m http.server 8000
```

## Deployment na Vercel

1. **Fork / clone repo** na swoje konto GitHub.
2. Wejdź na [vercel.com/new](https://vercel.com/new), wybierz repo.
3. Vercel automatycznie wykryje statyczny site — kliknij **Deploy**.
4. Po kilkunastu sekundach dostajesz URL `https://<nazwa-repo>.vercel.app`.

Nie ma żadnych zmiennych środowiskowych do ustawienia — klucz API jest wpisywany przez użytkownika w przeglądarce.

## Edycja promptu

Cały prompt znajduje się w pliku [`prompt.js`](./prompt.js) jako stała `PROMPT_MASTER`. Możesz go modyfikować — zmiana wymaga tylko zapisania pliku i odświeżenia strony (lub re-deploya).

## Limity Gemini

| Plan | Wolumen |
|---|---|
| Free | 15 RPM, 1500/dzień |
| Tier 1 (Pay-as-you-go) | ~2000 RPM |

Przy default delay 1000ms aplikacja robi 60 RPM. Dla Free zwiększ delay do co najmniej `4000ms` lub przetwarzaj partiami.

## Format wejściowego CSV

Aplikacja oczekuje pliku CSV z BaseLinker w formacie:
- Separator: `;`
- Kodowanie: UTF-8
- Wymagane kolumny: `produkt_nazwa`, `opis`
- Pozostałe kolumny (id, sku, ceny, zdjęcia, opis_dodatkowy_1-4 itd.) są zachowywane bez zmian.

## Stack

- Vanilla JavaScript (brak frameworka)
- [PapaParse 5.x](https://www.papaparse.com/) z CDN — parsowanie/budowanie CSV
- Google Gemini API (`generativelanguage.googleapis.com/v1beta`)
- Hosting: Vercel static
