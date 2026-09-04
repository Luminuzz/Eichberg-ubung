# Cheat Sheet — Passwortwiederherstellung (24SEA)

> John the Ripper **und** Hashcat nebeneinander. In der VM ist meist **John (CPU)** der Arbeiter,
> weil kein OpenCL/GPU-Backend da ist. Hashcat-Zeilen stehen als Referenz + für die mündliche Prüfung.
> **Bewertet wird der Weg, nicht nur das Ergebnis.** Ein gut begründeter, aber erfolgloser Versuch zählt.

---

## 0. Universelle Methodik (immer diese Reihenfolge)

1. **Dateityp identifizieren** → `file`, `binwalk`, `xxd`
2. **Hash extrahieren** → passendes `*2john`-Tool
3. **Hash aufbereiten** → für Hashcat Präfix/Suffix abschneiden; für John meist roh lassen
4. **Angriffsstrategie wählen** → Wörterbuch / Regeln / Maske / Hybrid / Kombinator
5. **Angriff ausführen** → John oder Hashcat
6. **Alles dokumentieren** → siehe Checkliste unten

### Doku-Checkliste (genau das will die Prüfung sehen)

| # | Punkt |
|---|-------|
| 01 | Schritte zur **Extraktion** des Hashes |
| 02 | Schritte zur **Aufbereitung** des Hashes |
| 03 | Verwendete **Quellen** für (externe) Wörterbücher |
| 04 | Schritte zur **Erstellung** des Angriffs-Wörterbuchs |
| 05 | Schritte zur **Bereitstellung** des Toolings (z. B. Kompilieren, pip install) |
| 06 | Der/die **Angriffsschritte** auf den/die Hashes |
| 07 | Das/die **wiederhergestellten Passwörter** |
| 08 | **Dauer** der Suche (grob in Minuten reicht) |

**Merke fürs Mündliche:** *warum* dieses Tool, *warum* diese Strategie, *warum* dieser Modus.

---

## 1. Syntax-Grundlagen: John ↔ Hashcat

Der Denkunterschied in einem Satz:
**Hashcat musst du sagen *was* (`-m`) und *wie* (`-a`); John rät das Format selbst und wählt den Modus über benannte Flags.**

### Hashcat — Grammatik
```
hashcat -m <modus> -a <angriff> [optionen] <hashdatei> <angriffs-argumente>
```
- `-m` = Hash-Typ als **Nummer** (0=MD5, 100=SHA1, 1400=SHA256, 1700=SHA512, 1800=sha512crypt, 11600=7z, 13400=KeePass, 18400=ODF)
- `-a` = Angriffsmodus: **0**=Wörterbuch · **1**=Kombinator · **3**=Maske/Brute · **6**=Hybrid Wort+Maske · **7**=Maske+Wort
- danach die Hashdatei, dann je nach `-a`: Wörterbuch (`-r rules`), Maske, oder zwei Wörterbücher
- oft nützlich: `-1 '$!.'` eigener Zeichensatz · `--show` · `-o treffer.txt` · `--stdout` (nur Kandidaten ausgeben, nicht angreifen)

### John — Grammatik
```
john [--format=NAME] <modus-flag> <hashdatei>
```
- Format wird **auto-erkannt**; nur bei Bedarf `--format=` erzwingen
- Modus = **Flag**, keine Nummer: `--wordlist=FILE` (+ `--rules=Jumbo`) · `--mask='?d?d?d?d'` · `--incremental` · `--single`
- Ergebnis/Verwaltung: `--show` · Pot-Datei `~/.john/john.pot` · `--session=name` · `--restore=name` · `--fork=4`

### Dasselbe Ziel, zwei Schreibweisen (SHA256 + rockyou + Regeln)
```bash
john    --wordlist=rockyou.txt --rules=Jumbo hash.txt
hashcat -m 1400 hash.txt -a 0 rockyou.txt -r /usr/share/hashcat/rules/best64.rule
```

---

## 2. Dateityp identifizieren

```bash
file verdaechtig.datei          # Typ raten (Magic Bytes)
xxd verdaechtig.datei | head    # ersten Bytes ansehen
binwalk verdaechtig.datei       # bekannte Header/Signaturen finden
binwalk -E verdaechtig.datei    # Entropie -> verschlüsselt? (hohe Entropie)
```

