
# Ausführungskontext / Execution context

Nest bietet mehrere Hilfsklassen, die es einfach machen, Anwendungen zu schreiben, die in verschiedenen Anwendungs-Kontexten funktionieren (z.B. Nest HTTP-Server-basiert, Microservices und WebSockets-Anwendungs-Kontexte). Diese Hilfsmittel liefern Informationen über den aktuellen Ausführungskontext, die verwendet werden können, um generische Guards, Filter und Interceptor zu erstellen, die in einer breiten Palette von Controllern, Methoden und Ausführungskontexten funktionieren.

In diesem Kapitel behandeln wir zwei solcher Klassen: ArgumentsHost und ExecutionContext.

## ArgumentsHost-Klasse / ArgumentsHost class

Die ArgumentsHost-Klasse bietet Methoden zum Abrufen der an einen Handler übergebenen Argumente. Sie ermöglicht die Wahl des entsprechenden Kontexts (z.B. HTTP, RPC (Microservice) oder WebSockets), um die Argumente abzurufen. Das Framework stellt eine Instanz von ArgumentsHost zur Verfügung, die typischerweise als Host-Parameter bezeichnet wird, an Stellen, an denen Sie darauf zugreifen möchten. Beispielsweise wird die catch()-Methode eines Exception-Filters mit einer ArgumentsHost-Instanz aufgerufen.

ArgumentsHost fungiert einfach als Abstraktion über die Argumente eines Handlers. Beispielsweise kapselt das Host-Objekt für HTTP-Server-Anwendungen (wenn @nestjs/platform-express verwendet wird) das Express-Array [request, response, next], wobei request das Anforderungsobjekt, response das Antwortobjekt und next eine Funktion ist, die den Anforderungs-Antwort-Zyklus der Anwendung steuert. Für GraphQL-Anwendungen enthält das Host-Objekt hingegen das Array [root, args, context, info].

## Aktueller Anwendungs-Kontext / Current application context

Beim Erstellen von generischen Guards, Filtern und Interceptoren, die in mehreren Anwendungs-Kontexten ausgeführt werden sollen, benötigen wir eine Möglichkeit, den Typ der Anwendung zu bestimmen, in der unsere Methode gerade ausgeführt wird. Dies kann mit der getType()-Methode von ArgumentsHost erfolgen:

```typescript
if (host.getType() === 'http') {
  // Führen Sie etwas aus, das nur im Kontext von regulären HTTP-Anfragen (REST) wichtig ist
} else if (host.getType() === 'rpc') {
  // Führen Sie etwas aus, das nur im Kontext von Microservice-Anfragen wichtig ist
} else if (host.getType<GqlContextType>() === 'graphql') {
  // Führen Sie etwas aus, das nur im Kontext von GraphQL-Anfragen wichtig ist
}
```

**TIPP**: Der GqlContextType wird aus dem @nestjs/graphql-Paket importiert.

Mit dem Anwendungstyp verfügbar, können wir generischere Komponenten schreiben, wie unten gezeigt.

## Host-Handler-Argumente / Host handler arguments

Um das Array der an den Handler übergebenen Argumente abzurufen, kann die getArgs()-Methode des Host-Objekts verwendet werden:

```typescript
const [req, res, next] = host.getArgs();
```

Sie können ein bestimmtes Argument anhand des Indexes mit der getArgByIndex()-Methode herausgreifen:

```typescript
const request = host.getArgByIndex(0);
const response = host.getArgByIndex(1);
```

In diesen Beispielen haben wir die Request- und Response-Objekte nach Index abgerufen, was typischerweise nicht empfohlen wird, da dies die Anwendung an einen bestimmten Ausführungskontext bindet. Stattdessen können Sie Ihren Code robuster und wiederverwendbarer machen, indem Sie eine der Hilfsmethoden des Host-Objekts verwenden, um in den für Ihre Anwendung geeigneten Kontext zu wechseln. Die Kontextwechsel-Hilfsmethoden sind unten gezeigt:

```typescript
/**
 * Wechselt den Kontext zu RPC.
 */
switchToRpc(): RpcArgumentsHost;
/**
 * Wechselt den Kontext zu HTTP.
 */
switchToHttp(): HttpArgumentsHost;
/**
 * Wechselt den Kontext zu WebSockets.
 */
switchToWs(): WsArgumentsHost;
```

Lassen Sie uns das vorherige Beispiel mit der switchToHttp()-Methode neu schreiben. Der Aufruf von host.switchToHttp() gibt ein HttpArgumentsHost-Objekt zurück, das für den HTTP-Anwendungskontext geeignet ist. Das HttpArgumentsHost-Objekt hat zwei nützliche Methoden, die wir verwenden können, um die gewünschten Objekte zu extrahieren. In diesem Fall verwenden wir die Typassertionen von Express, um native Express-typisierte Objekte zurückzugeben:

```typescript
const ctx = host.switchToHttp();
const request = ctx.getRequest<Request>();
const response = ctx.getResponse<Response>();
```

Ähnlich haben WsArgumentsHost und RpcArgumentsHost Methoden, um entsprechende Objekte im Microservices- und WebSockets-Kontext zurückzugeben. Hier sind die Methoden für WsArgumentsHost:

```typescript
export interface WsArgumentsHost {
  /**
   * Gibt das Datenobjekt zurück.
   */
  getData<T>(): T;
  /**
   * Gibt das Client-Objekt zurück.
   */
  getClient<T>(): T;
}
```

Folgende sind die Methoden für RpcArgumentsHost:

```typescript
export interface RpcArgumentsHost {
  /**
   * Gibt das Datenobjekt zurück.
   */
  getData<T>(): T;

  /**
   * Gibt das Kontextobjekt zurück.
   */
  getContext<T>(): T;
}
```

## ExecutionContext-Klasse / ExecutionContext class

ExecutionContext erweitert ArgumentsHost und liefert zusätzliche Details über den aktuellen Ausführungsprozess. Wie ArgumentsHost stellt Nest eine Instanz von ExecutionContext an Stellen zur Verfügung, an denen Sie sie benötigen, z.B. in der canActivate()-Methode eines Guards und der intercept()-Methode eines Interceptors. Es bietet die folgenden Methoden:

```typescript
export interface ExecutionContext extends ArgumentsHost {
  /**
   * Gibt den Typ der Controller-Klasse zurück, zu der der aktuelle Handler gehört.
   */
  getClass<T>(): Type<T>;
  /**
   * Gibt eine Referenz auf den Handler (Methode) zurück, der als nächstes in der
   * Anfrage-Pipeline aufgerufen wird.
   */
  getHandler(): Function;
}
```

Die getHandler()-Methode gibt eine Referenz auf den Handler zurück, der als nächstes aufgerufen wird. Die getClass()-Methode gibt den Typ der Controller-Klasse zurück, zu der dieser bestimmte Handler gehört. Beispielsweise, in einem HTTP-Kontext, wenn die aktuell verarbeitete Anfrage eine POST-Anfrage ist, die an die create()-Methode des CatsController gebunden ist, gibt getHandler() eine Referenz auf die create()-Methode zurück und getClass() gibt die CatsController-Klasse (nicht Instanz) zurück.

```typescript
const methodKey = ctx.getHandler().name; // "create"
const className = ctx.getClass().name; // "CatsController"
```

Die Möglichkeit, Referenzen sowohl auf die aktuelle Klasse als auch auf die Handler-Methode zuzugreifen, bietet große Flexibilität. Am wichtigsten ist, dass es uns die Möglichkeit gibt, auf die Metadaten zuzugreifen, die entweder durch über den Reflector#createDecorator erstellte Dekoratoren oder den eingebauten @SetMetadata()-Dekorator innerhalb von Guards oder Interceptors festgelegt wurden. Wir behandeln diesen Anwendungsfall unten.

