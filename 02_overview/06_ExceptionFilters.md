### Ausnahmefilter / Exception filters


NestJS verfügt über eine integrierte Ausnahmeschicht, die für die Verarbeitung aller unbehandelten Ausnahmen in einer Anwendung verantwortlich ist. Wenn eine Ausnahme nicht von Ihrem Anwendungscode behandelt wird, wird sie von dieser Schicht erfasst, die dann automatisch eine benutzerfreundliche Antwort sendet.

Standardmäßig wird diese Aktion von einem eingebauten globalen Ausnahmefilter durchgeführt, der Ausnahmen vom Typ `HttpException` (und dessen Unterklassen) behandelt. Wenn eine Ausnahme nicht erkannt wird (weder `HttpException` noch eine Klasse, die von `HttpException` erbt), generiert der eingebaute Ausnahmefilter die folgende Standard-JSON-Antwort:

```json
{
  "statusCode": 500,
  "message": "Internal server error"
}
```

### Hinweis

Der globale Ausnahmefilter unterstützt teilweise die Bibliothek `http-errors`. Im Grunde wird jede geworfene Ausnahme, die die Eigenschaften `statusCode` und `message` enthält, korrekt ausgefüllt und als Antwort zurückgesendet (anstatt der Standard-`InternalServerErrorException` für nicht erkannte Ausnahmen).

### Standardausnahmen werfen / Throwing standard exceptions

NestJS bietet eine eingebaute `HttpException`-Klasse, die aus dem Paket `@nestjs/common` bereitgestellt wird. Für typische HTTP-REST/GraphQL-API-basierte Anwendungen ist es Best Practice, standardisierte HTTP-Antwortobjekte zu senden, wenn bestimmte Fehlerbedingungen auftreten.

Beispielsweise haben wir im `CatsController` eine `findAll()` Methode (einen GET-Routen-Handler). Angenommen, dieser Routen-Handler wirft aus irgendeinem Grund eine Ausnahme. Um dies zu demonstrieren, kodieren wir dies wie folgt:

#### Beispiel: Ausnahme werfen

```typescript
@Get()
async findAll() {
  throw new HttpException('Forbidden', HttpStatus.FORBIDDEN);
}
```

### Hinweis

Wir haben hier `HttpStatus` verwendet. Dies ist ein Hilfs-Enum, das aus dem Paket `@nestjs/common` importiert wird.

Wenn der Client diesen Endpunkt aufruft, sieht die Antwort so aus:

```json
{
  "statusCode": 403,
  "message": "Forbidden"
}
```

Der `HttpException`-Konstruktor nimmt zwei erforderliche Argumente an, die die Antwort bestimmen:

- Das `response`-Argument definiert den JSON-Antwortkörper. Es kann ein String oder ein Objekt sein, wie unten beschrieben.
- Das `status`-Argument definiert den HTTP-Statuscode.

Standardmäßig enthält der JSON-Antwortkörper zwei Eigenschaften:

- `statusCode`: entspricht dem im `status`-Argument angegebenen HTTP-Statuscode
- `message`: eine kurze Beschreibung des HTTP-Fehlers basierend auf dem Status

Um nur den Nachrichtenanteil des JSON-Antwortkörpers zu überschreiben, geben Sie einen String im `response`-Argument an. Um den gesamten JSON-Antwortkörper zu überschreiben, übergeben Sie ein Objekt im `response`-Argument. NestJS serialisiert das Objekt und gibt es als JSON-Antwortkörper zurück.

Das zweite Konstruktorargument - `status` - sollte ein gültiger HTTP-Statuscode sein. Best Practice ist die Verwendung des `HttpStatus`-Enums, das aus `@nestjs/common` importiert wird.

Es gibt ein drittes optionales Konstruktorargument - `options` -, das verwendet werden kann, um eine Fehlerursache bereitzustellen. Dieses `cause`-Objekt wird nicht in das Antwortobjekt serialisiert, kann jedoch nützlich für Protokollierungszwecke sein und wertvolle Informationen über den inneren Fehler liefern, der zum Werfen der `HttpException` geführt hat.

