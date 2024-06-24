# Events / Ereignisse

Das Event Emitter Paket (@nestjs/event-emitter) bietet eine einfache Implementierung eines Beobachters, der es Ihnen ermöglicht, verschiedene Ereignisse zu abonnieren und darauf zu reagieren, die in Ihrer Anwendung auftreten. Ereignisse dienen als großartige Möglichkeit, verschiedene Aspekte Ihrer Anwendung zu entkoppeln, da ein einzelnes Ereignis mehrere Listener haben kann, die nicht voneinander abhängig sind.

EventEmitterModule verwendet intern das Paket eventemitter2.

## Erste Schritte / Getting started#

Zuerst das erforderliche Paket installieren:

```
$ npm i --save @nestjs/event-emitter
```

Sobald die Installation abgeschlossen ist, importieren Sie das EventEmitterModule in das Haupt-AppModule und führen Sie die statische Methode forRoot() wie unten gezeigt aus:

app.module.tsJS

```typescript
import { Module } from '@nestjs/common';
import { EventEmitterModule } from '@nestjs/event-emitter';

@Module({
  imports: [
    EventEmitterModule.forRoot()
  ],
})
export class AppModule {}
```

Der Aufruf von .forRoot() initialisiert den Event Emitter und registriert alle deklarativen Event Listener, die in Ihrer App existieren. Die Registrierung erfolgt, wenn der onApplicationBootstrap Lifecycle-Hook auftritt, um sicherzustellen, dass alle Module geladen wurden und alle geplanten Aufgaben deklariert sind.

Um die zugrunde liegende EventEmitter-Instanz zu konfigurieren, übergeben Sie das Konfigurationsobjekt an die .forRoot() Methode, wie folgt:

```typescript
EventEmitterModule.forRoot({
  // auf `true` setzen, um Platzhalter zu verwenden
  wildcard: false,
  // das Trennzeichen, das zur Segmentierung von Namensräumen verwendet wird
  delimiter: '.',
  // auf `true` setzen, wenn Sie das newListener-Ereignis auslösen möchten
  newListener: false,
  // auf `true` setzen, wenn Sie das removeListener-Ereignis auslösen möchten
  removeListener: false,
  // die maximale Anzahl von Listenern, die einem Ereignis zugewiesen werden können
  maxListeners: 10,
  // zeigt den Ereignisnamen in der Speicherleckmeldung an, wenn mehr als die maximale Anzahl von Listenern zugewiesen ist
  verboseMemoryLeak: false,
  // deaktiviert das Auslösen von uncaughtException, wenn ein Fehlerereignis ausgelöst wird und keine Listener vorhanden sind
  ignoreErrors: false,
});
```

## Ereignisse auslösen / Dispatching Events#

Um ein Ereignis auszulösen (d.h. zu feuern), injizieren Sie zunächst EventEmitter2 unter Verwendung der Standard-Konstruktorinjektion:

```typescript
constructor(private eventEmitter: EventEmitter2) {}
```

**HINWEIS**  
Importieren Sie EventEmitter2 aus dem @nestjs/event-emitter Paket.

Verwenden Sie es dann in einer Klasse wie folgt:

```typescript
this.eventEmitter.emit(
  'order.created',
  new OrderCreatedEvent({
    orderId: 1,
    payload: {},
  }),
);
```

## Auf Ereignisse hören / Listening to Events#

Um einen Event Listener zu deklarieren, dekorieren Sie eine Methode mit dem @OnEvent() Dekorator vor der Methodendefinition, die den auszuführenden Code enthält, wie folgt:

```typescript
@OnEvent('order.created')
handleOrderCreatedEvent(payload: OrderCreatedEvent) {
  // verarbeiten und behandeln Sie das "OrderCreatedEvent"-Ereignis
}
```

**WARNUNG**  
Ereignis-Abonnenten können nicht auf Anfrageebene sein.

Das erste Argument kann ein String oder Symbol für einen einfachen Event Emitter und ein String | Symbol | Array<String | Symbol> im Fall eines Platzhalteremitters sein.

Das zweite Argument (optional) ist ein Listener-Optionsobjekt wie folgt:

```typescript
export type OnEventOptions = OnOptions & {
  /**
   * Wenn "true", wird der gegebene Listener an den Anfang des Listener-Arrays eingefügt (anstatt am Ende).
   *
   * @see https://github.com/EventEmitter2/EventEmitter2#emitterprependlistenerevent-listener-options
   *
   * @default false
   */
  prependListener?: boolean;

  /**
   * Wenn "true", wird die onEvent Callback-Funktion keine Fehler werfen, während das Ereignis behandelt wird. Andernfalls, wenn "false", wird ein Fehler geworfen.
   * 
   * @default true
   */
  suppressErrors?: boolean;
};
```

**HINWEIS**  
Lesen Sie mehr über das OnOptions-Optionsobjekt von eventemitter2.

```typescript
@OnEvent('order.created', { async: true })
handleOrderCreatedEvent(payload: OrderCreatedEvent) {
  // verarbeiten und behandeln Sie das "OrderCreatedEvent"-Ereignis
}
```

Um Namensräume/Platzhalter zu verwenden, übergeben Sie die Platzhalteroption an die Methode EventEmitterModule#forRoot(). Wenn Namensräume/Platzhalter aktiviert sind, können Ereignisse entweder Strings (foo.bar), die durch ein Trennzeichen getrennt sind, oder Arrays (['foo', 'bar']) sein. Das Trennzeichen ist auch als Konfigurationseigenschaft (delimiter) konfigurierbar. Mit aktivierter Namensraumfunktion können Sie sich unter Verwendung eines Platzhalters auf Ereignisse abonnieren:

```typescript
@OnEvent('order.*')
handleOrderEvents(payload: OrderCreatedEvent | OrderRemovedEvent | OrderUpdatedEvent) {
  // verarbeiten und behandeln Sie ein Ereignis
}
```

Beachten Sie, dass ein solcher Platzhalter nur für einen Block gilt. Das Argument order.* wird zum Beispiel die Ereignisse order.created und order.shipped, aber nicht order.delayed.out_of_stock erfassen. Um auf solche Ereignisse zu hören, verwenden Sie das mehrstufige Platzhaltermuster (d.h. **), wie in der EventEmitter2-Dokumentation beschrieben.

Mit diesem Muster können Sie beispielsweise einen Event Listener erstellen, der alle Ereignisse erfasst:

```typescript
@OnEvent('**')
handleEverything(payload: any) {
  // verarbeiten und behandeln Sie ein Ereignis
}
```

**HINWEIS**  
Die EventEmitter2-Klasse bietet mehrere nützliche Methoden zur Interaktion mit Ereignissen, wie waitFor und onAny. Sie können [hier](https://github.com/EventEmitter2/EventEmitter2) mehr darüber lesen.

## Beispiel / Example#

Ein funktionierendes Beispiel finden Sie [hier](https://github.com/nestjs/nest/tree/master/sample/30-event-emitter).
