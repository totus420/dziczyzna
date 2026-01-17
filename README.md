# Szlachetna Dziczyzna — szybka dokumentacja

## Stos
- Front: `index.html` (single-page, Tailwind CDN, PapaParse, EmailJS).
- Backend: Google Apps Script Web App (zapis zamówień + licznik + update stanów magazynowych).
- Źródło produktów: Google Sheet CSV (`SHEET_URL`), kategorie i sortowanie w JS.

## Front (index.html)
- Pobiera CSV z arkusza przez PapaParse (header: true). Kluczowe kolumny arkusza produktów: `Nazwa`, `Kategoria`, `Opis`, `Skład`, `Cena`, `Dostępność`/`Dostepnosc`, `Jednostka`, opcjonalnie `Zdjęcie`.
- Kategorie: Kiełbasy, Wędliny, Przetwory, Inne; domyślnie zwinięte. Salami trafia do Kiełbas.
- Produkty z flagą wyprzedane (`Dostępność` <= 0) są nieklikalne.
- Detale produktu: akordeon „Szczegóły” (Opis/Skład/Ważność).
- Ceny:
  - Standard: cena × ilość (kg lub szt w zależności od `Jednostka`).
  - Szynka (słowo „szynk” w nazwie): wybór wariantu MAŁA (0,4–0,6) lub DUŻA (0,7–1,0). Ilość w szt., cena liczona jako cena/kg × 0.5 lub 0.85.
  - Polędwica (słowo „polęd”/„poled”): ilość w szt., cena liczona jako cena/kg × 0.6; dopisek „Kawałki ok. 0,5–0,7 kg”.
- Koszyk: sticky na desktopie, przyciski +/-/Usuń, rabaty z kodów (`ZNAJOMY`: 10%, `PRZYJACIEL`: 20%). Minimalne zamówienie 200 zł; przy pustym koszyku przycisk submit jest disabled.
- Formularz: walidacja podstawowa + `autocomplete` (name, tel, email, address-line1, address-level2).
- Numer zamówienia: najpierw próba pobrania z GAS (`action: nextOrderNumber`), fallback lokalny (licznik w localStorage, bez prefiksu).
- Zapis zamówienia: fetch POST do `GOOGLE_SCRIPT_URL` (najpierw `cors`, przy błędzie fallback `no-cors`). Payload: `sheet_id`, `sheet_tab`, dane klienta, `zamowienie/szczegoly`, `kwota`, `line_items` (name, qty, unit), `order_number`, `discount_code`.
- Email: EmailJS (service/template zdefiniowane w pliku).

## Backend (Google Apps Script)
- Web App (Deploy → Web App, Anyone). Kod w pliku GAS (patrz sekcja „Kod GAS” poniżej).
- Licznik numerów zamówień: `ScriptProperties.orderCounter`, start `COUNTER_MIN = 300`, `ORDER_PREFIX = ''`. Endpoint akcji: POST body `{ action: "nextOrderNumber" }` → zwraca JSON `{ ok: true, orderNumber, number }`.
- Zapisywanie zamówienia:
  - Arkusz docelowy: `sheet_tab` lub domyślnie `Zamowienia`. Tworzy nagłówki: `Numer zamówienia | Data | Imię i Nazwisko | Telefon | Email | Adres | Zamówienie | Kwota | Kod rabatowy | Uwagi`.
  - Numer zamówienia: bierze z `order_number` / `orderNumber` / `number`, inaczej generuje nowy.
  - Linie zamówienia: `line_items` (name, qty, unit) przekazywane do stock update.
  - Kod rabatowy: zapisuje `discount_code` (gdy obecny).
- Update stanów magazynowych:
  - Preferuje arkusz `Produkty`; jeśli brak, szuka pierwszego arkusza z kolumnami `Nazwa` i `Dostępność`/`Dostepnosc`.
  - Obsługa wariantów szynki: jeśli arkusz ma kolumny `Dostępność (mała)` i `Dostępność (duża)`, a w pozycji koszyka jest wariant MAŁA/DUŻA, aktualizuje odpowiednią kolumnę. Kolumna łączna `Dostępność` też jest zmniejszana (jeśli istnieje).
  - Dopasowanie nazw: znormalizowana nazwa bez sufiksu w nawiasach (np. `... (MAŁA 0,5 kg)`).
  - Logger wypisuje dopasowania i nowe stany.

