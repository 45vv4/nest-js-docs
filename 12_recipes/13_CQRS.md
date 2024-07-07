# CQRS / CQRS
Der Ablauf einfacher CRUD-Anwendungen (Erstellen, Lesen, Aktualisieren und Löschen) kann wie folgt beschrieben werden:

- Die Controllerschicht verarbeitet HTTP-Anfragen und delegiert Aufgaben an die Serviceschicht.
- Die Serviceschicht enthält die meisten Geschäftslogiken.
- Services verwenden Repositories / DAOs, um Entitäten zu ändern / zu speichern.
- Entitäten fungieren als Container für die Werte mit Setzern und Gettern.

Während dieses Muster für kleine und mittelgroße Anwendungen normalerweise ausreicht, ist es möglicherweise nicht die beste Wahl für größere, komplexere Anwendungen. In solchen Fällen kann das CQRS (Command and Query Responsibility Segregation) Modell geeigneter und skalierbarer sein (abhängig von den Anforderungen der Anwendung). Vorteile dieses Modells umfassen:

- Trennung der Verantwortlichkeiten: Das Modell trennt Lese- und Schreiboperationen in separate Modelle.
- Skalierbarkeit: Die Lese- und Schreiboperationen können unabhängig voneinander skaliert werden.
- Flexibilität: Das Modell ermöglicht die Verwendung verschiedener Datenspeicher für Lese- und Schreiboperationen.
- Leistung: Das Modell ermöglicht die Nutzung verschiedener Datenspeicher, die für Lese- und Schreiboperationen optimiert sind.

Um dieses Modell zu erleichtern, bietet Nest ein leichtgewichtiges CQRS-Modul. In diesem Kapitel wird beschrieben, wie man es verwendet.

## Installation / Installation
Zuerst das erforderliche Paket installieren:

```bash
$ npm install --save @nestjs/cqrs
```

## Befehle / Commands
Befehle werden verwendet, um den Zustand der Anwendung zu ändern. Sie sollten aufgabenbasiert und nicht datenorientiert sein. Wenn ein Befehl ausgeführt wird, wird er von einem entsprechenden Befehls-Handler bearbeitet. Der Handler ist für die Aktualisierung des Anwendungszustands verantwortlich.

```typescript
heroes-game.service.ts

@Injectable()
export class HeroesGameService {
  constructor(private commandBus: CommandBus) {}

  async killDragon(heroId: string, killDragonDto: KillDragonDto) {
    return this.commandBus.execute(
      new KillDragonCommand(heroId, killDragonDto.dragonId)
    );
  }
}
```

Im obigen Code-Snippet instanziieren wir die Klasse KillDragonCommand und übergeben sie an die Methode execute() des CommandBus. Dies ist die dargestellte Befehlsklasse:

```typescript
kill-dragon.command.ts

export class KillDragonCommand {
  constructor(
    public readonly heroId: string,
    public readonly dragonId: string,
  ) {}
}
```

Der CommandBus stellt einen Strom von Befehlen dar. Er ist für das Senden von Befehlen an die entsprechenden Handler verantwortlich. Die Methode execute() gibt ein Promise zurück, das den vom Handler zurückgegebenen Wert auflöst.

Erstellen wir einen Handler für den KillDragonCommand-Befehl.

```typescript
kill-dragon.handler.ts

@CommandHandler(KillDragonCommand)
export class KillDragonHandler implements ICommandHandler<KillDragonCommand> {
  constructor(private repository: HeroRepository) {}

  async execute(command: KillDragonCommand) {
    const { heroId, dragonId } = command;
    const hero = this.repository.findOneById(+heroId);

    hero.killEnemy(dragonId);
    await this.repository.persist(hero);
  }
}
```

Dieser Handler ruft die Hero-Entität aus dem Repository ab, ruft die Methode killEnemy() auf und speichert dann die Änderungen. Die Klasse KillDragonHandler implementiert das ICommandHandler-Interface, das die Implementierung der Methode execute() erfordert. Die Methode execute() empfängt das Befehlsobjekt als Argument.

## Abfragen / Queries
Abfragen werden verwendet, um Daten aus dem Anwendungszustand abzurufen. Sie sollten datenorientiert und nicht aufgabenbasiert sein. Wenn eine Abfrage ausgeführt wird, wird sie von einem entsprechenden Abfrage-Handler bearbeitet. Der Handler ist für das Abrufen der Daten verantwortlich.

Der QueryBus folgt demselben Muster wie der CommandBus. Abfrage-Handler sollten das IQueryHandler-Interface implementieren und mit dem @QueryHandler()-Dekorator annotiert werden.

## Ereignisse / Events
Ereignisse werden verwendet, um andere Teile der Anwendung über Änderungen im Anwendungszustand zu benachrichtigen. Sie werden von Modellen oder direkt über den EventBus gesendet. Wenn ein Ereignis gesendet wird, wird es von den entsprechenden Ereignis-Handlern bearbeitet. Handler können dann beispielsweise das Lesemodell aktualisieren.

