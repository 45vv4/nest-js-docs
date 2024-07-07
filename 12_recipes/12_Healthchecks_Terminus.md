# Gesundheitschecks (Terminus) / Healthchecks (Terminus)

Die Terminus-Integration bietet Ihnen Readiness/Liveness-Gesundheitschecks. Gesundheitschecks sind entscheidend bei komplexen Backend-Setups. Kurz gesagt, ein Gesundheitscheck im Bereich der Webentwicklung besteht normalerweise aus einer speziellen Adresse, zum Beispiel https://my-website.com/health/readiness. Ein Dienst oder eine Komponente Ihrer Infrastruktur (z.B. Kubernetes) überprüft kontinuierlich diese Adresse. Abhängig vom HTTP-Statuscode, der von einer GET-Anfrage an diese Adresse zurückgegeben wird, wird der Dienst Maßnahmen ergreifen, wenn er eine „ungesunde“ Antwort erhält. Da die Definition von „gesund“ oder „ungesund“ je nach Art des angebotenen Dienstes variiert, unterstützt Sie die Terminus-Integration mit einer Reihe von Gesundheitsindikatoren.

Zum Beispiel, wenn Ihr Webserver MongoDB zur Speicherung seiner Daten verwendet, wäre es von entscheidender Bedeutung zu wissen, ob MongoDB noch läuft. In diesem Fall können Sie den MongooseHealthIndicator nutzen. Wenn er richtig konfiguriert ist - mehr dazu später - wird Ihre Gesundheitscheck-Adresse je nach dem, ob MongoDB läuft, einen gesunden oder ungesunden HTTP-Statuscode zurückgeben.

## Einstieg / Getting started

Um mit @nestjs/terminus zu beginnen, müssen wir die erforderliche Abhängigkeit installieren.

```bash
$ npm install --save @nestjs/terminus
```

## Einrichten eines Gesundheitschecks / Setting up a Healthcheck

Ein Gesundheitscheck stellt eine Zusammenfassung von Gesundheitsindikatoren dar. Ein Gesundheitsindikator führt eine Überprüfung eines Dienstes durch, ob er sich in einem gesunden oder ungesunden Zustand befindet. Ein Gesundheitscheck ist positiv, wenn alle zugewiesenen Gesundheitsindikatoren laufen. Da viele Anwendungen ähnliche Gesundheitsindikatoren benötigen, bietet @nestjs/terminus eine Reihe vordefinierter Indikatoren wie:

- HttpHealthIndicator
- TypeOrmHealthIndicator
- MongooseHealthIndicator
- SequelizeHealthIndicator
- MikroOrmHealthIndicator
- PrismaHealthIndicator
- MicroserviceHealthIndicator
- GRPCHealthIndicator
- MemoryHealthIndicator
- DiskHealthIndicator

Um mit unserem ersten Gesundheitscheck zu beginnen, erstellen wir das HealthModule und importieren das TerminusModule in dessen Imports-Array.

HINWEIS
Um das Modul mit der Nest CLI zu erstellen, führen Sie einfach den Befehl $ nest g module health aus.

```typescript
import { Module } from '@nestjs/common';
import { TerminusModule } from '@nestjs/terminus';

@Module({
  imports: [TerminusModule]
})
export class HealthModule {}
```

Unsere Gesundheitschecks können mit einem Controller ausgeführt werden, der leicht mit der Nest CLI eingerichtet werden kann.

```bash
$ nest g controller health
```

INFO
Es wird dringend empfohlen, Shutdown-Hooks in Ihrer Anwendung zu aktivieren. Die Terminus-Integration nutzt dieses Lebenszyklusereignis, wenn es aktiviert ist. Lesen Sie hier mehr über Shutdown-Hooks.

## HTTP-Gesundheitscheck / HTTP Healthcheck

Sobald wir @nestjs/terminus installiert, unser TerminusModule importiert und einen neuen Controller erstellt haben, sind wir bereit, einen Gesundheitscheck zu erstellen.

Der HTTPHealthIndicator erfordert das Paket @nestjs/axios, daher stellen Sie sicher, dass es installiert ist.

