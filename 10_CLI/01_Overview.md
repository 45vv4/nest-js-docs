# Überblick / Overview

Das Nest CLI ist ein Befehlszeilentool, das Ihnen hilft, Ihre Nest-Anwendungen zu initialisieren, zu entwickeln und zu warten. Es unterstützt Sie in mehrfacher Hinsicht, einschließlich der Gerüstbildung des Projekts, der Bereitstellung im Entwicklungsmodus und dem Erstellen und Bündeln der Anwendung für die Produktionsverteilung. Es verkörpert bewährte architektonische Muster, um gut strukturierte Apps zu fördern.

# Installation / Installation

Hinweis: In diesem Leitfaden beschreiben wir die Verwendung von npm zur Installation von Paketen, einschließlich des Nest CLI. Andere Paketmanager können nach eigenem Ermessen verwendet werden. Mit npm haben Sie mehrere Optionen, wie Ihre OS-Befehlszeile den Speicherort der Nest CLI-Binärdatei auflöst. Hier beschreiben wir die Installation der Nest-Binärdatei global mit der Option -g. Dies bietet eine gewisse Bequemlichkeit und ist der Ansatz, den wir in der gesamten Dokumentation annehmen. Beachten Sie, dass die globale Installation eines npm-Pakets die Verantwortung des Benutzers bleibt, sicherzustellen, dass die richtige Version ausgeführt wird. Es bedeutet auch, dass, wenn Sie unterschiedliche Projekte haben, jedes die gleiche Version der CLI ausführt. Eine vernünftige Alternative ist die Verwendung des npx-Programms, das in die npm-CLI eingebaut ist (oder ähnliche Funktionen anderer Paketmanager), um sicherzustellen, dass Sie eine verwaltete Version des Nest CLI ausführen. Wir empfehlen, die npx-Dokumentation und/oder Ihr DevOps-Support-Team für weitere Informationen zu konsultieren.

Installieren Sie das CLI global mit dem Befehl npm install -g (siehe den obigen Hinweis zu globalen Installationen).

```bash
$ npm install -g @nestjs/cli
```

TIPP: Alternativ können Sie diesen Befehl npx @nestjs/cli@latest verwenden, ohne das CLI global zu installieren.

# Grundlegender Arbeitsablauf / Basic workflow

Sobald installiert, können Sie CLI-Befehle direkt über die nest ausführbare Datei von Ihrer OS-Befehlszeile aufrufen. Sehen Sie sich die verfügbaren Nest-Befehle an, indem Sie Folgendes eingeben:

```bash
$ nest --help
```

Holen Sie sich Hilfe zu einem einzelnen Befehl mit dem folgenden Konstrukt. Ersetzen Sie jeden Befehl, wie z.B. new, add, usw., durch generate im folgenden Beispiel, um detaillierte Hilfe zu diesem Befehl zu erhalten:

```bash
$ nest generate --help
```

Um ein neues grundlegendes Nest-Projekt im Entwicklungsmodus zu erstellen, zu bauen und auszuführen, wechseln Sie in den Ordner, der das übergeordnete Verzeichnis Ihres neuen Projekts sein soll, und führen Sie die folgenden Befehle aus:

```bash
$ nest new my-nest-project
$ cd my-nest-project
$ npm run start:dev
```

Öffnen Sie in Ihrem Browser http://localhost:3000, um die neue Anwendung zu sehen. Die App wird automatisch neu kompiliert und neu geladen, wenn Sie eine der Quelldateien ändern.

TIPP: Wir empfehlen die Verwendung des SWC-Builders für schnellere Builds (10-mal leistungsfähiger als der Standard-TypeScript-Compiler).

# Projektstruktur / Project structure

Wenn Sie nest new ausführen, generiert Nest eine Boilerplate-Anwendungsstruktur, indem ein neuer Ordner erstellt und ein Satz von Dateien initialisiert wird. Sie können weiterhin in dieser Standardstruktur arbeiten und neue Komponenten hinzufügen, wie in dieser Dokumentation beschrieben. Wir bezeichnen die durch nest new erzeugte Projektstruktur als Standardmodus. Nest unterstützt auch eine alternative Struktur zur Verwaltung mehrerer Projekte und Bibliotheken, die als Monorepo-Modus bezeichnet wird.