**⚠ AppleDouble-Falle (macOS):** Dateien die mit `._` anfangen sind oft **Köder** (163 Bytes,
Magic `00 05 16 07`) — kein echtes Archiv! Immer `file` + Größe prüfen, bevor man extrahiert.

---

## 3. Hash extrahieren — Tool je Dateityp

| Dateityp | Extraktionstool | Cracker | Hashcat-Modus* |
|----------|-----------------|---------|----------------|
| `.7z` | `7z2john <f>` (Perl; ggf. `libcompress-raw-lzma-perl`) | John | 11600 |
| `.pdf` | `pdf2john <f>` | John | 10500 (v1.4–1.6) |
| `.odt/.ods` (LibreOffice/ODF) | **`libreoffice2john.py`** (NICHT `odf2john`) | John | 18400 |
| `.kdbx` (KeePass, KDBX3) | `keepass2john <f>` | John | 13400 |
| `.kdbx` (KeePass, **KDBX4/Argon2**) | ⚠ `keepass2john` scheitert → **pykeepass** (siehe §8) | Python | — |
| `.numbers/.pages/.key` (iWork) | `iwork2john <f>` | John | (kein GPU-Modus → John) |
| `.dmg` (Mac Disk Image) | `dmg2john <f>` | John | — (nur John) |
| APFS-Volume | `apfs2hashcat` (kompilieren) | Hashcat | 18300 |
| Linux-Login | aus `/etc/shadow` kopieren | beide | 1800 ($6$) / 3200 ($2y$) |
| OpenSSL `enc` (`.aes256cbc_*`) | **kein Hash!** → Entschlüsselungs-Schleife (§8) | openssl | — |
| Java `hashCode()` | **kein Standard-Hash** → eigenes Python (§8) | Python | — |

\* Hashcat-Modi immer gegenprüfen: `hashcat --example-hashes | less` oder Hashcat-Wiki „Example Hashes".
John erkennt das Format meist automatisch — im Zweifel `--format=` explizit setzen.

**Extraktion ausführen (Beispiel 7z):**
```bash
7z2john Hurdle.jpg.7z > hurdle.hash    # Hash in Datei schreiben
cat hurdle.hash                        # NIE leer! leer = Extraktion fehlgeschlagen
```

### Hash-basiert vs. direkte Entschlüsselung — der wichtige Unterschied
- **Hash-basiert** (7z, PDF, KeePass, iWork, ODF, shadow): es gibt einen verifizierbaren Hash →
  John/Hashcat prüfen per Vergleich, Erfolg = eindeutig.
- **Direkte Entschlüsselung** (OpenSSL `enc`): **kein** Hash, kein Exit-Code der „richtig" sagt →
  Erfolg erkennt man daran, dass **lesbarer (printable) Klartext** herauskommt.

---

## 4. Hash aufbereiten (nur nötig, wenn Hashcat statt John)

John frisst die `*2john`-Ausgabe meist direkt. Für **Hashcat** muss man den `Dateiname:`-Präfix
(und bei ODF den Suffix) abschneiden:

```bash
# PDF: Dateinamen-Präfix entfernen
pdf2john Star.pdf | sed -E 's/^[^:]+://' > star.hashcat

# LibreOffice/ODF: Präfix UND Suffix entfernen
libreoffice2john.py Numbers1.odt | sed -E -e 's/^[^:]+://' -e 's/:::::[^:]+$//' > num.hashcat
```

---

## 5. Angriff — John ↔ Hashcat nebeneinander

Rockyou-Pfad in Kali: `/usr/share/wordlists/rockyou.txt`
(falls `.gz`: `sudo gunzip /usr/share/wordlists/rockyou.txt.gz`)
Weitere: `/usr/share/dict/cracklib-small`, `/usr/share/wordlists/metasploit/*.txt`

### a) Reines Wörterbuch
```bash
john    --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
hashcat -m 0 hash.txt -a 0 /usr/share/wordlists/rockyou.txt      # -a 0 = Wörterbuch
```

### b) Wörterbuch + Regeln (wenn rohes Rockyou scheitert)
```bash
john    --wordlist=rockyou.txt --rules=Jumbo hash.txt
hashcat -m 0 hash.txt -a 0 rockyou.txt -r /usr/share/hashcat/rules/best64.rule
```