```bash
$ npm i --save @nestjs/axios axios
```

Nun können wir unseren HealthController einrichten:

```typescript
import { Controller, Get } from '@nestjs/common';
import { HealthCheckService, HttpHealthIndicator, HealthCheck } from '@nestjs/terminus';

@Controller('health')
export class HealthController {
  constructor(
    private health: HealthCheckService,
    private http: HttpHealthIndicator,
  ) {}

  @Get()
  @HealthCheck()
  check() {
    return this.health.check([
      () => this.http.pingCheck('nestjs-docs', 'https://docs.nestjs.com'),
    ]);
  }
}
```

```typescript
import { Module } from '@nestjs/common';
import { TerminusModule } from '@nestjs/terminus';
import { HttpModule } from '@nestjs/axios';
import { HealthController } from './health.controller';

@Module({
  imports: [TerminusModule, HttpModule],
  controllers: [HealthController],
})
export class HealthModule {}
```

Unser Gesundheitscheck sendet nun eine GET-Anfrage an die Adresse https://docs.nestjs.com. Wenn wir von dieser Adresse eine gesunde Antwort erhalten, wird unsere Route unter http://localhost:3000/health das folgende Objekt mit einem 200-Statuscode zurückgeben.

```json
{
  "status": "ok",
  "info": {
    "nestjs-docs": {
      "status": "up"
    }
  },
  "error": {},
  "details": {
    "nestjs-docs": {
      "status": "up"
    }
  }
}
```

Die Schnittstelle dieses Antwortobjekts kann aus dem @nestjs/terminus-Paket mit der HealthCheckResult-Schnittstelle abgerufen werden.

- status: Wenn ein Gesundheitsindikator fehlschlägt, wird der Status 'error' sein. Wenn die NestJS-App heruntergefahren wird, aber immer noch HTTP-Anfragen akzeptiert, hat der Gesundheitscheck den Status 'shutting_down'.
  - 'error' | 'ok' | 'shutting_down'
- info: Objekt, das Informationen zu jedem Gesundheitsindikator enthält, der den Status 'up' hat, oder mit anderen Worten "gesund".
  - object
- error: Objekt, das Informationen zu jedem Gesundheitsindikator enthält, der den Status 'down' hat, oder mit anderen Worten "ungesund".
  - object
- details: Objekt, das alle Informationen zu jedem Gesundheitsindikator enthält
  - object

### Überprüfung auf spezifische HTTP-Antwortcodes / Check for specific HTTP response codes

In bestimmten Fällen möchten Sie möglicherweise auf spezifische Kriterien prüfen und die Antwort validieren. Angenommen, https://my-external-service.com gibt einen Antwortcode 204 zurück. Mit HttpHealthIndicator.responseCheck können Sie speziell auf diesen Antwortcode prüfen und alle anderen Codes als ungesund einstufen.

Falls ein anderer Antwortcode als 204 zurückgegeben wird, wäre das folgende Beispiel ungesund. Der dritte Parameter erfordert, dass Sie eine Funktion (synchron oder asynchron) bereitstellen, die einen booleschen Wert zurückgibt, ob die Antwort als gesund (true) oder ungesund (false) betrachtet wird.

```typescript
// Innerhalb der `HealthController`-Klasse

@Get()
@HealthCheck()
check() {
  return this.health.check([
    () =>
      this.http.responseCheck(
        'my-external-service',
        'https://my-external-service.com',
        (res) => res.status === 204,
      ),
  ]);
}
```

### TypeOrm-Gesundheitsindikator / TypeOrm health indicator

Terminus bietet die Möglichkeit, Datenbankprüfungen in Ihren Gesundheitscheck zu integrieren. Um mit diesem Gesundheitsindikator zu beginnen, sollten Sie das Datenbankkapitel durchlesen und sicherstellen, dass Ihre Datenbankverbindung innerhalb Ihrer Anwendung hergestellt ist.

