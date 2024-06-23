#### Erste Schritte / First steps
In dieser Artikelreihe lernen Sie die grundlegenden Prinzipien von Nest kennen. Um sich mit den essenziellen Bausteinen von Nest-Anwendungen vertraut zu machen, erstellen wir eine grundlegende CRUD-Anwendung, die viele Themen auf einem einführenden Niveau abdeckt.

#### Sprache / Language
Wir lieben TypeScript, aber vor allem lieben wir Node.js. Deshalb ist Nest sowohl mit TypeScript als auch mit reinem JavaScript kompatibel. Nest nutzt die neuesten Sprachfunktionen, daher benötigen wir einen Babel-Compiler, um es mit Vanilla JavaScript zu verwenden.

Wir verwenden hauptsächlich TypeScript in den Beispielen, die wir bereitstellen, aber Sie können die Codeausschnitte jederzeit auf Vanilla JavaScript umschalten (klicken Sie einfach auf die Sprachschaltfläche in der oberen rechten Ecke jedes Ausschnitts).

#### Voraussetzungen / Prerequisites
Stellen Sie sicher, dass Node.js (Version >= 16) auf Ihrem Betriebssystem installiert ist.

#### Einrichtung / Setup
Die Einrichtung eines neuen Projekts ist mit der Nest CLI recht einfach. Mit installiertem npm können Sie ein neues Nest-Projekt mit den folgenden Befehlen in Ihrem Betriebssystem-Terminal erstellen:

```bash
$ npm i -g @nestjs/cli
$ nest new projekt-name
```
TIPP: Um ein neues Projekt mit den strikteren Funktionen von TypeScript zu erstellen, geben Sie das Flag `--strict` beim `nest new`-Befehl an.

Das Verzeichnis `projekt-name` wird erstellt, Node-Module und einige andere Boilerplate-Dateien werden installiert, und ein `src/`-Verzeichnis wird erstellt und mit mehreren Kern-Dateien gefüllt.

```bash
src
  ├── app.controller.spec.ts
  ├── app.controller.ts
  ├── app.module.ts
  ├── app.service.ts
  └── main.ts
```
Hier ist eine kurze Übersicht über diese Kerndateien:

- **app.controller.ts**: Ein grundlegender Controller mit einer einzigen Route.
- **app.controller.spec.ts**: Die Unit-Tests für den Controller.
- **app.module.ts**: Das Root-Modul der Anwendung.
- **app.service.ts**: Ein grundlegender Service mit einer einzigen Methode.
- **main.ts**: Die Einstiegsdatei der Anwendung, die die Kernfunktion `NestFactory` verwendet, um eine Nest-Anwendungsinstanz zu erstellen.

Die `main.ts` enthält eine asynchrone Funktion, die unsere Anwendung bootstrapped:

```typescript
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(3000);
}
bootstrap();
```
Um eine Nest-Anwendungsinstanz zu erstellen, verwenden wir die `NestFactory`-Klasse. `NestFactory` stellt einige statische Methoden bereit, die es ermöglichen, eine Anwendungsinstanz zu erstellen. Die `create()`-Methode gibt ein Anwendungsobjekt zurück, das die `INestApplication`-Schnittstelle erfüllt. Dieses Objekt stellt eine Reihe von Methoden bereit, die in den folgenden Kapiteln beschrieben werden. Im obigen Beispiel `main.ts` starten wir einfach unseren HTTP-Listener, der die Anwendung auf eingehende HTTP-Anfragen warten lässt.

Beachten Sie, dass ein mit der Nest CLI erstelltes Projekt eine anfängliche Projektstruktur erstellt, die Entwickler ermutigt, das Muster zu befolgen, jedes Modul in einem eigenen dedizierten Verzeichnis zu halten.

TIPP / HINT: Standardmäßig wird Ihre App bei einem Fehler beim Erstellen der Anwendung mit dem Code 1 beendet. Wenn Sie möchten, dass stattdessen ein Fehler ausgelöst wird, deaktivieren Sie die Option `abortOnError` (z.B. `NestFactory.create(AppModule, { abortOnError: false })`).

