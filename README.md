Documentație tehnică — Formular SSM
Versiune curentă a formularului: v38
Tip fișier: HTML monolitic (HTML + CSS + JavaScript într-un singur fișier)
Dependențe externe (CDN): jsPDF 2.5.1, html2canvas 1.4.1, SignaturePad 5.1.3
---
1. Arhitectura generală
Formularul este o singură pagină HTML fără build step, fără framework, fără server propriu de backend (folosește un Google Apps Script — GAS ca proxy către GitHub).
```
Utilizator completează formular în browser
        │
        ▼
JavaScript generează un payload (obiect JS cu toate răspunsurile)
        │
        ▼
Payload-ul e transformat în HTML (pentru raport) → PDF (canvas → imagine → PDF)
        │
        ▼
PDF-ul (ca base64) + un rând CSV sunt trimise prin fetch() la GAS (POST)
        │
        ▼
GAS scrie fișierul în GitHub (folder rapoarte_raw) și actualizează un CSV centralizator
```
Toate întrebările, secțiunile, stilurile și logica de generare PDF trăiesc în acest singur fișier `.html`. Nu există pași de compilare — fișierul e editat direct și încărcat ca atare.
---
2. Structura HTML
2.1 Zone fixe (scrise direct în HTML)
Header pagină (`.page-header`) — titlu + etichetă versiune (`#formVersionLabel`, populată din `FORM_VERSION`)
Bară zoom text (`#zoomBar`) — butoane +/−/reset pentru mărirea textului din formular (vezi secțiunea 7)
Drawer navigare secțiuni (`#navDrawer`) — panou lateral cu lista tuturor secțiunilor și progresul fiecăreia
Secțiunile 14–19 — scrise static în HTML (feedback inspector, feedback șef echipă, observații, concluzii, semnătură, feedback formular)
Modal „Formular incomplet" (`#incompleteModal`) — confirmare înainte de descărcare/trimitere cu răspunsuri lipsă
Overlay de loading (`#overlay`) — afișat în timpul generării PDF-ului
Toast (`#toast`) — notificări scurte de succes/eroare
2.2 Zone generate dinamic din JavaScript
Secțiunile 1–13 (`#questions-container`) — construite la runtime din obiectul `QUESTIONS`, nu există în HTML-ul static
---
3. Sursa întrebărilor — `QUESTIONS` și `SECTION_META`
```javascript
const QUESTIONS = {
  1: [ "text întrebare 1.1", "text întrebare 1.2", ... ],
  2: [ ... ],
  ...
  13: [ ... ]
};
```
Cheile sunt numerele secțiunilor (1–13).
Valorile sunt array-uri de string-uri — fiecare string e o întrebare cu răspuns DA/NU/N/A.
`SECTION_META[sec].title` ține titlul afișat pentru fiecare secțiune.
Pentru a adăuga/edita o întrebare: se editează direct array-ul corespunzător din `QUESTIONS`. Nu e nevoie de nicio altă modificare — `buildFormHTML()` regenerează automat HTML-ul la încărcarea paginii, iar `buildPDFHtml()` citește din același obiect la generarea raportului.
Secțiunile 14 și 15 (întrebări cu răspuns liber, nu DA/NU/N/A) sunt în obiectul separat `TEXT_SECTIONS`, dar HTML-ul lor e scris static (nu generat dinamic ca 1–13).
---
4. Construcția dinamică a formularului — `buildFormHTML()` / `buildQuestionRow()`
La încărcarea paginii, `initForm()` apelează `buildFormHTML()`:
Iterează prin toate secțiunile din `QUESTIONS` (1 până la 13).
Pentru fiecare secțiune, creează un `<div class="section">` cu header și o listă de întrebări.
Pentru fiecare întrebare apelează `buildQuestionRow(sec, idx, questionText)`, care construiește:
3 butoane: DA / NU / N/A (`data-key` = `"{sectiune}-{index}"`, `data-val` = valoarea)
o zonă de comentariu ascunsă (`.comment-section`), afișată doar dacă răspunsul e DA sau NU
un buton de upload imagini specific întrebării respective (`#qimg-{key}`)
Cheia unei întrebări e mereu de forma `"1-0"`, `"1-1"`, `"2-0"` etc. — secțiune și index, separate prin `-`. Această cheie e folosită consecvent în `answers{}`, `qImages{}`, și în atributele `data-key`/`data-comment-key` din DOM.
---
5. Starea aplicației (variabile globale)
Variabilă	Tip	Scop
`answers`	`{ "1-0": {value, comment}, ... }`	răspunsurile DA/NU/N/A + comentariile pentru secțiunile 1–13
`qImages`	`{ "1-0": [{dataUrl, name}, ...], ... }`	imaginile atașate per întrebare
`textAnswers`	`{ "14-0": "text", ... }`	răspunsurile libere pentru secțiunile 14 și 15
`obsText`	string	textul din secțiunea 16 (Observații)
`concluziiText`	string	textul din secțiunea 17 (Concluzii finale)
`sigInspectorName/Functie/Firma/Data`	string	câmpurile text din secțiunea 18
`sigCanvases`	`{ inspector: SignaturePad }`	instanța SignaturePad pentru canvas-ul de semnătură
`cachedNrRaport` / `cachedNrUsed`	string / bool	numărul de raport preluat de la GAS și dacă a fost deja folosit la o salvare confirmată
Important: `answers` și `qImages` sunt populate prin manipulare directă a DOM-ului (`selectAnswer`, `handleQImg`), nu prin binding reactiv — orice citire a stării trebuie să se facă din aceste obiecte globale, nu din DOM direct (cu excepția câmpurilor meta, citite direct din `<input>`/`<textarea>` în `buildPayload()`).
---
6. Fluxul de selectare a unui răspuns — `selectAnswer()`
Apelat prin event delegation (`container.addEventListener('click', ...)` în `initForm`), nu prin `onclick` individual pe fiecare buton.
Logica:
Dacă butonul apăsat e deja selectat → deselectează (resetează valoarea la `null`, șterge comentariul și imaginile asociate).
Altfel → marchează butonul ca selectat (clasa `selected-yes`/`selected-no`/`selected-na`), salvează valoarea în `answers[key].value`.
Dacă noul răspuns e N/A → comentariul și imaginile existente se șterg automat și secțiunea de note se ascunde.
Dacă noul răspuns e DA/NU → secțiunea de note se deschide și primește focus.
La final apelează `updateProgress()` pentru a recalcula bara de progres și contorul din drawer-ul de navigare.
---
7. Sistemul de zoom text — `adjustZoom()`
Independent de zoom-ul browserului. Folosește `container.style.zoom` (proprietate CSS non-standard dar suportată de browserele Chromium) pentru a scala vizual tot conținutul `.container` între 70% și 160%, în pași de 10%.
Canvas-ul de semnătură (`#canvas-inspector`) e tratat separat — `initSignatures()` recalculează dimensiunile interne ale canvas-ului ținând cont de nivelul de zoom curent (`_zoomLevel`), pentru ca desenul semnăturii să rămână corect aliniat la `devicePixelRatio` indiferent de nivelul de zoom.
---
8. Upload imagini per întrebare — `handleQImg()` / `compressImage()`
Fiecare întrebare din secțiunile 1–13 are propriul buton de upload imagine (`accept="image/*" multiple`).
Flux:
`handleQImg(key, files)` validează fiecare fișier (tip imagine, dimensiune ≤ 10 MB).
Citește fișierul ca `dataUrl` prin `FileReader`.
Apelează `compressImage(dataUrl, 1000, 0.5)` — redimensionează la maxim 1000px lățime și recomprimă JPEG la calitate 0.5, folosind un `<canvas>` offscreen.
Imaginea comprimată înlocuiește originalul în `qImages[key]` și se afișează un thumbnail (`.q-thumb`) cu opțiune de redenumire și ștergere.
Imaginile sunt stocate doar în memorie (în `qImages`), nu sunt încărcate separat — sunt incluse direct ca `<img src="data:...">` în HTML-ul raportului generat de `buildPDFHtml()`.
---
9. Validare — `validate()`, modal incomplet
9.1 Validare obligatorie (`validate()`)
Verifică doar:
`meta-name` nevid
`meta-email` nevid și conține `@`
Toate celelalte câmpuri (inclusiv toate întrebările DA/NU/N/A) sunt opționale — un răspuns lipsă e salvat ca `null`/string vid, nu blochează trimiterea.
9.2 Modal „Formular incomplet"
Dacă există întrebări fără răspuns la apăsarea „Descarcă PDF" sau „Salvează și Descarcă", se afișează `#incompleteModal` cu numărul de întrebări lipsă (`getIncompleteCount()` / `getTotalQuestions()`), oferind opțiunea de a reveni sau de a continua oricum (`proceedAnyway()` → reapelează acțiunea cu `bypass=true`).
---
10. Generarea raportului — două metode diferite de PDF
Acesta e cel mai important lucru de înțeles din tot codul: există două căi complet separate de a produce un PDF, folosite în două contexte diferite.
10.1 `downloadPDF()` — PDF prin print nativ al browserului
Folosit la apăsarea butonului „Descarcă PDF".
Construiește `payload` din `buildPayload()`.
Generează HTML-ul raportului cu `buildPDFHtml(payload, imgData)`.
Deschide un tab nou (`window.open('', '_blank')`), scrie HTML-ul cu `document.write()`, injectând titlul corect în `<title>`.
Apelează `win.print()` — utilizatorul salvează manual ca PDF prin dialogul nativ de print al browserului.
Nu generează un fișier PDF binar — se bazează pe „Print to PDF" din browser. Nu trimite nimic la server.
10.2 `trimiteRaport()` — PDF binar real, generat client-side
Folosit la apăsarea butonului „Salvează și Descarcă".
Construiește `payload`, obține `nrRaport` (de la GAS sau din cache).
Generează `pdfHtml` cu `buildPDFHtml()`.
Randare HTML → Canvas → PDF binar, fără server de rendering:
parsează HTML-ul cu `DOMParser`
îl pune într-un `<div>` offscreen (`position:fixed;left:-9999px`) cu lățime fixă 860px
`html2canvas()` randează acel div într-un `<canvas>` la `scale:2`
calculează puncte de tăiere „sigure" între întrebări (`safeBottoms`) ca paginile PDF să nu taie un rând la mijloc
`jsPDF` adaugă fiecare bucată de canvas ca imagine JPEG pe câte o pagină A4
rezultatul e exportat ca `base64` (`pdf.output('datauristring').split(',')[1]`)
Trimite PDF-ul base64 + datele structurate către GAS prin `fetch(GAS_PROXY_URL, {method:'POST', ...})`.
La succes, trimite separat și:
feedback-ul din secțiunea 19, ca PDF propriu, în folderul `feedback-formular`
HTML-ul brut al raportului (necomprimat în PDF), în folderul `rapoarte_raw` — acesta e fișierul citit mai târziu de platforma de vizualizare a rapoartelor
Deschide local încă un tab cu raportul și declanșează `print()` pentru ca utilizatorul să aibă și o copie locală imediată.
De reținut pentru mentenanță: logica de tăiere pe pagini (`safeBottoms`, `breakPoints`) e duplicată identic în `trimiteRaport()` (pentru raportul principal) și încă o dată pentru PDF-ul de feedback. Dacă se modifică algoritmul de paginare, trebuie aplicat în ambele locuri.
---
11. `buildPDFHtml()` — generarea conținutului raportului
Funcție pură (nu modifică DOM-ul global), primește `payload` și `imgData`, returnează un string HTML complet, independent (cu `<style>` propriu, fără dependență de CSS-ul paginii principale).
Structura raportului generat:
Header cu titlul „FIȘA DE SUPRAVEGHERE ÎN ȘANTIERE"
Bloc `.meta` cu nr. raport, nume, email, dată, localitate, locație, firmă, tip lucrare
Pentru fiecare secțiune 1–13:
rând compact cu toate întrebările N/A fără note/imagini, grupate într-un singur rând (`compactNA`) pentru a economisi spațiu
câte un rând detaliat pentru fiecare întrebare DA/NU sau N/A cu note/imagini, cu badge colorat (verde=DA, roșu=NU, gri=N/A)
Secțiunile 14–15 (text liber)
Secțiunea 17 (concluzii), secțiunea 16 (observații) — atenție: ordinea în care apar în PDF e inversată față de numerotare (17 înainte de 16) — e o particularitate intenționată a codului actual, nu o eroare de citit greșit
Secțiunea 18 (tabel cu nume/funcție/firmă/dată + imaginea semnăturii, sau o linie goală dacă nu există semnătură)
CSS-ul intern include reguli `page-break-inside: avoid` pe rânduri și pe headerele de secțiune, pentru ca paginarea (atât la print browser, cât și la `html2canvas`) să nu rupă o întrebare la mijloc.
---
12. Trimiterea către GitHub prin GAS
Toate request-urile POST către `GAS_PROXY_URL` includ:
```javascript
{
  token: sessionStorage.getItem('form_token') || '',
  pdfHtmlBase64: ...,      // conținutul fișierului, encodat base64
  fileExt: 'pdf' | 'html',
  nrRaport: ...,
  folder: 'rapoarte_raw' | 'feedback-formular' (implicit GITHUB_FOLDER pentru raportul principal),
  inspector, data, localitate, tipLucrare
}
```
Tokenul (`form_token`) e preluat o singură dată la încărcarea paginii (`fetchNrRaportOnLoad()`) și e legat de data curentă — gestionat integral de partea de GAS, formularul doar îl stochează în `sessionStorage` și îl retrimite.
Numărul de raport (`nrRaport`) e obținut de la GAS prin `action=nextNr` și e cache-uit local (`cachedNrRaport`) pentru a putea fi refolosit dacă utilizatorul apasă mai întâi „Descarcă PDF" (fără upload) și apoi „Salvează și Descarcă" — astfel nu se „pierde" un număr de raport neutilizat.
---
13. Rândul CSV (`payload.csvRow`)
Construit manual înainte de trimitere, ca un array simplu de valori, în ordine fixă:
```
[nrRaport, nume, email, data, localitate, locatie, firma, tipLucrare,
 (răspuns, comentariu) × toate întrebările 1.1–13.5,
 răspuns × toate întrebările 14.1–15.1,
 observatii, concluzii,
 sigInspectorName, sigInspectorFunctie, sigInspectorFirma, sigInspectorData]
```
Acest array trebuie să corespundă exact ca ordine și numărare de coloane cu antetul CSV definit pe partea de GAS (`headerRow` în `updateCsv()`). Orice modificare a numărului de întrebări dintr-o secțiune (1–13) modifică numărul total de coloane și necesită actualizarea corespunzătoare a antetului CSV din GAS.
---
14. Navigarea pe secțiuni — `NAV_SECTIONS`, `buildNavItems()`
`NAV_SECTIONS` e o listă separată de `QUESTIONS`/`SECTION_META`, cu informația necesară pentru drawer-ul lateral de navigare: id-ul elementului DOM țintă, eticheta de afișat, și totalul de întrebări din secțiune (folosit pentru calculul progresului `done/total`).
`getSectionDone(sec)` calculează câte întrebări au răspuns, diferențiat pe tip de secțiune:
secțiuni DA/NU/N/A (`section-1`...`section-13`) → numără din `answers{}`
secțiuni text (`section-14-text`, `section-15-text`) → numără textarea-uri nevide
secțiuni cu un singur câmp (`obs-text`, `concluzii-text`) → 0 sau 1
La click pe un item din navigare, pagina face scroll smooth la secțiunea respectivă și evidențiază temporar marginea cu un outline.
---
15. Formatarea automată a datei — `formatDateInput()`
Aplicată pe `#meta-date` și `#sig-inspector-data`. Transformă input-ul brut în format `zz.ll.aaaa` pe măsură ce utilizatorul tastează, păstrând poziția cursorului corectă (calculează poziția pe baza numărului de cifre introduse înainte de cursor, nu pe baza poziției brute în string, pentru a evita „saltul" cursorului când se inserează automat punctele).
---
16. Avertisment Firefox
La încărcarea paginii, dacă `navigator.userAgent` conține `firefox`, se afișează un banner fix în partea de sus (`#firefox-warning`) care recomandă un browser bazat pe Chromium. Motivul: `html2canvas` și comportamentul de print pot avea inconsistențe pe Firefox care afectează calitatea PDF-ului generat.
---
17. Note pentru mentenanță viitoare
Adăugarea unei întrebări noi într-o secțiune 1–13: se adaugă string-ul în array-ul corespunzător din `QUESTIONS`. Trebuie actualizat și `total` din `NAV_SECTIONS` pentru acea secțiune, altfel bara de progres din navigare va arăta un procent greșit. Trebuie actualizat și antetul CSV din GAS (coloane noi de Răspuns/Comentariu).
Adăugarea unei secțiuni noi (14+): necesită HTML static nou (secțiunile 14+ nu sunt generate dinamic), plus integrare manuală în `buildPDFHtml()`, `buildPayload()`, `getIncompleteCount()`/`getTotalQuestions()`, și `NAV_SECTIONS`.
Limita de imagine (10 MB per fișier) e definită direct în `handleQImg()`; compresia (1000px, calitate 0.5) e fixă în apelul către `compressImage()`.
Două locuri generează PDF binar cu logică de paginare identică (raport principal + feedback) — orice fix la algoritmul de tăiere pe pagini trebuie aplicat simetric.