HINWEIS
Hinter den Kulissen führt der TypeOrmHealthIndicator einfach einen SELECT 1-SQL-Befehl aus, der häufig verwendet wird, um zu überprüfen, ob die Datenbank noch aktiv ist. Falls Sie eine Oracle-Datenbank verwenden, wird SELECT 1 FROM DUAL verwendet.

```typescript
@Controller('health')
export class HealthController {
  constructor(
    private health: HealthCheckService,
    private db: TypeOrmHealthIndicator,
  ) {}

  @Get()
  @HealthCheck()
  check() {
    return this.health.check([
      () => this.db.pingCheck('database'),
    ]);
  }
}
```

Wenn Ihre Datenbank erreichbar ist, sollten Sie nun das folgende JSON-Ergebnis sehen, wenn Sie http://localhost:3000/health mit einer GET-Anfrage anfordern:

```json
{
  "status": "ok",
  "info": {
    "database": {
      "status": "up"
    }
  },
  "error": {},
  "details": {
    "database": {
      "status": "up"
    }
  }
}
```

Falls Ihre App mehrere Datenbanken verwendet, müssen Sie jede Verbindung in Ihren HealthController injizieren. Anschließend können Sie einfach die Verbindungsreferenz an den TypeOrmHealthIndicator übergeben.

```typescript
@Controller('health')
export class HealthController {
  constructor(
    private health: HealthCheckService,
    private db: TypeOrmHealthIndicator,
    @InjectConnection('albumsConnection')
    private albumsConnection: Connection,
    @InjectConnection()
    private defaultConnection: Connection,
  ) {}

  @Get()
  @HealthCheck()
  check() {
    return this.health.check([
      () => this.db.pingCheck('albums-database', { connection: this.albumsConnection }),
      () => this.db.pingCheck('database', { connection: this.defaultConnection }),
    ]);
  }
}
```

### Festplatten-Gesundheitsindikator / Disk health indicator

Mit dem DiskHealthIndicator können wir überprüfen, wie viel Speicherplatz genutzt wird. Um zu beginnen, stellen Sie sicher, dass der DiskHealthIndicator in Ihren HealthController injiziert wird. Das folgende Beispiel überprüft den Speicherplatz des Pfads / (oder unter Windows können Sie C:\\ verwenden). Wenn dies mehr als 50% des gesamten Speicherplatzes überschreitet, würde es mit einem ungesunden Gesundheitscheck antworten.

```typescript
@Controller('health')
export class HealthController {
  constructor(
    private readonly health: HealthCheckService,
    private readonly disk: DiskHealthIndicator,
  ) {}

  @Get()
  @Health

Check()
  check() {
    return this.health.check([
      () => this.disk.checkStorage('storage', { path: '/', thresholdPercent: 0.5 }),
    ]);
  }
}
```

Mit der Funktion DiskHealthIndicator.checkStorage haben Sie auch die Möglichkeit, eine feste Menge an Speicherplatz zu überprüfen. Das folgende Beispiel wäre ungesund, falls der Pfad /my-app/ 250GB überschreiten würde.

```typescript
// Innerhalb der `HealthController`-Klasse

@Get()
@HealthCheck()
check() {
  return this.health.check([
    () => this.disk.checkStorage('storage', {  path: '/', threshold: 250 * 1024 * 1024 * 1024, })
  ]);
}
```

### Speicher-Gesundheitsindikator / Memory health indicator

Um sicherzustellen, dass Ihr Prozess ein bestimmtes Speicherlimit nicht überschreitet, kann der MemoryHealthIndicator verwendet werden. Das folgende Beispiel kann verwendet werden, um den Heap Ihres Prozesses zu überprüfen.

HINWEIS
Heap ist der Teil des Speichers, in dem sich dynamisch zugewiesener Speicher befindet (d.h. Speicher, der über malloc zugewiesen wurde). Speicher, der aus dem Heap zugewiesen wurde, bleibt zugewiesen, bis eines der folgenden Ereignisse eintritt:
- Der Speicher wird freigegeben
- Das Programm endet

