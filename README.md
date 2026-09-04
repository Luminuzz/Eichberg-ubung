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

## 1. Dateityp identifizieren

```bash
file verdaechtig.datei          # Typ raten (Magic Bytes)
xxd verdaechtig.datei | head    # ersten Bytes ansehen
binwalk verdaechtig.datei       # bekannte Header/Signaturen finden
binwalk -E verdaechtig.datei    # Entropie -> verschlüsselt? (hohe Entropie)
```

**⚠ AppleDouble-Falle (macOS):** Dateien die mit `._` anfangen sind oft **Köder** (163 Bytes,
Magic `00 05 16 07`) — kein echtes Archiv! Immer `file` + Größe prüfen, bevor man extrahiert.

---

## 2. Hash extrahieren — Tool je Dateityp

| Dateityp | Extraktionstool | Cracker | Hashcat-Modus* |
|----------|-----------------|---------|----------------|
| `.7z` | `7z2john <f>` (Perl; ggf. `libcompress-raw-lzma-perl`) | John | 11600 |
| `.pdf` | `pdf2john <f>` | John | 10500 (v1.4–1.6) |
| `.odt/.ods` (LibreOffice/ODF) | **`libreoffice2john.py`** (NICHT `odf2john`) | John | 18400 |
| `.kdbx` (KeePass, KDBX3) | `keepass2john <f>` | John | 13400 |
| `.kdbx` (KeePass, **KDBX4/Argon2**) | ⚠ `keepass2john` scheitert → **pykeepass** (siehe §7) | Python | — |
| `.numbers/.pages/.key` (iWork) | `iwork2john <f>` | John | (kein GPU-Modus → John) |
| `.dmg` (Mac Disk Image) | `dmg2john <f>` | John | — (nur John) |
| APFS-Volume | `apfs2hashcat` (kompilieren) | Hashcat | 18300 |
| Linux-Login | aus `/etc/shadow` kopieren | beide | 1800 ($6$) / 3200 ($2y$) |
| OpenSSL `enc` (`.aes256cbc_*`) | **kein Hash!** → Entschlüsselungs-Schleife (§7) | openssl | — |
| Java `hashCode()` | **kein Standard-Hash** → eigenes Python (§7) | Python | — |

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

## 3. Hash aufbereiten (nur nötig, wenn Hashcat statt John)

John frisst die `*2john`-Ausgabe meist direkt. Für **Hashcat** muss man den `Dateiname:`-Präfix
(und bei ODF den Suffix) abschneiden:

```bash
# PDF: Dateinamen-Präfix entfernen
pdf2john Star.pdf | sed -E 's/^[^:]+://' > star.hashcat

# LibreOffice/ODF: Präfix UND Suffix entfernen
libreoffice2john.py Numbers1.odt | sed -E -e 's/^[^:]+://' -e 's/:::::[^:]+$//' > num.hashcat
```

---

## 4. Angriff — John ↔ Hashcat nebeneinander

Rockyou-Pfad in Kali: `/usr/share/wordlists/rockyou.txt`
(falls `.gz`: `sudo gunzip /usr/share/wordlists/rockyou.txt.gz`)
Weitere: `/usr/share/dict/cracklib-small`, `/usr/share/wordlists/metasploit/*.txt`

### a) Reines Wörterbuch
```bash
# John
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
# Hashcat  (-a 0 = Wörterbuch)
hashcat -m 0 hash.txt -a 0 /usr/share/wordlists/rockyou.txt
```

### b) Wörterbuch + Regeln (wenn rohes Rockyou scheitert)
```bash
# John (eingebaute Regelsätze)
john --wordlist=rockyou.txt --rules=Jumbo hash.txt
# Hashcat (best64 = Wettbewerbssieger-Regelsatz)
hashcat -m 0 hash.txt -a 0 rockyou.txt -r /usr/share/hashcat/rules/best64.rule
```

### c) Maske / Brute-force (bekanntes Muster)
```bash
# Beispiel: 5-stellige PIN
# John
john --mask='?d?d?d?d?d' hash.txt
# Hashcat  (-a 3 = Maske/Brute-force)
hashcat -m 1400 hash.txt -a 3 '?d?d?d?d?d'
```