#### Beispiel: Anpassen des gesamten Antwortkörpers und Bereitstellen einer Fehlerursache

```typescript
@Get()
async findAll() {
  try {
    await this.service.findAll()
  } catch (error) {
    throw new HttpException({
      status: HttpStatus.FORBIDDEN,
      error: 'This is a custom message',
    }, HttpStatus.FORBIDDEN, {
      cause: error
    });
  }
}
```

Verwendet man das obige Beispiel, sieht die Antwort folgendermaßen aus:

```json
{
  "status": 403,
  "error": "This is a custom message"
}
```

### Eigene Ausnahmen / Custom exceptions

In vielen Fällen müssen Sie keine eigenen Ausnahmen schreiben und können die eingebauten NestJS-HTTP-Ausnahmen verwenden. Wenn Sie dennoch eigene Ausnahmen erstellen müssen, ist es ratsam, eine eigene Ausnahmenhierarchie zu erstellen, bei der Ihre benutzerdefinierten Ausnahmen von der Basis-`HttpException`-Klasse erben. Mit diesem Ansatz erkennt NestJS Ihre Ausnahmen automatisch und kümmert sich um die Fehlerantworten. Implementieren wir eine solche benutzerdefinierte Ausnahme:

#### Beispiel: Benutzerdefinierte Ausnahme

```typescript
export class ForbiddenException extends HttpException {
  constructor() {
    super('Forbidden', HttpStatus.FORBIDDEN);
  }
}
```

Da `ForbiddenException` von der Basis-`HttpException` erbt, funktioniert sie nahtlos mit dem eingebauten Ausnahme-Handler. Daher können wir sie in der `findAll()`-Methode verwenden.

#### Beispiel: Verwendung von `ForbiddenException`

```typescript
@Get()
async findAll() {
  throw new ForbiddenException();
}
```

### Eingebaute HTTP-Ausnahmen / Built-in HTTP exceptions

NestJS bietet eine Reihe von Standardausnahmen, die von der Basis-`HttpException` erben. Diese werden aus dem Paket `@nestjs/common` bereitgestellt und repräsentieren viele der häufigsten HTTP-Ausnahmen:

- `BadRequestException`
- `UnauthorizedException`
- `NotFoundException`
- `ForbiddenException`
- `NotAcceptableException`
- `RequestTimeoutException`
- `ConflictException`
- `GoneException`
- `HttpVersionNotSupportedException`
- `PayloadTooLargeException`
- `UnsupportedMediaTypeException`
- `UnprocessableEntityException`
- `InternalServerErrorException`
- `NotImplementedException`
- `ImATeapotException`
- `MethodNotAllowedException`
- `BadGatewayException`
- `ServiceUnavailableException`
- `GatewayTimeoutException`
- `PreconditionFailedException`

Alle eingebauten Ausnahmen können sowohl eine Fehlerursache als auch eine Fehlerbeschreibung mithilfe des `options`-Parameters bereitstellen:

```typescript
throw new BadRequestException('Something bad happened', { cause: new Error(), description: 'Some error description' })
```

Verwendet man das obige Beispiel, sieht die Antwort folgendermaßen aus:

```json
{
  "message": "Something bad happened",
  "error": "Some error description",
  "statusCode": 400
}
```

### Ausnahmefilter / Exception filters

Während der Basis-(eingebaute) Ausnahmefilter viele Fälle automatisch für Sie behandeln kann, möchten Sie möglicherweise die volle Kontrolle über die Ausnahmeschicht haben. Zum Beispiel möchten Sie möglicherweise Protokollierung hinzufügen oder ein anderes JSON-Schema basierend auf dynamischen Faktoren verwenden. Ausnahmefilter sind genau für diesen Zweck konzipiert. Sie ermöglichen es Ihnen, den genauen Kontrollfluss und den Inhalt der an den Client zurückgesendeten Antwort zu steuern.

