# Lignum Sacrum — Generator opisów produktów

Webowa aplikacja, która przepisuje opisy produktów z eksportu CSV BaseLinker za pomocą **Google Gemini**, **Anthropic Claude** lub **DeepSeek**. Wszystkie kolumny CSV pozostają niezmienione — nadpisywana jest tylko kolumna `opis` (kolumna J).

## Jak działa

1. Wgrywasz plik CSV (eksport produktów z BaseLinker, separator `;`, UTF-8).
2. Aplikacja czyta `produkt_nazwa` (kolumna B) i `opis` (kolumna J) z każdego wiersza.
3. Wysyła do Gemini API prompt copywriterski Lignum Sacrum.
4. Z odpowiedzi wyciąga **Body HTML** i wstawia w miejsce starego opisu.
5. Pobierasz nowy CSV z wszystkimi kolumnami nietkniętymi (oprócz `opis`).

## Funkcje

- **BYOK** — klucz API wpisujesz w przeglądarce, nie opuszcza Twojego komputera.
- **Wybór dostawcy** — Google Gemini (2.5 Flash / 2.5 Pro / 2.0 Flash), Anthropic Claude (Sonnet 4.6 / Opus 4.7 / Haiku 4.5) lub DeepSeek (v4-flash / v4-pro).
- **Rate limit handling** — automatyczny retry przy 429/529/5xx z exponential backoff.
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

## Skąd wziąć klucze API

| Dostawca | Link | Uwagi |
|---|---|---|
| Google Gemini | https://aistudio.google.com/apikey | Free tier: 15 RPM, 1500/dzień |
| Anthropic Claude | https://console.anthropic.com/settings/keys | Wymaga kredytu na koncie (brak free tier dla API) |
| DeepSeek | https://platform.deepseek.com/api_keys | Najtańszy: $0.14/1M input, $0.28/1M output (v4-flash) |

## Limity i koszty

**Gemini:** Free → 15 RPM. Tier 1 (paid) → ~2000 RPM.
**Anthropic:** zależy od tier — Tier 1 to 50 RPM dla Sonnet 4.6.
**DeepSeek:** ~60 RPM dla większości użytkowników. Najtańszy z trójki — Pro ma 75% rabat do 31.05.2026.

Przy default delay 1000ms aplikacja robi 60 RPM. Dla Gemini Free zwiększ delay do co najmniej `4000ms`.

## Format wejściowego CSV

Aplikacja oczekuje pliku CSV w formacie:
- Separator: `,` lub `;` (autodetekcja — eksport z BaseLinker domyślnie `;`, eksport z Google Sheets domyślnie `,`)
- Kodowanie: UTF-8 (z BOM lub bez)
- Wymagane kolumny: `produkt_nazwa`, `opis`
- Pozostałe kolumny (id, sku, ceny, zdjęcia, opis_dodatkowy_1-4 itd.) są zachowywane bez zmian
- Wynikowy CSV zachowuje ten sam separator co plik wejściowy

## Stack

- Vanilla JavaScript (brak frameworka)
- [PapaParse 5.x](https://www.papaparse.com/) z CDN — parsowanie/budowanie CSV
- Google Gemini API (`generativelanguage.googleapis.com/v1beta`)
- Anthropic Claude API (`api.anthropic.com/v1/messages`) — direct browser access z headerem `anthropic-dangerous-direct-browser-access`
- DeepSeek API (`api.deepseek.com/chat/completions`) — OpenAI-compatible format, CORS wspierany
- Hosting: Vercel static
