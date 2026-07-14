# Sefaria Text-Studio — Woher kommen die Bücher, und was sagt die README?

## Kurze Antwort auf deine Frage

**Ja**, die Bücher werden von genau dem in der README beschriebenen Datensatz bezogen — aber **nicht über GitHub selbst**, sondern über den öffentlichen **Google-Cloud-Storage-Bucket**, den die README als eigentlichen Fundort nennt:

> „The actual text data (~26GB, ~85K files) lives in a public GCS bucket and can be downloaded without authentication."

Das `Sefaria/Sefaria-Export`-Repository auf GitHub ist also kein Datenlager, sondern nur ein **Index und ein Werkzeugkasten** um den Bucket herum. Das ist ein wichtiger Unterschied, der auch meine erste Version der App betrifft — dazu gleich mehr.

## Was in der README steht (und was es für dich bedeutet)

**1. Zwei getrennte Orte**
- **GitHub-Repo** (`github.com/Sefaria/Sefaria-Export`): enthält nur `books.json` (Index-Datei mit Metadaten und Download-Links), Hilfsskripte (`scripts/`, `examples/`) und die GitHub-Action, die `books.json` monatlich neu erzeugt.
- **GCS-Bucket** (`gs://sefaria-export/` bzw. `https://storage.googleapis.com/sefaria-export/`): enthält die eigentlichen ~85.000 Textdateien, organisiert nach Format/Kategorie/Titel/Sprache/Version.

**2. Bucket-Struktur**
```
gs://sefaria-export/
  json/{Kategorien}/{Titel}/{Sprache}/{Versionstitel}.json
  txt/{Kategorien}/{Titel}/{Sprache}/{Versionstitel}.txt
  cltk-full/…       (Format für das Classical Language Toolkit)
  cltk-flat/…        (flache CLTK-Variante)
  schemas/{Titel}.json   (Struktur-/Schema-Metadaten je Werk)
  links/links0.csv … links12.csv   (alle intertextuellen Verknüpfungen)
  table_of_contents.json
```

**3. „merged"-Dateien sind der beste Einstieg.** Für jeden Titel/jede Sprache gibt es eine `merged.json`- bzw. `merged.txt`-Datei, die laut README „the maximal content available from all versions" zusammenführt — also die vollständigste verfügbare Fassung. Genau diese Datei lädt die App:
```
https://storage.googleapis.com/sefaria-export/json/Tanakh/Torah/Genesis/English/merged.json
```

**4. `books.json`** ist ein Gesamtindex über alle ~19.700 Texte mit Titel, Sprache, Kategorien und direkten Download-URLs (`json_url`, `txt_url`, …). Er wird einmal im Monat automatisch neu erzeugt. Für die Bücherliste in der App verwende ich stattdessen das Sefaria-API-Inhaltsverzeichnis (`/api/index`), weil es zusätzlich die hebräischen Titel und die Kategorie-Baumstruktur mitliefert, die `books.json` laut Doku nicht enthält — inhaltlich sind beide Quellen aber deckungsgleich, da sie aus derselben Sefaria-Datenbank stammen.

## Was ich an der App korrigiert habe

In meiner ersten Version hatte ich die Textdateien fälschlich von
`raw.githubusercontent.com/Sefaria/Sefaria-Export/master/json/…` geladen. Das war falsch, weil dieser Pfad **im Git-Repository gar nicht existiert** — dort liegen nur `books.json`, Skripte und Schemas, keine vollständigen Texte (siehe oben). Nach deiner README-Zusendung habe ich das korrigiert auf den tatsächlichen Bucket-Pfad:

```
https://storage.googleapis.com/sefaria-export/json/<Kategorien>/<Titel>/<Hebrew|English>/merged.json
```

Das ist jetzt in der Datei `sefaria-studio.html` hinterlegt (Konstante `EXPORT_RAW`).

## Ein offener Punkt: CORS

Ob ein Browser direkt auf `storage.googleapis.com` zugreifen darf, hängt von der **CORS-Konfiguration des Buckets** ab — das ist bei Google Cloud Storage nicht automatisch aktiviert, sondern muss vom Bucket-Betreiber (hier: Sefaria) separat eingerichtet werden. Da der Bucket laut README ausdrücklich für den anonymen, direkten Download vorgesehen ist, ist eine Freigabe wahrscheinlich, aber ich kann sie von hier aus nicht verlässlich prüfen (meine Sandbox hat keinen Netzwerkzugriff auf `storage.googleapis.com`, um das live zu testen).

