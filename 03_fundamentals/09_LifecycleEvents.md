# Lebenszyklus-Ereignisse / Lifecycle Events

Eine Nest-Anwendung, ebenso wie jedes Anwendungselement, hat einen Lebenszyklus, der von Nest verwaltet wird. Nest bietet Lebenszyklus-Hooks, die Einblick in wichtige Lebenszyklus-Ereignisse geben und die Möglichkeit bieten, bei deren Auftreten (registrierten Code auf Ihren Modulen, Providern oder Controllern ausführen) zu handeln.

## Lebenszyklus-Sequenz / Lifecycle sequence

Das folgende Diagramm zeigt die Sequenz der wichtigsten Anwendungs-Lebenszyklus-Ereignisse, vom Zeitpunkt des Anwendungsstarts bis zum Beenden des Node-Prozesses. Wir können den gesamten Lebenszyklus in drei Phasen unterteilen: Initialisierung, Ausführung und Beendigung. Mithilfe dieses Lebenszyklus können Sie die geeignete Initialisierung von Modulen und Diensten planen, aktive Verbindungen verwalten und Ihre Anwendung ordnungsgemäß herunterfahren, wenn sie ein Beendigungssignal erhält.

![lifecycle-events](lifecycle-events.png/)

## Lebenszyklus-Ereignisse / Lifecycle events

Lebenszyklus-Ereignisse treten während des Anwendungsstarts und -shutdowns auf. Nest ruft registrierte Lebenszyklus-Hook-Methoden auf Modulen, Providern und Controllern bei jedem der folgenden Lebenszyklus-Ereignisse auf (Shutdown-Hooks müssen zuerst aktiviert werden, wie unten beschrieben). Wie im obigen Diagramm gezeigt, ruft Nest auch die entsprechenden zugrunde liegenden Methoden auf, um das Lauschen auf Verbindungen zu beginnen und das Lauschen auf Verbindungen zu beenden.

In der folgenden Tabelle werden onModuleDestroy, beforeApplicationShutdown und onApplicationShutdown nur ausgelöst, wenn Sie app.close() explizit aufrufen oder wenn der Prozess ein spezielles Systemsignal (wie SIGTERM) erhält und Sie enableShutdownHooks beim Anwendungsstart korrekt aufgerufen haben (siehe unten Anwendungsshutdown).

| Lifecycle-Hook-Methode           | Lebenszyklus-Ereignis, das den Hook-Methodenaufruf auslöst                   |
|----------------------------------|--------------------------------------------------------------------------------|
| onModuleInit()                   | Aufgerufen, sobald die Abhängigkeiten des Hostmoduls aufgelöst wurden.         |
| onApplicationBootstrap()         | Aufgerufen, sobald alle Module initialisiert wurden, aber bevor auf Verbindungen gelauscht wird. |
| onModuleDestroy()*               | Aufgerufen, nachdem ein Beendigungssignal (z.B. SIGTERM) empfangen wurde.      |
| beforeApplicationShutdown()*     | Aufgerufen, nachdem alle onModuleDestroy() Handler abgeschlossen sind (Promises aufgelöst oder abgelehnt); nach Abschluss (Promises aufgelöst oder abgelehnt) werden alle bestehenden Verbindungen geschlossen (app.close() aufgerufen). |
| onApplicationShutdown()*         | Aufgerufen, nachdem Verbindungen geschlossen wurden (app.close() wird aufgelöst). |

* Für diese Ereignisse, wenn Sie nicht explizit app.close() aufrufen, müssen Sie sich entscheiden, damit sie mit Systemsignalen wie SIGTERM funktionieren. Siehe unten Anwendungsshutdown.

**WARNUNG**: Die oben aufgeführten Lebenszyklus-Hooks werden nicht für request-scoped Klassen ausgelöst. Request-scoped Klassen sind nicht an den Anwendungslebenszyklus gebunden und ihre Lebensdauer ist unvorhersehbar. Sie werden ausschließlich für jede Anfrage erstellt und automatisch nach dem Senden der Antwort vom Garbage Collector entfernt.

**TIPP**: Die Ausführungsreihenfolge von onModuleInit() und onApplicationBootstrap() hängt direkt von der Reihenfolge der Modulimporte ab und wartet auf den vorherigen Hook.

## Verwendung / Usage