```typescript
@Controller('health')
export class HealthController {
  constructor(
    private health: HealthCheckService,
    private memory: MemoryHealthIndicator,
  ) {}

  @Get()
  @HealthCheck()
  check() {
    return this.health.check([
      () => this.memory.checkHeap('memory_heap', 150 * 1024 * 1024),
    ]);
  }
}
```

Es ist auch möglich, die Speicher-RSS Ihres Prozesses mit MemoryHealthIndicator.checkRSS zu überprüfen. Dieses Beispiel würde einen ungesunden Antwortcode zurückgeben, falls Ihr Prozess mehr als 150MB zugewiesen hat.

HINWEIS
RSS ist die Resident Set Size und wird verwendet, um anzuzeigen, wie viel Speicher diesem Prozess zugewiesen wurde und sich im RAM befindet. Es umfasst keinen Speicher, der ausgelagert wurde. Es umfasst Speicher von gemeinsam genutzten Bibliotheken, solange die Seiten dieser Bibliotheken tatsächlich im Speicher sind. Es umfasst den gesamten Stapel- und Heap-Speicher.

```typescript
// Innerhalb der `HealthController`-Klasse

@Get()
@HealthCheck()
check() {
  return this.health.check([
    () => this.memory.checkRSS('memory_rss', 150 * 1024 * 1024),
  ]);
}
```

### Benutzerdefinierter Gesundheitsindikator / Custom health indicator

In einigen Fällen decken die von @nestjs/terminus bereitgestellten vordefinierten Gesundheitsindikatoren nicht alle Ihre Anforderungen an den Gesundheitscheck ab. In diesem Fall können Sie einen benutzerdefinierten Gesundheitsindikator gemäß Ihren Anforderungen einrichten.

Beginnen wir damit, einen Service zu erstellen, der unseren benutzerdefinierten Indikator darstellt. Um ein grundlegendes Verständnis dafür zu bekommen, wie ein Indikator strukturiert ist, erstellen wir ein Beispiel namens DogHealthIndicator. Dieser Service sollte den Zustand 'up' haben, wenn jeder Dog-Objekt den Typ 'goodboy' hat. Wenn diese Bedingung nicht erfüllt ist, sollte er einen Fehler auslösen.

```typescript
import { Injectable } from '@nestjs/common';
import { HealthIndicator, HealthIndicatorResult, HealthCheckError } from '@nestjs/terminus';

export interface Dog {
  name: string;
  type: string;
}

@Injectable()
export class DogHealthIndicator extends HealthIndicator {
  private dogs: Dog[] = [
    { name: 'Fido', type: 'goodboy' },
    { name: 'Rex', type: 'badboy' },
  ];

  async isHealthy(key: string): Promise<HealthIndicatorResult> {
    const badboys = this.dogs.filter(dog => dog.type === 'badboy');
    const isHealthy = badboys.length === 0;
    const result = this.getStatus(key, isHealthy, { badboys: badboys.length });

    if (isHealthy) {
      return result;
    }
    throw new HealthCheckError('Dogcheck failed', result);
  }
}
```

Der nächste Schritt besteht darin, den Gesundheitsindikator als Anbieter zu registrieren.

```typescript
import { Module } from '@nestjs/common';
import { TerminusModule } from '@nestjs/terminus';
import { DogHealthIndicator } from './dog.health';

@Module({
  controllers: [HealthController],
  imports: [TerminusModule],
  providers: [DogHealthIndicator]
})
export class HealthModule { }
```

HINWEIS
In einer realen Anwendung sollte der DogHealthIndicator in einem separaten Modul, zum Beispiel DogModule, bereitgestellt werden, das dann vom HealthModule importiert wird.

Der letzte erforderliche Schritt besteht darin, den nun verfügbaren Gesundheitsindikator im erforderlichen Gesundheitscheck-Endpunkt hinzuzufügen. Dazu kehren wir zu unserem HealthController zurück und fügen ihn unserer Check-Funktion hinzu.

