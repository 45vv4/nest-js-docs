# Bibliotheken / Libraries

Viele Anwendungen müssen dieselben allgemeinen Probleme lösen oder eine modulare Komponente in verschiedenen Kontexten wiederverwenden. Nest bietet verschiedene Möglichkeiten, dies zu adressieren, wobei jede auf einer anderen Ebene arbeitet, um das Problem auf eine Weise zu lösen, die unterschiedliche architektonische und organisatorische Ziele unterstützt.

Nest-Module sind nützlich, um einen Ausführungskontext bereitzustellen, der das Teilen von Komponenten innerhalb einer einzelnen Anwendung ermöglicht. Module können auch mit npm verpackt werden, um eine wiederverwendbare Bibliothek zu erstellen, die in verschiedenen Projekten installiert werden kann. Dies kann eine effektive Methode sein, um konfigurierbare, wiederverwendbare Bibliotheken zu verteilen, die von verschiedenen, lose verbundenen oder nicht verbundenen Organisationen verwendet werden können (z. B. durch Verteilen/Installieren von Drittanbieterbibliotheken).

Für das Teilen von Code innerhalb eng organisierter Gruppen (z. B. innerhalb von Unternehmens- oder Projektgrenzen) kann es nützlich sein, einen leichteren Ansatz für das Teilen von Komponenten zu haben. Monorepos sind als Konstrukt entstanden, um dies zu ermöglichen, und innerhalb eines Monorepos bietet eine Bibliothek eine Möglichkeit, Code auf einfache, leichte Weise zu teilen. In einem Nest-Monorepo ermöglicht die Verwendung von Bibliotheken die einfache Zusammenstellung von Anwendungen, die Komponenten teilen. Tatsächlich fördert dies die Zerlegung monolithischer Anwendungen und Entwicklungsprozesse, um sich auf den Aufbau und die Zusammensetzung modularer Komponenten zu konzentrieren.

# Nest-Bibliotheken / Nest libraries

Eine Nest-Bibliothek ist ein Nest-Projekt, das sich von einer Anwendung dadurch unterscheidet, dass es nicht eigenständig ausgeführt werden kann. Eine Bibliothek muss in eine enthaltene Anwendung importiert werden, damit ihr Code ausgeführt wird. Die in diesem Abschnitt beschriebene eingebaute Unterstützung für Bibliotheken ist nur für Monorepos verfügbar (Standardmodusprojekte können ähnliche Funktionen mit npm-Paketen erreichen).

Beispielsweise kann eine Organisation ein AuthModule entwickeln, das die Authentifizierung verwaltet, indem es Unternehmensrichtlinien implementiert, die alle internen Anwendungen regeln. Anstatt dieses Modul separat für jede Anwendung zu erstellen oder den Code physisch mit npm zu verpacken und jedes Projekt zu zwingen, es zu installieren, kann ein Monorepo dieses Modul als Bibliothek definieren. Wenn es auf diese Weise organisiert ist, können alle Nutzer des Bibliotheksmoduls eine aktuelle Version des AuthModule sehen, sobald es committet wird. Dies kann erhebliche Vorteile bei der Koordinierung der Komponentenentwicklung und -zusammenstellung sowie bei der Vereinfachung von End-to-End-Tests haben.

# Erstellen von Bibliotheken / Creating libraries

Jede Funktionalität, die sich zur Wiederverwendung eignet, ist ein Kandidat für die Verwaltung als Bibliothek. Die Entscheidung, was eine Bibliothek sein soll und was Teil einer Anwendung sein soll, ist eine architektonische Designentscheidung. Das Erstellen von Bibliotheken erfordert mehr als nur das Kopieren von Code aus einer bestehenden Anwendung in eine neue Bibliothek. Wenn der Code als Bibliothek verpackt wird, muss er von der Anwendung entkoppelt werden. Dies kann anfänglich mehr Zeit erfordern und einige Designentscheidungen erzwingen, denen man bei enger gekoppeltem Code möglicherweise nicht begegnet. Dieser zusätzliche Aufwand kann sich jedoch auszahlen, wenn die Bibliothek verwendet werden kann, um eine schnellere Anwendungszusammenstellung über mehrere Anwendungen hinweg zu ermöglichen.

Um mit der Erstellung einer Bibliothek zu beginnen, führen Sie den folgenden Befehl aus:

```bash
$ nest g library my-library
```

Wenn Sie den Befehl ausführen, fordert Sie das Bibliotheksschema auf, ein Präfix (Alias) für die Bibliothek anzugeben:

```bash
What prefix would you like to use for the library (default: @app)?
```

