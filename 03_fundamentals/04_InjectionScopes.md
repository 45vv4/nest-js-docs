# Injektionsbereiche / Injection scopes

Für Menschen aus unterschiedlichen Programmierhintergründen mag es überraschend sein zu erfahren, dass in Nest fast alles über eingehende Anfragen hinweg geteilt wird. Wir haben einen Verbindungspool zur Datenbank, Singleton-Dienste mit globalem Zustand usw. Denken Sie daran, dass Node.js nicht dem Multi-Threaded Stateless Model für Anfrage/Antwort folgt, bei dem jede Anfrage von einem separaten Thread verarbeitet wird. Daher ist die Verwendung von Singleton-Instanzen für unsere Anwendungen völlig sicher.

Es gibt jedoch Grenzfälle, in denen eine auf Anfragen basierende Lebensdauer das gewünschte Verhalten sein kann, zum Beispiel pro Anfrage-Caching in GraphQL-Anwendungen, Anfrageverfolgung und Multi-Tenancy. Injektionsbereiche bieten einen Mechanismus, um das gewünschte Lebensdauerverhalten von Providern zu erreichen.

## Provider-Bereich / Provider scope

Ein Provider kann einen der folgenden Bereiche haben:

- **DEFAULT**: Eine einzige Instanz des Providers wird in der gesamten Anwendung geteilt. Die Lebensdauer der Instanz ist direkt mit dem Lebenszyklus der Anwendung verknüpft. Sobald die Anwendung gestartet ist, wurden alle Singleton-Provider instanziiert. Singleton-Bereich wird standardmäßig verwendet.
- **REQUEST**: Eine neue Instanz des Providers wird exklusiv für jede eingehende Anfrage erstellt. Die Instanz wird nach Abschluss der Anfrage verarbeitet und vom Garbage Collector entfernt.
- **TRANSIENT**: Transiente Provider werden nicht über Verbraucher hinweg geteilt. Jeder Verbraucher, der einen transienten Provider injiziert, erhält eine neue, dedizierte Instanz.
- **TIPP**: Die Verwendung des Singleton-Bereichs wird für die meisten Anwendungsfälle empfohlen. Das Teilen von Providern über Verbraucher und Anfragen hinweg bedeutet, dass eine Instanz zwischengespeichert werden kann und ihre Initialisierung nur einmal, beim Start der Anwendung, erfolgt.

## Verwendung / Usage

Legen Sie den Injektionsbereich fest, indem Sie die Scope-Eigenschaft an das @Injectable() Dekorator-Optionsobjekt übergeben:

```typescript
import { Injectable, Scope } from '@nestjs/common';

@Injectable({ scope: Scope.REQUEST })
export class CatsService {}
```

Ähnlich für benutzerdefinierte Provider, setzen Sie die Scope-Eigenschaft in der ausführlichen Form einer Provider-Registrierung:

```typescript
{
  provide: 'CACHE_MANAGER',
  useClass: CacheManager,
  scope: Scope.TRANSIENT,
}
```

**TIPP**: Importieren Sie das Scope-Enum aus @nestjs/common. Singleton-Bereich wird standardmäßig verwendet und muss nicht deklariert werden. Wenn Sie einen Provider explizit als Singleton-Bereich deklarieren möchten, verwenden Sie den Wert Scope.DEFAULT für die Scope-Eigenschaft.

## Hinweis / NOTICE

Websocket-Gateways sollten keine request-scoped Provider verwenden, da sie als Singletons agieren müssen. Jedes Gateway kapselt einen echten Socket und kann nicht mehrfach instanziiert werden. Diese Einschränkung gilt auch für einige andere Provider, wie Passport-Strategien oder Cron-Controller.

## Controller-Bereich / Controller scope

Controller können ebenfalls einen Bereich haben, der für alle in diesem Controller deklarierten Anfragemethoden-Handler gilt. Wie beim Provider-Bereich erklärt der Bereich eines Controllers seine Lebensdauer. Für einen request-scoped Controller wird für jede eingehende Anfrage eine neue Instanz erstellt und nach Abschluss der Anfrage vom Garbage Collector entfernt.