## Reflektion und Metadaten / Reflection and metadata

Nest bietet die Möglichkeit, benutzerdefinierte Metadaten über Dekoratoren, die über die Reflector#createDecorator-Methode erstellt wurden, an Routen-Handler anzuhängen, sowie den eingebauten @SetMetadata()-Dekorator. In diesem Abschnitt vergleichen wir die beiden Ansätze und sehen, wie man innerhalb eines Guards oder Interceptors auf die Metadaten zugreift.

Um stark typisierte Dekoratoren mit Reflector#createDecorator zu erstellen, müssen wir das Typ-Argument angeben. Lassen Sie uns zum Beispiel einen Roles-Dekorator erstellen, der ein Array von Zeichenfolgen als Argument nimmt.

```typescript
// roles.decorator.ts

import { Reflector } from '@nestjs/core';

export const Roles = Reflector.createDecorator<string[]>();
```

Der Roles-Dekorator hier ist eine Funktion, die ein einzelnes Argument vom Typ string[] nimmt.

Um diesen Dekorator zu verwenden, annotieren wir den Handler einfach damit:

```typescript
// cats.controller.ts

@Post()
@Roles(['admin'])
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
```

Hier haben wir die Roles-Dekorator-Metadaten an die create()-Methode angehängt, was bedeutet, dass nur Benutzer mit der admin-Rolle auf diese Route zugreifen dürfen.

Um auf die Rollen der Route (benutzerdefinierte Metadaten) zuzugreifen, verwenden wir erneut die Reflector-Hilfsklasse. Reflector kann auf normale Weise in eine Klasse injiziert werden:

```typescript
// roles.guard.ts

@Injectable()
export class RolesGuard {
  constructor(private reflector: Reflector) {}
}
```

**TIPP**: Die Reflector-Klasse wird aus dem @nestjs/core-Paket importiert.

Um die Handler-Metadaten zu lesen, verwenden Sie die get()-Methode:

```typescript
const roles = this.reflector.get(Roles, context.getHandler());
```

Die Reflector#get-Methode ermöglicht es uns, einfach auf die Metadaten zuzugreifen, indem wir zwei Argumente übergeben:

 eine Dekorator-Referenz und einen Kontext (Dekorator-Ziel), um die Metadaten abzurufen. In diesem Beispiel ist der angegebene Dekorator Roles (siehe oben im roles.decorator.ts-Datei). Der Kontext wird durch den Aufruf von context.getHandler() bereitgestellt, der das Extrahieren der Metadaten für den aktuell verarbeiteten Routen-Handler zur Folge hat. Denken Sie daran, getHandler() gibt uns eine Referenz auf die Routen-Handler-Funktion.

Alternativ können wir unseren Controller organisieren, indem wir Metadaten auf der Controller-Ebene anwenden, die für alle Routen in der Controller-Klasse gelten.

```typescript
// cats.controller.ts

@Roles(['admin'])
@Controller('cats')
export class CatsController {}
```

In diesem Fall, um Controller-Metadaten zu extrahieren, übergeben wir context.getClass() als zweites Argument (um die Controller-Klasse als Kontext für die Metadatenextraktion bereitzustellen) anstelle von context.getHandler():

```typescript
// roles.guard.ts

const roles = this.reflector.get(Roles, context.getClass());
```

Da es möglich ist, Metadaten auf mehreren Ebenen bereitzustellen, müssen Sie möglicherweise Metadaten aus mehreren Kontexten extrahieren und zusammenführen. Die Reflector-Klasse bietet zwei Hilfsmethoden, um dabei zu helfen. Diese Methoden extrahieren sowohl Controller- als auch Methoden-Metadaten auf einmal und kombinieren sie auf verschiedene Weise.

Betrachten Sie das folgende Szenario, in dem Sie Rollen-Metadaten auf beiden Ebenen bereitgestellt haben.