### c) Maske / Brute-force (bekanntes Muster)
```bash
# Beispiel: 5-stellige PIN
john    --mask='?d?d?d?d?d' hash.txt
hashcat -m 1400 hash.txt -a 3 '?d?d?d?d?d'                        # -a 3 = Maske
```

### d) Kombinator = zwei Wörterlisten kreuzen  ⭐ (schnell, KEINE Zwischendatei nötig)
`-a 1` hängt **jedes** Wort aus Liste 1 an **jedes** aus Liste 2 — direkt, ohne vorher eine
Datei zu erzeugen. Unter Zeitdruck fast immer besser als „erst per Skript eine Liste bauen".
```bash
# Grundform: links.txt × rechts.txt  (z.B. BerlinHamburg)
hashcat -m 1400 hash.txt -a 1 links.txt rechts.txt

# gleiche Liste mit sich selbst (StadtStadt, HausHaus, ...)
hashcat -m 1400 hash.txt -a 1 cities.txt cities.txt

# Regel je Seite:  -j = linkes/erstes Wort,  -k = rechtes/zweites Wort
hashcat -m 1400 hash.txt -a 1 a.txt b.txt -j '$-'    # "-" hinter das linke Wort  -> Kuh-Haus
```
**Grenzen:** `-a 1` kreuzt **genau zwei** Listen und braucht Hashcat (GPU-Vorteil).
- **Drei+ Listen:** verketten über `--stdout`, dann als Wörterbuch nutzen
  ```bash
  hashcat --stdout -a 1 base1.txt base2.txt > b12.txt
  hashcat --stdout -a 1 b12.txt  base3.txt > final.txt   # oder direkt angreifen
  ```
  (oder `combinator` / `combinator3` aus **hashcat-utils**)
- **Nur John in der VM (kein GPU):** John hat keinen nativen Kombinator → Liste einmal vorab
  erzeugen und mit `--wordlist` fahren:
  ```bash
  hashcat --stdout -a 1 cities.txt cities.txt > combined.txt
  john --wordlist=combined.txt hash.txt
  ```

### e) Hybrid = Wörterbuch + Maske (z. B. `ILuvU2023!!`)
```bash
# Eigener Zeichensatz -1='$.!', dann in Maske als ?1 referenzieren
hashcat -m 1400 hash.txt -a 6 candidates.txt -1 '$.!' '?d?d?d?d?1?1'   # -a 6 = Wort + Maske hinten
# -a 7 = Maske + Wort vorne
```

### f) Kandidaten aus stdin / eigener Liste
```bash
john    --stdin hash.txt < candidates.txt
cat candidates.txt | hashcat -m 0 hash.txt
```

---

## 6. Zeichensätze & Regeln (Kurzreferenz)

**Hashcat-Zeichensätze:**
```
?l = a-z        ?u = A-Z        ?d = 0-9
?s = Sonderz.   ?a = ?l?u?d?s   ?1..?4 = eigene (mit -1..-4 definieren)
```

**Wichtige Regeln (John- & Hashcat-kompatibel):**
```
l  lowercase           p@ssW0rd -> p@ssw0rd
u  uppercase           p@ssW0rd -> P@SSW0RD
c  Capitalize (1. groß)p@ssW0rd -> P@ssw0rd
t  Toggle Case
r  Reverse             p@ssW0rd -> dr0Wss@p
d  Duplicate (doppeln) p@ssW0rd -> p@ssW0rdp@ssW0rd
$X  X hinten anhängen
^X  X vorne voranstellen
```
Kombination: z. B. `cd` = erst Capitalize, dann doppeln → deckt `TreeTree`, `HausHaus`.

**Eigenen Regelsatz nutzen:**
```bash
# Hashcat: Regeln in datei.rule (eine pro Zeile), dann -r
printf 'cd\ndc\nd\n' > case.rule
hashcat -m 1700 hash.txt candidates.txt -r case.rule
# John: Section in ~/.john/john.conf anlegen:  [List.Rules:MyCase]  ... dann --rules=MyCase
```

---

## 7. Wörterbücher bauen mit Shell (grep/tr/sed/sort/awk)

