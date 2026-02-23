# Write-up: Herleitung der NBL-2 Längenprüfung (18 Byte) vs. 0xFF-Dokuangaben (21 Byte)

**Datum:** 2026-02-23  
**Ziel:** Nachvollziehbar dokumentieren, wie die reale Längenprüfung im Teltonika/Netronix-NBLX-Decoder gefunden wurde und warum Doku-/Tool-Angaben ("0xFF mit 21 Byte") im Feld leicht zu Verwirrung führen können.

---

## Kurzfazit

- Die NBL-2-Decoderfunktion prüft **hart auf `len == 18`** (in alter **und** neuer Firmware).
- Diese `len` bezieht sich auf den **an die NBL-2-Funktion übergebenen Sub-Block** (Payload-Slice), **nicht** direkt auf die komplette BLE-AD-Struktur "on-air".
- Deshalb können Dokumente/Tools mit "0xFF = 21 Byte" trotzdem existieren, obwohl der funktionale Decoder nur mit einem 18-Byte-Subblock arbeitet.
- Der beobachtete Effekt "20 Byte funktioniert, 21 Byte nicht" passt zu einem **Off-by-one auf Wrapper-Ebene** (z. B. AD Len/Type/Company-ID mitgezählt oder nicht).

---

## 1) Methodik: Wie der Fund zustande kam

### Schritt A – Entropieblock in dekodierbaren Code überführen

1. VIVA/ALICE-Region extrahiert (old/new).
2. `mtk_fw_tools/unalice.py` in venv ausgeführt.
3. Ergebnis: disassemblierbare `ALICE_old.unalice.bin` und `ALICE_new.unalice.bin`.

Ohne `unalice` war die Region weitgehend high-entropy; mit `unalice` wurden String-/Codeanker sichtbar.

### Schritt B – String-Anker gesetzt (source-file + Fehlermeldungen)

In `ALICE_new.unalice.bin` gefunden:

- `teltonika_app\src\bluetooth\m2m_ble_netronix_nblx.c`
- `Unknown NBL-1 UUID 0x%02X%02X`
- `Unknown NBL-2 UUID 0x%02X%02X`
- `Unknown NBL-T UUID 0x%02X%02X`

Das verankert den relevanten Decoderbereich zuverlässig im NBLX-Modul.

### Schritt C – Rückwärts in die umgebenden Funktionen disassembliert

Vom Stringcluster aus den Funktionsprolog/Control-Flow gelesen.

Für NBL-2 (neue FW):
- Bereich um `0x212eda...`
- **`cmp r6, #18`** bei `0x212ef2`
- bei Ungleichheit Abbruchpfad
- bei Header-Fehler Logger mit `Unknown NBL-2 UUID...`

Für NBL-2 (alte FW):
- Bereich um `0x2038fa...`
- **`cmp r6, #18`** bei `0x20390e`
- identisches Muster

=> Reproduzierbar in beiden Generationen.

---

## 2) Harte Belege (Adressen)

## Neue Firmware (`ALICE_new.unalice.bin`)

- NBL-2 Funktionsbereich: `0x212eda..0x212f8e`
- Längenprüfung: `cmp r6, #18` @ `0x212ef2`
- Prefix/UUID-Check-Call: `blx 0x13207c` @ `0x212efe`
- Fehlerpfad bei unbekannter UUID:
  - Logger: `bl 0x0e0bfc` @ `0x212fa2`
  - Message: `Unknown NBL-2 UUID 0x%02X%02X` @ `0x212fac`
  - Source-Datei: `m2m_ble_netronix_nblx.c` @ `0x212fcc`

## Alte Firmware (`ALICE_old.unalice.bin`)

- NBL-2 Funktionsbereich: `0x2038fa..0x2039aa`
- Längenprüfung: `cmp r6, #18` @ `0x20390e`
- Prefix/UUID-Check-Call: `blx 0x1281e8` @ `0x20391a`
- Fehlerpfad bei unbekannter UUID:
  - Logger: `bl 0x0dfe00` @ `0x2039be`
  - Message: `Unknown NBL-2 UUID 0x%02X%02X` @ `0x2039c8`
  - Source-Datei: `m2m_ble_netronix_nblx.c` @ `0x2039e8`

---

## 3) Warum die 21-Byte-Doku trotzdem "stimmen kann"

Typischer Stolperstein bei BLE Manufacturer Data (`AD type 0xFF`):

- Manche Beschreibungen zählen:
  - AD-Len-Feld,
  - AD-Type,
  - Company-ID,
  - Payload
  **alles zusammen**.
- Andere meinen nur den "nutzbaren" Payload-Teil nach internem Demux.

Der NBL-2-Decoder bekommt offenbar bereits einen **normalisierten Ausschnitt** (Sub-Block), für den `len == 18` gilt.

Daher sind Aussagen wie
- "0xFF hat 21 Byte" (Dokuebene / On-Air-Rahmen)
und
- "Parser akzeptiert 18" (interne Funktionsschnittstelle)
kein Widerspruch, sondern unterschiedliche Zählebenen.

---

## 4) Einordnung der Feldbeobachtung (20 funktioniert, 21 nicht)

Die beobachtete Praxis (20 statt 21) passt sehr gut zu:

- einem zusätzlichen Byte auf äußerer AD-Ebene (z. B. mitgezähltes Wrapper-Feld),
- während der NBL-2-Decoder intern weiterhin seinen 18-Byte-Slice erwartet.

Wenn ein zusätzliches Byte in den falschen Abschnitt rutscht, kann die interne Aufteilung kippen (falscher Startoffset, Headervergleich scheitert, Klassifikation fällt auf Default/Fallback).

---

## 5) "NBLTools zeigt dann NBL-4" – plausible Erklärung

Wenn NBL-2-Gates nicht sauber matchen (Länge/Prefix/UUID), greifen Tools oft auf heuristische Klassifikation/Fallback zurück.  
Das kann ein "NBL-4"-Label erzeugen, obwohl semantisch eher ein NBL-2-ähnliches Paket vorliegt.

**Wichtig:** Für diese konkrete Label-Logik in *NBLTools* liegt hier noch kein disassemblierter Tool-Codebeweis vor; die Aussage ist aktuell als **plausible Heuristik-Folge** zu behandeln.

---

## 6) Praktische Debug-Regel

Bei Testvektoren immer getrennt prüfen:

1. **On-Air-AD-Layout** (`len|type|company|payload`)  
2. **Interner Decoder-Input** (welcher Slice landet in `r0/r1`)

Erst wenn (2) exakt reproduziert ist, sind Längenurteile belastbar.

---

## 7) Reproduktions-Notiz (Kurz)

- Disassembly (new): Bereich `0x212eda..0x212f8e` prüfen (`cmp r6,#18`).
- Disassembly (old): Bereich `0x2038fa..0x2039aa` prüfen (`cmp r6,#18`).
- Stringanker im gleichen Cluster verifizieren (`Unknown NBL-2 UUID...`, `m2m_ble_netronix_nblx.c`).

Damit ist der Fund robust und versionsübergreifend bestätigt.