Lassen Sie uns einen Ausnahmefilter erstellen, der für das Abfangen von Ausnahmen zuständig ist, die eine Instanz der `HttpException`-Klasse sind, und eine benutzerdefinierte Antwortlogik dafür implementieren. Dazu müssen wir auf die zugrunde liegenden Plattform-Request- und Response-Objekte zugreifen. Wir greifen auf das Request-Objekt zu, um die ursprüngliche URL herauszuziehen und in die Protokollierungsinformationen aufzunehmen. Wir verwenden das Response-Objekt, um die Kontrolle über die zurückgesendete Antwort zu übernehmen, indem wir die `response.json()`-Methode verwenden.

#### Beispiel: HttpExceptionFilter

```typescript
import { ExceptionFilter, Catch, ArgumentsHost, HttpException } from '@nestjs/common';
import { Request, Response } from 'express';

@Catch(HttpException)
export class HttpExceptionFilter implements ExceptionFilter {
  catch(exception: HttpException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();
    const status = exception.getStatus();

    response
      .status(status)
      .json({
        statusCode: status,
        timestamp: new Date().toISOString(),
        path: request.url,
      });
  }
}
```

### Hinweis

Alle Ausnahmefilter sollten das generische `ExceptionFilter<T>`-Interface implementieren. Dies erfordert, dass Sie die Methode `catch(exception: T, host: ArgumentsHost)` mit ihrer angegebenen Signatur bereitstellen. `T` gibt den Typ der Ausnahme an.

### Hinweis

Wenn Sie `@nestjs/platform-fastify` verwenden, können Sie `response.send()` anstelle von `response.json()` verwenden. Vergessen Sie nicht, die korrekten Typen von Fastify zu importieren.

Der `@Catch(HttpException)`-Dekorator bindet die erforderlichen Metadaten an den Ausnahmefilter und teilt NestJS mit, dass dieser spezielle Filter nach Ausnahmen vom Typ `HttpException` sucht und nichts anderes. Der `@Catch()`-Dekorator kann einen einzelnen Parameter oder eine durch Kommata getrennte Liste von Parametern annehmen. Dies ermöglicht es Ihnen, den Filter gleichzeitig für mehrere Ausnahmetypen einzurichten.

### Parameter von `ArgumentsHost` / Arguments host

Schauen wir uns die Parameter der `catch()`-Methode an. Der `exception`-Parameter ist das derzeit verarbeitete Ausnahmeobjekt. Der `host`-Parameter ist ein `ArgumentsHost`-Objekt. `ArgumentsHost` ist ein leistungsfähiges Hilfsobjekt, das wir im Kapitel zum Ausführungskontext weiter untersuchen werden. In diesem Codebeispiel verwenden wir es, um eine Referenz auf die Request- und Response-Objekte zu erhalten, die an den ursprünglichen Request-Handler (im Controller, in dem die Ausnahme auftritt) übergeben werden. In diesem Codebeispiel haben wir einige Hilfsmethoden von `ArgumentsHost` verwendet, um die gewünschten Request- und Response-Objekte zu erhalten.

### Hinweis

Der Grund für dieses Abstraktionsniveau ist,

 dass `ArgumentsHost` in allen Kontexten funktioniert (z.B. im HTTP-Serverkontext, mit dem wir jetzt arbeiten, aber auch in Microservices und WebSockets). Im Kapitel zum Ausführungskontext werden wir sehen, wie wir mit den Fähigkeiten von `ArgumentsHost` und seinen Hilfsfunktionen auf die entsprechenden zugrunde liegenden Argumente für jeden Ausführungskontext zugreifen können. Dies ermöglicht es uns, generische Ausnahmefilter zu schreiben, die in allen Kontexten funktionieren.

### Binden von Filtern / Binding filters

Lassen Sie uns unseren neuen `HttpExceptionFilter` an die `create()`-Methode des `CatsController` binden.

