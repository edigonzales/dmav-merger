# dmav

Werkzeugkasten für das Handling mit DMAV-Transferdateien:

- `merge`: Erzeugt aus mehreren Transferdateien eine einzelne DMAV-Transferdatei.
- `split`: Trennt eine DMAV-Transferdatei in einzelne Transferdateien pro Datenmodell.

Die Verarbeitung erfolgt ereignisbasiert mit **iox-ili**. Die aktuelle DMAV-Modellbasis ist `DMAVTYM_Alles_V1_1` (DMAV Version 1.1). Legacy-Daten auf Basis `DMAVTYM_Alles_V1_0` werden weiterhin unterstützt.

Snapshot-/Entwicklungsbuilds:
- GitHub Actions-Artefakte aus dem `main`-Branch: https://github.com/sogis/dmav/actions
- Der JVM-Snapshot-JAR wird als Actions-Artefakt mit 14 Tagen Aufbewahrung bereitgestellt.

Native-Build manuell testen:
1. In GitHub unter **Actions** den Workflow `build and release` öffnen.
2. **Run workflow** auswählen und als Branch `main` setzen.
3. Den Schalter `run_native` auf `true` setzen.
4. Nach Abschluss die Artefakte `native-linux-x86_64`, `native-windows-x86_64` und `native-osx-aarch_64` herunterladen.

Dieser manuelle Lauf baut die Native-ZIPs, erzeugt aber keinen GitHub-Release. Ein Release wird weiterhin ausschließlich durch einen `vX.Y.Z`-Tag ausgelöst.

Release-Versionen:
- https://github.com/sogis/dmav/releases

## Anforderungen

Java 11 oder grösser zur Ausführung. Für den Gradle-Build wird ein JDK 17 oder grösser benötigt; die CI verwendet JDK 21.

## Benutzung

### Dateien mergen

```bash
java -jar dmav.jar merge --config myconfig.ini --fosnr 449 --out /path/to/directory
```

| Name | Beschreibung | Required |
|-----|-----|-----|
| `--config` | Pfad zur Konfig-Datei (siehe unten). | ja |
| `--fosnr` | BFS-Nummer; `${fosnr}` in der Konfiguration wird damit ersetzt. | ja |
| `--out` | Zielverzeichnis für `DMAV.<fosnr>.xtf`. | ja |

Beispiel für DMAV 1.1:

```ini
DMAV_FixpunkteAVKategorie3_V1_1=/data/DMAV_FixpunkteAVKategorie3_V1_1.${fosnr}.xtf
KGKCGC_FPDS2_V1_1=https://example.org/fixpunkte_v1_1.zip
```

Die Keys entsprechen den direkt von `DMAVTYM_Alles_V1_1` importierten Datenmodellen. Values können lokale XTF-/ZIP-Dateien oder HTTP(S)-URLs auf XTF-/ZIP-Dateien sein. Bei ZIP-Dateien wird die erste enthaltene XTF-Datei verwendet.

Der Merger kompiliert das passende DMAV-Umbrella-Modell und leitet daraus die erwarteten Transfer-Topics ab. Vorhandene Baskets werden mit ihren BIDs, TIDs und Referenzen übernommen. Für fehlende Topics wird programmgesteuert ein leerer Basket mit deterministischer BID erzeugt; statische `*.empty.xtf`-Vorlagen werden nicht benötigt.

Für `FixpunkteLV_V1_0` wird ein normaler, vollständiger Input über den Key `FixpunkteLV_V1_0` unterstützt. Aus Kompatibilitätsgründen sind auch die bisherigen Keys

```ini
FixpunkteLV_V1_0_LFP=/data/lfp.xtf
FixpunkteLV_V1_0_HFP=/data/hfp.xtf
```

möglich. In diesem Fall werden `LFP1` und `HFP1` wie bisher in einen gemeinsamen `FixpunkteLV`-Basket geschrieben; die bisherige historische Basket-ID bleibt für diesen Spezialfall erhalten.

### Datei aufsplitten

```bash
java -jar dmav.jar split --input path/to/dmav.xtf --fosnr 449 --out /path/to/directory
```

| Name | Beschreibung | Required |
|-----|-----|-----|
| `--input` | DMAV-Transferdatei mit mehreren Themen/Modellen. | ja |
| `--fosnr` | BFS-Nummer als Suffix der Ausgabedateien. | ja |
| `--out` | Zielverzeichnis für die resultierenden Transferdateien. | ja |

Der Splitter liest den XTF-Eventstrom mit iox-ili und schreibt pro Datenmodell eine Ausgabedatei. Mehrere Baskets desselben Modells bleiben in derselben Datei. BIDs, TIDs und Referenzen werden unverändert übertragen.

## Architektur

Splitter und Merger arbeiten direkt auf IOX-Events (`StartBasketEvent`, `ObjectEvent`, `EndBasketEvent`) und laden nicht die vollständige Transferdatei in einen DOM-Baum. Die INTERLIS-Modelle werden zur Laufzeit über ili2c geladen. Für DMAV 1.1 ist `DMAVTYM_Alles_V1_1` die verbindliche Modellbasis; bei erkannten DMAV-1.0-Inputs wird `DMAVTYM_Alles_V1_0` verwendet.

Die Testbasis enthält einen reduzierten Ausschnitt des offiziellen DMAV-1.1-Testdatensatzes. Ein Roundtrip-Test prüft `DMAV 1.1 -> split -> merge` semantisch auf Basket-IDs, Objekt-IDs und Referenzen sowie auf die modellgetrieben erzeugten leeren Baskets.

## Tests

```bash
./gradlew clean test
```

## Native Image

Für das Native-Image-Build wird lokal GraalVM CE 25.2.4 verwendet. Der Gradle-Wrapper 9.1.0 kann direkt mit dieser GraalVM laufen:

```bash
export GRAALVM_HOME=/Users/stefan/.sdkman/candidates/java/25.2.4-graalce
export JAVA_HOME="$GRAALVM_HOME"
export PATH="$JAVA_HOME/bin:$PATH"

./gradlew nativeCompile
build/native/nativeCompile/dmav --help
```

Auf GraalVM Community Edition wird automatisch Serial GC verwendet. Falls der konkrete GraalVM-Build G1 anbietet, wird G1 automatisch aktiviert. Die Release-Binaries werden als ZIPs bereitgestellt: `dmav-linux-x86_64.zip`, `dmav-windows-x86_64.zip` und `dmav-osx-aarch_64.zip`. Jedes ZIP enthält genau eine Datei, die unter Linux/macOS `dmav` beziehungsweise unter Windows `dmav.exe` heißt.

## Release

Releases werden über Versions-Tags auf dem `main`-Branch ausgelöst. Die Basisversion steht in `build.gradle`; normale Builds tragen den Suffix `-SNAPSHOT`. Für einen Release wird die Version zusätzlich aus dem Tag an Gradle übergeben.

Beispiel für Version `0.0.12`:

```bash
git switch main
git pull --ff-only
# baseVersion in build.gradle auf 0.0.12 setzen
git commit -am "Prepare release 0.0.12"
git push origin main

# Nach erfolgreichem Main-Build:
git tag -a v0.0.12 -m "Release v0.0.12"
git push origin v0.0.12
```

Der Tag startet den Release-Workflow. Dieser baut den JVM-JAR sowie Native Images für Linux (`x86_64`), Windows (`x86_64`) und macOS (`aarch_64`) und publiziert sie als GitHub-Release. Bestehende Releases werden nicht überschrieben.
