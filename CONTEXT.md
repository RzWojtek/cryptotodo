# CONTEXT.md — CryptoToDo / TODO Task Crypto

> Plik kontekstowy do wklejenia na początku nowego czatu z Claude.  
> Ostatnia aktualizacja: kwiecień 2026

---

## 1. Czym jest aplikacja

**CryptoToDo** (tytuł: „TODO Task Crypto") to osobista webowa aplikacja do zarządzania projektami kryptowalutowymi. Wojtek używa jej do śledzenia projektów DeFi/Web3, codziennych zadań, alertów terminowych, portfeli krypto, zapisów na whitelisty/waitlisty i notatek. Aplikacja działa wyłącznie dla jednego zalogowanego użytkownika (Google Auth), dane są prywatne i synchronizowane między urządzeniami przez Firebase.

---

## 2. Stack technologiczny

| Warstwa | Technologia |
|---|---|
| Frontend | Czysty HTML + CSS + JavaScript (vanilla, bez frameworka) |
| Baza danych | Firebase Firestore (compat SDK v10.12.0) |
| Autoryzacja | Firebase Authentication — Google Sign-In (signInWithPopup) |
| Hosting | GitHub Pages lub Vercel (plik `index.html` w repozytorium) |
| Fonty | Rajdhani (główny), Share Tech Mono (mono/kod) — Google Fonts |
| Ikony | Emoji + SVG inline (X/Twitter, Discord) |
| Brak | Node.js, npm, bundlera, frameworka — wszystko w jednym pliku HTML |

**Firebase projekt:** `cryptotodo`  
**authDomain:** `cryptotodo.firebaseapp.com`  
**API Key:** `AIzaSyClnrWiuGIxHZwHuRxNbhOKw8ARaZYutE8`

---

## 3. Struktura plików

Cała aplikacja to **jeden plik** `index.html` (~3080 linii). Nie ma build step, node_modules ani osobnych plików CSS/JS.

```
repo/
├── index.html          ← CAŁA aplikacja (HTML + CSS + JS w jednym pliku)
├── manifest.json       ← PWA manifest
├── sw.js               ← Service Worker (opcjonalny)
├── icon-16.png
├── icon-32.png
├── icon-192.png
├── przyklad-import.json  ← Przykładowy plik importu
└── przyklad-import.csv   ← Przykładowy plik importu CSV
```

**Deployment:** Wojtek wgrywa `index.html` przez GitHub web UI (nie terminal). Vercel automatycznie deployuje po każdym pushu.

---

## 4. Struktura UI — zakładki i sekcje

### Nawigacja
Górny pasek z 6 zakładkami (tab-nav), stan `currentTab` w JS.

### Zakładka 1: 📁 Projekty
- Siatka kart projektów (widok kart lub kompaktowy — toggle ▦/☰)
- Każda karta: nagłówek (nazwa, status badge, badge linki/dni/sieć/WL), body (2 kolumny: linki | notatki), stopka (edytuj/alert/duplikuj/usuń)
- Filtry: Wszystkie / Ważne (starred) + filtry sieci (Wszystkie/Mainnet/Testnet/Bez oznaczenia)
- Wyszukiwarka projektu (nazwa, notatki, URL linków)
- Statusy projektu: 🟢 W toku / 👀 Obserwuję / ⏸ Wstrzymany / ✅ Zakończony / ❌ Porzucony
- Zakończone i Porzucone automatycznie na końcu listy + wyszarzone (opacity: 0.6)
- Drag & drop kolejności
- Import/Eksport JSON i CSV
- Powiązania: badge WL na karcie → kliknięcie filtruje Whitelist; przycisk "+Alert" prefilluje modal alertu

### Zakładka 2: 📋 Daily Tasks
- Statystyki (4 karty): ukończone dziś, pozostało, passa dni (localStorage), wszystkich zadań
- Pasek postępu
- Zadania w **2 kolumnach** (grid 1fr 1fr, mobile 1 kolumna)
- Każde zadanie: checkbox + tekst + badge projektu (opcjonalnie) + przyciski edytuj/usuń
- Pod każdym zadaniem: **zwijany wykres 30 dni** (domyślnie zwinięty, kliknięcie toggleChart())
- Wykres: słupki per dzień (zielony=wykonany, żółty=dziś, ciemny=pominięty)
- Brak kategorii zadań (usunięte)

### Zakładka 3: 🔔 Alerty
- Karty alertów sortowane: nieukończone najpierw, po dacie
- Kolory pilności: czerwony ≤3 dni, żółty ≤7 dni, zielony OK
- Akcje: ✓ toggle ukończenia, ✏️ edytuj, 🗑 usuń
- Badge z liczbą pilnych alertów na zakładce nawigacji

### Zakładka 4: 👛 Portfele
- Karty portfeli z adresami
- Auto-ikona portfela (MetaMask🦊, Phantom👻, Rabby🐰, itp.)
- Notatka do każdego adresu
- Eksport CSV

### Zakładka 5: 🎯 Whitelist
- Zapisy WL/waitlisty z polami: nazwa, URL, status, metoda zapisu, liczba kont, data zapisu, priorytet, data odbioru nagrody, notatka
- Statusy: 🟢 Aktywne / ⏳ Oczekuję / 🏆 Wygrałem / ❌ Nie dostałem
- Priorytety: 🔴 Wysoki / 🟡 Średni / 🟢 Niski
- Metody zapisu (checkboxy): 👛 Wallet, 𝕏 Twitter, 📧 Email, 💬 Discord, ✈️ Telegram, 🔗 Inne
- Data odbioru nagrody → badge z licznikiem dni (czerwony ≤3, żółty ≤7)
- Przycisk "🔔 Alert" na karcie → tworzy alert z datą odbioru
- Statystyki: łączna liczba, aktywne, wygrane, win rate %
- Filtry + wyszukiwarka + sortowanie

### Zakładka 6: 📝 Notatnik
- Scratch pad z autosave (debounce 1.5s)
- Zapisywany w Firestore: `users/{uid}/scratch/pad`

---

## 5. Struktura danych Firestore

Wszystkie dane użytkownika są pod ścieżką:
```
users/{uid}/
  ├── projects/{projectId}     ← dokumenty projektów
  ├── tasks/{taskId}           ← dokumenty zadań daily
  ├── taskhistory/data         ← jeden dokument: { history: { taskId: { 'YYYY-MM-DD': true } } }
  ├── alerts/{alertId}         ← dokumenty alertów
  ├── wallets/{walletId}       ← dokumenty portfeli
  ├── wl/{wlId}                ← dokumenty whitelist
  └── scratch/pad              ← jeden dokument: { content: string, updatedAt: number }
```

### Schemat dokumentu projektu
```js
{
  id: string,           // genId() — Date.now().toString(36) + random
  name: string,
  links: [{ id, url, name, note }],
  notes: string,
  starred: boolean,
  network: 'mainnet' | 'testnet' | '',
  status: 'active' | 'watching' | 'paused' | 'done' | 'abandoned',
  socialX: string,
  socialDiscord: string,
  createdAt: number,    // Date.now()
  order: number         // do drag & drop
}
```

### Schemat dokumentu zadania (task)
```js
{
  id: string,
  text: string,
  projectId: string,    // opcjonalne — powiązanie z projektem
  projectName: string,  // cache nazwy projektu
  createdAt: number
  // BRAK pola cat (kategorie usunięte)
}
```

### Schemat dokumentu WL
```js
{
  id: string,
  name: string,
  url: string,
  status: 'active' | 'pending' | 'won' | 'lost',
  priority: 'high' | 'mid' | 'low',
  methods: ['wallet', 'twitter', 'email', 'discord', 'telegram', 'other'],
  accounts: number,
  date: 'YYYY-MM-DD',       // data zapisu
  claimDate: 'YYYY-MM-DD',  // data odbioru nagrody (opcjonalne)
  note: string,
  createdAt: number
}
```

---

## 6. Flow danych

```
Firebase Firestore
      ↕ (onSnapshot — real-time)
  STATE w pamięci JS
  let projects = []
  let tasks = []
  let taskHistory = {}
  let alertsList = []
  let wallets = []
  let wlEntries = []
      ↕
  render*() functions → innerHTML gridu
      ↕ (user action)
  save*ToDB() → Firestore → onSnapshot → re-render
```

**localStorage** (per user, per device):
- `taskdone_{uid}` — które zadania odhaczone dziś (reset o północy przez todayStr())
- `todocrypto_theme` — 'light' lub 'dark'
- `todocrypto_view` — 'cards' lub 'compact'
- `streak_{uid}` — passa dni (ile dni z rzędu wszystkie zadania ukończone)

**Historia zadań** (`taskHistory`) — w Firestore jako jeden dokument `taskhistory/data`, nie w localStorage — dostępna na wszystkich urządzeniach.

---

## 7. Powiązania między zakładkami

| Skąd | Dokąd | Mechanizm |
|---|---|---|
| Projekty | Alerty | Przycisk "🔔 Alert" w stopce karty → `quickAddAlert(projectName)` prefilluje tytuł alertu |
| Projekty | Whitelist | Badge "🎯 N WL" na karcie → `goToWL(name)` wpisuje nazwę do wyszukiwarki WL |
| Whitelist | Alerty | Przycisk "🔔 Alert" na karcie WL → `wlCreateAlert(wlId)` tworzy alert z claimDate |
| Daily Tasks | Projekty | Badge projektu na zadaniu → kliknięcie przełącza na zakładkę Projekty |

---

## 8. Motyw / Design System

### Paleta kolorów (dark mode — domyślny)
```css
--bg: #2e2e48          /* tło strony */
--bg2: #35355a
--bg3: #3c3c62
--card: #404068        /* tło karty */
--card2: #484862       /* nagłówki kart, stopki */
--border: #5a5a80
--border-glow: #6a6a96
--yellow: #f5e642      /* akcent główny */
--green: #39ff6e
--cyan: #00d4ff
--red: #ff3864
--text: #e0e0f0
--text-dim: #aaaacc
--text-muted: #7777aa
```

### Fonty
- Główny: `Rajdhani` (400/500/600/700) — tekst UI
- Mono: `Share Tech Mono` — kody, badge'e, daty, labele

### Konwencje UI
- Radius: `--radius: 8px`, `--radius-lg: 12px`
- Animacje wejścia: `@keyframes fadeIn` (translateY + opacity)
- Toast notyfikacje: zielony (sukces), żółty (info), czerwony (błąd)
- Modalne: `.modal-overlay` z klasą `.open`, zamykanie przez ESC lub klik w overlay
- Przyciski: `.btn-primary` (żółty), `.btn-secondary` (obramowanie), `.btn-danger` (czerwony), `.btn-ghost` (przezroczysty)

---

## 9. Konwencje kodu

- **Jeden plik** — cały HTML + CSS + JS w `index.html`
- **Vanilla JS** — brak frameworków, brak build step
- **genId()** — `Date.now().toString(36) + Math.random().toString(36).slice(2)` — unikalne ID
- **esc()** — escapowanie HTML w template literalach
- **todayStr()** — `new Date().toISOString().slice(0,10)` — format YYYY-MM-DD
- **onSnapshot** — real-time listeners zamiast jednorazowych getów
- **Modyfikacje: zawsze str_replace**, nigdy pełny rewrite — Wojtek preferuje inkrementalne zmiany
- **Przed wygenerowaniem pliku**: zawsze uruchomić audyt JS (`node --check`) i sprawdzić duplikaty funkcji, brakujące ID
- **Deployment**: Wojtek wgrywa przez GitHub web UI → Vercel auto-deploy. Nie używa terminal git.

---

## 10. Znane pułapki i naprawione bugi

### Krytyczne — zawsze pamiętaj:
1. **Jeden `</div>\`;` na końcu renderCard** — historycznie duplikował się po edycji, powodując błąd składni JS i brak działania całej aplikacji (w tym logowania)
2. **`auth.getRedirectResult()` przy starcie** — niszczy automatyczne logowanie w Firebase compat SDK. **Nigdy nie dodawać**.
3. **`setNetFilter` musi używać selektora `[data-net]`** a nie `.net-filter-btn` — inaczej zeruje filtry WL też
4. **`let wlEntries = []` musi być w STATE (góra)** — `renderCard` używa go przed sekcją WL
5. **Projekty puste na starcie** — nie używać `setTimeout` + `setView` w `render()`. Widok stosować synchronicznie przy pierwszym załadowaniu w `initApp`

### Logowanie Google:
- Używa **wyłącznie `signInWithPopup`** — nie dodawać `signInWithRedirect` ani `getRedirectResult`
- `onAuthStateChanged` zawsze ukrywa "Logowanie..." i odblokowuje przycisk

### Task historia:
- Przechowywana w Firebase (`taskhistory/data`), nie localStorage
- Format: `{ history: { taskId: { 'YYYY-MM-DD': true } } }`

---

## 11. Co zostało zbudowane (historia sesji)

### Funkcjonalności aplikacji (zbudowane od zera w tej sesji):
- ✅ Pełna aplikacja z Firebase Auth + Firestore
- ✅ Zakładka Projekty z kartami, drag&drop, importem/eksportem JSON+CSV
- ✅ Linki do projektu z notatkami i favicon, przycisk "Otwórz wszystkie"
- ✅ Edycja całego projektu (modal z nazwą, notatkami, siecią, X/Discord, statusem)
- ✅ Statusy projektu (W toku/Obserwuję/Wstrzymany/Zakończony/Porzucony) + sortowanie zakończonych na koniec
- ✅ Daily Tasks z historią 30-dniową (wykres słupkowy), pasta dni (streak)
- ✅ Wykres zwijany (domyślnie zamknięty), 2 kolumny zadań
- ✅ Edycja zadań, brak kategorii
- ✅ Historia zadań w Firebase (sync między urządzeniami)
- ✅ Alerty terminowe z kolorowymi badge'ami pilności
- ✅ Portfele z auto-ikoną i notatkami do adresów
- ✅ Whitelist/Waitlist z pełnym formularzem, filtrami, statystykami, win rate
- ✅ Data odbioru nagrody WL z licznikiem dni
- ✅ Notatnik z autosave do Firebase
- ✅ Powiązania między zakładkami (4 rodzaje)
- ✅ Tryb jasny/ciemny
- ✅ Widok kart/kompaktowy
- ✅ Nowa paleta kolorów (`#2e2e48` / `#484862`)
- ✅ Import/eksport projektów (JSON + CSV) ze statusem, siecią, notatkami linków
- ✅ PWA (manifest + theme-color)

---

## 12. Aktualny stan

| Co | Status |
|---|---|
| Logowanie Google | ✅ Działa (po naprawieniu błędu składni JS) |
| Synchronizacja Firestore | ✅ Real-time, all tabs |
| Projekty | ✅ W pełni funkcjonalne |
| Daily Tasks | ✅ Działa, historia w Firebase |
| Alerty | ✅ Działa |
| Portfele | ✅ Działa |
| Whitelist | ✅ Działa, edycja naprawiona |
| Notatnik | ✅ Działa |
| Powiązania między zakładkami | ✅ Zaimplementowane |
| Import/Eksport | ✅ JSON + CSV |
| Mobilna responsywność | ✅ Naprawiony header karty projektu |

### Potencjalnie do zrobienia (nie zaimplementowane):
- Wykres miesięczny w Projektach
- Powiadomienia push (alerty terminowe)
- Filtrowanie Daily Tasks po projekcie
- Historia zmian statusu projektu

---

## 13. Firestore Security Rules (wymagane)

```js
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /users/{userId}/{document=**} {
      allow read, write: if request.auth != null && request.auth.uid == userId;
    }
  }
}
```

---

## 14. Przydatne komendy

```bash
# Sprawdzenie błędów składni JS (wyciągnij JS z HTML)
python3 -c "
c=open('index.html').read(); lines=c.split('\n')
start=next(i for i,l in enumerate(lines) if '<script>' in l and 'firebase' not in l.lower())
js='\n'.join(lines[start+1:-2])
open('/tmp/check.js','w').write(js)
"
node --check /tmp/check.js

# Wgranie na GitHub przez web UI:
# github.com/[repo] → index.html → ołówek (edit) → wklej → Commit changes

# Firebase Authorized Domains (gdy logowanie nie działa):
# Firebase Console → Authentication → Settings → Authorized domains
# Dodaj: twoja-domena.vercel.app
```

---

## PROMPT STARTOWY

> Wklej ten blok jako pierwszą wiadomość w nowym czacie:

---

```
Cześć! Pracujemy razem nad moją aplikacją webową **CryptoToDo** (plik index.html, ~3080 linii).

## Stack
- Frontend: czysty HTML + CSS + vanilla JS, jeden plik index.html
- Backend/DB: Firebase Firestore (compat SDK v10.12.0), projekt: "cryptotodo"
- Auth: Firebase Google Sign-In (signInWithPopup TYLKO — nigdy nie dodawać getRedirectResult ani signInWithRedirect)
- Hosting: GitHub Pages / Vercel (wgrywam przez GitHub web UI, nie terminal)
- Fonty: Rajdhani + Share Tech Mono

## Zakładki aplikacji
1. 📁 Projekty — karty z linkami, notatkami, statusami (W toku/Zakończony/Porzucony/itp.), drag&drop, import/eksport
2. 📋 Daily Tasks — zadania (2 kolumny), zwijany wykres 30-dni, historia w Firebase, passa dni
3. 🔔 Alerty — terminy z kolorowymi badge'ami pilności
4. 👛 Portfele — adresy krypto z auto-ikoną
5. 🎯 Whitelist — zapisy WL z datą odbioru nagrody, win rate, filtry
6. 📝 Notatnik — scratch pad z autosave

## Struktura Firestore
users/{uid}/projects | tasks | taskhistory/data | alerts | wallets | wl | scratch/pad

## Kluczowe konwencje
- Modyfikacje ZAWSZE przez str_replace, nigdy pełny rewrite
- Przed wygenerowaniem pliku: sprawdź składnię JS (node --check) i duplikaty funkcji
- Paleta: --bg:#2e2e48 --card:#404068 --card2:#484862 --yellow:#f5e642 --green:#39ff6e --cyan:#00d4ff
- setNetFilter używa [data-net] (nie .net-filter-btn) — inaczej zeruje filtry WL
- let wlEntries musi być w STATE (góra pliku) — renderCard używa go wcześniej
- Nie używać setTimeout + setView w render() — powoduje puste projekty na starcie
- Nie dodawać auth.getRedirectResult() — niszczy automatyczne logowanie

## Znany krytyczny bug do unikania
Duplikat </div>`; na końcu renderCard powoduje błąd składni JS i brak działania CAŁEJ aplikacji.
Zawsze po edycji renderCard sprawdź czy nie ma dwóch linii zamykających template literal.

Proszę zapoznaj się z powyższym kontekstem. Co chcesz żebym zrobił?
```