#### Beispiel: Filter binden

```typescript
@Post()
@UseFilters(new HttpExceptionFilter())
async create(@Body() createCatDto: CreateCatDto) {
  throw new ForbiddenException();
}
```

### Hinweis

Der `@UseFilters()`-Dekorator wird aus dem Paket `@nestjs/common` importiert.

Wir haben hier den `@UseFilters()`-Dekorator verwendet. Ähnlich wie der `@Catch()`-Dekorator kann er eine einzelne Filterinstanz oder eine durch Kommata getrennte Liste von Filterinstanzen annehmen. Hier haben wir die Instanz des `HttpExceptionFilter` direkt erstellt. Alternativ können Sie die Klasse (anstatt einer Instanz) übergeben, wodurch die Verantwortung für die Instanziierung dem Framework überlassen wird und die Dependency Injection ermöglicht wird.

#### Beispiel: Verwendung der Filterklasse

```typescript
@Post()
@UseFilters(HttpExceptionFilter)
async create(@Body() createCatDto: CreateCatDto) {
  throw new ForbiddenException();
}
```

### Hinweis

Es ist vorzuziehen, Filter nach Möglichkeit mithilfe von Klassen anstelle von Instanzen anzuwenden. Dies reduziert den Speicherverbrauch, da NestJS Instanzen derselben Klasse leicht in Ihrem gesamten Modul wiederverwenden kann.

Im obigen Beispiel wird der `HttpExceptionFilter` nur auf den einzelnen `create()`-Routen-Handler angewendet und ist somit methodenspezifisch. Ausnahmefilter können auf verschiedenen Ebenen angewendet werden: methodenspezifisch für den Controller/Resolver/Gateway, controller-spezifisch oder global. Um einen Filter controller-spezifisch zu machen, würden Sie folgendes tun:

#### Beispiel: Controller-spezifischer Filter

```typescript
@UseFilters(new HttpExceptionFilter())
export class CatsController {}
```

Diese Konstruktion richtet den `HttpExceptionFilter` für jeden Routen-Handler ein, der im `CatsController` definiert ist.

Um einen globalen Filter zu erstellen, würden Sie folgendes tun:

#### Beispiel: Globaler Filter

main.ts
```typescript
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalFilters(new HttpExceptionFilter());
  await app.listen(3000);
}
bootstrap();
```

### Hinweis

Die Methode `useGlobalFilters()` richtet keine Filter für Gateways oder hybride Anwendungen ein.

Global-spezifische Filter werden in der gesamten Anwendung verwendet, für jeden Controller und jeden Routen-Handler. In Bezug auf Dependency Injection können global registrierte Filter von außerhalb eines Moduls (mit `useGlobalFilters()` wie im obigen Beispiel) keine Abhängigkeiten injizieren, da dies außerhalb des Kontexts eines Moduls geschieht. Um dieses Problem zu lösen, können Sie einen globalen Filter direkt aus einem Modul heraus registrieren, indem Sie die folgende Konstruktion verwenden:

#### Beispiel: Globaler Filter mit Dependency Injection

```typescript
import { Module } from '@nestjs/common';
import { APP_FILTER } from '@nestjs/core';

@Module({
  providers: [
    {
      provide: APP_FILTER,
      useClass: HttpExceptionFilter,
    },
  ],
})
export class AppModule {}
```

### Hinweis