```typescript
import { HealthCheckService, HealthCheck } from '@nestjs/terminus';
import { Injectable, Dependencies, Get } from '@nestjs/common';
import { DogHealthIndicator } from './dog.health';

@Injectable()
export class HealthController {
  constructor(
    private health: HealthCheckService,
    private dogHealthIndicator: DogHealthIndicator
  ) {}

  @Get()
  @HealthCheck()
  healthCheck() {
    return this.health.check([
      () => this.dogHealthIndicator.isHealthy('dog'),
    ])
  }
}
```

## Logging

Terminus protokolliert nur Fehlermeldungen, z. B. wenn ein Gesundheitscheck fehlgeschlagen ist. Mit der Methode TerminusModule.forRoot() haben Sie mehr Kontrolle darüber, wie Fehler protokolliert werden, und können das Logging vollständig übernehmen.

In diesem Abschnitt führen wir Sie durch die Erstellung eines benutzerdefinierten Loggers, TerminusLogger. Dieser Logger erweitert den integrierten Logger. Daher können Sie auswählen, welchen Teil des Loggers Sie überschreiben möchten.

INFO
Wenn Sie mehr über benutzerdefinierte Logger in NestJS erfahren möchten, lesen Sie hier mehr.

```typescript
import { Injectable, Scope, ConsoleLogger } from '@nestjs/common';

@Injectable({ scope: Scope.TRANSIENT })
export class TerminusLogger extends ConsoleLogger {
  error(message: any, stack?: string, context?: string): void;
  error(message: any, ...optionalParams: any[]): void;
  error(
    message: unknown,
    stack?: unknown,
    context?: unknown,
    ...rest: unknown[]
  ): void {
    // Hier überschreiben, wie Fehlermeldungen protokolliert werden sollen
  }
}
```

Sobald Sie Ihren benutzerdefinierten Logger erstellt haben, müssen Sie ihn nur noch in das TerminusModule.forRoot() einfügen.

```typescript
@Module({
imports: [
  TerminusModule.forRoot({
    logger: TerminusLogger,
  }),
],
})
export class HealthModule {}
```

Um alle Logmeldungen von Terminus, einschließlich Fehlermeldungen, vollständig zu unterdrücken, konfigurieren Sie Terminus wie folgt.

```typescript
@Module({
imports: [
  TerminusModule.forRoot({
    logger: false,
  }),
],
})
export class HealthModule {}
```

Terminus ermöglicht es Ihnen, zu konfigurieren, wie Gesundheitscheck-Fehler in Ihren Protokollen angezeigt werden sollen.

| Error Log Style | Beschreibung | Beispiel |
|-----------------|--------------|----------|
| json (default)  | Gibt eine Zusammenfassung des Gesundheitscheck-Ergebnisses im Fehlerfall als JSON-Objekt aus |  |
| pretty          | Gibt eine Zusammenfassung des Gesundheitscheck-Ergebnisses im Fehlerfall in formatierten Boxen aus und hebt erfolgreiche/fehlgeschlagene Ergebnisse hervor |  |

Sie können den Protokollstil mithilfe der errorLogStyle-Konfigurationsoption ändern, wie im folgenden Snippet gezeigt.

```typescript
@Module({
  imports: [
    TerminusModule.forRoot({
      errorLogStyle: 'pretty',
    }),
  ]
})
export class HealthModule {}
```

### Zeitüberschreitung für das sanfte Herunterfahren / Graceful shutdown timeout

Wenn Ihre Anwendung es erfordert, den Herunterfahrprozess zu verzögern, kann Terminus dies für Sie übernehmen. Diese Einstellung kann besonders nützlich sein, wenn Sie mit einem Orchestrator wie Kubernetes arbeiten. Durch das Festlegen einer Verzögerung, die etwas länger als das Readiness-Check-Intervall ist, können Sie bei der Herunterfahren von Containern eine Ausfallzeit von null erreichen.

```typescript
@Module({
  imports: [
    TerminusModule.forRoot({
      gracefulShutdownTimeoutMs: 1000,
    }),
  ]
})
export class HealthModule {}
```

## Weitere Beispiele / More examples

Weitere funktionierende Beispiele finden Sie hier.