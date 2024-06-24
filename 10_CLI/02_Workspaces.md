# Arbeitsbereiche / Workspaces

Nest hat zwei Modi zur Organisation von Code:

Standardmodus: Nützlich für den Aufbau einzelner projektorientierter Anwendungen, die ihre eigenen Abhängigkeiten und Einstellungen haben und keine Optimierung für das Teilen von Modulen oder die Optimierung komplexer Builds benötigen. Dies ist der Standardmodus.
Monorepo-Modus: Dieser Modus behandelt Code-Artefakte als Teil eines leichtgewichtigen Monorepos und kann für Entwicklerteams und/oder Multi-Projekt-Umgebungen besser geeignet sein. Er automatisiert Teile des Build-Prozesses, um es einfach zu machen, modulare Komponenten zu erstellen und zusammenzustellen, fördert die Wiederverwendung von Code, erleichtert Integrationstests, ermöglicht das einfache Teilen von projektweiten Artefakten wie eslint-Regeln und anderen Konfigurationsrichtlinien und ist einfacher zu verwenden als Alternativen wie GitHub-Submodule. Der Monorepo-Modus verwendet das Konzept eines Arbeitsbereichs, das in der nest-cli.json Datei dargestellt wird, um die Beziehung zwischen den Komponenten des Monorepos zu koordinieren.
Es ist wichtig zu beachten, dass praktisch alle Funktionen von Nest unabhängig von Ihrem Code-Organisationsmodus sind. Die einzige Auswirkung dieser Wahl ist, wie Ihre Projekte zusammengesetzt sind und wie Build-Artefakte generiert werden. Alle anderen Funktionen, von der CLI über Kernmodule bis hin zu Zusatzmodulen, funktionieren in beiden Modi gleich.

Außerdem können Sie jederzeit vom Standardmodus in den Monorepo-Modus wechseln, sodass Sie diese Entscheidung aufschieben können, bis die Vorteile des einen oder anderen Ansatzes deutlicher werden.

# Standardmodus / Standard mode

Wenn Sie nest new ausführen, wird ein neues Projekt für Sie mit einem eingebauten Schema erstellt. Nest führt Folgendes aus:

1. Erstellen eines neuen Ordners, der dem Namensargument entspricht, das Sie nest new übergeben.
2. Befüllen dieses Ordners mit Standarddateien, die einer minimalen Basis-Nest-Anwendung entsprechen. Sie können diese Dateien im typescript-starter-Repository einsehen.
3. Bereitstellen zusätzlicher Dateien wie nest-cli.json, package.json und tsconfig.json, die verschiedene Werkzeuge zum Kompilieren, Testen und Bereitstellen Ihrer Anwendung konfigurieren und aktivieren.
Von dort aus können Sie die Starter-Dateien ändern, neue Komponenten hinzufügen, Abhängigkeiten hinzufügen (z.B. npm install) und Ihre Anwendung wie in der restlichen Dokumentation beschrieben weiterentwickeln.

# Monorepo-Modus / Monorepo mode

Um den Monorepo-Modus zu aktivieren, beginnen Sie mit einer Standardmodus-Struktur und fügen Projekte hinzu. Ein Projekt kann eine vollständige Anwendung sein (die Sie mit dem Befehl nest generate app zum Arbeitsbereich hinzufügen) oder eine Bibliothek (die Sie mit dem Befehl nest generate library zum Arbeitsbereich hinzufügen). Wir werden die Details dieser speziellen Projekttypen weiter unten besprechen. Der entscheidende Punkt, den man jetzt beachten sollte, ist, dass es das Hinzufügen eines Projekts zu einer bestehenden Standardmodus-Struktur ist, das diese in den Monorepo-Modus umwandelt. Schauen wir uns ein Beispiel an.

Wenn wir ausführen:

```bash
$ nest new my-project
```

haben wir eine Standardmodus-Struktur konstruiert, mit einer Ordnerstruktur, die so aussieht:

```
node_modules
src
  app.controller.ts
  app.module.ts
  app.service.ts
  main.ts
nest-cli.json
package.json
tsconfig.json
.eslintrc.js
```

Wir können dies in eine Monorepo-Modus-Struktur umwandeln, indem wir Folgendes ausführen:

```bash
$ cd my-project
$ nest generate app my-app
```

An diesem Punkt konvertiert Nest die bestehende Struktur in eine Monorepo-Modus-Struktur. Dies führt zu einigen wichtigen Änderungen. Die Ordnerstruktur sieht nun so aus:

```
apps
  my-app
    src
      app.controller.ts
      app.module.ts
      app.service.ts
      main.ts
    tsconfig.app.json
  my-project
    src
      app.controller.ts
      app.module.ts
      app.service.ts
      main.ts
    tsconfig.app.json
nest-cli.json
package.json
tsconfig.json
.eslintrc.js
```

Das generate app-Schema hat den Code umorganisiert - indem es jedes Anwendungsprojekt unter den apps-Ordner verschoben und eine projektspezifische tsconfig.app.json-Datei im Stammordner jedes Projekts hinzugefügt hat. Unsere ursprüngliche my-project-App ist zum Standardprojekt für das Monorepo geworden und befindet sich nun als Peer mit der gerade hinzugefügten my-app unter dem Ordner apps. Wir werden die Standardprojekte weiter unten behandeln.

WARNUNG: Die Konvertierung einer Standardmodus-Struktur in den Monorepo-Modus funktioniert nur für Projekte, die der kanonischen Nest-Projektstruktur folgen. Insbesondere versucht das Schema während der Konvertierung, die src- und test-Ordner in einen Projektordner unter dem apps-Ordner im Stammverzeichnis zu verschieben. Wenn ein Projekt diese Struktur nicht verwendet, schlägt die Konvertierung fehl oder erzeugt unzuverlässige Ergebnisse.

# Arbeitsbereichsprojekte / Workspace projects

Ein Monorepo verwendet das Konzept eines Arbeitsbereichs, um seine Mitgliedseinheiten zu verwalten. Arbeitsbereiche bestehen aus Projekten. Ein Projekt kann entweder sein:

- eine Anwendung: eine vollständige Nest-Anwendung einschließlich einer main.ts-Datei zum Bootstrappen der Anwendung. Abgesehen von Kompilierungs- und Build-Überlegungen ist ein Anwendungsprojekt innerhalb eines Arbeitsbereichs funktional identisch mit einer Anwendung innerhalb einer Standardmodus-Struktur.
- eine Bibliothek: Eine Bibliothek ist eine Möglichkeit, eine allgemeine Sammlung von Funktionen (Module, Anbieter, Controller usw.) zu verpacken, die innerhalb anderer Projekte verwendet werden können. Eine Bibliothek kann nicht eigenständig ausgeführt werden und hat keine main.ts-Datei. Lesen Sie [hier](https://docs.nestjs.com/cli/libraries) mehr über Bibliotheken.

Alle Arbeitsbereiche haben ein Standardprojekt (das ein Anwendungsprojekt sein sollte). Dies wird durch die oberste "root"-Eigenschaft in der nest-cli.json-Datei definiert, die auf das Stammverzeichnis des Standardprojekts verweist (siehe CLI-Eigenschaften unten für weitere Details). Normalerweise ist dies die Standardmodus-Anwendung, mit der Sie begonnen haben und die später mit nest generate app in ein Monorepo konvertiert wurde. Wenn Sie diese Schritte befolgen, wird diese Eigenschaft automatisch ausgefüllt.

Standardprojekte werden von nest-Befehlen wie nest build und nest start verwendet, wenn kein Projektname angegeben wird.

Zum Beispiel startet in der obigen Monorepo-Struktur das Ausführen von:

```bash
$ nest start
```

die my-project-App. Um my-app zu starten, würden wir Folgendes verwenden:

```bash
$ nest start my-app
```

# Anwendungen / Applications

Anwendungsprojekte oder informell als "Anwendungen" bezeichnet, sind vollständige Nest-Anwendungen, die Sie ausführen und bereitstellen können. Sie erstellen ein Anwendungsprojekt mit nest generate app.

Dieser Befehl generiert automatisch ein Projektskelett, einschließlich der Standard-src- und test-Ordner aus dem TypeScript-Starter. Im Gegensatz zum Standardmodus hat ein Anwendungsprojekt in einem Monorepo keine Abhängigkeitspakete (package.json) oder andere Projektkonfigurationsartefakte wie .prettierrc und .eslintrc.js. Stattdessen werden die monorepo-weiten Abhängigkeiten und Konfigurationsdateien verwendet.

Das Schema generiert jedoch eine projektspezifische tsconfig.app.json-Datei im Stammordner des Projekts. Diese Konfigurationsdatei setzt automatisch geeignete Build-Optionen, einschließlich der korrekten Einstellung des Kompilierungsausgabeverzeichnisses. Die Datei erweitert die oberste (monorepo) tsconfig.json-Datei, sodass Sie globale Einstellungen monorepo-weit verwalten können, diese jedoch bei Bedarf auf Projektebene überschreiben können.

# Bibliotheken / Libraries

Wie erwähnt, sind Bibliotheksprojekte oder einfach "Bibliotheken" Pakete von Nest-Komponenten, die in Anwendungen eingebunden werden müssen, um ausgeführt zu werden. Sie erstellen ein Bibliotheksprojekt mit nest generate library. Die Entscheidung, was in eine Bibliothek gehört, ist eine architektonische Designentscheidung. Wir diskutieren Bibliotheken ausführlich im Bibliothekenkapitel.

# CLI-Eigenschaften / CLI properties

Nest speichert die Metadaten, die benötigt werden, um sowohl Standard- als auch Monorepo-strukturierte Projekte zu organisieren, zu erstellen und bereitzustellen, in der nest-cli.json-Datei. Nest fügt diese Datei automatisch hinzu und aktualisiert sie, wenn Sie Projekte hinzufügen, sodass Sie normalerweise nicht darüber nachdenken oder deren Inhalt bearbeiten müssen. Es gibt jedoch einige Einstellungen, die Sie möglicherweise manuell ändern möchten, daher ist es hilfreich, einen Überblick über die Datei zu haben.

Nach dem Ausführen der oben genannten Schritte zur Erstellung eines Monorepos sieht unsere nest-cli.json-Datei so aus:

```json
{
  "collection": "@nestjs/schematics",
  "sourceRoot": "apps/my-project/src",
  "monorepo": true,
  "root": "apps/my-project",
  "compilerOptions": {
    "webpack": true,
    "tsConfigPath": "apps/my-project/tsconfig.app.json"
  },
  "projects": {
    "my-project": {
      "type": "application",
      "root": "apps/my-project",
      "entryFile": "main",
      "sourceRoot": "apps/my-project/src",
      "compilerOptions": {
        "ts

ConfigPath": "apps/my-project/tsconfig.app.json"
      }
    },
    "my-app": {
      "type": "application",
      "root": "apps/my-app",
      "entryFile": "main",
      "sourceRoot": "apps/my-app/src",
      "compilerOptions": {
        "tsConfigPath": "apps/my-app/tsconfig.app.json"
      }
    }
  }
}
```

Die Datei ist in Abschnitte unterteilt:

- ein globaler Abschnitt mit obersten Eigenschaften, die Standard- und monorepo-weite Einstellungen steuern
- eine oberste Eigenschaft ("projects") mit Metadaten zu jedem Projekt. Dieser Abschnitt ist nur für Monorepo-Modus-Strukturen vorhanden.

Die obersten Eigenschaften sind wie folgt:

- "collection": zeigt auf die Sammlung von Schemas, die zum Generieren von Komponenten verwendet werden; Sie sollten diesen Wert im Allgemeinen nicht ändern
- "sourceRoot": zeigt auf das Stammverzeichnis des Quellcodes für das einzige Projekt in Standardmodus-Strukturen oder das Standardprojekt in Monorepo-Modus-Strukturen
- "compilerOptions": eine Karte mit Schlüsseln, die Compiler-Optionen angeben, und Werten, die die Optionseinstellungen angeben; siehe Details unten
- "generateOptions": eine Karte mit Schlüsseln, die globale Generierungsoptionen angeben, und Werten, die die Optionseinstellungen angeben; siehe Details unten
- "monorepo": (nur Monorepo) für eine Monorepo-Modus-Struktur ist dieser Wert immer true
- "root": (nur Monorepo) zeigt auf das Projektstammverzeichnis des Standardprojekts

# Globale Compiler-Optionen / Global compiler options

Diese Eigenschaften geben den zu verwendenden Compiler sowie verschiedene Optionen an, die jeden Kompilierungsschritt betreffen, unabhängig davon, ob er Teil von nest build oder nest start ist, und unabhängig davon, welcher Compiler verwendet wird, ob tsc oder webpack.

| Eigenschaftsname | Eigenschaftswerttyp | Beschreibung |
| --- | --- | --- |
| webpack | boolean | Wenn true, verwenden Sie den webpack-Compiler. Wenn false oder nicht vorhanden, verwenden Sie tsc. Im Monorepo-Modus ist der Standardwert true (verwenden Sie webpack), im Standardmodus ist der Standardwert false (verwenden Sie tsc). Siehe unten für Details. (veraltet: verwenden Sie stattdessen builder) |
| tsConfigPath | string | (nur Monorepo) Zeigt auf die Datei mit den tsconfig.json-Einstellungen, die verwendet werden, wenn nest build oder nest start ohne Projektoption aufgerufen wird (z.B. wenn das Standardprojekt gebaut oder gestartet wird). |
| webpackConfigPath | string | Zeigt auf eine webpack-Optionsdatei. Wenn nicht angegeben, sucht Nest nach der Datei webpack.config.js. Siehe unten für weitere Details. |
| deleteOutDir | boolean | Wenn true, wird bei jedem Aufruf des Compilers zuerst das Kompilierungsausgabeverzeichnis (wie in tsconfig.json konfiguriert, Standardwert ist ./dist) entfernt. |
| assets | array | Ermöglicht das automatische Verteilen von Nicht-TypeScript-Assets, sobald ein Kompilierungsschritt beginnt (Asset-Verteilung erfolgt nicht bei inkrementellen Kompilierungen im --watch-Modus). Siehe unten für Details. |
| watchAssets | boolean | Wenn true, im Watch-Modus ausführen und alle Nicht-TypeScript-Assets beobachten. (Für eine genauere Kontrolle der zu beobachtenden Assets siehe Abschnitt Assets unten). |
| manualRestart | boolean | Wenn true, ermöglicht die Abkürzung rs das manuelle Neustarten des Servers. Standardwert ist false. |
| builder | string/object | Weist die CLI an, welchen Builder sie zum Kompilieren des Projekts verwenden soll (tsc, swc oder webpack). Um das Verhalten des Builders anzupassen, können Sie ein Objekt mit zwei Attributen übergeben: type (tsc, swc oder webpack) und options. |
| typeCheck | boolean | Wenn true, wird die Typüberprüfung für SWC-gesteuerte Projekte (wenn builder swc ist) aktiviert. Standardwert ist false. |

# Globale Generierungsoptionen / Global generate options

Diese Eigenschaften geben die Standardgenerierungsoptionen an, die vom Befehl nest generate verwendet werden.

| Eigenschaftsname | Eigenschaftswerttyp | Beschreibung |
| --- | --- | --- |
| spec | boolean or object | Wenn der Wert boolean ist, aktiviert ein Wert von true die Spezifikationsgenerierung standardmäßig und ein Wert von false deaktiviert sie. Eine auf der CLI-Befehlszeile angegebene Option überschreibt diese Einstellung, ebenso wie eine projektspezifische generateOptions-Einstellung (siehe unten). Wenn der Wert ein Objekt ist, repräsentiert jeder Schlüssel einen Schemanamen und der boolesche Wert bestimmt, ob die Standard-Spezifikationsgenerierung für dieses spezifische Schema aktiviert/deaktiviert ist. |
| flat | boolean | Wenn true, werden alle Generierbefehle eine flache Struktur erzeugen. |

Das folgende Beispiel verwendet einen booleschen Wert, um anzugeben, dass die Spezifikationsdateigenerierung standardmäßig für alle Projekte deaktiviert werden soll:

```json
{
  "generateOptions": {
    "spec": false
  }
}
```

Das folgende Beispiel verwendet einen booleschen Wert, um anzugeben, dass die flache Dateigenerierung standardmäßig für alle Projekte aktiviert werden soll:

```json
{
  "generateOptions": {
    "flat": true
  }
}
```

Im folgenden Beispiel ist die Spezifikationsdateigenerierung nur für Dienstschemata deaktiviert (z.B. nest generate service...):

```json
{
  "generateOptions": {
    "spec": {
      "service": false
    }
  }
}
```

WARNUNG: Beim Angeben der spec als Objekt unterstützt der Schlüssel für das Generierungsschema derzeit keine automatische Alias-Behandlung. Das bedeutet, dass bei der Angabe eines Schlüssels wie z.B. service: false und dem Versuch, einen Dienst über den Alias s zu generieren, die Spezifikationsdatei trotzdem generiert würde. Um sicherzustellen, dass sowohl der normale Schemaname als auch der Alias wie beabsichtigt funktionieren, geben Sie sowohl den normalen Befehlsnamen als auch den Alias an, wie unten gezeigt.

```json
{
  "generateOptions": {
    "spec": {
      "service": false,
      "s": false
    }
  }
}
```

# Projektspezifische Generierungsoptionen / Project-specific generate options

Zusätzlich zur Bereitstellung globaler Generierungsoptionen können Sie auch projektspezifische Generierungsoptionen angeben. Die projektspezifischen Generierungsoptionen folgen genau demselben Format wie die globalen Generierungsoptionen, werden jedoch direkt für jedes Projekt angegeben.

Projektspezifische Generierungsoptionen überschreiben globale Generierungsoptionen.

```json
{
  "projects": {
    "cats-project": {
      "generateOptions": {
        "spec": {
          "service": false
        }
      }
    }
  }
}
```

WARNUNG: Die Reihenfolge der Priorität für Generierungsoptionen ist wie folgt. Auf der CLI-Befehlszeile angegebene Optionen haben Vorrang vor projektspezifischen Optionen. Projektspezifische Optionen überschreiben globale Optionen.

# Angegebener Compiler / Specified compiler

Der Grund für die unterschiedlichen Standard-Compiler ist, dass für größere Projekte (z.B. typischer in einem Monorepo) webpack erhebliche Vorteile bei den Build-Zeiten und bei der Erstellung einer einzigen Datei hat, die alle Projektkomponenten zusammenfasst. Wenn Sie einzelne Dateien generieren möchten, setzen Sie "webpack" auf false, wodurch der Build-Prozess tsc (oder swc) verwendet.

# Webpack-Optionen / Webpack options

Die Webpack-Optionsdatei kann standardmäßige Webpack-Konfigurationsoptionen enthalten. Um beispielsweise Webpack anzuweisen, node_modules zu bündeln (die standardmäßig ausgeschlossen sind), fügen Sie Folgendes zur webpack.config.js hinzu:

```javascript
module.exports = {
  externals: [],
};
```

Da die Webpack-Konfigurationsdatei eine JavaScript-Datei ist, können Sie sogar eine Funktion bereitstellen, die Standardoptionen übernimmt und ein modifiziertes Objekt zurückgibt:

```javascript
module.exports = function (options) {
  return {
    ...options,
    externals: [],
  };
};
```

# Assets / Assets

Die TypeScript-Kompilierung verteilt automatisch Compiler-Ausgaben (.js und .d.ts-Dateien) in das angegebene Ausgabeverzeichnis. Es kann auch praktisch sein, Nicht-TypeScript-Dateien, wie .graphql-Dateien, Bilder, .html-Dateien und andere Assets zu verteilen. Dies ermöglicht es Ihnen, nest build (und jeden initialen Kompilierungsschritt) als leichten Entwicklungsschritt zu behandeln, bei dem Sie möglicherweise Nicht-TypeScript-Dateien bearbeiten und iterativ kompilieren und testen. Die Assets sollten sich im src-Ordner befinden, andernfalls werden sie nicht kopiert.

Der Wert des assets-Schlüssels sollte ein Array von Elementen sein, das die zu verteilenden Dateien angibt. Die Elemente können einfache Zeichenfolgen mit glob-ähnlichen Dateispezifikationen sein, zum Beispiel:

```json
"assets": ["**/*.graphql"],
"watchAssets": true,
```

Für eine genauere Kontrolle können die Elemente Objekte mit den folgenden Schlüsseln sein:

- "include": glob-ähnliche Dateispezifikationen für die zu ver

teilenden Assets
- "exclude": glob-ähnliche Dateispezifikationen für Assets, die von der Include-Liste ausgeschlossen werden sollen
- "outDir": eine Zeichenfolge, die den Pfad (relativ zum Stammverzeichnis) angibt, in dem die Assets verteilt werden sollen. Standardmäßig wird das gleiche Ausgabeverzeichnis wie für die Compiler-Ausgabe verwendet.
- "watchAssets": boolean; wenn true, im Watch-Modus ausführen und die angegebenen Assets beobachten

Zum Beispiel:

```json
"assets": [
  { "include": "**/*.graphql", "exclude": "**/omitted.graphql", "watchAssets": true },
]
```

WARNUNG: Das Festlegen von watchAssets in einer obersten compilerOptions-Eigenschaft überschreibt alle watchAssets-Einstellungen innerhalb der assets-Eigenschaft.

# Projekteigenschaften / Project properties

Dieses Element existiert nur für Monorepo-Modus-Strukturen. Sie sollten diese Eigenschaften im Allgemeinen nicht bearbeiten, da sie von Nest verwendet werden, um Projekte und deren Konfigurationsoptionen innerhalb des Monorepos zu lokalisieren.