Deklarieren Sie den Controller-Bereich mit der Scope-Eigenschaft des ControllerOptions-Objekts:

```typescript
@Controller({
  path: 'cats',
  scope: Scope.REQUEST,
})
export class CatsController {}
```

## Bereichshierarchie / Scope hierarchy

Der REQUEST-Bereich erstreckt sich die Injektionskette hinauf. Ein Controller, der von einem request-scoped Provider abhängt, wird selbst request-scoped.

Stellen Sie sich das folgende Abhängigkeitsdiagramm vor: CatsController <- CatsService <- CatsRepository. Wenn CatsService request-scoped ist (und die anderen sind standardmäßige Singletons), wird der CatsController request-scoped, da er vom injizierten Service abhängig ist. Das CatsRepository, das nicht abhängig ist, bleibt singleton-scoped.

Transiente Abhängigkeiten folgen diesem Muster nicht. Wenn ein singleton-scoped DogsService einen transienten LoggerService-Provider injiziert, erhält er eine frische Instanz davon. DogsService bleibt jedoch singleton-scoped, sodass das Injizieren von ihm überall nicht zu einer neuen Instanz von DogsService führen würde. Falls dieses Verhalten gewünscht ist, muss DogsService explizit als TRANSIENT gekennzeichnet werden.

## Anfrage-Provider / Request provider

In einer auf HTTP-Servern basierenden Anwendung (z.B. mit @nestjs/platform-express oder @nestjs/platform-fastify) möchten Sie möglicherweise eine Referenz auf das ursprüngliche Anforderungsobjekt erhalten, wenn Sie request-scoped Provider verwenden. Dies können Sie tun, indem Sie das REQUEST-Objekt injizieren.

Der REQUEST-Provider ist request-scoped, daher müssen Sie den REQUEST-Bereich in diesem Fall nicht explizit verwenden:

```typescript
import { Injectable, Scope, Inject } from '@nestjs/common';
import { REQUEST } from '@nestjs/core';
import { Request } from 'express';

@Injectable({ scope: Scope.REQUEST })
export class CatsService {
  constructor(@Inject(REQUEST) private request: Request) {}
}
```

Aufgrund unterschiedlicher Plattform-/Protokollunterschiede greifen Sie in Microservice- oder GraphQL-Anwendungen etwas anders auf die eingehende Anfrage zu. In GraphQL-Anwendungen injizieren Sie CONTEXT anstelle von REQUEST:

```typescript
import { Injectable, Scope, Inject } from '@nestjs/common';
import { CONTEXT } from '@nestjs/graphql';

@Injectable({ scope: Scope.REQUEST })
export class CatsService {
  constructor(@Inject(CONTEXT) private context) {}
}
```

Sie konfigurieren dann Ihren Kontextwert (im GraphQLModule), um die Anfrage als Eigenschaft zu enthalten.

## Inquirer-Provider / Inquirer provider

Wenn Sie die Klasse erhalten möchten, in der ein Provider erstellt wurde, beispielsweise in Logging- oder Metrik-Providern, können Sie das INQUIRER-Token injizieren:

```typescript
import { Inject, Injectable, Scope } from '@nestjs/common';
import { INQUIRER } from '@nestjs/core';

@Injectable({ scope: Scope.TRANSIENT })
export class HelloService {
  constructor(@Inject(INQUIRER) private parentClass: object) {}

  sayHello(message: string) {
    console.log(`${this.parentClass?.constructor?.name}: ${message}`);
  }
}
```

Und dann wie folgt verwenden:

```typescript
import { Injectable } from '@nestjs/common';
import { HelloService } from './hello.service';

@Injectable()
export class AppService {
  constructor(private helloService: HelloService) {}

  getRoot(): string {
    this.helloService.sayHello('My name is getRoot');

    return 'Hello world!';
  }
}
```

Im obigen Beispiel, wenn AppService#getRoot aufgerufen wird, wird "AppService: My name is getRoot" in die Konsole protokolliert.

## Leistung / Performance