```bash
# Alle 3-4-buchstabigen Wörter aus Rockyou (Lookahead filtert längere)
grep -Po "^[a-zA-Z]{3,4}(?=[^a-zA-Z])" /usr/share/wordlists/rockyou.txt \
  | tr '[:upper:]' '[:lower:]' | sort -u > candidates.txt

# Wörter mit love/luv/liebe drin
grep -oE "[a-zA-Z]*[Ll]((uv)|(ove)|(iebe))[a-zA-Z]*" /usr/share/wordlists/rockyou.txt \
  | sort -u > candidates.txt

# Wörter mit >=4 gleichen Buchstaben am Stück, Länge 4-16, nach Länge sortiert
grep -E "([a-zA-Z])\1{3,}" rockyou.txt | grep -E "^.{4,16}$" \
  | sed -E 's/[^a-zA-Z]//g' | sort -u \
  | awk '{print length" "$0}' | sort -n | sed -E 's/^[0-9]+ //'
```
> ⚠ `sort -u` **zerstört die Häufigkeits-Reihenfolge** eines Leaks. Nur nutzen, wenn Reihenfolge egal ist.

**Fragmente kombinieren (princeprocessor):**
```bash
princeprocessor --pw-min=6 --pw-max=16 base.txt \
  | hashcat -m 1400 hash.txt -r number_prepend.rule -r sc_append.rule
```
> Kreuzprodukt von Regelsätzen: aufpassen, dass keine doppelten Regeln entstehen
> (`^1$.` und `:$1` etc. — sonst redundant).

---

## 7b. Kandidatenlisten-Kochbuch (gezielte Listen)

> Die wichtigste Fähigkeit: Suchraum durch **Wissen über die Zielperson** klein machen.
> Ablauf immer: **Fakten sammeln → Muster erkennen → Kandidaten erzeugen → angreifen.**

**Fakten-Quellen:** Profil (`.md`), geleakte `Passworte.txt`, Hinweise im Aufgabentext.
Achte auf: Namen (Person, Partner, Kinder, Haustiere), Jahreszahlen, Orte, Interessen, bevorzugte Sonderzeichen.

### Rezept 1 — Winzige Liste (Handvoll) → direkt tippen
```bash
printf 'DerSprung\nDerGang\nDerRuf\n' > candidates.txt
john --wordlist=candidates.txt hash.txt
```

### Rezept 2 — Doppelwörter (`HausMaus`, `TestTest`) → Liste mit sich selbst kreuzen
```bash
hashcat --stdout -a 1 words.txt words.txt > doubled.txt
# oder ganz ohne Zwischendatei direkt angreifen:
hashcat -m <mode> hash.txt -a 1 words.txt words.txt
```

### Rezept 3 — Mehrteiliges Schema (`<Jahr><Stadt><Stadt><Sonderz.>`) → Kern kombinieren, Deko per Hybrid
```bash
printf 'Berlin\nHamburg\nMuenchen\nKoeln\n' > cities.txt
hashcat --stdout -a 1 cities.txt cities.txt > pairs.txt      # BerlinHamburg, ...
awk '{print "2025"$0}' pairs.txt > y_pairs.txt               # 2025BerlinHamburg, ...
# vier Sonderzeichen aus kleinem Satz als Maske hinten (Hybrid -a 6):
hashcat -m 10500 star.hashcat -a 6 y_pairs.txt -1 '$€!' '?1?1?1?1'
```

### Rezept 4 — Namen aus Profil (`#janejudy1`) → Namen kreuzen, dann dekorieren
```bash
printf 'jane\njudy\nemma\n' > names.txt
hashcat --stdout -a 1 names.txt names.txt > namepairs.txt    # janejudy, ...
printf '^#$1\n' > deco.rule                                  # '#' davor, '1' dahinter
hashcat -m <mode> hash.txt namepairs.txt -r deco.rule        # #janejudy1
```

**Groß-/Kleinschreibung nie von Hand** — Regeln erledigen das: `c` (Capitalize), `d` (doppeln), `$1` (Ziffer dran), `t` (Toggle). Basisliste klein halten, Regeln decken die Varianten ab.

