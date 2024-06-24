# Logger / Logger

Nest enthält einen integrierten textbasierten Logger, der während des Anwendungs-Bootstrappings und in mehreren anderen Fällen verwendet wird, wie z.B. bei der Anzeige abgefangener Ausnahmen (d.h. Systemprotokollierung). Diese Funktionalität wird über die Logger-Klasse im @nestjs/common-Paket bereitgestellt. Sie können das Verhalten des Protokollierungssystems vollständig steuern, einschließlich der folgenden Punkte:

- Protokollierung vollständig deaktivieren
- Das Detailniveau der Protokollierung festlegen (z.B. Fehler, Warnungen, Debug-Informationen anzeigen, etc.)
- Zeitstempel im Standard-Logger überschreiben (z.B. ISO8601-Standard als Datumsformat verwenden)
- Den Standard-Logger vollständig überschreiben
- Den Standard-Logger durch Erweiterung anpassen
- Abhängigkeitsinjektion verwenden, um das Zusammenstellen und Testen Ihrer Anwendung zu vereinfachen

Sie können auch den integrierten Logger verwenden oder Ihre eigene benutzerdefinierte Implementierung erstellen, um Ihre eigenen anwendungsbezogenen Ereignisse und Nachrichten zu protokollieren.

Für fortschrittlichere Protokollierungsfunktionen können Sie ein beliebiges Node.js-Logging-Paket, wie z.B. Winston, verwenden, um ein vollständig benutzerdefiniertes, produktionsreifes Protokollierungssystem zu implementieren.

## Grundlegende Anpassung / Basic customization

Um die Protokollierung zu deaktivieren, setzen Sie die logger-Eigenschaft im (optional) Nest-Anwendungsoptionsobjekt, das als zweites Argument an die Methode NestFactory.create() übergeben wird, auf false.

```typescript
const app = await NestFactory.create(AppModule, {
  logger: false,
});
await app.listen(3000);
```

Um bestimmte Protokollierungsebenen zu aktivieren, setzen Sie die logger-Eigenschaft auf ein Array von Zeichenfolgen, die die anzuzeigenden Protokollierungsebenen angeben, wie folgt:

```typescript
const app = await NestFactory.create(AppModule, {
  logger: ['error', 'warn'],
});
await app.listen(3000);
```

Werte im Array können eine beliebige Kombination aus 'log', 'fatal', 'error', 'warn', 'debug' und 'verbose' sein.

**HINWEIS**
Um die Farbe in den Nachrichten des Standard-Loggers zu deaktivieren, setzen Sie die Umgebungsvariable NO_COLOR auf einen nicht leeren String.

## Benutzerdefinierte Implementierung / Custom implementation

Sie können eine benutzerdefinierte Logger-Implementierung bereitstellen, die von Nest für die Systemprotokollierung verwendet wird, indem Sie den Wert der logger-Eigenschaft auf ein Objekt setzen, das das LoggerService-Interface erfüllt. Zum Beispiel können Sie Nest anweisen, das eingebaute globale JavaScript-Konsole-Objekt zu verwenden (das das LoggerService-Interface implementiert), wie folgt:

```typescript
const app = await NestFactory.create(AppModule, {
  logger: console,
});
await app.listen(3000);
```

Die Implementierung eines eigenen benutzerdefinierten Loggers ist einfach. Implementieren Sie einfach jede der Methoden des LoggerService-Interfaces wie unten gezeigt.

```typescript
import { LoggerService, Injectable } from '@nestjs/common';

@Injectable()
export class MyLogger implements LoggerService {
  /**
   * Protokoll auf 'log'-Ebene schreiben.
   */
  log(message: any, ...optionalParams: any[]) {}

  /**
   * Protokoll auf 'fatal'-Ebene schreiben.
   */
  fatal(message: any, ...optionalParams: any[]) {}

  /**
   * Protokoll auf 'error'-Ebene schreiben.
   */
  error(message: any, ...optionalParams: any[]) {}

  /**
   * Protokoll auf 'warn'-Ebene schreiben.
   */
  warn(message: any, ...optionalParams: any[]) {}

  /**
   * Protokoll auf 'debug'-Ebene schreiben.
   */
  debug?(message: any, ...optionalParams: any[]) {}

  /**
   * Protokoll auf 'verbose'-Ebene schreiben.
   */
  verbose?(message: any, ...optionalParams: any[]) {}
}
```

