
#### Einführung / Introduction
Nest (NestJS) ist ein Framework zum Erstellen effizienter, skalierbarer serverseitiger Node.js-Anwendungen. Es verwendet modernes JavaScript, ist mit TypeScript vollständig kompatibel (ermöglicht es Entwicklern jedoch auch, in reinem JavaScript zu programmieren) und kombiniert Elemente der OOP (Objektorientierte Programmierung), FP (Funktionale Programmierung) und FRP (Funktionale Reaktive Programmierung).

Unter der Haube nutzt Nest robuste HTTP-Server-Frameworks wie Express (Standard) und kann optional so konfiguriert werden, dass es Fastify verwendet.

Nest bietet eine Abstraktionsebene über diesen gängigen Node.js-Frameworks (Express/Fastify), stellt aber auch deren APIs direkt dem Entwickler zur Verfügung. Dies gibt Entwicklern die Freiheit, die Vielzahl von Drittanbieter-Modulen zu verwenden, die für die zugrunde liegende Plattform verfügbar sind.

#### Philosophie / Philosophy
In den letzten Jahren hat sich JavaScript dank Node.js zur „Lingua Franca“ des Webs für sowohl Front- als auch Backend-Anwendungen entwickelt. Dies hat zu großartigen Projekten wie Angular, React und Vue geführt, die die Produktivität der Entwickler steigern und die Erstellung schneller, testbarer und erweiterbarer Frontend-Anwendungen ermöglichen. Während jedoch viele hervorragende Bibliotheken, Helfer und Tools für Node (und serverseitiges JavaScript) existieren, löst keiner von ihnen das Hauptproblem der Architektur effektiv.

Nest bietet eine einsatzbereite Anwendungsarchitektur, die es Entwicklern und Teams ermöglicht, hoch testbare, skalierbare, lose gekoppelte und leicht wartbare Anwendungen zu erstellen. Die Architektur ist stark von Angular inspiriert.

#### Installation / Installation
Um loszulegen, können Sie entweder das Projekt mit der Nest CLI erstellen oder ein Starter-Projekt klonen (beide Optionen führen zum gleichen Ergebnis).

Um das Projekt mit der Nest CLI zu erstellen, führen Sie die folgenden Befehle aus. Dies erstellt ein neues Projektverzeichnis und füllt das Verzeichnis mit den grundlegenden Nest-Dateien und unterstützenden Modulen, wodurch eine konventionelle Basisstruktur für Ihr Projekt entsteht. Die Erstellung eines neuen Projekts mit der Nest CLI wird für Erstbenutzer empfohlen. Wir fahren mit diesem Ansatz in den ersten Schritten fort.

```bash
$ npm i -g @nestjs/cli
$ nest new project-name
```

**HINWEIS / HINT**  
Um ein neues TypeScript-Projekt mit strengerem Funktionsumfang zu erstellen, verwenden Sie das Flag `--strict` mit dem Befehl `nest new`.

#### Alternativen / Alternatives
Alternativ können Sie das TypeScript-Starterprojekt mit Git installieren:

```bash
$ git clone https://github.com/nestjs/typescript-starter.git project
$ cd project
$ npm install
$ npm run start
```

**HINWEIS / HINT**  
Wenn Sie das Repository ohne die Git-Historie klonen möchten, können Sie `degit` verwenden.

Öffnen Sie Ihren Browser und navigieren Sie zu [http://localhost:3000/](http://localhost:3000/).

Um die JavaScript-Version des Starterprojekts zu installieren, verwenden Sie `javascript-starter.git` im obigen Befehl.

Sie können auch manuell ein neues Projekt von Grund auf erstellen, indem Sie die Kern- und unterstützenden Dateien mit npm (oder yarn) installieren. In diesem Fall sind Sie natürlich selbst für die Erstellung der Projektvorlagendateien verantwortlich.

```bash
$ npm i --save @nestjs/core @nestjs/common rxjs reflect-metadata
```