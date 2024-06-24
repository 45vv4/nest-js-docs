# Aufgabenplanung / Task Scheduling

Aufgabenplanung ermöglicht es Ihnen, beliebigen Code (Methoden/Funktionen) zu einem festgelegten Datum/Uhrzeit, in wiederkehrenden Intervallen oder einmal nach einem bestimmten Intervall auszuführen. In der Linux-Welt wird dies oft durch Pakete wie cron auf Betriebssystemebene gehandhabt. Für Node.js-Apps gibt es mehrere Pakete, die eine cron-ähnliche Funktionalität nachahmen. Nest bietet das @nestjs/schedule-Paket, das sich in das beliebte Node.js cron-Paket integriert. In diesem Kapitel werden wir dieses Paket behandeln.

## Installation / Installation

Um es zu verwenden, installieren wir zunächst die erforderlichen Abhängigkeiten.

```bash
$ npm install --save @nestjs/schedule
```

Um die Aufgabenplanung zu aktivieren, importieren Sie das ScheduleModule in das Root AppModule und führen Sie die statische Methode forRoot() wie unten gezeigt aus:

```typescript
// app.module.ts
import { Module } from '@nestjs/common';
import { ScheduleModule } from '@nestjs/schedule';

@Module({
  imports: [
    ScheduleModule.forRoot()
  ],
})
export class AppModule {}
```

Der Aufruf von .forRoot() initialisiert den Scheduler und registriert alle deklarativen cron-Jobs, Timeouts und Intervalle, die in Ihrer App vorhanden sind. Die Registrierung erfolgt, wenn der onApplicationBootstrap-Lebenszyklus-Hook auftritt, wodurch sichergestellt wird, dass alle Module geladen wurden und geplante Jobs deklariert haben.

## Deklarative cron-Jobs / Declarative cron jobs

Ein cron-Job plant eine beliebige Funktion (Methodenaufruf) zur automatischen Ausführung. Cron-Jobs können ausgeführt werden:

- Einmalig zu einem festgelegten Datum/Uhrzeit.
- Wiederkehrend; wiederkehrende Jobs können zu einem festgelegten Zeitpunkt innerhalb eines bestimmten Intervalls ausgeführt werden (zum Beispiel einmal pro Stunde, einmal pro Woche, alle 5 Minuten).

Deklarieren Sie einen cron-Job mit dem @Cron()-Dekorator vor der Methodendefinition, die den auszuführenden Code enthält, wie folgt:

```typescript
import { Injectable, Logger } from '@nestjs/common';
import { Cron } from '@nestjs/schedule';

@Injectable()
export class TasksService {
  private readonly logger = new Logger(TasksService.name);

  @Cron('45 * * * * *')
  handleCron() {
    this.logger.debug('Aufgerufen, wenn die aktuelle Sekunde 45 ist');
  }
}
```

In diesem Beispiel wird die handleCron()-Methode jedes Mal aufgerufen, wenn die aktuelle Sekunde 45 ist. Mit anderen Worten, die Methode wird einmal pro Minute zur 45-Sekunden-Marke ausgeführt.

Der @Cron()-Dekorator unterstützt alle Standard-cron-Muster:

- Sternchen (z.B. *)
- Bereiche (z.B. 1-3,5)
- Schritte (z.B. */2)

Im obigen Beispiel haben wir 45 * * * * * an den Dekorator übergeben. Der folgende Schlüssel zeigt, wie jede Position in der cron-Musterzeichenfolge interpretiert wird:

```
* * * * * *
| | | | | |
| | | | | Wochentag
| | | | Monate
| | | Monatstag
| | Stunden
| Minuten
Sekunden (optional)
```

Einige Beispiel-cron-Muster sind:

- `* * * * * *`: jede Sekunde
- `45 * * * * *`: jede Minute, zur 45. Sekunde
- `0 10 * * * *`: jede Stunde, zu Beginn der 10. Minute
- `0 */30 9-17 * * *`: alle 30 Minuten zwischen 9 Uhr und 17 Uhr
- `0 30 11 * * 1-5`: Montag bis Freitag um 11:30 Uhr

Das @nestjs/schedule-Paket bietet ein praktisches Enum mit häufig verwendeten cron-Mustern. Sie können dieses Enum wie folgt verwenden:

```typescript
import { Injectable, Logger } from '@nestjs/common';
import { Cron, CronExpression } from '@nestjs/schedule';

@Injectable()
export class TasksService {
  private readonly logger = new Logger(TasksService.name);

  @Cron(CronExpression.EVERY_30_SECONDS)
  handleCron() {
    this.logger.debug('Alle 30 Sekunden aufgerufen');
  }
}
```