Erstellen wir zu Demonstrationszwecken eine Ereignisklasse:

```typescript
hero-killed-dragon.event.ts

export class HeroKilledDragonEvent {
  constructor(
    public readonly heroId: string,
    public readonly dragonId: string,
  ) {}
}
```

Während Ereignisse direkt mit der Methode EventBus.publish() gesendet werden können, können wir sie auch vom Modell aus senden. Aktualisieren wir das Hero-Modell, um das Ereignis HeroKilledDragonEvent zu senden, wenn die Methode killEnemy() aufgerufen wird.

```typescript
hero.model.ts

export class Hero extends AggregateRoot {
  constructor(private id: string) {
    super();
  }

  killEnemy(enemyId: string) {
    // Geschäftslogik
    this.apply(new HeroKilledDragonEvent(this.id, enemyId));
  }
}
```

Die Methode apply() wird verwendet, um Ereignisse zu senden. Sie akzeptiert ein Ereignisobjekt als Argument. Da unser Modell jedoch nichts über den EventBus weiß, müssen wir es mit dem Modell verknüpfen. Das können wir mit der Klasse EventPublisher tun.

```typescript
kill-dragon.handler.ts

@CommandHandler(KillDragonCommand)
export class KillDragonHandler implements ICommandHandler<KillDragonCommand> {
  constructor(
    private repository: HeroRepository,
    private publisher: EventPublisher,
  ) {}

  async execute(command: KillDragonCommand) {
    const { heroId, dragonId } = command;
    const hero = this.publisher.mergeObjectContext(
      await this.repository.findOneById(+heroId),
    );
    hero.killEnemy(dragonId);
    hero.commit();
  }
}
```

Die Methode EventPublisher#mergeObjectContext fügt den Ereignispublisher in das bereitgestellte Objekt ein, was bedeutet, dass das Objekt jetzt in der Lage ist, Ereignisse an den Ereignisstrom zu senden.

Beachten Sie, dass wir in diesem Beispiel auch die Methode commit() auf dem Modell aufrufen. Diese Methode wird verwendet, um alle ausstehenden Ereignisse zu senden. Um Ereignisse automatisch zu senden, können wir die Eigenschaft autoCommit auf true setzen:

```typescript
export class Hero extends AggregateRoot {
  constructor(private id: string) {
    super();
    this.autoCommit = true;
  }
}
```

Falls wir den Ereignispublisher in ein nicht vorhandenes Objekt, sondern in eine Klasse einfügen möchten, können wir die Methode EventPublisher#mergeClassContext verwenden:

```typescript
const HeroModel = this.publisher.mergeClassContext(Hero);
const hero = new HeroModel('id'); // <-- HeroModel ist eine Klasse
```

Jetzt kann jede Instanz der HeroModel-Klasse Ereignisse senden, ohne die Methode mergeObjectContext() zu verwenden.

Zusätzlich können wir Ereignisse manuell mit EventBus senden:

```typescript
this.eventBus.publish(new HeroKilledDragonEvent());
```

HINWEIS
Der EventBus ist eine injizierbare Klasse.
Jedes Ereignis kann mehrere Ereignis-Handler haben.

```typescript
hero-killed-dragon.handler.ts

@EventsHandler(HeroKilledDragonEvent)
export class HeroKilledDragonHandler implements IEventHandler<HeroKilledDragonEvent> {
  constructor(private repository: HeroRepository) {}

  handle(event: HeroKilledDragonEvent) {
    // Geschäftslogik
  }
}
```

HINWEIS
Beachten Sie, dass Sie, wenn Sie Ereignis-Handler verwenden, den traditionellen HTTP-Webkontext verlassen.
Fehler in CommandHandlers können weiterhin von integrierten Ausnahmefiltern abgefangen werden.
Fehler in EventHandlers können nicht von Ausnahmefiltern abgefangen werden: Sie müssen sie manuell behandeln. Entweder durch ein einfaches try/catch, durch Verwendung von Sagas, indem ein kompensierendes Ereignis ausgelöst wird, oder durch eine andere Lösung Ihrer Wahl.
HTTP-Antworten in CommandHandlers können weiterhin an den Client gesendet werden.
HTTP-Antworten in EventHandlers können nicht gesendet werden. Wenn Sie Informationen an den Client senden möchten, könnten Sie WebSocket, SSE oder eine andere Lösung Ihrer Wahl verwenden.

## Sagas / Sagas
Eine Saga ist ein lang laufender Prozess, der auf Ereignisse hört und möglicherweise neue Befehle auslöst. Sie wird normalerweise verwendet, um komplexe Workflows in der Anwendung zu verwalten. Beispielsweise könnte eine Saga, wenn sich ein Benutzer registriert, auf das Ereignis UserRegisteredEvent hören und eine Willkommens-E-Mail an den Benutzer senden.