### Kod GAS
```
const DEFAULT_SHEET_ID = '18bHIaug8Jv424muP0_r9mrmMglJbNqHGhnHTLELrzLE';
const DEFAULT_TAB = 'Zamowienia';
const PRODUCT_TAB = 'Produkty';
const ORDER_PREFIX = '';
const COUNTER_MIN = 300;

function doPost(e) {
  try {
    const data = parseRequest(e);
    if (data.action === 'nextOrderNumber') {
      const nr = nextOrderNumber();
      return jsonResponse({ ok: true, orderNumber: nr.full, number: nr.num });
    }
    const sheetId = data.sheet_id || DEFAULT_SHEET_ID;
    const tabName = data.sheet_tab || DEFAULT_TAB;
    const ss = SpreadsheetApp.openById(sheetId);
    let orderSheet = ss.getSheetByName(tabName);
    if (!orderSheet) {
      orderSheet = ss.insertSheet(tabName);
      orderSheet.appendRow(['Numer zamówienia', 'Data', 'Imię i Nazwisko', 'Telefon', 'Email', 'Adres', 'Zamówienie', 'Kwota', 'Kod rabatowy', 'Uwagi']);
    }
    const date = new Date();
    const orderNr = data.order_number || data.orderNumber || data.number || nextOrderNumber().full;
    orderSheet.appendRow([
      orderNr,
      date,
      "'" + (data.imie_nazwisko || ''),
      "'" + (data.telefon || ''),
      data.email || '',
      data.adres || '',
      data.zamowienie || data.szczegoly || '',
      data.kwota || '',
      data.discount_code || '',
      data.uwagi || ''
    ]);
    const items = Array.isArray(data.line_items) ? data.line_items : [];
    if (items.length) updateStock(ss, items);
    return jsonResponse({ ok: true, orderNumber: orderNr });
  } catch (err) {
    return jsonResponse({ ok: false, error: err.toString() });
  }
}

function doGet() { return jsonResponse({ ok: true }); }

function parseRequest(e) {
  if (e && e.postData && e.postData.contents) { try { return JSON.parse(e.postData.contents); } catch (err) {} }
  return e && e.parameter ? e.parameter : {};
}
function nextOrderNumber() {
  const props = PropertiesService.getScriptProperties();
  let current = parseInt(props.getProperty('orderCounter') || COUNTER_MIN, 10);
  if (isNaN(current) || current < COUNTER_MIN) current = COUNTER_MIN;
  const next = current + 1;
  props.setProperty('orderCounter', String(next));
  const full = ORDER_PREFIX ? ORDER_PREFIX + next : String(next);
  return { full, num: next };
}
function jsonResponse(obj) {
  const output = ContentService.createTextOutput(JSON.stringify(obj));
  output.setMimeType(ContentService.MimeType.JSON);
  return output;
}
function normalize(str) {
  return (str || '').toString().toLowerCase()
    .replace(/[ąćęłńóśźż]/g, c => ({'ą':'a','ć':'c','ę':'e','ł':'l','ń':'n','ó':'o','ś':'s','ź':'z','ż':'z'}[c] || c))
    .trim();
}
function normalizeProductName(name) {
  return normalize(name).replace(/\(.*?\)/g, '').trim();
}
function getVariantKey(name) {
  const n = normalize(name);
  if (n.includes('duz')) return 'large';
  if (n.includes('mal')) return 'small';
  return '';
}
function updateStock(ss, items) {
  Logger.log('Line items: %s', JSON.stringify(items));
  let productSheet = ss.getSheetByName(PRODUCT_TAB);
  let nameCol, stockCol, smallCol, largeCol;
  if (!productSheet) {
    ss.getSheets().some(sh => {
      const headers = sh.getRange(1, 1, 1, sh.getLastColumn()).getValues()[0];
      Logger.log('Sheet %s headers: %s', sh.getName(), headers);
      const idxName = headers.findIndex(h => normalize(h) === 'nazwa');
      const idxStock = headers.findIndex(h => { const n = normalize(h); return n === 'dostepnosc' || n === 'dostępność'; });
      const idxSmall = headers.findIndex(h => normalize(h) === 'dostepnosc (mala)');
      const idxLarge = headers.findIndex(h => normalize(h) === 'dostepnosc (duza)');
      if (idxName !== -1 && (idxStock !== -1 || idxSmall !== -1 || idxLarge !== -1)) {
        productSheet = sh; nameCol = idxName + 1; stockCol = idxStock + 1; smallCol = idxSmall + 1; largeCol = idxLarge + 1; return true;
      }
      return false;
    });
  }
  if (!productSheet) { Logger.log('Product sheet not found'); return; }
  if (!nameCol || !stockCol) {
    const headers = productSheet.getRange(1, 1, 1, productSheet.getLastColumn()).getValues()[0];
    nameCol = headers.findIndex(h => normalize(h) === 'nazwa') + 1;
    stockCol = headers.findIndex(h => { const n = normalize(h); return n === 'dostepnosc' || n === 'dostępność'; }) + 1;
    smallCol = headers.findIndex(h => normalize(h) === 'dostepnosc (mala)') + 1;
    largeCol = headers.findIndex(h => normalize(h) === 'dostepnosc (duza)') + 1;
  }
  if (nameCol <= 0 || (stockCol <= 0 && smallCol <= 0 && largeCol <= 0)) { Logger.log('Name/stock columns not found'); return; }
  const lastRow = productSheet.getLastRow();
  if (lastRow < 2) { Logger.log('No data rows'); return; }
  const nameVals = productSheet.getRange(2, nameCol, lastRow - 1, 1).getValues().map(r => normalizeProductName(r[0]));
  items.forEach(it => {
    const n = normalizeProductName(it.name);
    const qty = parseFloat(it.qty || 0);
    if (!n || !qty) return;
    const idx = nameVals.indexOf(n);
    Logger.log('Match %s -> row idx %s', n, idx);
    if (idx === -1) return;
    const row = idx + 2;
    const variant = getVariantKey(it.name);
    if (stockCol > 0) {
      const cur = parseFloat(productSheet.getRange(row, stockCol).getValue()) || 0;
      const next = Math.max(0, cur - qty);
      productSheet.getRange(row, stockCol).setValue(next);
      Logger.log('Row %s (global stock): %s -> %s', row, cur, next);
    }
    if (variant === 'small' && smallCol > 0) {
      const curS = parseFloat(productSheet.getRange(row, smallCol).getValue()) || 0;
      const nextS = Math.max(0, curS - qty);
      productSheet.getRange(row, smallCol).setValue(nextS);
      Logger.log('Row %s (small): %s -> %s', row, curS, nextS);
    } else if (variant === 'large' && largeCol > 0) {
      const curL = parseFloat(productSheet.getRange(row, largeCol).getValue()) || 0;
      const nextL = Math.max(0, curL - qty);
      productSheet.getRange(row, largeCol).setValue(nextL);
      Logger.log('Row %s (large): %s -> %s', row, curL, nextL);
    }
  });
}
```