**Zwei Punktebringer (auch mündlich):**
- **Reihenfolge lassen** bei nach Häufigkeit sortierten Leaks → **kein** `sort -u` (sonst „wahrscheinlichstes zuerst" kaputt).
- **Klein halten:** bei langsamen Hashes (7z, KeePass, PDF) zählt jeder Versuch — Kandidatenraum *verkleinern*, nicht aufblähen.

---

## 8. Sonderfälle (die typischen Stolperer)

### KeePass KDBX4 / Argon2 → pykeepass statt keepass2john
`keepass2john` kann nur KDBX3. Bei KDBX4 (Argon2) direkter Brute-Force in Python:
```bash
pip install pykeepass --break-system-packages
```
```python
from pykeepass import PyKeePass
for pw in open("candidates.txt", encoding="utf-8", errors="ignore"):
    pw = pw.strip()
    try:
        PyKeePass("Max Mueller.kdbx", password=pw); print("GEFUNDEN:", pw); break
    except Exception:
        pass
```

### OpenSSL `enc` (AES-256-CBC) → kein Hash, Entschlüsselungs-Schleife
Erfolg = **lesbarer Klartext** (Exit-Code lügt hier):
```bash
while read pw; do
  out=$(openssl enc -d -aes-256-cbc -md sha1 -in datei.encrypted -pass pass:"$pw" 2>/dev/null)
  if printf '%s' "$out" | LC_ALL=C grep -qP '^[[:print:][:space:]]+$'; then
    echo "TREFFER: $pw"; printf '%s\n' "$out"; break
  fi
done < candidates.txt
```
> Zuerst herausfinden, **welche Software** die Datei erzeugt hat (Dateiname/Header verraten `aes-256-cbc` + `sha1`).

### Java `String.hashCode()` → kein Krypto-Hash, eigenes Python
`h = 31*h + c` über alle Zeichen (32-bit int). Kurze Passwörter → Brute-Force,
längere → Meet-in-the-Middle. Kleiner Brute-Forcer:
```python
import itertools, string
def jhash(s):
    h = 0
    for c in s: h = (31*h + ord(c)) & 0xFFFFFFFF
    return h - 0x100000000 if h > 0x7FFFFFFF else h
targets = {int(x) for x in open("JavaHashcodes")}
alpha = string.ascii_letters + string.digits
for n in range(1, 6):
    for t in itertools.product(alpha, repeat=n):
        w = "".join(t)
        if jhash(w) in targets: print(jhash(w), w)
```

---

## 9. John Housekeeping (nützlich in der Prüfung)

```bash
john --show hash.txt              # bereits geknackte Passwörter anzeigen
cat ~/.john/john.pot              # Pot-Datei = alle Treffer (nach Neustart)
john --restore=meinjob            # unterbrochenen Lauf fortsetzen
john --session=meinjob --wordlist=rockyou.txt hash.txt   # benannte Session
john --fork=4 --wordlist=... hash.txt   # 4 CPU-Kerne nutzen
john --list=formats | tr ',' '\n' | grep -i keepass      # Format finden
```
> Läuft ein Angriff „leer" durch, ist oft die **Hash-Datei leer** (Extraktion in Schritt 3 gescheitert).
> Und: Shell-Hygiene — `VAR=wert` **ohne Leerzeichen** ums `=`, sonst stiller Fehler.

---

## 10. Plattform: VM, macOS & GPU

### Erst prüfen, was da ist (schlägt jede Vermutung)
```bash
hashcat -I     # listet Backends/Geräte — steht da eine GPU (Metal/CUDA/OpenCL)? sonst CPU/John
hashcat -b     # Benchmark: wie schnell ist die Maschine wirklich?
```

### Kali-VM (auch auf dem Mac: UTM/VirtualBox/Parallels)
- GPU-Durchreichung geht praktisch **nicht** → Hashcat bleibt CPU-only oder unbrauchbar.
- ⇒ **John (CPU) ist der Default.** Genau wie im Labor.

### Nativ auf macOS (Apple Silicon M1–M4)
- Hashcat nutzt das **Metal-Backend** → echte GPU-Beschleunigung (nur bei *schnellen* Hashes sinnvoll).
- Install über Homebrew:
  ```bash
  brew install hashcat john-jumbo
  ```
- `*2john`-Skripte finden (Pfad variiert je nach brew-Version):
  ```bash
  ls "$(brew --prefix)/share/john/"*2john*      # z.B. 7z2john.pl, pdf2john.pl, keepass2john
  ```
- rockyou ist auf macOS **nicht** vorinstalliert → aus SecLists/GitHub laden, Pfad selbst setzen.
- Metal hat gelegentlich Lücken bei exotischen Modi → John bleibt der sichere Fallback.

### Der entscheidende Punkt (fürs Mündliche!)
Bei **Container-Formaten** (7z, PDF, KeePass, iWork, ODF) hilft die GPU oft **kaum**: sie nutzen
absichtlich **langsame KDFs** (jeder Rateversuch ist teuer). Brute-Force ist da chancenlos — egal
wie schnell die Karte. ⇒ Gewinn kommt aus einer **kleinen, klugen Kandidatenliste**, nicht aus Hardware.
GPU lohnt nur bei *schnellen* Hashes (MD5/SHA + rockyou + Regeln).

---

## 11. Alternative Wege & „war das schlau?" (Labor-Rückblick)

> Kernaussage: Der Engpass war fast nie Rechenleistung, sondern die **Qualität der Kandidatenliste**.
> Genau das wurde optimiert → guter Weg. Alternativen zeigen, dass man das Werkzeug versteht.

| Aufgabe | Gewählter Weg | Alternative(n) | Warum schlau |
|---------|---------------|----------------|--------------|
| Hurdle.7z | Muster `Der <Nomen>` → Mini-Liste → `7z2john` + John | dieselbe Liste mit `hashcat -m 11600` | Muster erkannt statt brute |
| Star.pdf | Schema-Liste `<Jahr><Stadt><Stadt><Sz>` bauen | Kombinator `-a 1` (Städte×Städte) + Hybrid `-a 6` fürs Jahr/Sonderz. | Kandidatenraum gezielt klein |
| MySheet.numbers | `iwork2john` + Doppelwort-Liste | `-a 0` mit Regel `d` (→ `TestTest`) oder Kombinator + `c` | passende Regel statt Handarbeit |
| Java Hashcodes | eigenes Python + Meet-in-the-Middle | praktisch keine (kein Tool kann Java `hashCode()`) | Highlight — eigenes Werkzeug gebaut |
| Max Müller.kdbx | `keepass2john` scheitert → **pykeepass** + Profil-Kandidaten | neueres john-jumbo / `hashcat -m 13400` (KDBX4-Support **versionsabhängig**); `keepass4brute` | dokumentierter Pivot = genau das, was zählt |
| Passwords.pages | Muster `#name1` → Liste | Hybrid: Namen-Kombinator + `^#` + `$?d` | Muster direkt umgesetzt |
| OpenSSL-Datei | `enc -d`-Schleife + Printable-Check | `bruteforce-salted-openssl` | Printable-Check ist der Kniff |
| Numbers1.odt | **libreoffice2john** + rockyou | `hashcat -m 18400` | richtiger Extraktor (nicht `odf2john`) |

**Satz fürs Mündliche:** „Wir haben den *Kandidatenraum* durch Wissen über die Zielperson klein gemacht,
statt blind zu brute-forcen — deshalb reichten CPU/John, obwohl es langsame Hashes waren."

---

## 12. Muster aus den Übungsaufgaben (Strategie-Spickzettel)

> Die Prüfung ist „strukturell sehr vergleichbar". Hier das *Denkmuster* je Aufgabentyp — nicht die Lösungen.

| Aufgabentyp | Hinweis im Text → Strategie |
|-------------|------------------------------|
| Archiv + geleakte Wortliste | Muster im Leak erkennen (z. B. `Der <Substantiv>`) → gezielte Kandidatenliste |
| PDF, Schema bekannt | Policy + Aufbau `<Jahr><Stadt1><Stadt2><Sonderz.>` → Kandidatenliste generieren |
| iWork, „zusammengezogene Wörter, ≥8 Buchstaben, korrekte Groß-/Kleinschr." | Wortpaare + Capitalize-Regel |
| Java Hashcodes | eigenes Python (§8) |
| KeePass + Profil-`.md` | Fakten aus Profil → Kandidaten; KDBX4 → pykeepass (§8) |
| `.pages`, Muster wie `#name1` | gezielte Kandidaten nach erkanntem Muster |
| OpenSSL-verschlüsselt + Kandidatenliste | Software bestimmen → Entschlüsselungs-Schleife (§8) |
| ODF/Numbers, keine Infos | Best Practice: `libreoffice2john.py` + rockyou (+ Regeln) |