```typescript
// cats.controller.ts

@Roles(['user'])
@Controller('cats')
export class CatsController {
  @Post()
  @Roles(['admin'])
  async create(@Body() createCatDto: CreateCatDto) {
    this.catsService.create(createCatDto);
  }
}
```

Wenn Ihr Ziel darin besteht, 'user' als Standardrolle anzugeben und sie selektiv für bestimmte Methoden zu überschreiben, würden Sie wahrscheinlich die getAllAndOverride()-Methode verwenden:

```typescript
const roles = this.reflector.getAllAndOverride(Roles, [context.getHandler(), context.getClass()]);
```

Ein Guard mit diesem Code, der im Kontext der create()-Methode ausgeführt wird, würde dazu führen, dass roles ['admin'] enthält.

Um Metadaten für beide zu erhalten und sie zu kombinieren (diese Methode kombiniert sowohl Arrays als auch Objekte), verwenden Sie die getAllAndMerge()-Methode:

```typescript
const roles = this.reflector.getAllAndMerge(Roles, [context.getHandler(), context.getClass()]);
```

Dies würde dazu führen, dass roles ['user', 'admin'] enthält.

Für beide Merge-Methoden übergeben Sie den Metadatenschlüssel als erstes Argument und ein Array von Metadatenziel-Kontexten (d.h. Aufrufe der getHandler() und/oder getClass()-Methoden) als zweites Argument.

## Low-Level-Ansatz / Low-level approach

Wie bereits erwähnt, können Sie anstelle der Verwendung von Reflector#createDecorator auch den eingebauten @SetMetadata()-Dekorator verwenden, um Metadaten an einen Handler anzuhängen.

```typescript
// cats.controller.ts

@Post()
@SetMetadata('roles', ['admin'])
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
```

**TIPP**: Der @SetMetadata()-Dekorator wird aus dem @nestjs/common-Paket importiert.

Mit der obigen Konstruktion haben wir die Rollen-Metadaten (roles ist ein Metadatenschlüssel und ['admin'] ist der zugehörige Wert) an die create()-Methode angehängt. Obwohl dies funktioniert, ist es nicht empfehlenswert, @SetMetadata() direkt in Ihren Routen zu verwenden. Stattdessen können Sie Ihre eigenen Dekoratoren erstellen, wie unten gezeigt:

```typescript
// roles.decorator.ts

import { SetMetadata } from '@nestjs/common';

export const Roles = (...roles: string[]) => SetMetadata('roles', roles);
```

Dieser Ansatz ist viel sauberer und lesbarer und ähnelt dem Reflector#createDecorator-Ansatz. Der Unterschied besteht darin, dass Sie mit @SetMetadata mehr Kontrolle über den Metadatenschlüssel und -wert haben und auch Dekoratoren erstellen können, die mehr als ein Argument annehmen.

Jetzt, da wir einen benutzerdefinierten @Roles()-Dekorator haben, können wir ihn verwenden, um die create()-Methode zu dekorieren.

```typescript
// cats.controller.ts

@Post()
@Roles('admin')
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
```

Um auf die Rollen der Route (benutzerdefinierte Metadaten) zuzugreifen, verwenden wir erneut die Reflector-Hilfsklasse:

```typescript
// roles.guard.ts

@Injectable()
export class RolesGuard {
  constructor(private reflector: Reflector) {}
}
```

**TIPP**: Die Reflector-Klasse wird aus dem @nestjs/core-Paket importiert.

Um die Handler-Metadaten zu lesen, verwenden Sie die get()-Methode:

```typescript
const roles = this.reflector.get<string[]>('roles', context.getHandler());
```

Hier übergeben wir anstelle einer Dekorator-Referenz den Metadatenschlüssel als erstes Argument (der in unserem Fall 'roles' ist). Alles andere bleibt wie im Reflector#createDecorator-Beispiel.