Sagas sind eine äußerst leistungsstarke Funktion. Eine einzelne Saga kann auf 1..* Ereignisse hören. Mithilfe der RxJS-Bibliothek können wir Ereignisströme filtern, abbilden, verzweigen und zusammenführen, um komplexe Workflows zu erstellen. Jede Saga gibt ein Observable zurück, das eine Befehlsinstanz produziert. Dieser Befehl wird dann asynchron vom CommandBus gesendet.

Erstellen wir eine Saga, die auf das Ereignis HeroKilledDragonEvent hört und den Befehl DropAncientItemCommand sendet.

```typescript


heroes-game.saga.ts

@Injectable()
export class HeroesGameSagas {
  @Saga()
  dragonKilled = (events$: Observable<any>): Observable<ICommand> => {
    return events$.pipe(
      ofType(HeroKilledDragonEvent),
      map((event) => new DropAncientItemCommand(event.heroId, fakeItemID)),
    );
  }
}
```

HINWEIS
Der Operator ofType und der Dekorator @Saga() werden aus dem Paket @nestjs/cqrs exportiert.
Der Dekorator @Saga() markiert die Methode als Saga. Das Argument events$ ist ein Observable-Strom aller Ereignisse. Der Operator ofType filtert den Strom nach dem angegebenen Ereignistyp. Der Operator map ordnet das Ereignis einer neuen Befehlsinstanz zu.

In diesem Beispiel ordnen wir das Ereignis HeroKilledDragonEvent dem Befehl DropAncientItemCommand zu. Der Befehl DropAncientItemCommand wird dann automatisch vom CommandBus gesendet.

## Einrichtung / Setup
Um das Ganze abzuschließen, müssen wir alle Befehls-Handler, Ereignis-Handler und Sagas im HeroesGameModule registrieren:

```typescript
heroes-game.module.ts

export const CommandHandlers = [KillDragonHandler, DropAncientItemHandler];
export const EventHandlers =  [HeroKilledDragonHandler, HeroFoundItemHandler];

@Module({
  imports: [CqrsModule],
  controllers: [HeroesGameController],
  providers: [
    HeroesGameService,
    HeroesGameSagas,
    ...CommandHandlers,
    ...EventHandlers,
    HeroRepository,
  ]
})
export class HeroesGameModule {}
```

## Nicht behandelte Ausnahmen / Unhandled exceptions
Ereignis-Handler werden asynchron ausgeführt. Das bedeutet, dass sie immer alle Ausnahmen behandeln sollten, um zu verhindern, dass die Anwendung in einen inkonsistenten Zustand gerät. Wenn jedoch eine Ausnahme nicht behandelt wird, erstellt der EventBus das Objekt UnhandledExceptionInfo und schiebt es in den Strom UnhandledExceptionBus. Dieser Strom ist ein Observable, das verwendet werden kann, um nicht behandelte Ausnahmen zu verarbeiten.

```typescript
private destroy$ = new Subject<void>();

constructor(private unhandledExceptionsBus: UnhandledExceptionBus) {
  this.unhandledExceptionsBus
    .pipe(takeUntil(this.destroy$))
    .subscribe((exceptionInfo) => {
      // Ausnahme hier behandeln
      // z.B. an externen Dienst senden, Prozess beenden oder ein neues Ereignis senden
    });
}

onModuleDestroy() {
  this.destroy$.next();
  this.destroy$.complete();
}
```

Um Ausnahmen herauszufiltern, können wir den Operator ofType wie folgt verwenden:

```typescript
this.unhandledExceptionsBus.pipe(takeUntil(this.destroy$), UnhandledExceptionBus.ofType(TransactionNotAllowedException)).subscribe((exceptionInfo) => {
  // Ausnahme hier behandeln
});
```

Dabei ist TransactionNotAllowedException die Ausnahme, die wir herausfiltern möchten.

Das Objekt UnhandledExceptionInfo enthält die folgenden Eigenschaften:

```typescript
export interface UnhandledExceptionInfo<Cause = IEvent | ICommand, Exception = any> {
  /**
   * Die ausgelöste Ausnahme.
   */
  exception: Exception;
  /**
   * Der Grund für die Ausnahme (Ereignis- oder Befehlsreferenz).
   */
  cause: Cause;
}
```

## Abonnieren aller Ereignisse / Subscribing to all events
CommandBus, QueryBus und EventBus sind alle Observables. Das bedeutet, dass wir den gesamten Strom abonnieren und beispielsweise alle Ereignisse verarbeiten können. Wir könnten beispielsweise alle Ereignisse in der Konsole protokollieren oder sie im Ereignisspeicher speichern.

```typescript
private destroy$ = new Subject<void>();

constructor(private eventBus: EventBus) {
  this.eventBus
    .pipe(takeUntil(this.destroy$))
    .subscribe((event) => {
      // Ereignisse in Datenbank speichern
    });
}

onModuleDestroy() {
  this.destroy$.next();
  this.destroy$.complete();
}
```

## Beispiel / Example
Ein funktionierendes Beispiel ist hier verfügbar.