Die Verwendung von request-scoped Providern wirkt sich auf die Anwendungsleistung aus. Obwohl Nest versucht, so viele Metadaten wie möglich zwischenzuspeichern, muss dennoch eine Instanz Ihrer Klasse bei jeder Anfrage erstellt werden. Dies wird daher Ihre durchschnittliche Antwortzeit und Ihr Gesamtbenchmark-Ergebnis verlangsamen. Sofern ein Provider nicht request-scoped sein muss, wird dringend empfohlen, den standardmäßigen Singleton-Bereich zu verwenden.

**TIPP**: Obwohl es sehr einschüchternd klingen mag, sollte eine gut gestaltete Anwendung, die request-scoped Provider verwendet, die Latenz nicht um mehr als ~5% verlangsamen.

## Dauerhafte Provider / Durable providers

Wie im obigen Abschnitt erwähnt, können request-scoped Provider zu erhöhter Latenz führen, da mindestens ein request-scoped Provider (in den Controller injiziert oder tiefer - in einen seiner Provider injiziert) den Controller ebenfalls request-scoped macht. Das bedeutet, dass er für jede einzelne Anfrage neu erstellt (instanziiert) werden muss (und anschließend vom Garbage Collector entfernt wird). Für sagen wir 30.000 Anfragen gleichzeitig, werden also 30.000 ephemere Instanzen des Controllers (und seiner request-scoped Provider) existieren.

Ein gemeinsamer Provider, von dem die meisten Provider abhängen (denken Sie an eine Datenbankverbindung oder einen Logger-Service), macht automatisch alle diese Provider ebenfalls request-scoped. Dies kann eine Herausforderung in Multi-Tenancy-Anwendungen darstellen, insbesondere für diejenigen, die einen zentralen request-scoped "Datenquellen"-Provider haben, der Header/Token aus dem Anforderungsobjekt erfasst und basierend auf dessen Werten die entsprechende Datenbankverbindung/Schemata (spezifisch für diesen Mandanten) abruft.

Angenommen, Sie haben eine Anwendung, die abwechselnd von 10 verschiedenen Kunden verwendet wird. Jeder Kunde hat seine eigene dedizierte Datenquelle, und Sie möchten sicherstellen, dass Kunde A niemals auf die Datenbank von Kunde B zugreifen kann. Eine Möglichkeit, dies zu erreichen, könnte darin bestehen, einen request-scoped "Datenquellen"-Provider zu deklarieren, der - basierend auf dem Anforderungsobjekt - bestimmt, wer der "aktuelle Kunde" ist und dessen entsprechende Datenbank abruft. Mit diesem Ansatz können Sie Ihre Anwendung

 in nur wenigen Minuten in eine Multi-Tenancy-Anwendung verwandeln. Ein großer Nachteil dieses Ansatzes ist jedoch, dass wahrscheinlich ein großer Teil Ihrer Anwendungskomponenten auf den "Datenquellen"-Provider angewiesen ist und somit implizit "request-scoped" wird, was zweifellos Auswirkungen auf die Leistung Ihrer Anwendung haben wird.

Aber was wäre, wenn wir eine bessere Lösung hätten? Da wir nur 10 Kunden haben, könnten wir nicht 10 individuelle DI-Unterbäume pro Kunde haben (anstatt jeden Baum pro Anfrage neu zu erstellen)? Wenn Ihre Provider nicht von einer Eigenschaft abhängen, die für jede aufeinanderfolgende Anfrage wirklich einzigartig ist (z.B. Anfrage-UUID), sondern es stattdessen spezifische Attribute gibt, die uns aggregieren (klassifizieren) lassen, gibt es keinen Grund, den DI-Unterbaum bei jeder eingehenden Anfrage neu zu erstellen.

Und genau hier kommen die dauerhaften Provider ins Spiel.

Bevor wir beginnen, Provider als dauerhaft zu kennzeichnen, müssen wir zuerst eine Strategie registrieren, die Nest anweist, welche "gemeinsamen Anfrageattribute" vorhanden sind, und eine Logik bereitstellen, die Anfragen gruppiert und sie ihren entsprechenden DI-Unterbäumen zuordnet:

```typescript
import {
  HostComponentInfo,
  ContextId,
  ContextIdFactory,
  ContextIdStrategy,
} from '@nestjs/core';
import { Request } from 'express';

const tenants = new Map<string, ContextId>();

export class AggregateByTenantContextIdStrategy implements ContextIdStrategy {
  attach(contextId: ContextId, request: Request) {
    const tenantId = request.headers['x-tenant-id'] as string;
    let tenantSubTreeId: ContextId;

    if (tenants.has(tenantId)) {
      tenantSubTreeId = tenants.get(tenantId);
    } else {
      tenantSubTreeId = ContextIdFactory.create();
      tenants.set(tenantId, tenantSubTreeId);
    }

    // Wenn der Baum nicht dauerhaft ist, geben Sie das ursprüngliche "contextId"-Objekt zurück
    return (info: HostComponentInfo) =>
      info.isTreeDurable ? tenantSubTreeId : contextId;
  }
}
```

**TIPP**: Ähnlich wie beim Anfragebereich steigt die Dauerhaftigkeit die Injektionskette hinauf. Das bedeutet, wenn A von B abhängt, das als dauerhaft gekennzeichnet ist, wird A implizit auch dauerhaft (es sei denn, die Dauerhaftigkeit ist für A explizit auf false gesetzt).
**WARNUNG**: Diese Strategie ist nicht ideal für Anwendungen mit einer großen Anzahl von Mandanten.
Der Wert, der von der attach-Methode zurückgegeben wird, weist Nest an, welcher Kontextidentifikator für einen gegebenen Host verwendet werden soll. In diesem Fall haben wir angegeben, dass der tenantSubTreeId anstelle des ursprünglichen, automatisch generierten contextId-Objekts verwendet werden soll, wenn die Host-Komponente (z.B. ein request-scoped Controller) als dauerhaft gekennzeichnet ist (Sie können unten lernen, wie man Provider als dauerhaft markiert). Außerdem würde im obigen Beispiel keine Nutzlast registriert werden (wobei Nutzlast = REQUEST/CONTEXT-Provider, der die "Wurzel" - das Elternteil des Unterbaums - darstellt).

Wenn Sie die Nutzlast für einen dauerhaften Baum registrieren möchten, verwenden Sie stattdessen die folgende Konstruktion:

```typescript
// Die Rückgabe der `AggregateByTenantContextIdStrategy#attach`-Methode:
return {
  resolve: (info: HostComponentInfo) =>
    info.isTreeDurable ? tenantSubTreeId : contextId,
  payload: { tenantId },
}
```

Wann immer Sie den REQUEST-Provider (oder CONTEXT für GraphQL-Anwendungen) mit @Inject(REQUEST)/@Inject(CONTEXT) injizieren, würde das Nutzlastobjekt injiziert werden (bestehend aus einer einzelnen Eigenschaft - tenantId in diesem Fall).

Mit dieser Strategie können Sie es irgendwo in Ihrem Code registrieren (da es ohnehin global gilt), zum Beispiel könnten Sie es in die main.ts-Datei einfügen:

```typescript
ContextIdFactory.apply(new AggregateByTenantContextIdStrategy());
```

**TIPP**: Die ContextIdFactory-Klasse wird aus dem @nestjs/core-Paket importiert. Solange die Registrierung erfolgt, bevor eine Anfrage Ihre Anwendung erreicht, funktioniert alles wie beabsichtigt.

Zuletzt, um einen regulären Provider in einen dauerhaften Provider zu verwandeln, setzen Sie einfach die durable-Eigenschaft auf true und ändern dessen Bereich auf Scope.REQUEST (nicht erforderlich, wenn der REQUEST-Bereich bereits in der Injektionskette ist):

```typescript
import { Injectable, Scope } from '@nestjs/common';

@Injectable({ scope: Scope.REQUEST, durable: true })
export class CatsService {}
```

Ähnlich für benutzerdefinierte Provider, setzen Sie die durable-Eigenschaft in der ausführlichen Form einer Provider-Registrierung:

```typescript
{
  provide: 'foobar',
  useFactory: () => { ... },
  scope: Scope.REQUEST,
  durable: true,
}
```
