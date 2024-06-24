# Versionsverwaltung / Versioning

**HINWEIS**
Dieses Kapitel ist nur für HTTP-basierte Anwendungen relevant.

Versionsverwaltung ermöglicht es Ihnen, verschiedene Versionen Ihrer Controller oder einzelner Routen innerhalb derselben Anwendung auszuführen. Anwendungen ändern sich sehr oft und es ist nicht ungewöhnlich, dass es breaking changes gibt, die Sie vornehmen müssen, während Sie gleichzeitig die vorherige Version der Anwendung unterstützen müssen.

Es gibt 4 Arten der Versionsverwaltung, die unterstützt werden:

- **URI-Versionierung**: Die Version wird innerhalb der URI der Anfrage übermittelt (Standard).
- **Header-Versionierung**: Ein benutzerdefinierter Anforderungsheader gibt die Version an.
- **Media-Type-Versionierung**: Der Accept-Header der Anfrage gibt die Version an.
- **Benutzerdefinierte Versionierung**: Jeder Aspekt der Anfrage kann verwendet werden, um die Version(en) anzugeben. Eine benutzerdefinierte Funktion wird bereitgestellt, um die Version(en) zu extrahieren.

## URI-Versionierung / URI Versioning Type

Die URI-Versionierung verwendet die Version, die innerhalb der URI der Anfrage übermittelt wird, wie https://example.com/v1/route und https://example.com/v2/route.

**HINWEIS**
Bei der URI-Versionierung wird die Version automatisch nach dem globalen Pfadpräfix (falls vorhanden) und vor allen Controller- oder Routenpfaden zur URI hinzugefügt.

Um die URI-Versionierung für Ihre Anwendung zu aktivieren, tun Sie Folgendes:

```typescript
// main.ts
const app = await NestFactory.create(AppModule);
// oder "app.enableVersioning()"
app.enableVersioning({
  type: VersioningType.URI,
});
await app.listen(3000);
```

**HINWEIS**
Die Version in der URI wird standardmäßig automatisch mit v vorangestellt. Dieser Präfixwert kann jedoch konfiguriert werden, indem der Schlüssel prefix auf den gewünschten Präfix oder false gesetzt wird, wenn Sie ihn deaktivieren möchten.

**HINWEIS**
Das VersioningType-Enum ist für die Verwendung der Eigenschaft type verfügbar und wird aus dem @nestjs/common-Paket importiert.

## Header-Versionierung / Header Versioning Type

Die Header-Versionierung verwendet einen benutzerdefinierten, vom Benutzer angegebenen Anforderungsheader, um die Version anzugeben, wobei der Wert des Headers die zu verwendende Version für die Anfrage darstellt.

Beispiel für HTTP-Anfragen zur Header-Versionierung:

Um die Header-Versionierung für Ihre Anwendung zu aktivieren, tun Sie Folgendes:

```typescript
// main.ts
const app = await NestFactory.create(AppModule);
app.enableVersioning({
  type: VersioningType.HEADER,
  header: 'Custom-Header',
});
await app.listen(3000);
```

Die header-Eigenschaft sollte der Name des Headers sein, der die Version der Anfrage enthält.

**HINWEIS**
Das VersioningType-Enum ist für die Verwendung der Eigenschaft type verfügbar und wird aus dem @nestjs/common-Paket importiert.

## Media-Type-Versionierung / Media Type Versioning Type

Die Media-Type-Versionierung verwendet den Accept-Header der Anfrage, um die Version anzugeben.

Innerhalb des Accept-Headers wird die Version mit einem Semikolon (;) vom Medientyp getrennt. Es sollte dann ein Schlüssel-Wert-Paar enthalten, das die zu verwendende Version für die Anfrage darstellt, wie z.B. Accept: application/json;v=2. Der Schlüssel wird eher als Präfix behandelt, wenn die Version ermittelt wird, und wird so konfiguriert, dass der Schlüssel und der Separator enthalten sind.

Um die Media-Type-Versionierung für Ihre Anwendung zu aktivieren, tun Sie Folgendes:

```typescript
// main.ts
const app = await NestFactory.create(AppModule);
app.enableVersioning({
  type: VersioningType.MEDIA_TYPE,
  key: 'v=',
});
await app.listen(3000);
```