### Plattform / Platform
Nest zielt darauf ab, ein plattformunabhängiges Framework zu sein. Plattformunabhängigkeit ermöglicht es, wiederverwendbare logische Teile zu erstellen, die Entwickler in verschiedenen Anwendungsarten nutzen können. Technisch gesehen kann Nest mit jedem Node HTTP-Framework arbeiten, sobald ein Adapter erstellt wurde. Es gibt zwei HTTP-Plattformen, die standardmäßig unterstützt werden: `express` und `fastify`. Sie können diejenige auswählen, die am besten zu Ihren Bedürfnissen passt.

- **platform-express**: Express ist ein bekanntes minimalistisches Web-Framework für Node. Es ist eine kampferprobte, produktionsreife Bibliothek mit vielen von der Community implementierten Ressourcen. Das Paket `@nestjs/platform-express` wird standardmäßig verwendet. Viele Benutzer sind mit Express gut bedient und müssen nichts weiter tun, um es zu aktivieren.
- **platform-fastify**: Fastify ist ein hochleistungsfähiges und ressourcenschonendes Framework, das stark darauf ausgerichtet ist, maximale Effizienz und Geschwindigkeit zu bieten. Lesen Sie hier, wie Sie es verwenden können.

Unabhängig von der verwendeten Plattform stellt jede ihre eigene Anwendungsoberfläche bereit. Diese sind jeweils als `NestExpressApplication` und `NestFastifyApplication` zu sehen.

Wenn Sie einen Typ an die Methode `NestFactory.create()` übergeben, wie im Beispiel unten, hat das `app`-Objekt Methoden, die exklusiv für diese spezifische Plattform verfügbar sind. Beachten Sie jedoch, dass Sie keinen Typ angeben müssen, es sei denn, Sie möchten tatsächlich auf die zugrunde liegende Plattform-API zugreifen.

```typescript
const app = await NestFactory.create<NestExpressApplication>(AppModule);
```

### Anwendung ausführen
Sobald der Installationsprozess abgeschlossen ist, können Sie den folgenden Befehl in Ihrem OS-Terminal ausführen, um die Anwendung zu starten und auf eingehende HTTP-Anfragen zu warten:

```bash
$ npm run start
```
TIPP: Um den Entwicklungsprozess zu beschleunigen (20-mal schnellere Builds), können Sie den SWC-Builder verwenden, indem Sie das `-b swc`-Flag zum Startskript hinzufügen, wie folgt: `npm run start -- -b swc`.

Dieser Befehl startet die App mit dem HTTP-Server, der auf dem im `src/main.ts`-Datei definierten Port lauscht. Sobald die Anwendung läuft, öffnen Sie Ihren Browser und navigieren Sie zu `http://localhost:3000/`. Sie sollten die Nachricht "Hello World!" sehen.

Um auf Änderungen in Ihren Dateien zu achten, können Sie den folgenden Befehl ausführen, um die Anwendung zu starten:

```bash
$ npm run start:dev
```
Dieser Befehl überwacht Ihre Dateien, kompiliert sie automatisch neu und lädt den Server neu.

### Linting und Formatierung
Die CLI bietet bestmögliche Unterstützung, um einen zuverlässigen Entwicklungsworkflow im großen Maßstab zu erstellen. Daher wird ein generiertes Nest-Projekt sowohl mit einem Code-Linter als auch einem Formatter vorinstalliert (jeweils `eslint` und `prettier`).

TIPP: Nicht sicher über die Rolle von Formatierern vs. Lintern? Erfahren Sie hier den Unterschied.

Um maximale Stabilität und Erweiterbarkeit zu gewährleisten, verwenden wir die Basis-CLI-Pakete `eslint` und `prettier`. Diese Konfiguration ermöglicht eine saubere IDE-Integration mit offiziellen Erweiterungen.

Für kopflose Umgebungen, in denen eine IDE nicht relevant ist (Continuous Integration, Git-Hooks, etc.), kommt ein Nest-Projekt mit einsatzbereiten npm-Skripten.

```bash
# Lint und Auto-Fix mit eslint
$ npm run lint

# Formatieren mit prettier
$ npm run format
```