Sie können dann eine Instanz von MyLogger über die logger-Eigenschaft des Nest-Anwendungsoptionsobjekts bereitstellen.

```typescript
const app = await NestFactory.create(AppModule, {
  logger: new MyLogger(),
});
await app.listen(3000);
```

Diese Technik, obwohl einfach, nutzt keine Abhängigkeitsinjektion für die MyLogger-Klasse. Dies kann einige Herausforderungen darstellen, insbesondere für Tests, und die Wiederverwendbarkeit von MyLogger einschränken. Für eine bessere Lösung siehe den Abschnitt Abhängigkeitsinjektion unten.

## Eingebauten Logger erweitern / Extend built-in logger

Anstatt einen Logger von Grund auf neu zu schreiben, können Sie Ihre Anforderungen erfüllen, indem Sie die eingebaute ConsoleLogger-Klasse erweitern und ausgewähltes Verhalten der Standardimplementierung überschreiben.

```typescript
import { ConsoleLogger } from '@nestjs/common';

export class MyLogger extends ConsoleLogger {
  error(message: any, stack?: string, context?: string) {
    // Fügen Sie hier Ihre maßgeschneiderte Logik hinzu
    super.error(...arguments);
  }
}
```

Sie können einen solchen erweiterten Logger in Ihren Feature-Modulen verwenden, wie im Abschnitt Verwendung des Loggers für die Anwendungsprotokollierung unten beschrieben.

Sie können Nest anweisen, Ihren erweiterten Logger für die Systemprotokollierung zu verwenden, indem Sie eine Instanz davon über die logger-Eigenschaft des Anwendungsoptionsobjekts übergeben (wie im Abschnitt Benutzerdefinierte Implementierung oben gezeigt) oder indem Sie die im Abschnitt Abhängigkeitsinjektion unten gezeigte Technik verwenden. Wenn Sie dies tun, sollten Sie darauf achten, super aufzurufen, wie im obigen Beispielcode gezeigt, um den spezifischen Logmethodenaufruf an die übergeordnete (eingebaute) Klasse zu delegieren, damit Nest sich auf die erwarteten eingebauten Funktionen verlassen kann.

## Abhängigkeitsinjektion / Dependency injection

Für fortschrittlichere Protokollierungsfunktionen sollten Sie die Abhängigkeitsinjektion nutzen. Beispielsweise möchten Sie möglicherweise einen ConfigService in Ihren Logger injizieren, um ihn anzupassen, und wiederum Ihren benutzerdefinierten Logger in andere Controller und/oder Provider injizieren. Um die Abhängigkeitsinjektion für Ihren benutzerdefinierten Logger zu aktivieren, erstellen Sie eine Klasse, die LoggerService implementiert, und registrieren Sie diese Klasse als Provider in einem Modul. Zum Beispiel können Sie

Definieren Sie eine MyLogger-Klasse, die entweder die eingebaute ConsoleLogger-Klasse erweitert oder vollständig überschreibt, wie in den vorherigen Abschnitten gezeigt. Stellen Sie sicher, dass Sie das LoggerService-Interface implementieren.
Erstellen Sie ein LoggerModule wie unten gezeigt und stellen Sie MyLogger aus diesem Modul bereit.

```typescript
import { Module } from '@nestjs/common';
import { MyLogger } from './my-logger.service';

@Module({
  providers: [MyLogger],
  exports: [MyLogger],
})
export class LoggerModule {}
```

Mit dieser Konstruktion stellen Sie jetzt Ihren benutzerdefinierten Logger zur Verwendung durch andere Module bereit. Da Ihre MyLogger-Klasse Teil eines Moduls ist, kann sie die Abhängigkeitsinjektion nutzen (z.B. um einen ConfigService zu injizieren). Es gibt eine weitere Technik, die benötigt wird, um diesen benutzerdefinierten Logger für die Systemprotokollierung durch Nest (z.B. für Bootstrapping und Fehlerbehandlung) bereitzustellen.