Die key-Eigenschaft sollte der Schlüssel und der Separator des Schlüssel-Wert-Paares sein, das die Version enthält. Für das Beispiel Accept: application/json;v=2 wäre die key-Eigenschaft auf v= gesetzt.

**HINWEIS**
Das VersioningType-Enum ist für die Verwendung der Eigenschaft type verfügbar und wird aus dem @nestjs/common-Paket importiert.

## Benutzerdefinierte Versionierung / Custom Versioning Type

Die benutzerdefinierte Versionierung verwendet jeden Aspekt der Anfrage, um die Version (oder Versionen) anzugeben. Die eingehende Anfrage wird mithilfe einer Extraktorfunktion analysiert, die eine Zeichenfolge oder ein Array von Zeichenfolgen zurückgibt.

Wenn mehrere Versionen vom Anforderer bereitgestellt werden, kann die Extraktorfunktion ein Array von Zeichenfolgen zurückgeben, sortiert in der Reihenfolge von der größten/höchsten Version zur kleinsten/niedrigsten Version. Versionen werden in der Reihenfolge von der höchsten zur niedrigsten mit Routen abgeglichen.

Wenn eine leere Zeichenfolge oder ein Array zurückgegeben wird, werden keine Routen abgeglichen und es wird eine 404-Antwort zurückgegeben.

Beispielsweise, wenn eine eingehende Anfrage angibt, dass sie die Versionen 1, 2 und 3 unterstützt, MUSS der Extraktor [3, 2, 1] zurückgeben. Dies stellt sicher, dass zuerst die höchstmögliche Routenversion ausgewählt wird.

Wenn Versionen [3, 2, 1] extrahiert werden, aber nur Routen für die Versionen 2 und 1 existieren, wird die Route ausgewählt, die der Version 2 entspricht (Version 3 wird automatisch ignoriert).

**HINWEIS**
Die Auswahl der höchsten übereinstimmenden Version basierend auf dem vom Extraktor zurückgegebenen Array funktioniert aufgrund von Designbeschränkungen nicht zuverlässig mit dem Express-Adapter. Eine einzelne Version (entweder eine Zeichenfolge oder ein Array von einem Element) funktioniert einwandfrei in Express. Fastify unterstützt sowohl die Auswahl der höchsten übereinstimmenden Version als auch die Auswahl einer einzelnen Version korrekt.

Um die benutzerdefinierte Versionierung für Ihre Anwendung zu aktivieren, erstellen Sie eine Extraktorfunktion und übergeben Sie sie wie folgt in Ihre Anwendung:

```typescript
// main.ts
// Beispiel-Extraktor, der eine Liste von Versionen aus einem benutzerdefinierten Header extrahiert und in ein sortiertes Array umwandelt.
// Dieses Beispiel verwendet Fastify, aber Express-Anfragen können auf ähnliche Weise verarbeitet werden.
const extractor = (request: FastifyRequest): string | string[] =>
  [request.headers['custom-versioning-field'] ?? '']
    .flatMap(v => v.split(','))
    .filter(v => !!v)
    .sort()
    .reverse();

const app = await NestFactory.create(AppModule);
app.enableVersioning({
  type: VersioningType.CUSTOM,
  extractor,
});
await app.listen(3000);
```

## Nutzung / Usage

Versionsverwaltung ermöglicht es Ihnen, Controller, einzelne Routen zu versionieren und bietet auch eine Möglichkeit für bestimmte Ressourcen, sich von der Versionierung abzumelden. Die Verwendung der Versionsverwaltung ist unabhängig von dem Versionsverwaltungstyp, den Ihre Anwendung verwendet, gleich.

**HINWEIS**
Wenn die Versionsverwaltung für die Anwendung aktiviert ist, der Controller oder die Route jedoch keine Version angibt, wird jede Anfrage an diesen Controller/diese Route mit einem 404-Antwortstatus zurückgegeben. Ebenso, wenn eine Anfrage mit einer Version empfangen wird, die keinen entsprechenden Controller oder Route hat, wird sie ebenfalls mit einem 404-Antwortstatus zurückgegeben.

## Controller-Versionen / Controller versions

Eine Version kann auf einen Controller angewendet werden, indem die Version für alle Routen innerhalb des Controllers festgelegt wird.

Um einem Controller eine Version hinzuzufügen, tun Sie Folgendes:

```typescript
// cats.controller.ts
@Controller({
  version: '1',
})
export class CatsControllerV1 {
  @Get('cats')
  findAll(): string {
    return 'Diese Aktion gibt alle Katzen für Version 1 zurück';
  }
}
```

## Routen-Versionen / Route versions

Eine Version kann auf eine einzelne Route angewendet werden. Diese Version überschreibt jede andere Version, die die Route betreffen würde, wie z.B. die Controller-Version.

Um einer einzelnen Route eine Version hinzuzufügen, tun Sie Folgendes:

```typescript
// cats.controller.ts
import { Controller, Get, Version } from '@nestjs/common';

@Controller()
export class CatsController {
  @Version('1')
  @Get('cats')
  findAllV1(): string {
    return 'Diese Aktion gibt alle Katzen für Version 1 zurück';
  }

  @Version('2')
  @Get('cats')
  findAllV2(): string {
    return 'Diese Aktion gibt alle Katzen für Version 2 zurück';
  }
}
```

## Mehrere Versionen / Multiple versions

Mehrere Versionen können auf einen Controller oder eine Route angewendet werden. Um mehrere Versionen zu verwenden, würden Sie die Version als ein Array festlegen.

Um mehrere Versionen hinzuzufügen, tun Sie Folgendes:

```typescript
// cats.controller.ts
@Controller({
  version: ['1', '2'],
})
export class CatsController {
  @Get('cats')
  findAll(): string {
    return 'Diese Aktion gibt alle Katzen für Version 1 oder 2 zurück';
  }
}
```

## Version "Neutral" / Version "Neutral"

Einige Controller oder Routen interessieren sich möglicherweise nicht für die Version und würden unabhängig von der Version dieselbe Funktionalität haben. Um dies zu berücksichtigen, kann die Version auf das Symbol VERSION_NEUTRAL gesetzt werden.

Eine eingehende Anfrage wird zusätzlich zu

 der, wenn die Anfrage überhaupt keine Version enthält, einer VERSION_NEUTRAL-Controller oder -Route zugeordnet.

**HINWEIS**
Für die URI-Versionierung wäre eine VERSION_NEUTRAL-Ressource nicht in der URI vorhanden.

Um einem Controller oder einer Route eine versionneutrale Version hinzuzufügen, tun Sie Folgendes:

```typescript
// cats.controller.ts
import { Controller, Get, VERSION_NEUTRAL } from '@nestjs/common';

@Controller({
  version: VERSION_NEUTRAL,
})
export class CatsController {
  @Get('cats')
  findAll(): string {
    return 'Diese Aktion gibt alle Katzen unabhängig von der Version zurück';
  }
}
```

## Globale Standardversion / Global default version

Wenn Sie nicht für jeden Controller oder jede einzelne Route eine Version angeben möchten oder wenn Sie eine bestimmte Version als Standardversion für jeden Controller/Route festlegen möchten, der/die keine Version angegeben hat, können Sie die Standardversion wie folgt festlegen:

```typescript
// main.ts
app.enableVersioning({
  // ...
  defaultVersion: '1'
  // oder
  defaultVersion: ['1', '2']
  // oder
  defaultVersion: VERSION_NEUTRAL
});
```

## Middleware-Versionierung / Middleware versioning

Middlewares können auch Versionsverwaltungsmetadaten verwenden, um die Middleware für eine spezifische Version der Route zu konfigurieren. Dazu geben Sie die Versionsnummer als einen der Parameter für die Methode MiddlewareConsumer.forRoutes() an:

```typescript
// app.module.ts
import { Module, NestModule, MiddlewareConsumer } from '@nestjs/common';
import { LoggerMiddleware } from './common/middleware/logger.middleware';
import { CatsModule } from './cats/cats.module';
import { CatsController } from './cats/cats.controller';

@Module({
  imports: [CatsModule],
})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(LoggerMiddleware)
      .forRoutes({ path: 'cats', method: RequestMethod.GET, version: '2' });
  }
}
```

Mit dem obigen Code wird die LoggerMiddleware nur auf die Version '2' des /cats-Endpunkts angewendet.

**HINWEIS**
Middlewares funktionieren mit jedem in diesem Abschnitt beschriebenen Versionsverwaltungstyp: URI, Header, Media Type oder Custom.
