# Nest CLI und Skripte / Nest CLI and scripts

Dieser Abschnitt bietet zusätzlichen Hintergrund, wie der Nest-Befehl mit Compilern und Skripten interagiert, um DevOps-Mitarbeiter bei der Verwaltung der Entwicklungsumgebung zu unterstützen.

Eine Nest-Anwendung ist eine Standard-Typescript-Anwendung, die in JavaScript kompiliert werden muss, bevor sie ausgeführt werden kann. Es gibt verschiedene Möglichkeiten, den Kompilierungsschritt durchzuführen, und Entwickler/Teams können die für sie am besten geeignete Methode wählen. Mit diesem Gedanken im Hinterkopf bietet Nest eine Reihe von Tools, die Folgendes tun sollen:

- Bereitstellung eines standardmäßigen Build-/Ausführungsprozesses, der über die Befehlszeile verfügbar ist und mit vernünftigen Standardeinstellungen „einfach funktioniert“.
- Sicherstellen, dass der Build-/Ausführungsprozess offen ist, sodass Entwickler direkten Zugriff auf die zugrunde liegenden Tools haben, um sie mit nativen Funktionen und Optionen anzupassen.
- Beibehaltung eines völlig standardmäßigen Typescript/Node.js-Frameworks, sodass die gesamte Kompilierungs-/Bereitstellungs-/Ausführungspipeline von beliebigen externen Tools verwaltet werden kann, die das Entwicklungsteam verwenden möchte.

Dieses Ziel wird durch eine Kombination aus dem Nest-Befehl, einem lokal installierten Typescript-Compiler und package.json-Skripten erreicht. Im Folgenden beschreiben wir, wie diese Technologien zusammenarbeiten. Dies sollte Ihnen helfen zu verstehen, was bei jedem Schritt des Build-/Ausführungsprozesses passiert und wie Sie dieses Verhalten bei Bedarf anpassen können.

## Die Nest-Binärdatei / The nest binary

Der Nest-Befehl ist eine auf Betriebssystemebene ausgeführte Binärdatei (d. h. er läuft über die OS-Befehlszeile). Dieser Befehl umfasst tatsächlich drei verschiedene Bereiche, die unten beschrieben werden. Wir empfehlen, die Build- (nest build) und Ausführungsunterbefehle (nest start) über die package.json-Skripte auszuführen, die automatisch erstellt werden, wenn ein Projekt eingerichtet wird (siehe Typescript-Starter, wenn Sie mit dem Klonen eines Repos beginnen möchten, anstatt nest new auszuführen).

### Build

nest build ist ein Wrapper über dem Standard-TS-Compiler oder SWC-Compiler (für Standardprojekte) oder dem webpack-Bundler unter Verwendung des ts-loaders (für Monorepos). Es fügt keine weiteren Kompilierungsfunktionen oder -schritte hinzu, außer das tsconfig-paths direkt zu handhaben. Der Grund für seine Existenz ist, dass die meisten Entwickler, insbesondere wenn sie mit Nest beginnen, keine Compiler-Optionen anpassen müssen (z. B. die tsconfig.json-Datei), was manchmal schwierig sein kann.

Siehe die nest build-Dokumentation für weitere Details.

### Ausführung / Execution

nest start stellt einfach sicher, dass das Projekt gebaut wurde (dasselbe wie nest build) und ruft dann den node-Befehl auf eine tragbare, einfache Weise auf, um die kompilierte Anwendung auszuführen. Wie bei Builds können Sie diesen Prozess nach Bedarf anpassen, entweder indem Sie den nest start-Befehl und seine Optionen verwenden oder ihn vollständig ersetzen. Der gesamte Prozess ist eine Standard-Typescript-Anwendungs-Build- und Ausführungspipeline, und Sie können den Prozess entsprechend verwalten.

Siehe die nest start-Dokumentation für weitere Details.

### Generierung / Generation

Die nest generate-Befehle generieren, wie der Name schon sagt, neue Nest-Projekte oder Komponenten innerhalb dieser.

## Paket-Skripte / Package scripts

Das Ausführen der Nest-Befehle auf OS-Befehlsebene erfordert, dass die Nest-Binärdatei global installiert ist. Dies ist eine Standardfunktion von npm und liegt außerhalb der direkten Kontrolle von Nest. Eine Folge davon ist, dass die global installierte Nest-Binärdatei nicht als Projektabhängigkeit in package.json verwaltet wird. Zum Beispiel können zwei verschiedene Entwickler zwei verschiedene Versionen der Nest-Binärdatei ausführen. Die Standardlösung hierfür besteht darin, Paket-Skripte zu verwenden, sodass Sie die Tools, die in den Build- und Ausführungsschritten verwendet werden, als Entwicklungsabhängigkeiten behandeln können.

Wenn Sie nest new ausführen oder den Typescript-Starter klonen, füllt Nest die package.json-Skripte des neuen Projekts mit Befehlen wie build und start. Es installiert auch die zugrunde liegenden Compiler-Tools (wie Typescript) als Entwicklungsabhängigkeiten.

Sie führen die Build- und Ausführungsskripte mit Befehlen wie aus:

```bash
$ npm run build
```

und

```bash
$ npm run start
```

Diese Befehle verwenden die Skriptausführungsfunktionen von npm, um nest build oder nest start mit der lokal installierten Nest-Binärdatei auszuführen. Durch die Verwendung dieser integrierten Paket-Skripte haben Sie die vollständige Abhängigkeitsverwaltung über die Nest CLI-Befehle. Dies bedeutet, dass alle Mitglieder Ihrer Organisation durch die empfohlene Verwendung der Skripte sicherstellen können, dass sie dieselbe Version der Befehle ausführen.

*Dies gilt für die Befehle build und start. Die Befehle nest new und nest generate sind nicht Teil der Build-/Ausführungspipeline, sodass sie in einem anderen Kontext arbeiten und keine integrierten package.json-Skripte haben.

Für die meisten Entwickler/Teams wird empfohlen, die Paket-Skripte für das Bauen und Ausführen ihrer Nest-Projekte zu verwenden. Sie können das Verhalten dieser Skripte über ihre Optionen (--path, --webpack, --webpackPath) vollständig anpassen und/oder die TS- oder Webpack-Compiler-Optionsdateien (z. B. tsconfig.json) nach Bedarf anpassen. Es steht Ihnen auch frei, einen vollständig benutzerdefinierten Build-Prozess auszuführen, um das Typescript zu kompilieren (oder sogar das Typescript direkt mit ts-node auszuführen).

## Abwärtskompatibilität / Backward compatibility

Da Nest-Anwendungen reine Typescript-Anwendungen sind, funktionieren die vorherigen Versionen der Nest-Build-/Ausführungsskripte weiterhin. Es ist nicht erforderlich, diese zu aktualisieren. Sie können die neuen nest build- und nest start-Befehle nutzen, wenn Sie bereit sind, oder weiterhin vorherige oder angepasste Skripte ausführen.

## Migration / Migration

Obwohl Sie keine Änderungen vornehmen müssen, möchten Sie möglicherweise auf die Verwendung der neuen CLI-Befehle anstelle von Tools wie tsc-watch oder ts-node migrieren. Installieren Sie in diesem Fall die neueste Version von @nestjs/cli sowohl global als auch lokal:

```bash
$ npm install -g @nestjs/cli
$ cd /some/project/root/folder
$ npm install -D @nestjs/cli
```

Sie können dann die in package.json definierten Skripte durch die folgenden ersetzen:

```json
"build": "nest build",
"start": "nest start",
"start:dev": "nest start --watch",
"start:debug": "nest start --debug --watch",
```