Da die Anwendungsinstanziierung (NestFactory.create()) außerhalb des Kontexts eines Moduls erfolgt, nimmt sie nicht an der normalen Abhängigkeitsinjektionsphase der Initialisierung teil. Daher müssen wir sicherstellen, dass mindestens ein Anwendungsmodul das LoggerModule importiert, um Nest zu veranlassen, eine Singleton-Instanz unserer MyLogger-Klasse zu instanziieren.

Wir können Nest dann anweisen, dieselbe Singleton-Instanz von MyLogger mit der folgenden Konstruktion zu verwenden:

```typescript
const app = await NestFactory.create(AppModule, {
  bufferLogs: true,
});
app.useLogger(app.get(MyLogger));
await app.listen(3000);
```

**HINWEIS**
Im obigen Beispiel haben wir bufferLogs auf true gesetzt, um sicherzustellen, dass alle Protokolle gepuffert werden, bis ein benutzerdefinierter Logger (in diesem Fall MyLogger) angehängt wird und der Initialisierungsprozess der Anwendung entweder abgeschlossen ist oder fehlschlägt. Wenn der Initialisierungsprozess fehlschlägt, fällt Nest auf den ursprünglichen ConsoleLogger zurück, um alle gemeldeten Fehlermeldungen auszugeben. Außerdem können Sie autoFlushLogs auf false setzen (standardmäßig true), um Protokolle manuell zu leeren (mithilfe der Methode Logger#flush()).

Hier verwenden wir die Methode get() auf der NestApplication-Instanz, um die Singleton-Instanz des MyLogger-Objekts abzurufen. Diese Technik ist im Wesentlichen eine Möglichkeit, eine Instanz eines Loggers für die Verwendung durch Nest zu "injektieren". Der Aufruf app.get() ruft die Singleton-Instanz von MyLogger ab und hängt davon ab, dass diese Instanz zuerst in einem anderen Modul injiziert wird, wie oben beschrieben.

Sie können diesen MyLogger-Provider auch in Ihren Feature-Klassen injizieren, um so ein konsistentes Protokollierungsverhalten sowohl bei der Systemprotok

ollierung von Nest als auch bei der Anwendungsprotokollierung sicherzustellen. Siehe Verwendung des Loggers für die Anwendungsprotokollierung und Injektion eines benutzerdefinierten Loggers unten für weitere Informationen.

## Verwendung des Loggers für die Anwendungsprotokollierung / Using the logger for application logging

Wir können mehrere der obigen Techniken kombinieren, um konsistentes Verhalten und Formatierung sowohl bei der Systemprotokollierung von Nest als auch bei unserer eigenen Anwendungsereignis-/Nachrichtenprotokollierung bereitzustellen.

Eine gute Praxis besteht darin, die Logger-Klasse aus @nestjs/common in jedem unserer Dienste zu instanziieren. Wir können unseren Dienstnamen als Kontextargument im Logger-Konstruktor übergeben, wie folgt:

```typescript
import { Logger, Injectable } from '@nestjs/common';

@Injectable()
class MyService {
  private readonly logger = new Logger(MyService.name);

  doSomething() {
    this.logger.log('Doing something...');
  }
}
```

In der Standard-Logger-Implementierung wird der Kontext in eckigen Klammern gedruckt, wie NestFactory im folgenden Beispiel:

```plaintext
[Nest] 19096   - 12/08/2019, 7:12:59 AM   [NestFactory] Starting Nest application...
```

Wenn wir einen benutzerdefinierten Logger über app.useLogger() bereitstellen, wird er tatsächlich intern von Nest verwendet. Das bedeutet, dass unser Code implementierungsagnostisch bleibt, während wir den Standard-Logger leicht durch unseren benutzerdefinierten Logger ersetzen können, indem wir app.useLogger() aufrufen.

Wenn wir also die Schritte aus dem vorherigen Abschnitt befolgen und app.useLogger(app.get(MyLogger)) aufrufen, würden die folgenden Aufrufe von this.logger.log() aus MyService zu Aufrufen der Methode log der MyLogger-Instanz führen.

Dies sollte für die meisten Fälle geeignet sein. Wenn Sie jedoch mehr Anpassungen benötigen (z.B. das Hinzufügen und Aufrufen benutzerdefinierter Methoden), gehen Sie zum nächsten Abschnitt.

## Injektion eines benutzerdefinierten Loggers / Injecting a custom logger

Um zu beginnen, erweitern Sie den eingebauten Logger mit einem Code wie dem folgenden. Wir geben die scope-Option als Konfigurationsmetadaten für die ConsoleLogger-Klasse an, wobei ein vorübergehender Scope angegeben wird, um sicherzustellen, dass wir eine eindeutige Instanz von MyLogger in jedem Feature-Modul haben. In diesem Beispiel erweitern wir nicht die einzelnen ConsoleLogger-Methoden (wie log(), warn(), etc.), obwohl Sie dies tun können.

```typescript
import { Injectable, Scope, ConsoleLogger } from '@nestjs/common';

@Injectable({ scope: Scope.TRANSIENT })
export class MyLogger extends ConsoleLogger {
  customLog() {
    this.log('Please feed the cat!');
  }
}
```

Erstellen Sie als Nächstes ein LoggerModule mit einer Konstruktion wie dieser:

```typescript
import { Module } from '@nestjs/common';
import { MyLogger } from './my-logger.service';

@Module({
  providers: [MyLogger],
  exports: [MyLogger],
})
export class LoggerModule {}
```

Importieren Sie als Nächstes das LoggerModule in Ihr Feature-Modul. Da wir den Standard-Logger erweitert haben, haben wir den Vorteil, die setContext-Methode zu verwenden. So können wir den kontextbezogenen benutzerdefinierten Logger wie folgt verwenden:

```typescript
import { Injectable } from '@nestjs/common';
import { MyLogger } from './my-logger.service';

@Injectable()
export class CatsService {
  private readonly cats: Cat[] = [];

  constructor(private myLogger: MyLogger) {
    // Aufgrund des transienten Scopes hat CatsService seine eigene eindeutige Instanz von MyLogger,
    // sodass das Festlegen des Kontexts hier keine anderen Instanzen in anderen Diensten beeinflusst
    this.myLogger.setContext('CatsService');
  }

  findAll(): Cat[] {
    // Sie können alle Standardmethoden aufrufen
    this.myLogger.warn('About to return cats!');
    // Und Ihre benutzerdefinierten Methoden
    this.myLogger.customLog();
    return this.cats;
  }
}
```

Instruieren Sie schließlich Nest, eine Instanz des benutzerdefinierten Loggers in Ihrer main.ts-Datei wie unten gezeigt zu verwenden. Natürlich haben wir in diesem Beispiel das Verhalten des Loggers nicht wirklich angepasst (durch Erweiterung der Logger-Methoden wie log(), warn(), etc.), daher ist dieser Schritt eigentlich nicht erforderlich. Aber er wäre notwendig, wenn Sie benutzerdefinierte Logik zu diesen Methoden hinzufügen und möchten, dass Nest dieselbe Implementierung verwendet.

```typescript
const app = await NestFactory.create(AppModule, {
  bufferLogs: true,
});
app.useLogger(new MyLogger());
await app.listen(3000);
```

**HINWEIS**
Alternativ können Sie anstelle des Setzens von bufferLogs auf true den Logger vorübergehend mit der Anweisung logger: false deaktivieren. Beachten Sie, dass wenn Sie logger: false an NestFactory.create übergeben, nichts protokolliert wird, bis Sie useLogger aufrufen, sodass Sie möglicherweise einige wichtige Initialisierungsfehler übersehen. Wenn es Ihnen nichts ausmacht, dass einige Ihrer anfänglichen Nachrichten mit dem Standard-Logger protokolliert werden, können Sie die Option logger: false einfach weglassen.

## Externe Logger verwenden / Use external logger

Produktionsanwendungen haben oft spezifische Protokollierungsanforderungen, einschließlich fortschrittlicher Filterung, Formatierung und zentralisierter Protokollierung. Der eingebaute Logger von Nest wird verwendet, um das Verhalten von Nest zu überwachen, und kann auch nützlich für die grundlegende formatierte Textprotokollierung in Ihren Feature-Modulen während der Entwicklung sein. Produktionsanwendungen nutzen jedoch oft dedizierte Protokollierungsmodule wie Winston. Wie bei jeder Standard-Node.js-Anwendung können Sie solche Module in Nest vollständig nutzen.