Deshalb hat die App eine **automatische Rückfalllogik**: Schlägt der direkte Bucket-Zugriff fehl (egal ob wegen CORS oder eines fehlenden Titels), lädt sie den Text stattdessen abschnittsweise über die offizielle Sefaria-API (`/api/texts`), die von Sefarias eigener Web-Oberfläche selbst genutzt wird und daher für Browser-Zugriffe freigegeben ist. Du merkst als Nutzer im Normalfall nichts davon — nur falls beide Wege scheitern, zeigt die App eine Fehlermeldung mit beiden Fehlerursachen.

**Falls der Bucket-Zugriff in der Praxis blockiert werden sollte**, lässt sich das ohne App-Änderung beheben, z. B. durch:
- einen kleinen serverlosen Proxy (Cloudflare Worker, Netlify Function) vor dem Bucket, oder
- Umstellen der App auf ausschließlich die Sefaria-API als Quelle (dann entfällt der Bucket-Zugriff ganz).

## Zusammenfassung

| Frage | Antwort |
|---|---|
| Kommen die Bücher aus dem in der README beschriebenen Datensatz? | Ja, direkt aus dem GCS-Bucket `gs://sefaria-export`, der Kern dieses Datensatzes. |
| Kommen sie aus dem GitHub-Repo selbst? | Nein — das Repo enthält nur Index/Werkzeuge, keine Texte. |
| Welche Datei pro Buch wird geladen? | Die `merged.json`-Datei je Titel/Sprache (vollständigste Fassung). |
| Ist die README für dich relevant? | Ja — sie erklärt exakt die Struktur, auf der die App aufbaut, und war der Grund für die Korrektur der Bucket-URL. |

---

## Neue Funktionen (Version mit Overlay-Fenstern)

### 1. Automatische hebräische Umschrift (vereinfachtes modernes Ivrit)

Die Umschrift folgt dem Stil „Simple / Modern Israeli" (wie bei alittlehebrew.com): `sh`, `ch`, `tz`, `kh`, Vokale nur als `a e i o u`, keine Längenzeichen, keine Sonderzeichen. Beispiele: בְּרֵאשִׁית → *bereshit*, שָׁלוֹם → *shalom*, רוּחַ → *ruach* (furtives Patach), מֹשֶׁה → *moshe*, מִצְוָה → *mitzvah*.

Zwei Zugänge:
- **Inline je Buchfeld:** Kästchen „Umschrift" in der Abschnittsleiste — die Umschrift erscheint automatisch unter jedem hebräischen Vers und wandert beim Blättern mit.
- **Overlay-Fenster** (Button „ABC Umschrift" in der Kopfleiste): beliebigen Text einfügen, Live-Umschrift, Kopieren oder direkt in den Editor übernehmen. Markierter hebräischer Text wird beim Öffnen automatisch übernommen.

Die Engine läuft vollständig im Browser (keine externe Abhängigkeit) und verarbeitet Nikud, Dagesch, Schin/Sin-Punkte, Schuruk/Cholam male, Lesemütter, furtives Patach und Geresch-Buchstaben (ג׳ = j usw.). Bei unvokalisiertem Text (z. B. Talmud) ist nur eine Konsonanten-Näherung möglich — darauf weist das Overlay hin.

### 2. KI-Übersetzung über die OpenAI-API

Overlay-Fenster (Button „KI-Übersetzung" oder „KI"-Button in jedem Buchfeld):
- **API-Schlüssel** wird eingegeben und nur im Arbeitsspeicher der Seite gehalten — nie dauerhaft gespeichert, nie in die Datei geschrieben. Er geht direkt vom Browser an api.openai.com.
- **Modell** frei wählbar (Standard: gpt-4o-mini), **Zielsprache** Deutsch oder Englisch.
- **Quelle:** ein geladenes Buchfeld oder „Eigener Text" (Passage einfügen; markierter Text wird vorbefüllt).
- **Umfang:** „aktueller Abschnitt" oder „ganzes Buch". Beim ganzen Buch wird der Text automatisch aufbereitet: alle Abschnitte werden als Klartext extrahiert, mit Segmentmarkierungen [1], [2], … versehen und in Häppchen von ~4500 Zeichen gebündelt, die nacheinander übersetzt werden — mit Fortschrittsbalken, Live-Ergebnisanzeige, Abbrechen-Knopf und Fehlerbehandlung (bei 401/403/429 stoppt der Vorgang).
- **Ergebnis:** im Overlay lesbar, „→ In Editor übernehmen" oder „Als .txt speichern".

Kostenhinweis: Jede Anfrage wird bei OpenAI abgerechnet; ein ganzes großes Werk kann viele Teilanfragen bedeuten. Die App zeigt die Teilanzahl während der Übersetzung an.