Jeder Lebenszyklus-Hook wird durch eine Schnittstelle dargestellt. Schnittstellen sind technisch optional, da sie nach der TypeScript-Kompilierung nicht existieren. Dennoch ist es eine gute Praxis, sie zu verwenden, um von starkem Typing und Editor-Tools zu profitieren. Um einen Lebenszyklus-Hook zu registrieren, implementieren Sie die entsprechende Schnittstelle. Zum Beispiel, um eine Methode zu registrieren, die während der Modulinitialisierung auf einer bestimmten Klasse (z.B. Controller, Provider oder Modul) aufgerufen wird, implementieren Sie die OnModuleInit-Schnittstelle, indem Sie eine onModuleInit()-Methode bereitstellen, wie unten gezeigt:

```typescript
import { Injectable, OnModuleInit } from '@nestjs/common';

@Injectable()
export class UsersService implements OnModuleInit {
  onModuleInit() {
    console.log(`Das Modul wurde initialisiert.`);
  }
}
```

## Asynchrone Initialisierung / Asynchronous initialization

Sowohl die OnModuleInit- als auch die OnApplicationBootstrap-Hooks ermöglichen es Ihnen, den Initialisierungsprozess der Anwendung zu verzögern (eine Promise zurückgeben oder die Methode als async markieren und auf den Abschluss einer asynchronen Methode im Methodenkörper warten).

```typescript
async onModuleInit(): Promise<void> {
  await this.fetch();
}
```

## Anwendungsshutdown / Application shutdown

Die Hooks onModuleDestroy(), beforeApplicationShutdown() und onApplicationShutdown() werden in der Beendigungsphase aufgerufen (als Reaktion auf einen expliziten Aufruf von app.close() oder beim Empfang von Systemsignalen wie SIGTERM, wenn Sie sich dafür entschieden haben). Diese Funktion wird häufig mit Kubernetes verwendet, um Container-Lebenszyklen zu verwalten, von Heroku für Dynos oder ähnliche Dienste.

Shutdown-Hook-Listener verbrauchen Systemressourcen und sind daher standardmäßig deaktiviert. Um Shutdown-Hooks zu verwenden, müssen Sie Listener aktivieren, indem Sie enableShutdownHooks() aufrufen:

```typescript
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Beginnt, auf Shutdown-Hooks zu lauschen
  app.enableShutdownHooks();

  await app.listen(3000);
}
bootstrap();
```

**WARNUNG**: Aufgrund inhärenter Plattformbeschränkungen hat NestJS nur begrenzte Unterstützung für Anwendungsshutdown-Hooks unter Windows. Sie können erwarten, dass SIGINT funktioniert, ebenso wie SIGBREAK und bis zu einem gewissen Grad SIGHUP - mehr lesen. SIGTERM wird jedoch niemals unter Windows funktionieren, da das Beenden eines Prozesses im Task-Manager bedingungslos ist, "d.h., es gibt keine Möglichkeit für eine Anwendung, dies zu erkennen oder zu verhindern". Hier sind einige relevante Dokumentationen von libuv, um mehr darüber zu erfahren, wie SIGINT, SIGBREAK und andere unter Windows gehandhabt werden. Siehe auch die Node.js-Dokumentation zu Prozesssignalereignissen.

**INFO**: enableShutdownHooks verbraucht Speicher, indem Listener gestartet werden. In Fällen, in denen Sie mehrere Nest-Apps in einem einzigen Node-Prozess ausführen (z.B. bei parallelen Tests mit Jest), kann Node über übermäßige Listener-Prozesse klagen. Aus diesem Grund ist enableShutdownHooks standardmäßig nicht aktiviert. Beachten Sie diese Bedingung, wenn Sie mehrere Instanzen in einem einzigen Node-Prozess ausführen.

Wenn die Anwendung ein Beendigungssignal erhält, werden alle registrierten onModuleDestroy(), beforeApplicationShutdown() und dann onApplicationShutdown()-Methoden (in der oben beschriebenen Reihenfolge) mit dem entsprechenden Signal als erstem Parameter aufgerufen. Wenn eine registrierte Funktion auf einen asynchronen Aufruf wartet (eine Promise zurückgibt), wird Nest die Sequenz nicht fortsetzen, bis die Promise aufgelöst oder abgelehnt wird.

```typescript
@Injectable()
class UsersService implements OnApplicationShutdown {
  onApplicationShutdown(signal: string) {
    console.log(signal); // z.B. "SIGINT"
  }
}
```

**INFO**: Der Aufruf von app.close() beendet den Node-Prozess nicht, sondern löst nur die Hooks onModuleDestroy() und onApplicationShutdown() aus, sodass der Prozess nicht automatisch beendet wird, wenn es einige Intervalle, langlaufende Hintergrundaufgaben usw. gibt.
