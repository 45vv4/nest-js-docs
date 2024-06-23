# Modulreferenz / Module reference

Nest stellt die ModuleRef-Klasse zur Verfügung, um die interne Liste der Provider zu durchsuchen und eine Referenz zu jedem Provider zu erhalten, indem dessen Injektions-Token als Suchschlüssel verwendet wird. Die ModuleRef-Klasse bietet auch eine Möglichkeit, sowohl statische als auch scoped Provider dynamisch zu instanziieren. ModuleRef kann auf normale Weise in eine Klasse injiziert werden:

```typescript
// cats.service.ts

@Injectable()
export class CatsService {
  constructor(private moduleRef: ModuleRef) {}
}
```

**TIPP**: Die ModuleRef-Klasse wird aus dem @nestjs/core-Paket importiert.

## Instanzen abrufen / Retrieving instances

Die ModuleRef-Instanz (im Folgenden als Modulreferenz bezeichnet) hat eine get()-Methode. Diese Methode ruft einen Provider, Controller oder Injectable (z.B. Guard, Interceptor usw.) ab, der im aktuellen Modul existiert (instanziiert wurde), indem dessen Injektions-Token/Klassenname verwendet wird.

```typescript
// cats.service.ts

@Injectable()
export class CatsService implements OnModuleInit {
  private service: Service;
  constructor(private moduleRef: ModuleRef) {}

  onModuleInit() {
    this.service = this.moduleRef.get(Service);
  }
}
```

**WARNUNG**: Sie können keine scoped Provider (transient oder request-scoped) mit der get()-Methode abrufen. Verwenden Sie stattdessen die unten beschriebene Technik. Erfahren Sie hier, wie Sie Bereiche steuern können.

Um einen Provider aus dem globalen Kontext abzurufen (z.B. wenn der Provider in einem anderen Modul injiziert wurde), übergeben Sie die Option { strict: false } als zweites Argument an get():

```typescript
this.moduleRef.get(Service, { strict: false });
```

## Scoped Provider auflösen / Resolving scoped providers

Um einen scoped Provider (transient oder request-scoped) dynamisch aufzulösen, verwenden Sie die resolve()-Methode und übergeben das Injektions-Token des Providers als Argument.

```typescript
// cats.service.ts

@Injectable()
export class CatsService implements OnModuleInit {
  private transientService: TransientService;
  constructor(private moduleRef: ModuleRef) {}

  async onModuleInit() {
    this.transientService = await this.moduleRef.resolve(TransientService);
  }
}
```

Die resolve()-Methode gibt eine eindeutige Instanz des Providers aus dessen eigenem DI-Container-Unterbaum zurück. Jeder Unterbaum hat einen eindeutigen Kontextbezeichner. Wenn Sie diese Methode mehrmals aufrufen und Instanzreferenzen vergleichen, werden Sie sehen, dass sie nicht gleich sind.

```typescript
// cats.service.ts

@Injectable()
export class CatsService implements OnModuleInit {
  constructor(private moduleRef: ModuleRef) {}

  async onModuleInit() {
    const transientServices = await Promise.all([
      this.moduleRef.resolve(TransientService),
      this.moduleRef.resolve(TransientService),
    ]);
    console.log(transientServices[0] === transientServices[1]); // false
  }
}
```

Um eine einzige Instanz über mehrere resolve()-Aufrufe hinweg zu generieren und sicherzustellen, dass sie denselben generierten DI-Container-Unterbaum teilen, können Sie einen Kontextbezeichner an die resolve()-Methode übergeben. Verwenden Sie die ContextIdFactory-Klasse, um einen Kontextbezeichner zu generieren. Diese Klasse bietet eine create()-Methode, die einen geeigneten eindeutigen Bezeichner zurückgibt.

```typescript
// cats.service.ts

@Injectable()
export class CatsService implements OnModuleInit {
  constructor(private moduleRef: ModuleRef) {}

  async onModuleInit() {
    const contextId = ContextIdFactory.create();
    const transientServices = await Promise.all([
      this.moduleRef.resolve(TransientService, contextId),
      this.moduleRef.resolve(TransientService, contextId),
    ]);
    console.log(transientServices[0] === transientServices[1]); // true
  }
}
```

**TIPP**: Die ContextIdFactory-Klasse wird aus dem @nestjs/core-Paket importiert.

## REQUEST-Provider registrieren / Registering REQUEST provider

Manuell generierte Kontextbezeichner (mit ContextIdFactory.create()) repräsentieren DI-Unterbäume, in denen der REQUEST-Provider undefiniert ist, da sie nicht vom Nest-Abhängigkeitsinjektionssystem instanziiert und verwaltet werden.

Um ein benutzerdefiniertes REQUEST-Objekt für einen manuell erstellten DI-Unterbaum zu registrieren, verwenden Sie die ModuleRef#registerRequestByContextId()-Methode wie folgt:

```typescript
const contextId = ContextIdFactory.create();
this.moduleRef.registerRequestByContextId(/* IHR_REQUEST_OBJEKT */, contextId);
```

## Aktuellen Unterbaum abrufen / Getting current sub-tree

Gelegentlich möchten Sie eine Instanz eines request-scoped Providers innerhalb eines Anfragekontexts auflösen. Angenommen, CatsService ist request-scoped und Sie möchten die CatsRepository-Instanz auflösen, die ebenfalls als request-scoped Provider markiert ist. Um denselben DI-Container-Unterbaum zu teilen, müssen Sie den aktuellen Kontextbezeichner anstelle eines neuen (z.B. mit der ContextIdFactory.create()-Funktion, wie oben gezeigt) erhalten. Um den aktuellen Kontextbezeichner zu erhalten, beginnen Sie, indem Sie das Anforderungsobjekt mit dem @Inject()-Dekorator injizieren.

```typescript
// cats.service.ts

@Injectable()
export class CatsService {
  constructor(
    @Inject(REQUEST) private request: Record<string, unknown>,
  ) {}
}
```

**TIPP**: Erfahren Sie hier mehr über den request-Provider.

Nun verwenden Sie die getByRequest()-Methode der ContextIdFactory-Klasse, um eine Kontext-ID basierend auf dem Anforderungsobjekt zu erstellen und diese an den resolve()-Aufruf zu übergeben:

```typescript
const contextId = ContextIdFactory.getByRequest(this.request);
const catsRepository = await this.moduleRef.resolve(CatsRepository, contextId);
```

## Benutzerdefinierte Klassen dynamisch instanziieren / Instantiating custom classes dynamically

Um eine Klasse, die zuvor nicht als Provider registriert wurde, dynamisch zu instanziieren, verwenden Sie die create()-Methode der Modulreferenz.

```typescript
// cats.service.ts

@Injectable()
export class CatsService implements OnModuleInit {
  private catsFactory: CatsFactory;
  constructor(private moduleRef: ModuleRef) {}

  async onModuleInit() {
    this.catsFactory = await this.moduleRef.create(CatsFactory);
  }
}
```

Diese Technik ermöglicht es Ihnen, verschiedene Klassen bedingt außerhalb des Framework-Containers zu instanziieren.
