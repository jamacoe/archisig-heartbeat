# ArchiSig Heartbeat

Öffentlicher Beweiswert-Anker eines privaten, revisionssicheren Archivs nach
**BSI TR-03125 (TR-ESOR)**, Modul **ArchiSig** (TR-ESOR-M.3).

Der Archivinhalt bleibt privat. Veröffentlicht wird stündlich nur, was sich
nachträglich nicht mehr ändern lässt, ohne dass es auffällt:

| Pfad | Inhalt |
|---|---|
| `heartbeat/YYYY/MM/DD/HHmmssZ.json` | ein Heartbeat: Anzahl und Gesamtgröße der Archivobjekte, Befunde der Integritätsprüfung, Zustand der Beweiskette |
| `heartbeat/chain.log` | fortlaufende Kette: `seq  utc  sha256(datei)  sha256(vorgänger)  pfad` |
| `heartbeat/latest.json` | Kopie des jüngsten Heartbeats |
| `archisig/roots.jsonl` | je Zeile ein Archivzeitstempel: Wurzel des Merkle-Hashbaums, TSA-Zeit, Seriennummer |
| `archisig/ats/<runId>.tsr` | das zugehörige RFC-3161-Zeitstempel-Token |

Keine Dateinamen, keine Inhalte, keine personenbezogenen Daten.

## Prüfen

### Mit ArchiSig, in einem Schritt

```powershell
.\Test-ArchiSigHeartbeat.ps1            # lokale Arbeitskopie
.\Test-ArchiSigHeartbeat.ps1 -Remote    # frischer Klon dieses Repositories
.\Test-ArchiSigHeartbeat.ps1 -Remote -Deep   # zusätzlich das Archiv vollständig nachrechnen
```

Das Skript macht automatisch, was unten von Hand steht: Kette nachrechnen,
Commit-Signaturen prüfen, die veröffentlichten Zeitstempel gegen das Archiv
halten. Jede Abweichung wird einzeln benannt — welcher Eintrag, welche Datei,
welcher Lauf. Rückgabewert `0` ohne Befund, `1` bei Befunden, `2` wenn nicht
prüfbar. `-Remote` prüft, was GitHub Dritten tatsächlich ausliefert, statt der
Arbeitskopie auf dem eigenen Rechner.

### Von Hand, ohne ArchiSig

Die Kette der Heartbeats nachrechnen:

```bash
while read -r seq utc digest prev path; do
  actual=$(sha256sum "$path" | cut -d' ' -f1)
  [ "$actual" = "$digest" ] || echo "Abweichung in $path"
done < heartbeat/chain.log
```

Jeder Heartbeat nennt im Feld `prev` den SHA-256 seines Vorgängers. Ein
nachträglich entfernter oder geänderter Eintrag bricht damit jeden späteren
Eintrag und jeden darüber liegenden signierten Commit.

Ein Archivzeitstempel lässt sich einzeln prüfen:

```bash
openssl ts -reply -in archisig/ats/<runId>.tsr -token_in -text
```

Das Feld `messageImprint` muss der `root` derselben Zeile in
`archisig/roots.jsonl` entsprechen. Diese Wurzel deckt alle Datenobjekte des
Laufes ab; welches Objekt darunterhängt, weist der Inhaber mit dem jeweiligen
Evidence Record nach (RFC 4998), ohne die übrigen offenlegen zu müssen.

## Signaturen

Alle Commits sind mit einem SSH-Schlüssel signiert. Lokal prüfbar mit:

```bash
git log --show-signature
```

Der erwartete Schlüssel steht in `.allowed_signers`.
