# Lazy Loading von Modulen / Lazy loading modules

Standardmäßig werden Module eagerly geladen, was bedeutet, dass alle Module geladen werden, sobald die Anwendung geladen wird, unabhängig davon, ob sie sofort benötigt werden oder nicht. Während dies für die meisten Anwendungen in Ordnung ist, kann es zu einem Engpass für Apps/Worker werden, die in einer serverlosen Umgebung laufen, wo die Startlatenz ("Cold Start") entscheidend ist.

Lazy Loading kann dazu beitragen, die Bootstrap-Zeit zu verkürzen, indem nur Module geladen werden, die für den spezifischen Aufruf der serverlosen Funktion erforderlich sind. Darüber hinaus könnten Sie auch andere Module asynchron laden, sobald die serverlose Funktion "warm" ist, um die Bootstrap-Zeit für nachfolgende Aufrufe weiter zu verkürzen (deferred modules registration).

**TIPP**: Wenn Sie mit dem Angular-Framework vertraut sind, haben Sie möglicherweise schon den Begriff "lazy-loading modules" gehört. Beachten Sie, dass diese Technik in Nest funktional anders ist und daher als eine völlig andere Funktion betrachtet werden sollte, die ähnliche Namenskonventionen teilt.

**WARNUNG**: Beachten Sie, dass Lebenszyklus-Hook-Methoden in lazy geladenen Modulen und Diensten nicht aufgerufen werden.

## Erste Schritte / Getting started

Um Module bei Bedarf zu laden, stellt Nest die LazyModuleLoader-Klasse zur Verfügung, die auf normale Weise in eine Klasse injiziert werden kann:

```typescript
// cats.service.ts

@Injectable()
export class CatsService {
  constructor(private lazyModuleLoader: LazyModuleLoader) {}
}
```

**TIPP**: Die LazyModuleLoader-Klasse wird aus dem @nestjs/core-Paket importiert.

Alternativ können Sie eine Referenz auf den LazyModuleLoader-Provider aus Ihrer Anwendungs-Bootstrap-Datei (main.ts) abrufen, wie folgt:

```typescript
// "app" stellt eine Nest-Anwendungsinstanz dar
const lazyModuleLoader = app.get(LazyModuleLoader);
```

Damit können Sie nun jedes Modul mit der folgenden Konstruktion laden:

```typescript
const { LazyModule } = await import('./lazy.module');
const moduleRef = await this.lazyModuleLoader.load(() => LazyModule);
```

**TIPP**: "Lazy loaded" Module werden beim ersten Aufruf der LazyModuleLoader#load-Methode zwischengespeichert. Das bedeutet, dass jeder nachfolgende Versuch, LazyModule zu laden, sehr schnell sein wird und eine zwischengespeicherte Instanz zurückgibt, anstatt das Modul erneut zu laden.

```plaintext
Ladevorgang "LazyModule" Versuch: 1
Zeit: 2.379ms
Ladevorgang "LazyModule" Versuch: 2
Zeit: 0.294ms
Ladevorgang "LazyModule" Versuch: 3
Zeit: 0.303ms
```

Außerdem teilen sich "lazy loaded" Module denselben Modul-Graphen wie die beim Anwendungs-Bootstrap eagerly geladenen Module sowie alle später in Ihrer App registrierten lazy Module.

Wo lazy.module.ts eine TypeScript-Datei ist, die ein reguläres Nest-Modul exportiert (keine zusätzlichen Änderungen erforderlich).

Die LazyModuleLoader#load-Methode gibt die Modulreferenz (von LazyModule) zurück, die es Ihnen ermöglicht, die interne Liste der Provider zu durchsuchen und eine Referenz zu jedem Provider zu erhalten, indem dessen Injektions-Token als Suchschlüssel verwendet wird.

Zum Beispiel, nehmen wir an, wir haben ein LazyModule mit der folgenden Definition:

```typescript
@Module({
  providers: [LazyService],
  exports: [LazyService],
})
export class LazyModule {}
```

**TIPP**: Lazy geladenen Module können nicht als globale Module registriert werden, da dies einfach keinen Sinn ergibt (da sie lazy registriert werden, auf Abruf, wenn alle statisch registrierten Module bereits instanziiert wurden). Ebenso werden registrierte globale Enhancer (Guards/Interceptor usw.) auch nicht richtig funktionieren.

Damit könnten wir eine Referenz auf den LazyService-Provider wie folgt erhalten:

```typescript
const { LazyModule } = await import('./lazy.module');
const moduleRef = await this.lazyModuleLoader.load(() => LazyModule);

const { LazyService } = await import('./lazy.service');
const lazyService = moduleRef.get(LazyService);
```

**WARNUNG**: Wenn Sie Webpack verwenden, stellen Sie sicher, dass Sie Ihre tsconfig.json-Datei aktualisieren - setzen Sie compilerOptions.module auf "esnext" und fügen Sie die compilerOptions.moduleResolution-Eigenschaft mit dem Wert "node" hinzu:

```json
{
  "compilerOptions": {
    "module": "esnext",
    "moduleResolution": "node",
    ...
  }
}
```

Mit diesen Optionen können Sie die Code-Splitting-Funktion nutzen.

## Lazy Loading von Controllern, Gateways und Resolvern / Lazy loading controllers, gateways, and resolvers

Da Controller (oder Resolver in GraphQL-Anwendungen) in Nest Sätze von Routen/Pfaden/Themen (oder Abfragen/Mutationen) darstellen, können Sie diese nicht mit der LazyModuleLoader-Klasse lazy laden.

**WARNUNG**: Controller, Resolver und Gateways, die innerhalb von lazy geladenen Modulen registriert sind, verhalten sich nicht wie erwartet. Ebenso können Sie Middleware-Funktionen (durch Implementierung der MiddlewareConsumer-Schnittstelle) nicht auf Abruf registrieren.

Angenommen, Sie bauen eine REST-API (HTTP-Anwendung) mit einem Fastify-Treiber unter der Haube (unter Verwendung des @nestjs/platform-fastify-Pakets). Fastify erlaubt es nicht, Routen zu registrieren, nachdem die Anwendung bereit ist/erfolgreich auf Nachrichten hört. Das bedeutet, selbst wenn wir Routenabbildungen analysieren, die in den Controllern des Moduls registriert sind, wären alle lazy geladenen Routen nicht zugänglich, da es keine Möglichkeit gibt, sie zur Laufzeit zu registrieren.

Ebenso erfordern einige Transportstrategien, die wir als Teil des @nestjs/microservices-Pakets bereitstellen (einschließlich Kafka, gRPC oder RabbitMQ), das Abonnieren/Hören auf spezifische Themen/Kanäle, bevor die Verbindung hergestellt wird. Sobald Ihre Anwendung beginnt, auf Nachrichten zu hören, könnte das Framework nicht in der Lage sein, auf neue Themen zu abonnieren/hören.

Schließlich generiert das @nestjs/graphql-Paket mit dem Code-First-Ansatz das GraphQL-Schema automatisch on-the-fly basierend auf den Metadaten. Das bedeutet, dass alle Klassen vorher geladen werden müssen. Andernfalls wäre es nicht möglich, das entsprechende, gültige Schema zu erstellen.

## Häufige Anwendungsfälle / Common use-cases

Am häufigsten werden Sie lazy geladene Module in Situationen sehen, in denen Ihr Worker/Cron-Job/Lambda & serverlose Funktion/Webhook verschiedene Dienste (unterschiedliche Logik) basierend auf den Eingabeparametern (Routenpfad/Datum/Abfrageparameter usw.) auslösen muss. Andererseits macht das Lazy Loading von Modulen möglicherweise nicht viel Sinn für monolithische Anwendungen, bei denen die Startzeit eher irrelevant ist.

