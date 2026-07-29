# HR-Briefing (HR-Wissen Weekly)

Wöchentliches HR-Wissen-Weekly per Email, automatisiert über GitHub Actions.
Gebaut nach demselben Muster wie das bestehende `Send-AI-Briefing-Newsletter`
(KI-Briefing) – inkl. aller dort gesammelten Lessons Learned zu Grounding,
Modell-Auswahl und Fehlerbehandlung.

## Wie es funktioniert

1. `hr_briefing.py` ruft feste, konfigurierte Quellen **wirklich per HTTP ab**
   (Bundestag, Bundesregierung, BMAS, BMF, BAG, BFH, BSG, Bundesrat, EU-
   Kommission, Haufe, LTO – siehe `SOURCES`-Dictionary im Skript).
2. Nur der tatsächlich abgerufene Text wird als Kontext an ein LLM
   (über OpenRouter, kostenlose Modelle, live abgefragt) geschickt. Das
   Modell darf keine Aktenzeichen, Daten, Gerichte oder Links erfinden,
   die nicht im Kontext stehen.
3. Nach der LLM-Antwort läuft ein automatischer Grounding-Check: alle im
   Report genannten URLs werden gegen die Liste tatsächlich abgerufener
   Quellen geprüft, und jede "Quelle:"-Angabe gegen die konfigurierten
   Quellennamen (auf Wort-Ebene – verkürzte Zitate wie "BAG" statt
   "Bundesarbeitsgericht (BAG)" sind korrekt und lösen keinen Fehlalarm aus).
4. Versand per Gmail an `REPORT_RECIPIENT_EMAIL`.

Kategorien (aus der ursprünglichen PhiBox-Vorlage übernommen):
Gesetzesvorhaben · BMF-Schreiben · Urteile · Verordnungen ·
Gesetzgebungsverfahren · HR-Digitalisierung.

## Setup

### 1. GitHub Secrets anlegen

Unter *Settings → Secrets and variables → Actions*:

| Secret | Beschreibung |
|---|---|
| `GMAIL_CREDENTIALS_JSON` | Inhalt der Google OAuth `credentials.json` (Desktop-App) |
| `GMAIL_TOKEN_JSON` | Erzeugt via `generate_token.py` |
| `REPORT_RECIPIENT_EMAIL` | Empfänger-Adresse (z. B. `l.dashoefer@dashoefer.de`) |
| `OPENROUTER_API_KEY` | Kostenloser API-Key von [openrouter.ai](https://openrouter.ai) |

**Hinweis:** Wenn bereits ein `Send-AI-Briefing-Newsletter`- oder
`Newsletter-Analyse`-Repo mit denselben Secrets läuft, können
`GMAIL_CREDENTIALS_JSON`, `GMAIL_TOKEN_JSON` und `OPENROUTER_API_KEY`
1:1 wiederverwendet werden (gleicher Google-Account / gleicher
OpenRouter-Key), solange der Scope `gmail.send` im Token enthalten ist.

### 2. Gmail-Token erzeugen (falls noch nicht vorhanden)

```bash
pip install google-auth-oauthlib google-api-python-client
python generate_token.py
```

### 3. Quellen anpassen

Die Liste der abgerufenen Seiten steht im `SOURCES`-Dictionary am Anfang
von `hr_briefing.py`. Behörden-/Presseseiten ändern gelegentlich ihre
Struktur oder URL – wenn eine Quelle im Log dauerhaft als ❌ auftaucht,
einfach die URL in `SOURCES` anpassen. Das Skript bricht dadurch nicht
ab, es lässt die betroffene Quelle nur weg (lieber fehlende als
erfundene Inhalte).

### 4. Manuell testen

Im Tab *Actions* → *HR-Briefing* → *Run workflow* auslösen, oder lokal:

```bash
export GMAIL_CREDENTIALS_JSON='...'
export GMAIL_TOKEN_JSON='...'
export REPORT_RECIPIENT_EMAIL='du@example.com'
export OPENROUTER_API_KEY='...'
python hr_briefing.py --mode weekly
```

## Bekannte Grenzen

- Kostenlose OpenRouter-Modelle können bei Rate-Limits (`429`) einzelne
  Anfragen verzögern; das Skript versucht automatisch mehrere Modelle
  nacheinander.
- Wenn eine Quelle nicht erreichbar ist, wird sie **nicht** ins Briefing
  aufgenommen – das Skript füllt Lücken nicht mit Vermutungen auf.
  Fehlgeschlagene Quellen werden am Ende des Reports aufgelistet
  (aufklappbar).
- Manche Behördenseiten laden Inhalte dynamisch per JavaScript nach;
  ein reiner `requests`-Abruf sieht dann ggf. weniger Text als im
  Browser sichtbar ist. Die `SOURCES`-Einträge wurden bewusst auf
  möglichst textlastige, serverseitig gerenderte Übersichtsseiten
  (Pressemitteilungslisten, Schreiben-Verzeichnisse) ausgerichtet.
- Die konfigurierten URLs wurden per Web-Recherche ermittelt, aber
  **nicht** aus dieser Sandbox heraus per `requests` gegen die
  Zielserver getestet (die Sandbox hat keinen Netzwerkzugriff auf
  Behördendomains) – ein erster Testlauf über *Run workflow* sollte
  vor der ersten produktiven Woche gemacht werden.
- Noch offen: Mailversand hängt am Gmail-Token-Thema
  (`ki@dashoefer.de` / Microsoft-Graph-Freigabe). Bis das geklärt ist,
  kann `mail_graph.py` als Fallback eingebunden werden (siehe
  Kontext-Übergabe).

## Geplant, noch nicht enthalten

- Kombination mit dem `Newsletter-Analyse`-Repo (Gmail-Newsletter mit
  HR-Keyword-Vorfilter) – als zweites Modul vorgesehen, sobald dieses
  Kern-Skript stabil läuft.
- GitLab-Migration (reines Kopieren + Workflow-Syntax übersetzen, erst
  nach Stabilisierung auf GitHub).