Dies erstellt ein neues Projekt in Ihrem Arbeitsbereich namens my-library. Ein Bibliotheksprojekt, ähnlich wie ein Anwendungsprojekt, wird in einem benannten Ordner mit einem Schema erstellt. Bibliotheken werden unter dem libs-Ordner im Stammverzeichnis des Monorepos verwaltet. Nest erstellt den libs-Ordner beim ersten Erstellen einer Bibliothek.

Die für eine Bibliothek generierten Dateien unterscheiden sich geringfügig von denen, die für eine Anwendung generiert werden. Hier ist der Inhalt des libs-Ordners nach Ausführung des obigen Befehls:

```
libs
  my-library
    src
      index.ts
      my-library.module.ts
      my-library.service.ts
    tsconfig.lib.json
```

Die nest-cli.json-Datei wird einen neuen Eintrag für die Bibliothek unter dem "projects"-Schlüssel haben:

```json
...
{
  "my-library": {
    "type": "library",
    "root": "libs/my-library",
    "entryFile": "index",
    "sourceRoot": "libs/my-library/src",
    "compilerOptions": {
      "tsConfigPath": "libs/my-library/tsconfig.lib.json"
    }
  }
}
...
```

Es gibt zwei Unterschiede in den nest-cli.json-Metadaten zwischen Bibliotheken und Anwendungen:

- Die Eigenschaft "type" ist auf "library" statt auf "application" gesetzt
- Die Eigenschaft "entryFile" ist auf "index" statt auf "main" gesetzt

Diese Unterschiede schlüsseln den Build-Prozess so auf, dass Bibliotheken angemessen behandelt werden. Beispielsweise exportiert eine Bibliothek ihre Funktionen über die index.js-Datei.

Wie bei Anwendungsprojekten hat jede Bibliothek ihre eigene tsconfig.lib.json-Datei, die die root (monorepo-weite) tsconfig.json-Datei erweitert. Sie können diese Datei bei Bedarf ändern, um bibliotheksspezifische Compiler-Optionen bereitzustellen.

Sie können die Bibliothek mit dem CLI-Befehl erstellen:

```bash
$ nest build my-library
```

# Verwendung von Bibliotheken / Using libraries

Mit den automatisch generierten Konfigurationsdateien ist die Verwendung von Bibliotheken unkompliziert. Wie würden wir MyLibraryService aus der my-library-Bibliothek in die my-project-Anwendung importieren?

Zunächst ist zu beachten, dass die Verwendung von Bibliotheksmodulen derselbe ist wie die Verwendung jedes anderen Nest-Moduls. Was das Monorepo tut, ist Pfade so zu verwalten, dass das Importieren von Bibliotheken und das Generieren von Builds jetzt transparent ist. Um MyLibraryService zu verwenden, müssen wir sein deklarierendes Modul importieren. Wir können my-project/src/app.module.ts wie folgt ändern, um MyLibraryModule zu importieren:

```typescript
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { MyLibraryModule } from '@app/my-library';

@Module({
  imports: [MyLibraryModule],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

Beachten Sie oben, dass wir im ES-Modul-Import einen Pfadalias von @app verwendet haben, was das Präfix ist, das wir mit dem nest g library-Befehl oben angegeben haben. Im Hintergrund verwaltet Nest dies durch tsconfig-Pfadzuordnung. Beim Hinzufügen einer Bibliothek aktualisiert Nest den globalen (monorepo) tsconfig.json-Datei "paths"-Schlüssel wie folgt:

```json
"paths": {
  "@app/my-library": [
    "libs/my-library/src"
  ],
  "@app/my-library/*": [
    "libs/my-library/src/*"
  ]
}
```

Zusammenfassend hat die Kombination aus Monorepo- und Bibliotheksfunktionen das Einbinden von Bibliotheksmodulen in Anwendungen einfach und intuitiv gemacht.

Dieser gleiche Mechanismus ermöglicht das Erstellen und Bereitstellen von Anwendungen, die Bibliotheken zusammenstellen. Sobald Sie das MyLibraryModule importiert haben, verwaltet das Ausführen von nest build die gesamte Modulauflösung automatisch und bündelt die App zusammen mit allen Bibliotheksabhängigkeiten zur Bereitstellung. Der Standardcompiler für ein Monorepo ist webpack, sodass die resultierende Distributionsdatei eine einzelne Datei ist, die alle transpilierten JavaScript-Dateien in einer einzigen Datei bündelt. Sie können auch zu tsc wechseln, wie [hier](https://docs.nestjs.com/cli/monorepo#global-compiler-options) beschrieben.
