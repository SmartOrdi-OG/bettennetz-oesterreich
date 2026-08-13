# TODO — BettenNetz Österreich

## Erledigt

### Grundgerüst & Deployment
- Erster Prototyp (Karte, Dashboard, Login-Mock) ins Repo übernommen
- GitHub Pages aktiviert, zusätzlich Deployment über Vercel (https://bettennetz-oesterreich.vercel.app/) — beide folgen automatisch jedem Push auf `main`
- Repo von `ahmadalikaram89-pixel` zu `SmartOrdi-OG` transferiert
- Karte + Dashboard responsive für Mobile (Sidebar stapelt sich, Detailpanel als Bottom Sheet, Tabellen scrollen horizontal statt das Layout zu sprengen)

### Branding & Design
- Echtes Logo eingebaut (`assets/logo.png`) — ersetzt das 🏥-Emoji in Login-Screen, Topbar und Favicon
- Alle Akzentfarben auf die Logo-Palette abgestimmt (Teal `--accent` statt Blau, eigenes `--info`-Blaugrau für GÖG-Elemente, passende Karten-/Login-Hintergründe)
- Hover-Tooltip auf Karten-Pins (voller Krankenhausname + Ort + Bundesland) — mehrere Häuser teilen sich denselben Anfangsbuchstaben, der bisher als einziges Label diente
- Fonts von Google Fonts auf lokal gehostet umgestellt (`assets/fonts/`, DM Sans/Inter/JetBrains Mono) — keine Verbindung mehr zu Google, vermeidet ein bekanntes DSGVO-Problem (IP-Übertragung bei jedem Seitenaufruf)

### Kritische Bugfixes (bei umfassendem Codecheck gefunden)
- Karten-Detailpanel war fälschlich in `.map-area` verschachtelt statt eigenständiges Flex-Element in `#view-map` zu sein — erschien dadurch als kleine, falsch positionierte Box statt als volle Seitenleiste
- Kartenbereich war oben "eingequetscht" mit großer Lücke darunter, weil ein `flex:1` auf einem Element ohne Flex-Parent wirkungslos war — jetzt vertikal zentriert
- "Speichern & veröffentlichen" schrieb nirgends in die Kartendaten zurück — Karte/Liste zeigten nie die eigenen Änderungen. Neue `publishToMap()`-Funktion verbindet Dashboard-Eingaben jetzt tatsächlich mit der öffentlichen Kartenansicht
- Annehmen/Ablehnen einer Anfrage tat im lokalen Fallback-Modus gar nichts (String/Number-Vergleich der IDs)
- Manuelle OP-Kapazität-Eingaben gingen beim Hinzufügen einer neuen Kategorie ("+ Kategorie hinzufügen") stillschweigend verloren, weil Änderungen nur im DOM, nicht im Datenmodell standen
- Die Kapazitätsbalken in der Krankenhaus-Liste zeigten feste Fake-Prozente (40/35/45 %) statt echter, aus den Zahlen berechneter Werte
- Alle Fixes mit Playwright end-to-end nachgetestet (nicht nur Code gelesen)

### Sicherheit
- XSS-Härtung: Anfragedaten (`from_hospital`, `fach`, `prio`, …) werden jetzt escaped bevor sie ins DOM geschrieben werden — relevant sobald Supabase mit offener anon-Insert-Policy verbunden ist
- `supabase/schema.sql`: CHECK-Constraints auf `fach`/`prio` (nur erlaubte Werte) und Längenlimits auf Namensfeldern als Defense-in-Depth
- Echtes Login/Auth vorbereitet: `index.html` nutzt Supabase Auth (`signInWithPassword`) sobald `SUPABASE_CONFIGURED` ist, mit `hospital_profiles`-Tabelle statt Klartext-Passwörtern im JS
- `supabase/schema.sql` hat verschärfte RLS-Policies: jedes Krankenhaus sieht/verwaltet nur eigene Anfragen, GÖG sieht alles, Status-Updates sind auf die annehmende Klinik + ausschließlich die Spalte `status` beschränkt (Column-Level-Grant)
- Der lokale Demo-Fallback (`ACCS`, Klartext-Passwörter) bleibt aktiv bis ein echtes Supabase-Projekt verbunden ist — siehe „Offen" unten

### Produkt-Feinschliff
- UX-Politur: Modal schließt per Klick auf den Hintergrund oder Escape, mobiles Bottom-Sheet hat abgedunkeltes Overlay, "Anfrage senden" zeigt Ladezustand, Enter im Benutzername-Feld loggt ein, Suchfeld hat ×-Button
- Anfragen sind an ein Ziel-Krankenhaus geroutet (`to_hospital`) statt globaler Posteingang; "Eingehend" zeigt nur an uns gerichtete Anfragen
- Bestätigte Reservierung: Annahme einer Anfrage reduziert sofort die passende OP-Kapazität des annehmenden Krankenhauses (Doppelbuchungsschutz), Anfrage-ID erscheint im "Gesendet"-Tab als Code fürs Patienten-Handover
- OP-Kapazität-Kategorien erweiterbar: Krankenhäuser können per "+ Kategorie hinzufügen" eigene Kategorien (z. B. Geburtshilfe / Entbindung) ergänzen

### Neue Seiten & Features (für professionellen Auftritt ggü. GÖG/WIGEV)
- **Landing Page** vor dem Login: Hero, Problem-Statement, drei Feature-Karten, CTA zur Anmeldung — bisher landete jeder Besucher direkt auf einem Login-Formular mit sichtbaren Demo-Passwörtern
- **Berichte & Trends**: neues Dashboard-Panel mit 14-Tage-Auslastungs-Liniendiagramm (eigenes SVG, keine Chart-Library), Hover-Crosshair + Tooltip, 85 %-Schwellenlinie, Tabellenansicht als Alternative. Aktuell Demo-Daten; `capacity_history`-Tabelle inkl. RLS in `supabase/schema.sql` vorbereitet für echte Verlaufsdaten sobald Supabase verbunden ist
- **CSV-/PDF-Export**: Export-Buttons auf Berichte, eingehenden/gesendeten OP-Anfragen und der GÖG-Gesamtübersicht. CSV mit Semikolon-Trennung + UTF-8-BOM (passend für Excel AT/DE), PDF über den nativen Druckdialog des Browsers mit eigenem Print-Stylesheet
- **Impressum & Datenschutzerklärung**: rechtlich erforderliche Seiten mit echten Angaben (SmartOrdi OG, Steingasse A6, 4020 Linz, team@smartordi.eu), verlinkt im Landing-Page-Footer. Datenschutzerklärung beschreibt die tatsächliche Datenverarbeitung (Zugangsdaten, betriebliche Daten, **keine Patientendaten**, eingesetzte Dienste)

### Weitere Bugfixes (zweiter Codecheck)
- OP-Kapazität-Kategorien: `reserveCapacity()` griff nur für 4 von 10 Fachbereichen — Annahme einer Anfrage für z. B. Gynäkologie/Onkologie markierte sie als "Angenommen" und vergab einen Handover-Code, reservierte aber nie echte Kapazität. `DEFAULT_OP_CATEGORIES`/`FACH_TO_OPS_KEY` decken jetzt alle 10 Fachbereiche ab; Karte/Liste/Detailpanel lesen die Kategorien jetzt dynamisch statt vier Felder hart zu kodieren
- Annahme einer Anfrage veröffentlichte fälschlich *alle* OP-Kategorien (inkl. anderswo gerade unfertig editierter Werte) statt nur der reservierten — `publishToMap()` nimmt jetzt einen optionalen Fachbereich-Filter
- `P(0,0)` zeigte eine frisch hinzugefügte Kategorie sofort als 100 % "Ausgelastet" statt 0 %
- **Supabase-Sync für OP-Kapazität**: neue `hospital_capacity`-Tabelle (inkl. RLS + Realtime) in `supabase/schema.sql`, mit Seed-Daten für alle 21 Demo-Krankenhäuser. `index.html` lädt/abonniert/schreibt sie jetzt (`loadHospitalCapacity`, `subscribeHospitalCapacity`, `persistCapacity`) — vorher schrieb `publishToMap()`/`save()` nur ins lokale `HOSPITALS`-Array, die Kartendaten synchronisierten sich also nie zwischen Sessions, selbst mit verbundenem Supabase
- `transfers.fach`-CHECK-Constraint in `schema.sql` erlaubte nur 8 von 10 Fachbereichen — Anfragen für Pädiatrische Chirurgie/Gefäßchirurgie wären mit verbundenem Supabase am Insert gescheitert

## Offen
- [ ] **Impressum ergänzen**: Firmenbuchnummer und UID-Nummer (falls vorhanden) fehlen noch — rechtlich für eine OG relevant, aktuell bewusst weggelassen statt geraten
- [ ] Supabase-Projekt anlegen, `supabase/schema.sql` ausführen (Reihenfolge steht im Dateikopf)
- [ ] Pro Krankenhaus einen echten Supabase-Auth-User anlegen (`<username>@bettennetz.local` + echtes Passwort) und per `hospital_profiles`-Insert verknüpfen — danach die alten Demo-Passwörter aus `ACCS` in `index.html` entfernen
- [ ] Projekt-URL + anon key in `index.html` eintragen (`SUPABASE_URL`/`SUPABASE_ANON_KEY`)
- [ ] Transfer-Sync (inkl. Reservierungs-Sperre) zwischen mehreren Krankenhäusern/Sessions live testen, sobald Supabase verbunden ist
- [ ] Vercel-Git-Integration nach dem Repo-Umzug zu `SmartOrdi-OG` prüfen/neu verbinden (Settings → Git im Vercel-Dashboard) — ein Deployment blieb zwischenzeitlich hängen
- [ ] `save()` später an `capacity_history` anbinden, damit "Berichte" echte statt Demo-Verlaufsdaten zeigt
- [ ] Weitere UX-Politur nach Bedarf (Login-Screen auf sehr schmalen Displays, Fehlermeldungen in der UI)