In diesem Beispiel wird die handleCron()-Methode alle 30 Sekunden aufgerufen.

Alternativ können Sie dem @Cron()-Dekorator ein JavaScript Date-Objekt übergeben. Dadurch wird der Job genau einmal zum angegebenen Datum ausgeführt.

**HINWEIS**
Verwenden Sie JavaScript-Datumsarithmetik, um Jobs relativ zum aktuellen Datum zu planen. Zum Beispiel, @Cron(new Date(Date.now() + 10 * 1000)) plant einen Job, der 10 Sekunden nach dem Start der App ausgeführt wird.

Sie können auch zusätzliche Optionen als zweiten Parameter an den @Cron()-Dekorator übergeben.

- **name**: Nützlich, um auf einen cron-Job zuzugreifen und ihn zu steuern, nachdem er deklariert wurde.
- **timeZone**: Geben Sie die Zeitzone für die Ausführung an. Dies ändert die tatsächliche Zeit relativ zu Ihrer Zeitzone. Wenn die Zeitzone ungültig ist, wird ein Fehler ausgelöst. Sie können alle verfügbaren Zeitzonen auf der Moment Timezone-Website überprüfen.
- **utcOffset**: Damit können Sie den Offset Ihrer Zeitzone angeben, anstatt den Parameter timeZone zu verwenden.
- **disabled**: Gibt an, ob der Job überhaupt ausgeführt wird.

```typescript
import { Injectable } from '@nestjs/common';
import { Cron, CronExpression } from '@nestjs/schedule';

@Injectable()
export class NotificationService {
  @Cron('* * 0 * * *', {
    name: 'notifications',
    timeZone: 'Europe/Paris',
  })
  triggerNotifications() {}
}
```

Sie können auf einen cron-Job zugreifen und ihn steuern, nachdem er deklariert wurde, oder einen cron-Job dynamisch erstellen (wobei sein cron-Muster zur Laufzeit definiert wird) mit der dynamischen API. Um auf einen deklarativen cron-Job über die API zuzugreifen, müssen Sie den Job mit einem Namen verknüpfen, indem Sie die Eigenschaft name in einem optionalen options-Objekt als zweiten Parameter des Dekorators übergeben.

## Deklarative Intervalle / Declarative intervals

Um zu deklarieren, dass eine Methode in einem (wiederkehrenden) festgelegten Intervall ausgeführt werden soll, setzen Sie den @Interval()-Dekorator vor die Methodendefinition. Übergeben Sie den Intervallwert als Zahl in Millisekunden an den Dekorator, wie unten gezeigt:

```typescript
@Interval(10000)
handleInterval() {
  this.logger.debug('Alle 10 Sekunden aufgerufen');
}
```

**HINWEIS**
Dieser Mechanismus verwendet die JavaScript setInterval()-Funktion im Hintergrund. Sie können auch einen cron-Job verwenden, um wiederkehrende Jobs zu planen.

Wenn Sie Ihr deklaratives Intervall von außerhalb der deklarierenden Klasse über die dynamische API steuern möchten, verknüpfen Sie das Intervall mit einem Namen, indem Sie die folgende Konstruktion verwenden:

```typescript
@Interval('notifications', 2500)
handleInterval() {}
```

Die dynamische API ermöglicht auch das Erstellen dynamischer Intervalle, bei denen die Eigenschaften des Intervalls zur Laufzeit definiert werden, sowie deren Auflistung und Löschung.

## Deklarative Timeouts / Declarative timeouts

Um zu deklarieren, dass eine Methode (einmalig) nach einem festgelegten Timeout ausgeführt werden soll, setzen Sie den @Timeout()-Dekorator vor die Methodendefinition. Übergeben Sie den relativen Zeitversatz (in Millisekunden) ab dem Start der Anwendung an den Dekorator, wie unten gezeigt:

```typescript
@Timeout(5000)
handleTimeout() {
  this.logger.debug('Einmal nach 5 Sekunden aufgerufen');
}
```

**HINWEIS**
Dieser Mechanismus verwendet die JavaScript setTimeout()-Funktion im Hintergrund.

Wenn Sie Ihr deklaratives Timeout von außerhalb der deklarierenden Klasse über die dynamische API steuern möchten, verknüpfen Sie das Timeout mit einem Namen, indem Sie die folgende Konstruktion verwenden:

```typescript
@Timeout('notifications', 2500)
handleTimeout() {}
```

Die dynamische API ermöglicht auch das Erstellen dynamischer Timeouts, bei denen die Eigenschaften des Timeouts zur Laufzeit definiert werden, sowie deren Auflistung und Löschung.

## Dynamische API des Schedule-Moduls / Dynamic schedule module API

Das @nestjs/schedule-Modul bietet eine dynamische API, die die Verwaltung deklarativer cron-Jobs, Timeouts und Intervalle ermöglicht. Die API ermöglicht auch das Erstellen und Verwalten dynamischer cron-Jobs, Timeouts und Intervalle, bei denen die Eigenschaften zur Laufzeit definiert werden.

## Dynamische cron-Jobs / Dynamic cron jobs

Erhalten Sie eine Referenz zu einer CronJob-Instanz nach Namen von überall in Ihrem Code mithilfe der SchedulerRegistry API. Zuerst injizieren Sie SchedulerRegistry mit standardmäßiger Konstruktorinjektion:

```typescript
constructor(private schedulerRegistry: SchedulerRegistry) {}
```

**HINWEIS**
Importieren Sie SchedulerRegistry aus dem @nestjs/schedule-Paket.

Verwenden Sie es dann in einer Klasse wie folgt. Angenommen, ein cron-Job wurde mit der folgenden Deklaration erstellt:

```typescript


@Cron('* * 8 * * *', {
  name: 'notifications',
})
triggerNotifications() {}
```

Greifen Sie mit folgendem Code auf diesen Job zu:

```typescript
const job = this.schedulerRegistry.getCronJob('notifications');

job.stop();
console.log(job.lastDate());
```

Die Methode getCronJob() gibt den benannten cron-Job zurück. Das zurückgegebene CronJob-Objekt hat die folgenden Methoden:

- **stop()** - Stoppt einen Job, der geplant ist.
- **start()** - Startet einen Job neu, der gestoppt wurde.
- **setTime(time: CronTime)** - Stoppt einen Job, legt eine neue Zeit fest und startet ihn dann neu.
- **lastDate()** - Gibt eine Datum/Zeit-Darstellung des Datums zurück, an dem die letzte Ausführung eines Jobs stattfand.
- **nextDate()** - Gibt eine Datum/Zeit-Darstellung des Datums zurück, an dem die nächste Ausführung eines Jobs geplant ist.
- **nextDates(count: number)** - Gibt ein Array (Größe count) von Datum/Zeit-Darstellungen für die nächsten Ausführungsdaten zurück. Count ist standardmäßig 0 und gibt ein leeres Array zurück.

**HINWEIS**
Verwenden Sie toJSDate() für Datum/Zeit-Objekte, um sie als JavaScript-Datum zu rendern, das diesem Datum/Zeit-Objekt entspricht.

Erstellen Sie einen neuen cron-Job dynamisch mit der Methode SchedulerRegistry#addCronJob wie folgt:

```typescript
addCronJob(name: string, seconds: string) {
  const job = new CronJob(`${seconds} * * * * *`, () => {
    this.logger.warn(`Zeit (${seconds}) für Job ${name} zum Ausführen!`);
  });

  this.schedulerRegistry.addCronJob(name, job);
  job.start();

  this.logger.warn(
    `Job ${name} hinzugefügt für jede Minute bei ${seconds} Sekunden!`,
  );
}
```

In diesem Code verwenden wir das CronJob-Objekt aus dem cron-Paket, um den cron-Job zu erstellen. Der CronJob-Konstruktor nimmt ein cron-Muster (genauso wie der @Cron()-Dekorator) als erstes Argument und einen Callback, der ausgeführt wird, wenn der cron-Timer auslöst, als zweites Argument. Die Methode SchedulerRegistry#addCronJob nimmt zwei Argumente: einen Namen für den CronJob und das CronJob-Objekt selbst.

**WARNUNG**
Denken Sie daran, SchedulerRegistry zu injizieren, bevor Sie darauf zugreifen. Importieren Sie CronJob aus dem cron-Paket.

Löschen Sie einen benannten cron-Job mit der Methode SchedulerRegistry#deleteCronJob wie folgt:

```typescript
deleteCron(name: string) {
  this.schedulerRegistry.deleteCronJob(name);
  this.logger.warn(`Job ${name} gelöscht!`);
}
```

Listen Sie alle cron-Jobs mit der Methode SchedulerRegistry#getCronJobs wie folgt auf:

```typescript
getCrons() {
  const jobs = this.schedulerRegistry.getCronJobs();
  jobs.forEach((value, key, map) => {
    let next;
    try {
      next = value.nextDate().toJSDate();
    } catch (e) {
      next = 'Fehler: nächstes Ausführungsdatum liegt in der Vergangenheit!';
    }
    this.logger.log(`Job: ${key} -> Nächstes: ${next}`);
  });
}
```

Die Methode getCronJobs() gibt eine Map zurück. In diesem Code durchlaufen wir die Map und versuchen, auf die Methode nextDate() jedes CronJob zuzugreifen. In der CronJob-API, wenn ein Job bereits ausgeführt wurde und kein zukünftiges Ausführungsdatum hat, wird eine Ausnahme ausgelöst.

## Dynamische Intervalle / Dynamic intervals

Erhalten Sie eine Referenz zu einem Intervall mit der Methode SchedulerRegistry#getInterval. Wie oben, injizieren Sie SchedulerRegistry mit standardmäßiger Konstruktorinjektion:

```typescript
constructor(private schedulerRegistry: SchedulerRegistry) {}
```

Und verwenden Sie es wie folgt:

```typescript
const interval = this.schedulerRegistry.getInterval('notifications');
clearInterval(interval);
```

Erstellen Sie ein neues Intervall dynamisch mit der Methode SchedulerRegistry#addInterval wie folgt:

```typescript
addInterval(name: string, milliseconds: number) {
  const callback = () => {
    this.logger.warn(`Intervall ${name} wird zur Zeit (${milliseconds}) ausgeführt!`);
  };

  const interval = setInterval(callback, milliseconds);
  this.schedulerRegistry.addInterval(name, interval);
}
```

In diesem Code erstellen wir ein Standard-JavaScript-Intervall und übergeben es dann an die Methode SchedulerRegistry#addInterval. Diese Methode nimmt zwei Argumente: einen Namen für das Intervall und das Intervall selbst.

Löschen Sie ein benanntes Intervall mit der Methode SchedulerRegistry#deleteInterval wie folgt:

```typescript
deleteInterval(name: string) {
  this.schedulerRegistry.deleteInterval(name);
  this.logger.warn(`Intervall ${name} gelöscht!`);
}
```

Listen Sie alle Intervalle mit der Methode SchedulerRegistry#getIntervals wie folgt auf:

```typescript
getIntervals() {
  const intervals = this.schedulerRegistry.getIntervals();
  intervals.forEach(key => this.logger.log(`Intervall: ${key}`));
}
```

## Dynamische Timeouts / Dynamic timeouts

Erhalten Sie eine Referenz zu einem Timeout mit der Methode SchedulerRegistry#getTimeout. Wie oben, injizieren Sie SchedulerRegistry mit standardmäßiger Konstruktorinjektion:

```typescript
constructor(private readonly schedulerRegistry: SchedulerRegistry) {}
```

Und verwenden Sie es wie folgt:

```typescript
const timeout = this.schedulerRegistry.getTimeout('notifications');
clearTimeout(timeout);
```

Erstellen Sie ein neues Timeout dynamisch mit der Methode SchedulerRegistry#addTimeout wie folgt:

```typescript
addTimeout(name: string, milliseconds: number) {
  const callback = () => {
    this.logger.warn(`Timeout ${name} wird nach (${milliseconds}) ausgeführt!`);
  };

  const timeout = setTimeout(callback, milliseconds);
  this.schedulerRegistry.addTimeout(name, timeout);
}
```

In diesem Code erstellen wir ein Standard-JavaScript-Timeout und übergeben es dann an die Methode SchedulerRegistry#addTimeout. Diese Methode nimmt zwei Argumente: einen Namen für das Timeout und das Timeout selbst.

Löschen Sie ein benanntes Timeout mit der Methode SchedulerRegistry#deleteTimeout wie folgt:

```typescript
deleteTimeout(name: string) {
  this.schedulerRegistry.deleteTimeout(name);
  this.logger.warn(`Timeout ${name} gelöscht!`);
}
```

Listen Sie alle Timeouts mit der Methode SchedulerRegistry#getTimeouts wie folgt auf:

```typescript
getTimeouts() {
  const timeouts = this.schedulerRegistry.getTimeouts();
  timeouts.forEach(key => this.logger.log(`Timeout: ${key}`));
}
```

## Beispiel / Example

Ein funktionierendes Beispiel ist [hier](https://github.com/nestjs/nest/tree/master/sample/27-scheduling) verfügbar.