## Deploy / konfiguracja
1. GAS: Deploy → New deployment → Web app → Anyone. Skopiuj URL do `GOOGLE_SCRIPT_URL` w `index.html`.
2. W arkuszu produktów zadbaj o kolumny `Nazwa`, `Dostępność` (lub `Dostepnosc`), `Cena`, `Jednostka`, `Opis`, `Skład`, opcjonalnie `Zdjęcie`.
3. Licznik numerów: zarządzany w `ScriptProperties` (reset przez usunięcie klucza `orderCounter` lub zmianę `COUNTER_MIN` i redeploy).

## Branching i publikacja
- Podgląd: obecnie brak skonfigurowanego brancha pod GitHub Pages (historycznie `ui-tweaks`). Jeśli potrzebny staging, utwórz gałąź (np. `ui-tweaks`) i włącz Pages dla niej.
- Produkcja: branch `main` jest źródłem deployu na home.pl przez FTP (workflow).
- Zasada: pracuj na gałęzi roboczej, po akceptacji merge/push do `main` → automatyczny deploy na home.pl.

## Automatyczny deploy (home.pl)
- Workflow: `.github/workflows/deploy.yml`, odpala się na push do `main` lub ręcznie z Actions.
- Wymagane GitHub Secrets (Repository secrets):
  - `FTP_HOST` – host FTP z panelu home.pl (np. `serwer12345.home.pl`)
  - `FTP_USERNAME` – użytkownik FTP
  - `FTP_PASSWORD` – hasło FTP
  - `FTP_REMOTE_DIR` – katalog docelowy, np. `/public_html/szlachetnadziczyzna/`
- Po puszu do `main`:
  - Actions → run „Deploy to home.pl”; w logach widać listę przesłanych plików.
  - Na serwerze pojawia się `.ftp-deploy-sync-state` (nie usuwać, to stan różnicowy).
- Jeżeli w panelu włączone FTPS/SFTP, w workflow można zmienić `protocol: ftp` na `ftps` lub `sftp`.

## Jak pracować na branchach
- Start pracy: `git checkout ui-tweaks` (lub inny feature branch), `git pull`.
- Wprowadzaj zmiany, commituj i `git push origin ui-tweaks` → podgląd na GitHub Pages.
- Akceptacja: merge `ui-tweaks` → `main`, potem `git push origin main` → deploy na home.pl ruszy automatycznie.
- Nie commitujemy `main` bez review/akceptacji (produkcyjny deploy rusza od razu).

## Znane ograniczenia / notatki
- Web App nie zwraca nagłówków CORS z `ContentService`; front ma fallback (CORS→no-cors). Brak wglądu w błędy sieciowe po stronie frontu w trybie no-cors.
- Numery zamówień są globalne dla deploymentu GAS (nie unikalne między różnymi wdrożeniami).
- Stany magazynowe: wymagają zgodności nazw w `line_items.name` z kolumną `Nazwa` po normalizacji (małe litery, bez polskich znaków).