Wenn Sie diesen Ansatz verwenden, um die Dependency Injection für den Filter durchzuführen, beachten Sie, dass der Filter tatsächlich global ist, unabhängig davon, in welchem Modul diese Konstruktion angewendet wird. Wo sollte dies getan werden? Wählen Sie das Modul, in dem der Filter (im obigen Beispiel `HttpExceptionFilter`) definiert ist. `useClass` ist nicht die einzige Möglichkeit, benutzerdefinierte Provider zu registrieren. Erfahren Sie mehr darüber [hier](https://docs.nestjs.com/fundamentals/custom-providers).

Sie können mit dieser Technik so viele Filter hinzufügen, wie benötigt werden; fügen Sie einfach jeden zum `providers`-Array hinzu.

### Alles abfangen

Um jede unbehandelte Ausnahme (unabhängig vom Ausnahmetyp) abzufangen, lassen Sie die Parameterliste des `@Catch()`-Dekorators leer, z.B. `@Catch()`.

Im folgenden Beispiel haben wir einen Code, der plattformunabhängig ist, da er den HTTP-Adapter verwendet, um die Antwort zu liefern, und keine der plattformspezifischen Objekte (Request und Response) direkt verwendet:

#### Beispiel: Alle Ausnahmen abfangen

```typescript
import {
  ExceptionFilter,
  Catch,
  ArgumentsHost,
  HttpException,
  HttpStatus,
} from '@nestjs/common';
import { HttpAdapterHost } from '@nestjs/core';

@Catch()
export class AllExceptionsFilter implements ExceptionFilter {
  constructor(private readonly httpAdapterHost: HttpAdapterHost) {}

  catch(exception: unknown, host: ArgumentsHost): void {
    // In bestimmten Situationen ist `httpAdapter` im Konstruktor
    // möglicherweise nicht verfügbar, daher sollten wir es hier auflösen.
    const { httpAdapter } = this.httpAdapterHost;

    const ctx = host.switchToHttp();

    const httpStatus =
      exception instanceof HttpException
        ? exception.getStatus()
        : HttpStatus.INTERNAL_SERVER_ERROR;

    const responseBody = {
      statusCode: httpStatus,
      timestamp: new Date().toISOString(),
      path: httpAdapter.getRequestUrl(ctx.getRequest()),
    };

    httpAdapter.reply(ctx.getResponse(), responseBody, httpStatus);
  }
}
```

### Hinweis

Wenn Sie einen Ausnahmefilter kombinieren, der alles abfängt, mit einem Filter, der an einen bestimmten Typ gebunden ist, sollte der "Alles abfangen"-Filter zuerst deklariert werden, damit der spezifische Filter den gebundenen Typ korrekt verarbeiten kann.

### Vererbung

Typischerweise erstellen Sie vollständig angepasste Ausnahmefilter, die Ihre Anwendungsanforderungen erfüllen. Es kann jedoch Fälle geben, in denen Sie einfach den eingebauten Standard-Global-Exception-Filter erweitern und das Verhalten basierend auf bestimmten Faktoren überschreiben möchten.

Um die Ausnahmeverarbeitung an den Basisfilter zu delegieren, müssen Sie `BaseExceptionFilter` erweitern und die geerbte `catch()`-Methode aufrufen.

#### Beispiel: Erweiterung des Basisfilters

```typescript
import { Catch, ArgumentsHost } from '@nestjs/common';
import { BaseExceptionFilter } from '@nestjs/core';

@Catch()
export class AllExceptionsFilter extends BaseExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    super.catch(exception, host);
  }
}
```

### Hinweis

Methoden-spezifische und Controller-spezifische Filter, die `BaseExceptionFilter` erweitern, sollten nicht mit `new` instanziiert werden. Lassen Sie stattdessen das Framework sie automatisch instanziieren.

Globale Filter können den Basisfilter erweitern. Dies kann auf zwei Arten geschehen.

Die erste Methode besteht darin, die HttpAdapter-Referenz bei der Instanziierung des benutzerdefinierten globalen Filters zu injizieren:

#### Beispiel: Injektion des HttpAdapters

```typescript
async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  const { httpAdapter } = app.get(HttpAdapterHost);
  app.useGlobalFilters(new AllExceptionsFilter(httpAdapter));

  await app.listen(3000);
}
bootstrap();
```

Die zweite Methode besteht darin, das `APP_FILTER`-Token wie hier gezeigt zu verwenden.

---

NestJS ist ein MIT-lizenziertes Open-Source-Projekt und kann dank der Unterstützung der Community wachsen.