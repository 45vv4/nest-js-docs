### Warteschlangen / Queues

Warteschlangen sind ein leistungsfähiges Entwurfsmuster, das Ihnen hilft, häufige Herausforderungen bei der Skalierung und Leistung von Anwendungen zu bewältigen. Einige Beispiele für Probleme, die Sie mit Warteschlangen lösen können, sind:

- Glätten von Verarbeitungsspitzen. Wenn Benutzer beispielsweise ressourcenintensive Aufgaben zu beliebigen Zeiten initiieren können, können Sie diese Aufgaben zu einer Warteschlange hinzufügen, anstatt sie synchron auszuführen. Dann können Worker-Prozesse die Aufgaben kontrolliert aus der Warteschlange abrufen. Sie können leicht neue Warteschlangenverbraucher hinzufügen, um die Backend-Aufgabenverarbeitung zu skalieren, wenn die Anwendung wächst.
- Aufteilen monolithischer Aufgaben, die sonst die Node.js-Ereignisschleife blockieren könnten. Wenn eine Benutzeranfrage beispielsweise eine CPU-intensive Arbeit wie Audio-Transkodierung erfordert, können Sie diese Aufgabe an andere Prozesse delegieren, sodass benutzerorientierte Prozesse reaktionsfähig bleiben.
- Bereitstellung eines zuverlässigen Kommunikationskanals über verschiedene Dienste hinweg. Beispielsweise können Sie Aufgaben (Jobs) in einem Prozess oder Dienst in die Warteschlange stellen und in einem anderen konsumieren. Sie können benachrichtigt werden (durch Statusereignisse), wenn ein Job abgeschlossen ist, ein Fehler auftritt oder andere Zustandsänderungen im Job-Lebenszyklus von jedem Prozess oder Dienst aus auftreten. Wenn Warteschlangenproduzenten oder -verbraucher ausfallen, bleibt ihr Zustand erhalten und die Aufgabenverarbeitung kann beim Neustart der Knoten automatisch neu gestartet werden.

Nest bietet das Paket `@nestjs/bull` als Abstraktion/Wrapper auf Bull, einem beliebten, gut unterstützten, leistungsstarken Warteschlangensystem für Node.js. Das Paket erleichtert die Integration von Bull-Warteschlangen auf Nest-freundliche Weise in Ihre Anwendung.

Bull verwendet Redis, um Job-Daten zu speichern, daher müssen Sie Redis auf Ihrem System installiert haben. Da es auf Redis basiert, kann Ihre Warteschlangenarchitektur vollständig verteilt und plattformunabhängig sein. Beispielsweise können Sie einige Warteschlangenproduzenten, -verbraucher und -listener auf einem (oder mehreren) Knoten in Nest ausführen und andere Produzenten, Verbraucher und Listener auf anderen Node.js-Plattformen auf anderen Netzwerkknoten.

Dieses Kapitel behandelt das Paket `@nestjs/bull`. Wir empfehlen auch, die Bull-Dokumentation für weitere Hintergrundinformationen und spezifische Implementierungsdetails zu lesen.

### Installation / Installation

Um mit der Nutzung zu beginnen, installieren wir zunächst die erforderlichen Abhängigkeiten:

```bash
$ npm install --save @nestjs/bull bull
```

Nachdem der Installationsprozess abgeschlossen ist, können wir das `BullModule` in das Haupt-App-Modul importieren.

```typescript
import { Module } from '@nestjs/common';
import { BullModule } from '@nestjs/bull';

@Module({
  imports: [
    BullModule.forRoot({
      redis: {
        host: 'localhost',
        port: 6379,
      },
    }),
  ],
})
export class AppModule {}
```

Die `forRoot()`-Methode wird verwendet, um ein Bull-Paket-Konfigurationsobjekt zu registrieren, das von allen in der Anwendung registrierten Warteschlangen verwendet wird (sofern nicht anders angegeben). Ein Konfigurationsobjekt besteht aus den folgenden Eigenschaften:

- `limiter`: `RateLimiter` - Optionen zur Steuerung der Rate, mit der die Jobs der Warteschlange verarbeitet werden. Weitere Informationen finden Sie unter `RateLimiter`. Optional.
- `redis`: `RedisOpts` - Optionen zur Konfiguration der Redis-Verbindung. Weitere Informationen finden Sie unter `RedisOpts`. Optional.
- `prefix`: `string` - Präfix für alle Warteschlangenschlüssel. Optional.
- `defaultJobOptions`: `JobOpts` - Optionen zur Steuerung der Standardeinstellungen für neue Jobs. Weitere Informationen finden Sie unter `JobOpts`. Optional.
- `settings`: `AdvancedSettings` - Erweiterte Konfigurationseinstellungen für die Warteschlange. Diese sollten normalerweise nicht geändert werden. Weitere Informationen finden Sie unter `AdvancedSettings`. Optional.

Alle Optionen sind optional und bieten detaillierte Kontrolle über das Verhalten der Warteschlange. Diese werden direkt an den Bull-Warteschlangenkonstruktor übergeben. Lesen Sie [hier](https://github.com/OptimalBits/bull/blob/master/REFERENCE.md#queue) mehr über diese Optionen.

Um eine Warteschlange zu registrieren, importieren Sie das dynamische Modul `BullModule.registerQueue()` wie folgt:

```typescript
BullModule.registerQueue({
  name: 'audio',
});
```

**HINWEIS:** Erstellen Sie mehrere Warteschlangen, indem Sie mehrere, durch Kommas getrennte Konfigurationsobjekte an die Methode `registerQueue()` übergeben.

Die `registerQueue()`-Methode wird verwendet, um Warteschlangen zu instanziieren und/oder zu registrieren. Warteschlangen werden über Module und Prozesse hinweg geteilt, die sich mit derselben zugrunde liegenden Redis-Datenbank mit denselben Anmeldeinformationen verbinden. Jede Warteschlange ist durch ihre `name`-Eigenschaft eindeutig. Ein Warteschlangenname wird sowohl als Injektionstoken (zum Injizieren der Warteschlange in Controller/Provider) als auch als Argument für Dekoratoren verwendet, um Verbraucherklassen und Listener mit Warteschlangen zu verknüpfen.

Sie können auch einige der vorkonfigurierten Optionen für eine bestimmte Warteschlange überschreiben, wie folgt:

```typescript
BullModule.registerQueue({
  name: 'audio',
  redis: {
    port: 6380,
  },
});
```

Da Jobs in Redis gespeichert werden, versucht jede spezifisch benannte Warteschlange, die instanziiert wird (z.B. wenn eine App gestartet/neugestartet wird), alle alten Jobs zu verarbeiten, die aus einer vorherigen unvollständigen Sitzung stammen können.

Jede Warteschlange kann einen oder mehrere Produzenten, Verbraucher und Listener haben. Verbraucher holen Jobs aus der Warteschlange in einer bestimmten Reihenfolge: FIFO (Standard), LIFO oder nach Prioritäten. Die Steuerung der Warteschlangenverarbeitungsreihenfolge wird [hier](https://docs.nestjs.com/techniques/queues#consumers) diskutiert.

### Benannte Konfigurationen / Named configurations

Wenn Ihre Warteschlangen eine Verbindung zu mehreren Redis-Instanzen herstellen, können Sie eine Technik namens benannte Konfigurationen verwenden. Diese Funktion ermöglicht es Ihnen, mehrere Konfigurationen unter angegebenen Schlüsseln zu registrieren, auf die Sie dann in den Warteschlangenoptionen verweisen können.

Beispielsweise, wenn Sie eine zusätzliche Redis-Instanz (neben der Standardinstanz) haben, die von einigen Warteschlangen in Ihrer Anwendung verwendet wird, können Sie ihre Konfiguration wie folgt registrieren:

```typescript
BullModule.forRoot('alternative-config', {
  redis: {
    port: 6381,
  },
});
```

Im obigen Beispiel ist `alternative-config` einfach ein Konfigurationsschlüssel (es kann eine beliebige Zeichenkette sein).

Mit dieser Einrichtung können Sie nun auf diese Konfiguration im Optionsobjekt der `registerQueue()`-Methode verweisen:

```typescript
BullModule.registerQueue({
  configKey: 'alternative-config',
  name: 'video'
});
```

### Produzenten / Producers

Job-Produzenten fügen Jobs zu Warteschlangen hinzu. Produzenten sind typischerweise Anwendungsdienste (Nest-Provider). Um Jobs zu einer Warteschlange hinzuzufügen, injizieren Sie zuerst die Warteschlange in den Dienst wie folgt:

```typescript
import { Injectable } from '@nestjs/common';
import { Queue } from 'bull';
import { InjectQueue } from '@nestjs/bull';

@Injectable()
export class AudioService {
  constructor(@InjectQueue('audio') private audioQueue: Queue) {}
}
```

**HINWEIS:** Der `@InjectQueue()`-Dekorator identifiziert die Warteschlange anhand ihres Namens, wie in der `registerQueue()`-Methode angegeben (z.B. 'audio').

Fügen Sie nun einen Job hinzu, indem Sie die `add()`-Methode der Warteschlange aufrufen und ein benutzerdefiniertes Job-Objekt übergeben. Jobs werden als serialisierbare JavaScript-Objekte dargestellt (da sie so in der Redis-Datenbank gespeichert werden). Die Form des übergebenen Jobs ist beliebig; verwenden Sie sie, um die Semantik Ihres Job-Objekts darzustellen.

```typescript
const job = await this.audioQueue.add({
  foo: 'bar',
});
```

### Benannte Jobs / Named jobs

Jobs können eindeutige Namen haben. Dies ermöglicht es Ihnen, spezialisierte Verbraucher zu erstellen, die nur Jobs mit einem bestimmten Namen verarbeiten.

```typescript
const job = await this.audioQueue.add('transcode', {
  foo: 'bar',
});
```

**WARNUNG:** Wenn Sie benannte Jobs verwenden, müssen Sie Prozessoren für jeden eindeutigen Namen erstellen, der zu einer Warteschlange hinzugefügt wird, sonst wird die Warteschlange melden, dass Ihnen ein Prozessor für den angegebenen Job fehlt. Weitere Informationen zum Verarbeiten benannter Jobs finden Sie [hier](https://docs.nestjs.com/techniques/queues#consumers).

### Job-Optionen / Job options

Jobs können zusätzliche Optionen haben. Übergeben Sie ein Optionsobjekt nach dem Job-Argument in der `Queue.add()`-Methode. Job-Options-Eigenschaften sind:

- `priority`: `number` - Optionaler Prioritätswert. Bereich von 1 (höchste Priorität) bis MAX_INT (niedrigste Priorität). Beachten Sie, dass die Verwendung von Prioritäten einen geringen Einfluss auf die Leistung hat, daher verwenden Sie sie mit Vorsicht.
- `delay`: `number` - Eine Zeitspanne (Millisekunden), die gewartet werden soll, bis dieser Job verarbeitet werden kann. Beachten

 Sie, dass für genaue Verzögerungen sowohl der Server als auch die Clients ihre Uhren synchronisieren sollten.
- `attempts`: `number` - Die Gesamtanzahl der Versuche, den Job abzuschließen.
- `repeat`: `RepeatOpts` - Wiederholungsjob gemäß einer Cron-Spezifikation. Weitere Informationen finden Sie unter `RepeatOpts`.
- `backoff`: `number | BackoffOpts` - Backoff-Einstellung für automatische Wiederholungen, falls der Job fehlschlägt. Weitere Informationen finden Sie unter `BackoffOpts`.
- `lifo`: `boolean` - Wenn `true`, fügt den Job an das rechte Ende der Warteschlange anstatt an das linke hinzu (Standard: `false`).
- `timeout`: `number` - Die Anzahl der Millisekunden, nach denen der Job mit einem Timeout-Fehler fehlschlagen soll.
- `jobId`: `number | string` - Überschreibt die Job-ID - standardmäßig ist die Job-ID eine eindeutige Ganzzahl, aber Sie können diese Einstellung verwenden, um sie zu überschreiben. Wenn Sie diese Option verwenden, liegt es an Ihnen, sicherzustellen, dass die Job-ID eindeutig ist. Wenn Sie versuchen, einen Job mit einer bereits vorhandenen ID hinzuzufügen, wird er nicht hinzugefügt.
- `removeOnComplete`: `boolean | number` - Wenn `true`, wird der Job nach erfolgreichem Abschluss entfernt. Eine Zahl gibt die Anzahl der zu behaltenden Jobs an. Standardmäßig wird der Job im abgeschlossenen Set behalten.
- `removeOnFail`: `boolean | number` - Wenn `true`, wird der Job nach allen Versuchen entfernt, wenn er fehlschlägt. Eine Zahl gibt die Anzahl der zu behaltenden Jobs an. Standardmäßig wird der Job im fehlgeschlagenen Set behalten.
- `stackTraceLimit`: `number` - Begrenzt die Anzahl der Stack-Trace-Zeilen, die im Stacktrace aufgezeichnet werden.

Hier sind einige Beispiele für die Anpassung von Jobs mit Job-Optionen:

Um den Start eines Jobs zu verzögern, verwenden Sie die `delay`-Konfigurationseigenschaft.

```typescript
const job = await this.audioQueue.add(
  {
    foo: 'bar',
  },
  { delay: 3000 }, // 3 Sekunden verzögert
);
```

Um einen Job an das rechte Ende der Warteschlange hinzuzufügen (den Job als LIFO (Last In First Out) zu verarbeiten), setzen Sie die `lifo`-Eigenschaft des Konfigurationsobjekts auf `true`.

```typescript
const job = await this.audioQueue.add(
  {
    foo: 'bar',
  },
  { lifo: true },
);
```

Um einen Job zu priorisieren, verwenden Sie die `priority`-Eigenschaft.

```typescript
const job = await this.audioQueue.add(
  {
    foo: 'bar',
  },
  { priority: 2 },
);
```

### Verbraucher / Consumers

Ein Verbraucher ist eine Klasse, die Methoden definiert, die entweder Jobs verarbeiten, die der Warteschlange hinzugefügt wurden, oder auf Ereignisse in der Warteschlange hören, oder beides. Deklarieren Sie eine Verbraucherklasse mithilfe des `@Processor()`-Dekorators wie folgt:

```typescript
import { Processor } from '@nestjs/bull';

@Processor('audio')
export class AudioConsumer {}
```

**HINWEIS:** Verbraucher müssen als Provider registriert werden, damit das `@nestjs/bull`-Paket sie erkennen kann.

Wo das Zeichenkettenargument des Dekorators (z.B. 'audio') der Name der Warteschlange ist, die der Klasse zugeordnet werden soll.

Deklarieren Sie innerhalb einer Verbraucherklasse Job-Handler, indem Sie Handler-Methoden mit dem `@Process()`-Dekorator versehen.

```typescript
import { Processor, Process } from '@nestjs/bull';
import { Job } from 'bull';

@Processor('audio')
export class AudioConsumer {
  @Process()
  async transcode(job: Job<unknown>) {
    let progress = 0;
    for (let i = 0; i < 100; i++) {
      await doSomething(job.data);
      progress += 1;
      await job.progress(progress);
    }
    return {};
  }
}
```

Die dekorierte Methode (z.B. `transcode()`) wird aufgerufen, wann immer der Worker im Leerlauf ist und es Jobs zur Verarbeitung in der Warteschlange gibt. Diese Handler-Methode erhält das Job-Objekt als einziges Argument. Der von der Handler-Methode zurückgegebene Wert wird im Job-Objekt gespeichert und kann später abgerufen werden, beispielsweise in einem Listener für das abgeschlossene Ereignis.

Job-Objekte haben mehrere Methoden, mit denen Sie ihren Zustand interagieren können. Zum Beispiel verwendet der obige Code die `progress()`-Methode, um den Fortschritt des Jobs zu aktualisieren. Weitere Informationen finden Sie in der vollständigen Job-Objekt-API-Referenz.

Sie können angeben, dass eine Job-Handler-Methode nur Jobs eines bestimmten Typs (Jobs mit einem bestimmten Namen) verarbeitet, indem Sie diesen Namen an den `@Process()`-Dekorator übergeben, wie unten gezeigt. Sie können mehrere `@Process()`-Handler in einer Verbraucherklasse haben, die jeweils einem Job-Typ (Namen) entsprechen. Wenn Sie benannte Jobs verwenden, stellen Sie sicher, dass Sie einen Handler für jeden Namen haben.

```typescript
@Process('transcode')
async transcode(job: Job<unknown>) { ... }
```

**WARNUNG:** Wenn Sie mehrere Verbraucher für dieselbe Warteschlange definieren, wird die `concurrency`-Option in `@Process({ concurrency: 1 })` keine Wirkung haben. Die minimale Parallelität entspricht der Anzahl der definierten Verbraucher. Dies gilt auch, wenn `@Process()`-Handler verschiedene Namen verwenden, um benannte Jobs zu verarbeiten.

### Anfragebereich-Verbraucher / Request-scoped consumers

Wenn ein Verbraucher als anfragebasiert gekennzeichnet ist (weitere Informationen zu Injektionsbereichen finden Sie [hier](https://docs.nestjs.com/fundamentals/injection-scopes#provider-scope)), wird für jeden Job eine neue Instanz der Klasse erstellt. Die Instanz wird nach Abschluss des Jobs vom Garbage Collector gesammelt.

```typescript
@Processor({
  name: 'audio',
  scope: Scope.REQUEST,
})
```

Da anfragebasierte Verbraucherklassen dynamisch instanziiert und auf einen einzelnen Job beschränkt sind, können Sie einen `JOB_REF` durch den Konstruktor mithilfe eines Standardansatzes injizieren.

```typescript
constructor(@Inject(JOB_REF) jobRef: Job) {
  console.log(jobRef);
}
```

**HINWEIS:** Der `JOB_REF`-Token wird aus dem Paket `@nestjs/bull` importiert.

### Ereignis-Listener / Event listeners

Bull generiert eine Reihe nützlicher Ereignisse, wenn Zustandsänderungen in der Warteschlange und/oder bei Jobs auftreten. Nest bietet eine Reihe von Dekoratoren, mit denen Sie sich auf eine Kernmenge von Standardereignissen abonnieren können. Diese werden aus dem Paket `@nestjs/bull` exportiert.

Ereignis-Listener müssen innerhalb einer Verbraucherklasse deklariert werden (d.h. innerhalb einer Klasse, die mit dem `@Processor()`-Dekorator versehen ist). Um auf ein Ereignis zu hören, verwenden Sie einen der unten stehenden Dekoratoren, um einen Handler für das Ereignis zu deklarieren. Um beispielsweise auf das Ereignis zu hören, das ausgelöst wird, wenn ein Job den aktiven Zustand in der `audio`-Warteschlange erreicht, verwenden Sie die folgende Konstruktion:

```typescript
import { Processor, Process, OnQueueActive } from '@nestjs/bull';
import { Job } from 'bull';

@Processor('audio')
export class AudioConsumer {

  @OnQueueActive()
  onActive(job: Job) {
    console.log(
      `Processing job ${job.id} of type ${job.name} with data ${job.data}...`,
    );
  }
  ...
```

Da Bull in einer verteilten (mehrknotigen) Umgebung arbeitet, definiert es das Konzept der Ereignis-Lokalität. Dieses Konzept erkennt an, dass Ereignisse entweder vollständig innerhalb eines einzelnen Prozesses oder auf geteilten Warteschlangen von verschiedenen Prozessen ausgelöst werden können. Ein lokales Ereignis ist eines, das ausgelöst wird, wenn eine Aktion oder Zustandsänderung in einer Warteschlange im lokalen Prozess ausgelöst wird. Mit anderen Worten, wenn Ihre Ereignisproduzenten und -verbraucher lokal in einem einzigen Prozess sind, sind alle Ereignisse, die auf Warteschlangen auftreten, lokal.

Wenn eine Warteschlange über mehrere Prozesse hinweg geteilt wird, stoßen wir auf die Möglichkeit globaler Ereignisse. Damit ein Listener in einem Prozess eine Ereignisbenachrichtigung erhält, die von einem anderen Prozess ausgelöst wurde, muss er sich für ein globales Ereignis registrieren.

Ereignis-Handler werden aufgerufen, wann immer das entsprechende Ereignis ausgelöst wird. Der Handler wird mit der in der folgenden Tabelle gezeigten Signatur aufgerufen und bietet Zugriff auf Informationen, die für das Ereignis relevant sind. Wir besprechen einen wichtigen Unterschied zwischen lokalen und globalen Ereignis-Handler-Signaturen unten.

| Lokale Ereignis-Listener            | Globale Ereignis-Listener        | Handler-Methode Signatur / Wann ausgelöst |
|-------------------------------------|----------------------------------|--------------------------------------------|
| @OnQueueError()                     | @OnGlobalQueueError()            | handler(error: Error) - Ein Fehler ist aufgetreten. `error` enthält den auslösenden Fehler. |
| @OnQueueWaiting()                  

 | @OnGlobalQueueWaiting()          | handler(jobId: number | string) - Ein Job wartet darauf, verarbeitet zu werden, sobald ein Worker im Leerlauf ist. `jobId` enthält die ID des Jobs, der diesen Zustand erreicht hat. |
| @OnQueueActive()                    | @OnGlobalQueueActive()           | handler(job: Job) - Job `job` hat begonnen. |
| @OnQueueStalled()                   | @OnGlobalQueueStalled()          | handler(job: Job) - Job `job` wurde als blockiert markiert. Dies ist nützlich zum Debuggen von Job-Workern, die abstürzen oder die Ereignisschleife anhalten. |
| @OnQueueProgress()                  | @OnGlobalQueueProgress()         | handler(job: Job, progress: number) - Der Fortschritt des Jobs `job` wurde auf `progress` aktualisiert. |
| @OnQueueCompleted()                 | @OnGlobalQueueCompleted()        | handler(job: Job, result: any) - Job `job` wurde erfolgreich mit dem Ergebnis `result` abgeschlossen. |
| @OnQueueFailed()                    | @OnGlobalQueueFailed()           | handler(job: Job, err: Error) - Job `job` ist mit dem Fehlergrund `err` fehlgeschlagen. |
| @OnQueuePaused()                    | @OnGlobalQueuePaused()           | handler() - Die Warteschlange wurde pausiert. |
| @OnQueueResumed()                   | @OnGlobalQueueResumed()          | handler(job: Job) - Die Warteschlange wurde fortgesetzt. |
| @OnQueueCleaned()                   | @OnGlobalQueueCleaned()          | handler(jobs: Job[], type: string) - Alte Jobs wurden aus der Warteschlange bereinigt. `jobs` ist ein Array bereinigter Jobs, und `type` ist der Typ der bereinigten Jobs. |
| @OnQueueDrained()                   | @OnGlobalQueueDrained()          | handler() - Wird ausgelöst, wenn die Warteschlange alle wartenden Jobs verarbeitet hat (auch wenn es noch verzögerte Jobs gibt, die noch nicht verarbeitet wurden). |
| @OnQueueRemoved()                   | @OnGlobalQueueRemoved()          | handler(job: Job) - Job `job` wurde erfolgreich entfernt. |

Wenn Sie auf globale Ereignisse hören, können die Methodensignaturen etwas anders sein als ihre lokalen Gegenstücke. Insbesondere erhält jede Methodensignatur, die in der lokalen Version Job-Objekte empfängt, stattdessen eine `jobId` (Nummer) in der globalen Version. Um eine Referenz auf das tatsächliche Job-Objekt in einem solchen Fall zu erhalten, verwenden Sie die Methode `Queue#getJob`. Dieser Aufruf sollte erwartet werden, und daher sollte der Handler als `async` deklariert werden. Beispiel:

```typescript
@OnGlobalQueueCompleted()
async onGlobalCompleted(jobId: number, result: any) {
  const job = await this.immediateQueue.getJob(jobId);
  console.log('(Global) on completed: job ', job.id, ' -> result: ', result);
}
```

**HINWEIS:** Um auf das `Queue`-Objekt zuzugreifen (um einen `getJob()`-Aufruf zu machen), müssen Sie es natürlich injizieren. Außerdem muss die Warteschlange in dem Modul registriert sein, in dem Sie sie injizieren.

Neben den spezifischen Ereignis-Listener-Dekoratoren können Sie auch den generischen `@OnQueueEvent()`-Dekorator in Kombination mit den Enums `BullQueueEvents` oder `BullQueueGlobalEvents` verwenden. Lesen Sie [hier](https://github.com/OptimalBits/bull/blob/master/REFERENCE.md#events) mehr über Ereignisse.

### Warteschlangenverwaltung / Queue management

Warteschlangen haben eine API, mit der Sie Verwaltungsfunktionen wie das Pausieren und Fortsetzen, das Abrufen der Anzahl von Jobs in verschiedenen Zuständen und mehrere weitere ausführen können. Die vollständige Warteschlangen-API finden Sie [hier](https://github.com/OptimalBits/bull/blob/master/REFERENCE.md#queue). Rufen Sie eine dieser Methoden direkt auf dem `Queue`-Objekt auf, wie unten mit den Pause/Fortsetzen-Beispielen gezeigt.

Pausieren Sie eine Warteschlange mit dem Aufruf der `pause()`-Methode. Eine pausierte Warteschlange wird keine neuen Jobs verarbeiten, bis sie fortgesetzt wird, aber aktuelle Jobs, die verarbeitet werden, werden weiter bearbeitet, bis sie abgeschlossen sind.

```typescript
await audioQueue.pause();
```

Um eine pausierte Warteschlange fortzusetzen, verwenden Sie die `resume()`-Methode wie folgt:

```typescript
await audioQueue.resume();
```

### Separate Prozesse / Separate processes

Job-Handler können auch in einem separaten (geforkten) Prozess ausgeführt werden (Quelle). Dies hat mehrere Vorteile:

- Der Prozess ist sandboxed, sodass er nicht den Worker beeinträchtigt, wenn er abstürzt.
- Sie können blockierenden Code ausführen, ohne die Warteschlange zu beeinträchtigen (Jobs werden nicht blockiert).
- Bessere Nutzung von Multi-Core-CPUs.
- Weniger Verbindungen zu Redis.

```typescript
import { Module } from '@nestjs/common';
import { BullModule } from '@nestjs/bull';
import { join } from 'path';

@Module({
  imports: [
    BullModule.registerQueue({
      name: 'audio',
      processors: [join(__dirname, 'processor.js')],
    }),
  ],
})
export class AppModule {}
```

Bitte beachten Sie, dass aufgrund der Ausführung Ihrer Funktion in einem geforkten Prozess Dependency Injection (und IoC-Container) nicht verfügbar sind. Das bedeutet, dass Ihre Prozessorfunktion alle Instanzen externer Abhängigkeiten, die sie benötigt, enthalten (oder erstellen) muss.

```typescript
import { Job, DoneCallback } from 'bull';

export default function (job: Job, cb: DoneCallback) {
  console.log(`[${process.pid}] ${JSON.stringify(job.data)}`);
  cb(null, 'It works');
}
```

### Asynchrone Konfiguration / Async configuration

Möglicherweise möchten Sie Bull-Optionen asynchron anstatt statisch übergeben. In diesem Fall verwenden Sie die Methode `forRootAsync()`, die mehrere Möglichkeiten zur Behandlung der asynchronen Konfiguration bietet. Ebenso verwenden Sie die Methode `registerQueueAsync()`, wenn Sie Warteschlangenoptionen asynchron übergeben möchten.

Ein Ansatz ist die Verwendung einer Fabrikfunktion:

```typescript
BullModule.forRootAsync({
  useFactory: () => ({
    redis: {
      host: 'localhost',
      port: 6379,
    },
  }),
});
```

Unsere Fabrik verhält sich wie jeder andere asynchrone Provider (z.B. kann sie async sein und sie kann Abhängigkeiten durch `inject` injizieren).

```typescript
BullModule.forRootAsync({
  imports: [ConfigModule],
  useFactory: async (configService: ConfigService) => ({
    redis: {
      host: configService.get('QUEUE_HOST'),
      port: configService.get('QUEUE_PORT'),
    },
  }),
  inject: [ConfigService],
});
```

Alternativ können Sie die `useClass`-Syntax verwenden:

```typescript
BullModule.forRootAsync({
  useClass: BullConfigService,
});
```

Die obige Konstruktion wird die `BullConfigService`-Klasse innerhalb des `BullModule` instanziieren und sie verwenden, um ein Optionsobjekt bereitzustellen, indem sie `createSharedConfiguration()` aufruft. Beachten Sie, dass dies bedeutet, dass die `BullConfigService`-Klasse das `SharedBullConfigurationFactory`-Interface implementieren muss, wie unten gezeigt:

```typescript
@Injectable()
class BullConfigService implements SharedBullConfigurationFactory {
  createSharedConfiguration(): BullModuleOptions {
    return {
      redis: {
        host: 'localhost',
        port: 6379,
      },
    };
  }
}
```

Um die Erstellung von `BullConfigService` innerhalb des `BullModule` zu verhindern und einen Provider zu verwenden, der aus einem anderen Modul importiert wird, können Sie die `useExisting`-Syntax verwenden.

```typescript
BullModule.forRootAsync({
  imports: [ConfigModule],
  useExisting: ConfigService,
});
```

Diese Konstruktion funktioniert genauso wie `useClass` mit einem entscheidenden Unterschied - `BullModule` wird importierte Module durchsuchen, um einen vorhandenen `ConfigService` wiederzuverwenden, anstatt einen neuen zu instanziieren.

### Beispiel / Example

Ein funktionierendes Beispiel finden Sie [hier](https://github.com/nestjs/nest/tree/master/sample/26-queues).