### d) Kombinator = Kreuzprodukt zweier Wörterbücher (z. B. `BerlinHamburg`)
```bash
# Hashcat  (-a 1)  — self-combine erlaubt (Liste mit sich selbst)
hashcat -m 1400 hash.txt -a 1 cities.txt cities.txt
# John: keinen nativen -a1-Modus -> Liste vorher erzeugen und mit --wordlist fahren:
hashcat --stdout -a 1 cities.txt cities.txt > combined.txt
john --wordlist=combined.txt hash.txt
```

### e) Hybrid = Wörterbuch + Maske (z. B. `ILuvU2023!!`)
```bash
# Eigener Zeichensatz -1='$.!', dann in Maske als ?1 referenzieren
# Hashcat  (-a 6 = Wort + Maske hinten)
hashcat -m 1400 hash.txt -a 6 candidates.txt -1 '$.!' '?d?d?d?d?1?1'
# (-a 7 = Maske + Wort vorne)
```

### f) Kandidaten aus stdin / eigener Liste
```bash
# John
john --stdin hash.txt < candidates.txt
# Hashcat
cat candidates.txt | hashcat -m 0 hash.txt
```

---

## 5. Zeichensätze & Regeln (Kurzreferenz)

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

## 6. Wörterbücher bauen mit Shell (grep/tr/sed/sort/awk)

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

## 7. Sonderfälle (die typischen Stolperer)

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

## 8. John Housekeeping (nützlich in der Prüfung)

```bash
john --show hash.txt              # bereits geknackte Passwörter anzeigen
cat ~/.john/john.pot              # Pot-Datei = alle Treffer (nach Neustart)
john --restore=meinjob            # unterbrochenen Lauf fortsetzen
john --session=meinjob --wordlist=rockyou.txt hash.txt   # benannte Session
john --fork=4 --wordlist=... hash.txt   # 4 CPU-Kerne nutzen
john --list=formats | tr ',' '\n' | grep -i keepass      # Format finden
```
> Läuft ein Angriff „leer" durch, ist oft die **Hash-Datei leer** (Extraktion in Schritt 2 gescheitert).
> Und: Shell-Hygiene — `VAR=wert` **ohne Leerzeichen** ums `=`, sonst stiller Fehler.

---

## 9. Muster aus den Übungsaufgaben (Strategie-Spickzettel)

> Die Prüfung ist „strukturell sehr vergleichbar". Hier das *Denkmuster* je Aufgabentyp — nicht die Lösungen.

| Aufgabentyp | Hinweis im Text → Strategie |
|-------------|------------------------------|
| Archiv + geleakte Wortliste | Muster im Leak erkennen (z. B. `Der <Substantiv>`) → gezielte Kandidatenliste |
| PDF, Schema bekannt | Policy + Aufbau `<Jahr><Stadt1><Stadt2><Sonderz.>` → Kandidatenliste generieren |
| iWork, „zusammengezogene Wörter, ≥8 Buchstaben, korrekte Groß-/Kleinschr." | Wortpaare + Capitalize-Regel |
| Java Hashcodes | eigenes Python (§7) |
| KeePass + Profil-`.md` | Fakten aus Profil → Kandidaten; KDBX4 → pykeepass (§7) |
| `.pages`, Muster wie `#name1` | gezielte Kandidaten nach erkanntem Muster |
| OpenSSL-verschlüsselt + Kandidatenliste | Software bestimmen → Entschlüsselungs-Schleife (§7) |
| ODF/Numbers, keine Infos | Best Practice: `libreoffice2john.py` + rockyou (+ Regeln) |

---

## 10. VM-Realität
- **Kein GPU/OpenCL in VirtualBox** → Hashcat läuft nicht (gut) → **John (CPU)** ist der Default.
- Hashcat trotzdem kennen: für die mündliche Prüfung und weil manche Extraktion Hashcat-Format braucht.
- Es kann sein, dass weder John noch Hashcat *direkt* geht → dann Vorverarbeitung (§2/§3) oder eigenes Skript.