Abgesehen von einigen spezifischen Überlegungen zur Funktionsweise des Build-Prozesses (im Wesentlichen vereinfacht der Monorepo-Modus die Build-Komplexitäten, die manchmal durch monorepoartige Projektstrukturen entstehen können) und der integrierten Bibliotheksunterstützung, gelten die übrigen Nest-Funktionen und diese Dokumentation gleichermaßen für die Projektstrukturen im Standard- und Monorepo-Modus. Tatsächlich können Sie jederzeit in der Zukunft problemlos vom Standardmodus in den Monorepo-Modus wechseln, sodass Sie diese Entscheidung sicher aufschieben können, während Sie noch mehr über Nest lernen.

Sie können beide Modi verwenden, um mehrere Projekte zu verwalten. Hier ist eine kurze Zusammenfassung der Unterschiede:

| Merkmal | Standardmodus | Monorepo-Modus |
| --- | --- | --- |
| Mehrere Projekte | Separate Dateisystemstruktur | Einzelne Dateisystemstruktur |
| node_modules & package.json | Separate Instanzen | Gemeinsam im Monorepo |
| Standard-Compiler | tsc | webpack |
| Compiler-Einstellungen | Separat angegeben | Monorepo-Standardwerte, die pro Projekt überschrieben werden können |
| Konfigurationsdateien wie .eslintrc.js, .prettierrc usw. | Separat angegeben | Gemeinsam im Monorepo |
| nest build und nest start Befehle | Zielvorgabe automatisch auf das (einzige) Projekt im Kontext | Zielvorgabe auf das Standardprojekt im Monorepo |
| Bibliotheken | Manuell verwaltet, normalerweise über npm-Paketierung | Integrierte Unterstützung, einschließlich Pfadverwaltung und Bündelung |

Lesen Sie die Abschnitte zu Arbeitsbereichen und Bibliotheken für detailliertere Informationen, die Ihnen bei der Entscheidung helfen, welcher Modus für Sie am besten geeignet ist.

# Lernen Sie den richtigen Weg! / Learn the right way!

80+ Kapitel
5+ Stunden Videos
Offizielles Zertifikat
Deep-Dive-Sitzungen

ERKUNDEN SIE OFFIZIELLE KURSE

# CLI-Befehlssyntax / CLI command syntax

Alle nest-Befehle folgen demselben Format:

```bash
nest commandOrAlias requiredArg [optionalArg] [options]
```

Zum Beispiel:

```bash
$ nest new my-nest-project --dry-run
```

Hier ist new der commandOrAlias. Der new-Befehl hat ein Alias n. my-nest-project ist der requiredArg. Wenn ein requiredArg nicht auf der Befehlszeile angegeben wird, fordert nest dazu auf. Außerdem hat --dry-run eine äquivalente Kurzform -d. In diesem Sinne ist der folgende Befehl gleichwertig mit dem obigen:

```bash
$ nest n my-nest-project -d
```

Die meisten Befehle und einige Optionen haben Aliase. Versuchen Sie, nest new --help auszuführen, um diese Optionen und Aliase zu sehen und Ihr Verständnis der oben genannten Konstrukte zu bestätigen.

# Befehlsübersicht / Command overview

Führen Sie nest <command> --help für einen der folgenden Befehle aus, um befehlspezifische Optionen zu sehen.

Siehe usage für detaillierte Beschreibungen zu jedem Befehl.

| Befehl | Alias | Beschreibung |
| --- | --- | --- |
| new | n | Gerüstet eine neue Standardmodus-Anwendung mit allen notwendigen Boilerplate-Dateien. |
| generate | g | Generiert und/oder ändert Dateien basierend auf einem Schema. |
| build |  | Kompiliert eine Anwendung oder einen Arbeitsbereich in einen Ausgabefolder. |
| start |  | Kompiliert und führt eine Anwendung (oder ein Standardprojekt in einem Arbeitsbereich) aus. |
| add |  | Importiert eine Bibliothek, die als Nest-Bibliothek verpackt ist und ihr Installationsschema ausführt. |
| info | i | Zeigt Informationen über installierte Nest-Pakete und andere hilfreiche Systeminformationen an. |

# Anforderungen / Requirements

Nest CLI erfordert eine Node.js-Binärdatei, die mit Internationalisierungsunterstützung (ICU) gebaut wurde, wie die offiziellen Binärdateien von der Node.js-Projektseite. Wenn Sie auf Fehler im Zusammenhang mit ICU stoßen, überprüfen Sie, ob Ihre Binärdatei diese Anforderung erfüllt.

```bash
node -p process.versions.icu
```

Wenn der Befehl undefined ausgibt, hat Ihre Node.js-Binärdatei keine Internationalisierungsunterstützung.