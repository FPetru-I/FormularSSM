## CUPRINS

1. [Prezentare generală](#1-prezentare-generală)
2. [Arhitectură și flux de date](#2-arhitectură-și-flux-de-date)
3. [Structura HTML](#3-structura-html)
4. [CSS — Clase și variabile](#4-css--clase-și-variabile)
5. [Configurare și constante](#5-configurare-și-constante)
6. [Variabile globale](#6-variabile-globale)
7. [Gestionarea imaginilor per întrebare](#7-gestionarea-imaginilor-per-întrebare)
8. [Sistemul de răspunsuri DA/NU/N/A](#8-sistemul-de-răspunsuri-danuna)
9. [Progres și validare formular](#9-progres-și-validare-formular)
10. [Upload imagini secțiunea 16](#10-upload-imagini-secțiunea-16)
11. [Modal completare incompletă](#11-modal-completare-incompletă)
12. [Generare PDF](#12-generare-pdf)
13. [Trimitere raport (trimiteRaport)](#13-trimitere-raport-trimiteraport)
14. [Construire HTML pentru PDF (buildPDFHtml)](#14-construire-html-pentru-pdf-buildpdfhtml)
15. [PDF Feedback formular](#15-pdf-feedback-formular)
16. [Semnătură digitală](#16-semnătură-digitală)
17. [Inițializare formular (initForm)](#17-inițializare-formular-initform)
18. [Utilitare](#18-utilitare)
19. [Referință completă funcții](#19-referință-completă-funcții)
20. [Jurnal modificări](#20-jurnal-modificări)

\---

## 1\. Prezentare generală

`index.html` este un formular de inspecție SSM (Securitate și Sănătate în Muncă) care rulează complet în browser, fără backend propriu. Utilizatorul completează **101 întrebări** structurate în 19 secțiuni, adaugă imagini, semnează digital, și trimite raportul care este:

* Generat ca PDF și descărcat local (fereastră nouă + dialog print)
* Uploadat în GitHub (folder `rapoarte/`) prin proxy GAS ca fișier PDF binar
* Salvat ca HTML raw în GitHub (folder `rapoarte\_raw/`)
* Înregistrat în centralizatorul CSV (`centralizator.csv`)
* Feedback-ul formularului (secțiunea 19) salvat separat (folder `feedback-formular/`)

**Tehnologii folosite:**

|Librărie|Versiune|CDN|Rol|
|-|-|-|-|
|jsPDF|2.5.1|cdnjs|Generare fișier PDF binar|
|html2canvas|1.4.1|cdnjs|Captură HTML→Canvas pentru PDF|
|SignaturePad|5.1.3|jsdelivr|Câmp semnătură digitală pe canvas|

\---

## 2\. Arhitectură și flux de date

```
Utilizator completează formular
        │
        ▼
Apasă "Salvează și Descarcă"
        │
        ├─► Verificare câmpuri obligatorii (validate())
        │       └─► Dacă incomplet → Modal avertizare (showIncompleteModal)
        │
        ├─► Citire imagini atașate (fileToBase64)
        │
        ├─► Obținere număr raport de la GAS (?action=nextNr)
        │       └─► Sau din cache (cachedNrRaport) dacă există
        │
        ├─► Construire payload (buildPayload)
        │
        ├─► Generare PDF binar (html2canvas + jsPDF)
        │
        ├─► Upload PDF → GAS → GitHub /rapoarte/RN\_Loc\_Data.pdf
        │       └─► Actualizare centralizator.csv
        │
        ├─► Generare + Upload PDF feedback → GitHub /feedback-formular/
        │
        ├─► Upload HTML raw → GitHub /rapoarte\_raw/RN\_Loc\_Data.html
        │
        └─► Deschidere PDF local în fereastră nouă + print dialog
```

**Fluxul unui upload prin GAS:**

```
Browser → POST GAS\_PROXY\_URL (JSON cu pdfHtmlBase64)
GAS → validare email (doar pentru folderul principal)
GAS → GitHub API PUT /contents/rapoarte/filename
GAS → actualizare centralizator.csv
GAS → răspuns { status: 'success', fileName, nrRaport }
```

**De ce GAS ca proxy:**
Tokenul GitHub este stocat exclusiv în GAS (server-side). Browserul nu îl vede niciodată. Validarea emailului se face tot în GAS, nu în frontend, pentru a nu expune domeniul acceptat în codul sursă al paginii.

\---

## 3\. Structura HTML

```
<head>
  ├── CSS Reset (normalizare cross-browser)
  ├── Variabile CSS custom (:root)
  ├── Stiluri componente
  └── Script-uri externe (jsPDF, html2canvas, SignaturePad)

<body>
  ├── #firefox-warning          — banner avertizare Firefox (ascuns implicit)
  ├── .progress-bar             — bara de progres completare (sticky top, 3px)
  ├── <header class="page-header"> — titlu "Formular de Raportare v33"
  └── .container (max-width: 860px)
        ├── .meta-grid          — câmpuri identificare:
        │     Nume complet\*, Email\*, Data inspecției\*, Localitate\*, Tip lucrare\*
        │
        ├── Secțiunile 1–13     — întrebări DA/NU/N/A 
        │     ├── 1. Verificare generală șantier
        │     ├── 2. Lucrări electrice 
        │     ├── 3. Lucru la înălțime 
        │     ├── 4. Risc cădere obiecte și manipulare încărcături
        │     ├── 5. Utilaje cu piese în mișcare 
        │     ├── 6. Excavații și lucrări în sol 
        │     ├── 7. Manipulare și depozitare materiale 
        │     ├── 8. Substanțe chimice și periculoase 
        │     ├── 9. Lucru izolat 
        │     ├── 10. Risc de incendiu 
        │     ├── 11. Spații închise 
        │     ├── 12. Operațiuni de tăiere 
        │     └── 13. Lucrări de betonare și turnare 
        │
        ├── Secțiunea 14 (#section-14-text)  — Feedback inspector 
        ├── Secțiunea 15 (#section-15-text)  — Feedback șef echipă 
        ├── Secțiunea 16       — Observații (16.1 text) + upload imagini (16.2)
        ├── Secțiunea 17       — Concluzii finale (text liber)
        ├── Secțiunea 18       — Semnătură digitală inspector
        ├── Secțiunea 19       — Feedback formular (3 câmpuri text)
        │
        ├── .actions-bar       — butoane "⬇ Descarcă PDF" / "📤 Salvează și Descarcă"
        │     └── .actions-left — contor "X din Y întrebări completate"
        │
        ├── .overlay           — ecran loading semi-transparent (generare PDF)
        ├── #incompleteModal   — modal avertizare formular incomplet
        └── .toast             — notificări temporare (colț dreapta-jos)
```

\*câmpuri obligatorii marcate cu \*

\---

## 4\. CSS — Clase și variabile

### Variabile CSS (:root)

```css
--bg            #f4f3ef   /\* fundal pagină \*/
--surface       #fff      /\* fundal carduri/secțiuni \*/
--surface2      #f9f8f5   /\* fundal câmpuri input \*/
--border        #dddbd4   /\* borduri normale \*/
--border-strong #b8b5ac   /\* borduri accentuate \*/
--text          #1a1917   /\* text principal \*/
--text-muted    #6b6860   /\* text secundar/label \*/
--text-faint    #9b9890   /\* text dezactivat/placeholder \*/
--accent        #1a4d8f   /\* albastru principal (butoane, focus, header) \*/
--accent-light  #e8eef7   /\* albastru deschis (fundal hover) \*/
--yes           #1a6b3c   /\* verde — răspuns DA \*/
--yes-bg        #eaf5ee   /\* fundal verde deschis \*/
--no            #8f1a1a   /\* roșu — răspuns NU \*/
--no-bg         #f5eaea   /\* fundal roșu deschis \*/
--na            #1a6b9e   /\* albastru — răspuns N/A \*/
--na-bg         #e0f0fa   /\* fundal albastru deschis \*/
--danger        #c0392b   /\* roșu eroare/validare \*/
--radius        6px       /\* raza colțuri rotunjite \*/
```

### Clase principale

|Clasă|Element|Descriere|
|-|-|-|
|`.section`|`<div>`|Container o secțiune de întrebări|
|`.section-header`|`<div>`|Header secțiune cu număr și titlu|
|`.section-num`|`<div>`|Cerc albastru cu numărul secțiunii|
|`.section-title`|`<div>`|Titlul secțiunii|
|`.question-row`|`<div>`|Rând pentru o întrebare DA/NU/N/A|
|`.q-top`|`<div>`|Container index + text + butoane răspuns|
|`.q-text`|`<span>`|Textul întrebării|
|`.q-index`|`<span>`|Indexul întrebării (ex. 1.3)|
|`.q-options`|`<div>`|Container butoane DA/NU/N/A|
|`.opt-btn`|`<button>`|Buton opțiune răspuns (stare implicită)|
|`.opt-btn.selected-yes`|stare|Buton DA selectat (fundal verde)|
|`.opt-btn.selected-no`|stare|Buton NU selectat (fundal roșu)|
|`.opt-btn.selected-na`|stare|Buton N/A selectat (fundal albastru)|
|`.comment-section`|`<div>`|Zona note + imagini (ascunsă implicit)|
|`.comment-section.open`|stare|Zona note vizibilă|
|`.q-img-upload`|`<div>`|Container upload imagini per întrebare|
|`.q-thumb`|`<div>`|Thumbnail imagine cu câmp redenumire|
|`.text-question-row`|`<div>`|Rând întrebare cu răspuns text liber|
|`.meta-grid`|`<div>`|Grid câmpuri identificare|
|`.field-group`|`<div>`|Container label + input|
|`.drop-zone`|`<div>`|Zona drag\&drop imagini secțiunea 16|
|`.actions-bar`|`<div>`|Bara cu butoane de acțiune (fixă jos)|
|`.actions-left`|`<div>`|Contor progres completare|
|`.overlay`|`<div>`|Ecran loading semi-transparent|
|`.toast`|`<div>`|Notificare tip toast|
|`.modal-overlay`|`<div>`|Fond semi-transparent modal|
|`.modal-card`|`<div>`|Cardul modal avertizare incomplet|
|`.progress-bar`|`<div>`|Container bara progres sticky|
|`.progress-fill`|`<div>`|Umplerea barei de progres|
|`.sig-canvas-wrap`|`<div>`|Container canvas semnătură|

### Media queries

```css
/\* Layout mobil \*/
@media (max-width: 600px) {
  .meta-grid { grid-template-columns: 1fr; }   /\* 1 coloană în loc de 2 \*/
  .opt-btn   { padding: 12px 16px; flex: 1; }   /\* butoane mai mari touch \*/
  .actions-bar { flex-direction: column; }
  .btn { width: 100%; }
  .field-group input, select, textarea { font-size: 16px; }  /\* previne zoom iOS \*/
  .sig-grid { grid-template-columns: 1fr; }
  .q-text { font-size: 20px; line-height: 1.6; }  /\* text întrebări mărit \*/
}
```

> \*\*Notă privind printarea:\*\* media query-ul `@media (max-width: 600px)` nu afectează PDF-ul generat. `html2canvas` captează containerul la lățime fixă de 860px într-un div ascuns în afara viewport-ului, independent de dimensiunea ecranului.

\---

## 5\. Configurare și constante

```javascript
// ── Întrebările formularului DA/NU/N/A ──
const QUESTIONS = {
  1:  \[...],   // 26 întrebări — Verificare generală șantier
  2:  \[...],   // 15 întrebări — Lucrări electrice
  3:  \[...],   // 10 întrebări — Lucru la înălțime
  4:  \[...],   //  7 întrebări — Risc cădere obiecte
  5:  \[...],   //  3 întrebări — Utilaje cu piese în mișcare
  6:  \[...],   //  3 întrebări — Excavații și lucrări în sol
  7:  \[...],   //  3 întrebări — Manipulare și depozitare materiale
  8:  \[...],   //  5 întrebări — Substanțe chimice
  9:  \[...],   //  2 întrebări — Lucru izolat
  10: \[...],   //  7 întrebări — Risc de incendiu
  11: \[...],   //  5 întrebări — Spații închise
  12: \[...],   //  4 întrebări — Operațiuni de tăiere
  13: \[...],   //  5 întrebări — Lucrări de betonare și turnare
  // TOTAL: 95 întrebări DA/NU/N/A
}

// ── Întrebările text (secțiunile 14 și 15) ──
const TEXT\_SECTIONS = {
  14: \[  // 3 întrebări text — Feedback inspector
    "Cum au întâmpinat membrii echipei acțiunea de control?...",
    "Care a fost reacția în fața neconformităților?...",
    "Cum apreciați calitatea interacțiunii cu șeful de lucrare?..."
  ],
  15: \[  // 1 întrebare text — Feedback șef echipă
    "Cum apreciați calitatea interacțiunii cu inspectorul?..."
  ],
}
// + 16.1 (observații) + 17.1 (concluzii) = 101 întrebări totale

// ── URL-ul proxy-ului GAS ──
const GAS\_PROXY\_URL = 'https://script.google.com/macros/s/.../exec'
// Toate uploadurile și obținerea numărului de raport merg prin acest endpoint
```

**Adăugarea de noi secțiuni:**

* Secțiuni DA/NU/N/A: adaugă array în `QUESTIONS{}` → se integrează automat în `answers{}`, `updateProgress()`, `buildPayload()` și `buildPDFHtml()`
* Secțiuni text: adaugă în `TEXT\_SECTIONS{}` + HTML corespunzător → se integrează în `textAnswers{}` și `getTotalQuestions()`

\---

## 6\. Variabile globale

```javascript
// ── Starea răspunsurilor ──
let answers = {}
// Stochează răspunsurile tuturor întrebărilor DA/NU/N/A
// Structură: { "1-0": { value: "DA", comment: "..." }, "2-3": { value: null, comment: "" } }
// Cheie = "secțiune-index" (ex. "1-0" = secțiunea 1, prima întrebare)
// value: "DA" | "NU" | "N/A" | null (null = fără răspuns)
// Populat automat din DOM la initForm()

let images = \[]
// Fișierele File din secțiunea 16 (Observații — imagini generale)
// Elementele șterse devin null (index-urile se păstrează)

let qImages = {}
// Imaginile atașate per întrebare, embedate în PDF sub răspuns
// Structură: { "1-0": \[ {dataUrl: "data:image/...", name: "foto"}, null, ... ] }

let textAnswers = {}
// Răspunsurile text din secțiunile 14 și 15
// Structură: { "14-0": "text...", "14-1": "...", "15-0": "..." }

// ── Câmpuri text libere ──
let obsText = ''          // 16.1 — Încălcări reguli siguranță identificate
var concluziiText = ''    // 17.1 — Note și concluzii generale

// ── Feedback formular secțiunea 19 ──
// Valorile sunt citite direct din DOM la momentul trimiterii (în trimiteRaport)
// Variabilele globale sunt păstrate pentru compatibilitate cu event delegation
let fbSugestii = ''       // 19.2 — Sugestii îmbunătățire
var fbProbleme = ''       // 19.1 — Probleme întâmpinate
var fbGeneral  = ''       // 19.3 — Feedback general

// ── Semnătură digitală ──
var sigInspectorName    = ''   // Câmpul "Nume" din secțiunea 18
var sigInspectorFunctie = ''   // Câmpul "Funcție"
var sigInspectorFirma   = ''   // Câmpul "Firma"
var sigInspectorData    = ''   // Câmpul "Data" semnăturii
var sigCanvases = {}           // Instanțele SignaturePad: { 'inspector': SignaturePad }

// ── Cache număr raport ──
var cachedNrRaport = null
// Numărul preîncărcat de la GAS la deschiderea paginii (ex. "R5")
// Scopul: evitarea unui delay la primul click pe butonul de salvare

var cachedNrUsed = false
// Rămâne false până la un upload reușit, permițând reutilizarea aceluiași număr
// pentru descărcarea locală și uploadul în GitHub

// ── Modal stare ──
var \_pendingAction = null
// Acțiunea în așteptare când apare modalul de incomplet: 'download' | 'submit'
```

\---

## 7\. Gestionarea imaginilor per întrebare

Fiecare întrebare DA/NU/N/A are un buton "📷 Adaugă imagine" care permite atașarea de fotografii direct la întrebarea respectivă. Imaginile sunt incluse în PDF imediat sub răspunsul și comentariul întrebării respective.

### `clearQImagesForKey(key)`

```
Parametri: key — "secțiune-index" (ex. "1-0")
Rol: Șterge toate imaginile din qImages\[key] și curăță thumbnail-urile din DOM.
Când e apelat: la deselectarea unui răspuns sau la selectarea N/A.
```

### `handleQImg(key, files)`

```
Parametri:
  key   — identificatorul întrebării (ex. "2-3")
  files — FileList din input\[type=file]

Flux:
1. Verifică tip fișier → acceptă doar image/\*
2. Verifică dimensiune → max 5MB per fișier; showToast dacă depășit
3. Citește cu FileReader → dataURL base64
4. Adaugă în qImages\[key]\[] obiect {dataUrl, name}
5. Creează thumbnail în DOM cu:
   - Preview vizual al imaginii
   - Buton ✕ pentru ștergere
   - Input text editabil pentru redenumire (legendă în PDF)
6. Resetează valoarea input-ului (permite re-selectarea aceluiași fișier)
```

### `renameQImg(key, idx, val)`

```
Actualizează qImages\[key]\[idx].name când utilizatorul editează câmpul text
din thumbnail. Numele este folosit ca legendă sub imagine în PDF.
```

### `removeQImg(key, idx)`

```
Marchează imaginea ca null în array (nu reindexează pentru a păstra
referințele DOM corecte) și îndepărtează thumbnail-ul din pagină.
```

\---

## 8\. Sistemul de răspunsuri DA/NU/N/A

### Inițializare

La `initForm()`, pentru fiecare buton `.opt-btn\[data-key]` din DOM:

```javascript
answers\["1-0"] = { value: null, comment: "" }
// value null = fără răspuns (contribuie la contorul de incomplete)
```

### `selectAnswer(btn)`

Funcția centrală, apelată prin event delegation (click pe `.container`).

```
1. Citește key (ex. "1-0") și val ("DA"/"NU"/"N/A") din data-\* ale butonului

2. Verifică toggle (re-click pe butonul deja selectat = deselectare):
   DACĂ toggle:
     → answers\[key].value = null
     → Șterge nota din textarea
     → clearQImagesForKey(key)
     → Închide .comment-section
     → updateProgress()
     → return

3. ALTFEL (selectare nouă):
   → Elimină toate clasele selected-\* din rândul curent
   → Aplică clasa corespunzătoare pe butonul apăsat:
       DA  → selected-yes (verde)
       NU  → selected-no  (roșu)
       N/A → selected-na  (albastru)
   → Salvează answers\[key].value = val

   DACĂ N/A:
     → Șterge nota și imaginile (irelevante pentru N/A)
     → Ascunde .comment-section

   DACĂ DA sau NU:
     → Afișează .comment-section (adaugă clasa .open)
     → Focus pe textarea pentru editare rapidă

4. updateProgress()
```

### `updateComment(key, val)`

Apelat prin event delegation pe `textarea\[data-comment-key]`.
Actualizează `answers\[key].comment` în timp real pe măsură ce utilizatorul tastează.

\---

## 9\. Progres și validare formular

### `updateProgress()`

Recalculează progresul completării și actualizează UI.

```
Numărare:
1. DA/NU/N/A:
   totalDA = Object.keys(answers).length          // toate întrebările existente
   doneDA  = answers cu value !== null             // cele cu răspuns dat

2. Întrebări text (secțiunile 14 și 15):
   querySelectorAll('#section-14-text textarea, #section-15-text textarea')
   doneText = câte au .value.trim() !== ''

3. Câmpuri libere:
   #obs-text (16.1 Observații)     → +1 total, +1 done dacă completat
   #concluzii-text (17.1 Concluzii) → +1 total, +1 done dacă completat

Afișare:
  "X din Y întrebări completate" în .actions-left
  .progress-fill width = (done / total) × 100%
```

### `getIncompleteCount()`

```
Returnează: (totalDA - doneDA) + (totalText - doneText)
Folosit de: showIncompleteModal() și condițiile de bypass din downloadPDF/trimiteRaport
```

### `getTotalQuestions()`

```
Returnează totalul dinamic:
  = Object.keys(answers).length
  + querySelectorAll('#section-14-text textarea, #section-15-text textarea').length
  + 2  (câmpuri 16.1 și 17.1)

Avantaj: dacă se adaugă secțiuni noi cu întrebări DA/NU/N/A, totalul se
actualizează automat fără modificări hardcodate.
```

### `validate()`

```
Verifică câmpurile obligatorii înainte de submit:
1. Nume complet nu e gol → focus + showToast + return false
2. Email conține "@" → focus + showToast + return false

Notă: validarea domeniului se face exclusiv în GAS,
nu în frontend, pentru a nu expune domeniul acceptat în codul sursă.

Returnează: true dacă toate câmpurile sunt valide
```

\---

## 10\. Upload imagini secțiunea 16

Secțiunea 16.2 permite atașarea de imagini generale (nu per întrebare). Acestea apar la final în PDF, într-o secțiune dedicată "Imagini atașate (N)".

```
Suportă: click pe zona drop (input\[type=file]) și drag \& drop
Limite:  max 10 fișiere, max 5 MB per fișier
Note:    imaginile șterse devin null (index-ul se păstrează)
```

### `handleFiles(files)`

Validează fiecare fișier (dimensiune, număr maxim) și apelează `addThumb()`.

### `addThumb(file, idx)`

Generează thumbnail vizual. Pentru imagini (`image/\*`) afișează preview. Pentru alte tipuri afișează icon emoji (🎬 video, 📄 PDF, 📎 altele).

### `removeImage(idx)`

Marchează `images\[idx] = null` și îndepărtează thumbnail-ul din DOM.

\---

## 11\. Modal completare incompletă

Dacă se apasă "Descarcă PDF" sau "Salvează și Descarcă" cu întrebări fără răspuns, apare un modal de confirmare în loc să se proceseze direct.

### `showIncompleteModal(action)`

```
Parametri: action = 'download' | 'submit'

1. incomplete = getIncompleteCount()
2. total = getTotalQuestions()
3. \_pendingAction = action
4. Construiește mesaj: "Ai X întrebări fără răspuns din Y total."
   + descrierea acțiunii pendinte (descărcare / salvare și descărcare)
5. Afișează #incompleteModal
```

### `closeIncompleteModal()`

Ascunde modalul și resetează `\_pendingAction = null`.

### `proceedAnyway()`

```
Utilizatorul a ales să continue cu formular incomplet:
1. Închide modalul
2. Citește și resetează \_pendingAction
3. Apelează downloadPDF(true) sau trimiteRaport(true)
   — parametrul true = bypass (sare verificarea de incomplet)
```

### Logica bypass

```javascript
async function downloadPDF(bypass) {
  if (!bypass \&\& getIncompleteCount() > 0) {
    showIncompleteModal('download');
    return;  // oprire — utilizatorul trebuie să aleagă din modal
  }
  // ... continuă cu generarea
}
// Identic pentru trimiteRaport(bypass)
```

\---

## 12\. Generare PDF

### `downloadPDF(bypass)`

Generează un PDF vizual și îl deschide în fereastră nouă cu dialog de print.

```
1. Dacă !bypass și incomplete > 0 → showIncompleteModal('download')
2. Overlay loading
3. Citește imaginile secțiunii 16 ca base64 (fileToBase64)
4. Obține numărul raportului (din cachedNrRaport sau de la GAS)
5. buildPayload() → buildPDFHtml(payload, imgData)
6. Injectează titlul: NR\_Localitate\_Data
7. window.open('', '\_blank')
8. Scrie HTML în fereastra nouă
9. La evenimentul 'load': setTimeout(() => win.print(), 300)
```

### Procesul de generare PDF binar (în trimiteRaport)

```
1. Parsare HTML cu DOMParser
2. Extrage CSS din <style> tags
3. Creează container div ascuns:
   position: fixed; left: -9999px; width: 860px
   (lățimea standard A4, în afara viewport-ului)

4. Așteaptă 1200ms (randarea completă: fonturi, imagini, layout)

5. html2canvas(container, { scale: X, useCORS: true, ... })
   — scale configurat de utilizator (ex. 2.5 sau 3)
   — useCORS: true pentru imaginile externe
   — backgroundColor: #ffffff

6. Calculează pageHpx = usableH × canvas.width / usableW
   (înălțimea unei pagini PDF în pixeli canvas)

7. Detectează puncte de tăiere sigure (evitare tăiere în mijlocul unui rând):
   — Iterează elementele .row și .sec-header din container
   — Pentru sec-header: sare la .row următor (nu taie între header și primul rând)
   — safeBottoms\[] = array de poziții Y sigure pentru tăiere

8. Algoritmul de tăiere:
   breakPoints = \[0]
   while (pageEnd < canvas.height):
     găsește cel mai mare safeBottom ≤ pageEnd
     adaugă ca punct de tăiere
     pageEnd += pageHpx

9. Per pagină:
   — Creează slice canvas (drawImage cu offset Y)
   — pdf.addImage(slice.toDataURL('image/jpeg', quality), ...)
   — quality configurabilă de utilizator (ex. 0.92 sau 1.0)

10. pdf.output('datauristring') → extrage base64 → upload GAS
```

### `fileToBase64(file)`

```
Parametri: File object
Returnează: Promise<{ name: string, dataUrl: string }>
Citește fișierul cu FileReader și returnează obiect cu numele
și conținutul ca data URL base64 (pentru embedding în PDF).
```

\---

## 13\. Trimitere raport (trimiteRaport)

Funcția principală care orchestrează întregul flux de salvare.

```
PASUL 1 — Validare și verificare completitudine
  validate() → câmpuri obligatorii
  getIncompleteCount() → avertizare dacă !bypass

PASUL 2 — Citire imagini
  fileToBase64() pentru fiecare images\[i] non-null

PASUL 3 — Obținere număr raport
  Dacă cachedNrRaport \&\& !cachedNrUsed → refolosit (același nr pentru
  PDF local și GitHub, evitând un request suplimentar)
  Altfel: GET ?action=nextNr → GAS numără PDF-urile din /rapoarte/
  și returnează "R" + (count + 1)

PASUL 4 — Generare PDF binar
  buildPayload() → buildPDFHtml() → html2canvas → jsPDF
  Output: string base64 al PDF-ului binar

PASUL 4.5 — Construire rând CSV
  payload.csvRow = \[nrRaport, name, email, date, location, tipLucrare,
    ...per întrebare: \[răspuns, comentariu],  // secțiunile 1-13
    ...per întrebare text: \[răspuns],         // secțiunile 14-15
    observatii, concluzii,
    sigInspectorName, sigInspectorFunctie, sigInspectorFirma, sigInspectorData]

PASUL 5 — Upload PDF principal
  POST GAS\_PROXY\_URL {
    pdfHtmlBase64, fileExt: 'pdf', nrRaport,
    inspector, email, data, localitate, tipLucrare,
    observatii, concluzii, sections, csvRow
  }
  GAS: salvează PDF în /rapoarte/ + actualizează centralizator.csv

PASUL 6 — Upload PDF feedback (secțiunea 19)
  buildFeedbackPdfHtml() → html2canvas → jsPDF
  POST GAS\_PROXY\_URL { folder: 'feedback-formular', fileExt: 'pdf', ... }
  (fără email → sare validarea de domeniu din GAS)

PASUL 7 — Upload HTML raw
  buildPDFHtml(payload, imgData) → btoa(unescape(encodeURIComponent(html)))
  POST GAS\_PROXY\_URL { folder: 'rapoarte\_raw', fileExt: 'html', ... }

PASUL 8 — Descărcare locală
  Deschide window.open() cu HTML-ul PDF → print dialog (delay 300ms)

PASUL 9 — Toast confirmare
  "✔ R5 salvat în GitHub: R5\_Bucuresti\_2026-05-20.pdf"
```

\---

## 14\. Construire HTML pentru PDF (buildPDFHtml)

### `buildPDFHtml(payload, imgData)`

Generează un document HTML complet, optimizat pentru printare A4, folosit atât pentru descărcarea locală cât și pentru generarea PDF-ului binar și a HTML raw.

**Structura documentului generat:**

```html
<html>
  <head>
    <style>
      /\* Layout A4, tabel cu 3 coloane, badge-uri, imagini \*/
      @page { size: A4 portrait; margin: 1cm 1.5cm; }
      @media print { /\* forțare culori, text aliniat stânga \*/ }
    </style>
  </head>
  <body>
    <div class="meta">  <!-- 6 câmpuri: Nr, Nume, Email, Data, Localitate, Tip -->
    <div class="tbl">
      <div class="tbl-head">  <!-- Antet: Întrebare | Răspuns | Comentariu -->
      <!-- Rânduri per secțiune (sec-header + rows) -->
    </div>
    <div class="imgs">  <!-- Imagini secțiunea 16, grila 2 per rând -->
  </body>
</html>
```

**Lățimile coloanelor tabel:**

```
.col-q → flex: 0 0 40%   textul întrebării
.col-r → flex: 0 0 12%   badge răspuns (DA/NU/N/A)
.col-c → flex: 1          comentariu text + imagini per întrebare
```

**Optimizarea compactă — reducere pagini PDF:**

Per secțiune, întrebările sunt grupate în 3 categorii:

1. **Compact DA** — toate întrebările cu DA fără note/imagini → un singur rând verde: `"DA — întrebările: 1.1, 1.3, 1.7..."`
2. **Compact N/A** — similar, un singur rând gri: `"N/A — întrebările: 1.2, 1.5..."`
3. **Detaliate** — întrebările NU, sau DA/N/A cu note sau imagini atașate → afișate individual cu textul complet al întrebării

Efectul: un raport cu 80% răspunsuri DA simple ocupă 3-4 pagini în loc de 15+.

\---

## 15\. PDF Feedback formular

### `buildFeedbackPdfHtml(p)`

Generează HTML separat pentru PDF-ul secțiunii 19 (feedback formular).

**Parametri:**

```javascript
{
  nrRaport, name, date,
  fbProbleme,  // 19.1 — citit din DOM: document.getElementById('fb-probleme').value
  fbSugestii,  // 19.2 — citit din DOM: document.getElementById('fb-sugestii-input').value
  fbGeneral    // 19.3 — citit din DOM: document.getElementById('fb-general').value
}
```

**Structura:** document simplu cu header (Inspector / Data / Nr. Raport) + 3 întrebări cu răspunsurile lor. Dacă un câmp e gol, afișează "Fără răspuns" în italic.

### `answerDiv(val)` (funcție internă în buildFeedbackPdfHtml)

```
Returnează HTML pentru afișarea unui răspuns text.
Dacă val e gol/undefined → <div class="q-answer empty">Fără răspuns</div>
Altfel → <div class="q-answer">val</div> (cu newline → <br>)
```

\---

## 16\. Semnătură digitală

Secțiunea 18 folosește librăria **SignaturePad 5.1.3** pentru captarea semnăturii inspectorului pe canvas HTML.

### `initSignatures()`

```
Apelat la 500ms după initForm() — delay necesar pentru randarea completă a DOM.

1. Găsește #canvas-inspector
2. Creează instanță SignaturePad(canvas, { backgroundColor: 'rgb(255,255,255)', penColor: 'rgb(0,0,0)' })
3. Apelează resizeCanvas() pentru dimensionare corectă
4. Adaugă window.addEventListener('resize', resizeCanvas)
```

### `resizeCanvas()` (funcție internă)

```
Necesară pentru ecrane Retina (devicePixelRatio = 2):
fără resize, semnătura apare pixelată sau scrisul e descentrat.

1. Salvează datele semnăturii: sigData = pad.toData()
2. canvas.width  = offsetWidth  × devicePixelRatio
3. canvas.height = offsetHeight × devicePixelRatio
4. ctx.scale(devicePixelRatio, devicePixelRatio)
5. pad.clear()
6. Restaurează: pad.fromData(sigData) — semnătura rămâne după resize
```

### `clearSig(id)`

```
Parametri: id = 'inspector'
Apelat de butonul "✕ Șterge semnătura"
Apelează sigCanvases\['inspector'].clear()
```

### `getSigDataUrl(id)`

```
Parametri: id = 'inspector'
Returnează: string PNG dataURL sau null dacă canvas-ul e gol
Folosit în buildPayload() și buildPDFHtml() pentru embedding în PDF
```

\---

## 17\. Inițializare formular (initForm)

IIFE (Immediately Invoked Function Expression) — se execută automat la încărcarea paginii.

```
1. Inițializează answers{}:
   querySelectorAll('.opt-btn\[data-key]') → pentru fiecare cheie unică
   answers\[key] = { value: null, comment: '' }

2. Inițializează textAnswers{}:
   Iterează TEXT\_SECTIONS → textAnswers\[key] = ''

3. Setează data curentă (dd/mm/yyyy) în:
   — #meta-date (data inspecției)
   — #sig-inspector-data (data semnăturii)
   — sigInspectorData (variabila globală)

4. updateProgress() — starea inițială (0 din 101)

5. Event delegation pe .container (un singur listener pentru toate elementele):
   
   Click:
   — .opt-btn\[data-key] → selectAnswer(btn)
   
   Input:
   — textarea\[data-comment-key] → updateComment(key, value)
   — #obs-text-input → obsText = value

6. setTimeout(initSignatures, 500) — inițializare canvas semnătură

7. window.addEventListener('beforeunload', e => {
     e.preventDefault();
     e.returnValue = '';
   })
   — Afișează dialog de confirmare la refresh/închidere accidentală
```

### `fetchNrRaportOnLoad()` (IIFE async)

```
Se execută în background la deschiderea paginii:
GET GAS\_PROXY\_URL?action=nextNr

Scopul: preîncărcarea numărului de raport, astfel încât la primul click
pe "Salvează" să nu existe delay pentru obținerea numărului.

Salvează în cachedNrRaport (ex. "R5")
cachedNrUsed rămâne false până la primul upload reușit.

Logica de reutilizare:
- downloadPDF() poate folosi același cachedNrRaport (cachedNrUsed = false)
- trimiteRaport() refolosește numărul dacă !cachedNrUsed, astfel PDF-ul
  local și cel din GitHub au același număr de raport
```

\---

## 18\. Utilitare

### `buildPayload()`

```
Construiește obiectul complet cu toate datele formularului pentru trimitere:
{
  nrRaport,              // '' — completat ulterior
  date, name, email,     // din câmpurile meta
  location, tipLucrare,
  observatii,            // obsText
  numFisiere,            // images.filter(Boolean).length
  concluzii,             // din #concluzii-text
  sigInspectorName, sigInspectorFunctie, sigInspectorFirma, sigInspectorData,
  sigInspector,          // getSigDataUrl('inspector') — PNG base64 sau null
  sections: {
    1: \[ {question, answer, comment}, ... ],   // secțiunile 1-13
    ...
    14: \[ {question, answer}, ... ],           // secțiunile text (fără comment)
    15: \[ {question, answer}, ... ],
  }
}
```

### `dateForFilename(dateStr)`

```
Convertește data în format ISO pentru numele fișierelor GitHub:
"21/05/2026" → "2026-05-21"
"2026-05-21" → "2026-05-21" (deja ISO, returnează neschimbat)
```

### `formatDateInput(el)`

```
Formatează input dată în timp real (oninput):
Elimină non-numerice și inserează "/" automat:
"2105" → "21/05"
"210526" → "21/05/26"
"21052026" → "21/05/2026"
```

### `showToast(msg, type)`

```
Notificare temporară în colțul dreapta-jos, 4500ms.
type: '' (negru neutru) | 'error' (roșu) | 'success' (verde)
```

\---

## 19\. Referință completă funcții

|Funcție|Parametri|Return|Descriere|
|-|-|-|-|
|`clearQImagesForKey(key)`|`string`|`void`|Șterge imaginile și thumbnailurile pentru o întrebare|
|`handleQImg(key, files)`|`string, FileList`|`void`|Procesează imaginile uploadate per întrebare|
|`renameQImg(key, idx, val)`|`string, number, string`|`void`|Redenumește o imagine atașată|
|`removeQImg(key, idx)`|`string, number`|`void`|Șterge o imagine atașată|
|`selectAnswer(btn)`|`HTMLElement`|`void`|Procesează selectarea DA/NU/N/A cu toggle și zone note|
|`updateComment(key, val)`|`string, string`|`void`|Actualizează nota pentru o întrebare în answers{}|
|`updateProgress()`|—|`void`|Recalculează și afișează progresul completării|
|`getIncompleteCount()`|—|`number`|Numărul de întrebări fără răspuns|
|`getTotalQuestions()`|—|`number`|Totalul dinamic de întrebări (adaptat automat)|
|`showIncompleteModal(action)`|`'download'\|'submit'`|`void`|Afișează modalul de avertizare formular incomplet|
|`closeIncompleteModal()`|—|`void`|Închide modalul și resetează \_pendingAction|
|`proceedAnyway()`|—|`void`|Continuă cu acțiunea ignorând completitudinea|
|`handleFiles(files)`|`FileList`|`void`|Procesează imagini secțiunea 16|
|`addThumb(file, idx)`|`File, number`|`void`|Creează thumbnail vizual pentru secțiunea 16|
|`removeImage(idx)`|`number`|`void`|Șterge imagine din secțiunea 16|
|`validate()`|—|`boolean`|Validează câmpurile obligatorii (Nume, Email)|
|`buildPayload()`|—|`Object`|Construiește obiectul complet de date formular|
|`fileToBase64(file)`|`File`|`Promise<{name, dataUrl}>`|Convertește fișier în base64 dataURL|
|`downloadPDF(bypass?)`|`boolean?`|`Promise<void>`|Generează PDF și îl deschide local pentru print|
|`trimiteRaport(bypass?)`|`boolean?`|`Promise<void>`|Flux complet: PDF + feedback + HTML raw → GitHub|
|`buildPDFHtml(payload, imgData)`|`Object, Array`|`string`|Generează HTML complet optimizat pentru PDF|
|`buildFeedbackPdfHtml(p)`|`Object`|`string`|Generează HTML pentru PDF-ul de feedback (secț. 19)|
|`answerDiv(val)`|`string`|`string`|HTML pentru câmp răspuns în feedback PDF|
|`dateForFilename(dateStr)`|`string`|`string`|Convertește data în format ISO (yyyy-mm-dd)|
|`formatDateInput(el)`|`HTMLElement`|`void`|Formatare live input dată (dd/mm/yyyy)|
|`showToast(msg, type)`|`string, string`|`void`|Notificare temporară tip toast|
|`initSignatures()`|—|`void`|Inițializează instanța SignaturePad pe canvas|
|`resizeCanvas()`|—|`void`|Redimensionează canvas pentru Retina (funcție internă)|
|`clearSig(id)`|`string`|`void`|Șterge semnătura de pe canvas|
|`getSigDataUrl(id)`|`string`|`string\|null`|Returnează semnătura ca PNG base64 sau null|
|`initForm()`|—|`void`|Inițializare completă formular (IIFE)|
|`fetchNrRaportOnLoad()`|—|`Promise<void>`|Preîncarcă numărul raportului în background (IIFE async)|

\--
