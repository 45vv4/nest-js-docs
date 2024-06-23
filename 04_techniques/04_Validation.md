# Validierung / Validation

Es ist bewährte Praxis, die Korrektheit aller Daten, die an eine Webanwendung gesendet werden, zu validieren. Um eingehende Anfragen automatisch zu validieren, bietet Nest mehrere Pipes direkt einsatzbereit:

- ValidationPipe
- ParseIntPipe
- ParseBoolPipe
- ParseArrayPipe
- ParseUUIDPipe

Die `ValidationPipe` nutzt das leistungsstarke `class-validator`-Paket und seine deklarativen Validierungsdekoratoren. Die `ValidationPipe` bietet einen bequemen Ansatz zur Durchsetzung von Validierungsregeln für alle eingehenden Client-Daten, wobei die spezifischen Regeln mit einfachen Anmerkungen in lokalen Klassen/DTO-Deklarationen in jedem Modul deklariert werden.

## Überblick / Overview

Im Kapitel "Pipes" haben wir den Prozess des Aufbaus einfacher Pipes und deren Bindung an Controller, Methoden oder die globale App durchlaufen, um zu demonstrieren, wie der Prozess funktioniert. Stellen Sie sicher, dass Sie dieses Kapitel durchgehen, um die Themen dieses Kapitels am besten zu verstehen. Hier werden wir uns auf verschiedene reale Anwendungsfälle der `ValidationPipe` konzentrieren und zeigen, wie einige ihrer erweiterten Anpassungsfunktionen genutzt werden können.

## Verwendung der integrierten `ValidationPipe` / Using the built-in ValidationPipe

Um mit der Nutzung zu beginnen, installieren wir zunächst die erforderliche Abhängigkeit:

```bash
$ npm i --save class-validator class-transformer
```

**TIPP:** Die `ValidationPipe` wird aus dem Paket `@nestjs/common` exportiert.

Da diese Pipe die `class-validator`- und `class-transformer`-Bibliotheken verwendet, stehen viele Optionen zur Verfügung. Sie konfigurieren diese Einstellungen über ein Konfigurationsobjekt, das an die Pipe übergeben wird. Nachfolgend sind die integrierten Optionen aufgeführt:

```typescript
export interface ValidationPipeOptions extends ValidatorOptions {
  transform?: boolean;
  disableErrorMessages?: boolean;
  exceptionFactory?: (errors: ValidationError[]) => any;
}
```

Zusätzlich zu diesen sind alle `class-validator`-Optionen (geerbt von der `ValidatorOptions`-Schnittstelle) verfügbar:

| Option | Typ | Beschreibung |
|---|---|---|
| enableDebugMessages | boolean | Wenn auf true gesetzt, druckt der Validator zusätzliche Warnmeldungen in die Konsole, wenn etwas nicht stimmt. |
| skipUndefinedProperties | boolean | Wenn auf true gesetzt, überspringt der Validator die Validierung aller Eigenschaften, die im zu validierenden Objekt undefiniert sind. |
| skipNullProperties | boolean | Wenn auf true gesetzt, überspringt der Validator die Validierung aller Eigenschaften, die im zu validierenden Objekt null sind. |
| skipMissingProperties | boolean | Wenn auf true gesetzt, überspringt der Validator die Validierung aller Eigenschaften, die im zu validierenden Objekt null oder undefiniert sind. |
| whitelist | boolean | Wenn auf true gesetzt, entfernt der Validator alle Eigenschaften aus dem validierten (zurückgegebenen) Objekt, die keine Validierungsdekorationen verwenden. |
| forbidNonWhitelisted | boolean | Wenn auf true gesetzt, wirft der Validator eine Ausnahme, anstatt nicht auf der Whitelist stehende Eigenschaften zu entfernen. |
| forbidUnknownValues | boolean | Wenn auf true gesetzt, schlägt der Versuch, unbekannte Objekte zu validieren, sofort fehl. |
| disableErrorMessages | boolean | Wenn auf true gesetzt, werden Validierungsfehler nicht an den Client zurückgegeben. |
| errorHttpStatusCode | number | Diese Einstellung ermöglicht es Ihnen, den Ausnahme-Typ zu spezifizieren, der im Fehlerfall verwendet wird. Standardmäßig wird `BadRequestException` geworfen. |
| exceptionFactory | Funktion | Nimmt ein Array der Validierungsfehler und gibt ein Ausnahmeobjekt zurück, das geworfen werden soll. |
| groups | string[] | Gruppen, die bei der Validierung des Objekts verwendet werden sollen. |
| always | boolean | Setzt den Standard für die always-Option der Dekoratoren. Der Standard kann in den Dekorator-Optionen überschrieben werden. |
| strictGroups | boolean | Wenn keine Gruppen angegeben sind oder leer sind, ignorieren Sie Dekoratoren mit mindestens einer Gruppe. |
| dismissDefaultMessages | boolean | Wenn auf true gesetzt, werden die Standardmeldungen der Validierung nicht verwendet. Fehlermeldungen sind immer undefiniert, wenn sie nicht explizit gesetzt sind. |
| validationError.target | boolean | Gibt an, ob das Ziel im `ValidationError` angezeigt werden soll. |
| validationError.value | boolean | Gibt an, ob der validierte Wert im `ValidationError` angezeigt werden soll. |
| stopAtFirstError | boolean | Wenn auf true gesetzt, wird die Validierung der angegebenen Eigenschaft nach dem ersten Fehler gestoppt. Standardmäßig auf false gesetzt. |

**HINWEIS:** Weitere Informationen zum `class-validator`-Paket finden Sie in seinem Repository.

## Automatische Validierung / Auto-validation

Wir beginnen damit, die `ValidationPipe` auf Anwendungsebene zu binden, um sicherzustellen, dass alle Endpunkte vor dem Empfang falscher Daten geschützt sind:

```typescript
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalPipes(new ValidationPipe());
  await app.listen(3000);
}
bootstrap();
```

Um unsere Pipe zu testen, erstellen wir einen einfachen Endpunkt:

```typescript
@Post()
create(@Body() createUserDto: CreateUserDto) {
  return 'This action adds a new user';
}
```

**TIPP:** Da TypeScript keine Metadaten über Generika oder Schnittstellen speichert, kann `ValidationPipe` möglicherweise keine eingehenden Daten richtig validieren, wenn Sie sie in Ihren DTOs verwenden. Verwenden Sie aus diesem Grund konkrete Klassen in Ihren DTOs.

**TIPP:** Wenn Sie Ihre DTOs importieren, verwenden Sie keinen reinen Typ-Import, da dieser zur Laufzeit gelöscht wird. Importieren Sie z.B. `import { CreateUserDto }` anstelle von `import type { CreateUserDto }`.

Nun können wir einige Validierungsregeln in unserem `CreateUserDto` hinzufügen. Dies tun wir mit den Dekoratoren des `class-validator`-Pakets, wie hier beschrieben. Auf diese Weise wird jede Route, die das `CreateUserDto` verwendet, diese Validierungsregeln automatisch durchsetzen:

```typescript
import { IsEmail, IsNotEmpty } from 'class-validator';

export class CreateUserDto {
  @IsEmail()
  email: string;

  @IsNotEmpty()
  password: string;
}
```

Mit diesen Regeln wird die Anwendung automatisch mit einem 400 Bad Request-Code und dem folgenden Antwortkörper antworten, wenn eine Anfrage unseren Endpunkt mit einer ungültigen `email`-Eigenschaft im Anforderungstext erreicht:

```json
{
  "statusCode": 400,
  "error": "Bad Request",
  "message": ["email must be an email"]
}
```

Neben der Validierung von Anforderungstexten kann die `ValidationPipe` auch mit anderen Anfrageobjekteigenschaften verwendet werden. Stellen Sie sich vor, dass wir :id im Endpunktpfad akzeptieren möchten. Um sicherzustellen, dass nur Zahlen für diesen Anforderungsparameter akzeptiert werden, können wir die folgende Konstruktion verwenden:

```typescript
@Get(':id')
findOne(@Param() params: FindOneParams) {
  return 'This action returns a user';
}
```

`FindOneParams`, wie ein DTO, ist einfach eine Klasse, die Validierungsregeln mit `class-validator` definiert. Es würde so aussehen:

```typescript
import { IsNumberString } from 'class-validator';

export class FindOneParams {
  @IsNumberString()
  id: number;
}
```

## Detaillierte Fehler deaktivieren / Disable detailed errors

Fehlermeldungen können hilfreich sein, um zu erklären, was in einer Anfrage falsch war. Einige Produktionsumgebungen ziehen es jedoch vor, detaillierte Fehler zu deaktivieren. Dies kann durch Übergeben eines Optionsobjekts an die `ValidationPipe` erreicht werden:

```typescript
app.useGlobalPipes(
  new ValidationPipe({
    disableErrorMessages: true,
  }),
);
```

Infolgedessen werden detaillierte Fehlermeldungen nicht im Antwortkörper angezeigt.

## Eigenschaften filtern / Stripping properties

Unsere `ValidationPipe` kann auch Eigenschaften herausfiltern, die nicht vom Methoden-Handler empfangen werden sollen. In diesem Fall können wir die akzeptablen Eigenschaften auf die Whitelist setzen, und jede Eigenschaft, die nicht in der Whitelist enthalten ist, wird automatisch aus dem resultierenden Objekt entfernt. Wenn unser Handler beispielsweise `email` und `password`-Eigenschaften erwartet, eine Anfrage jedoch auch eine `age`-Eigenschaft enthält, kann diese Eigenschaft automatisch aus dem resultierenden DTO entfernt werden. Um ein solches Verhalten zu ermöglichen, setzen Sie `whitelist` auf `true`:

```typescript
app.useGlobalPipes(
  new ValidationPipe({
    whitelist: true,
  }),
);
```

Wenn `whitelist` auf `true` gesetzt ist, werden automatisch alle nicht in der Whitelist enthaltenen Eigenschaften (die keine Dekoratoren in der Validierungsklasse haben) entfernt.

Alternativ können Sie die Verarbeitung der Anfrage stoppen, wenn nicht auf der Whitelist stehende Eigenschaften vorhanden sind, und eine Fehlermeldung an den Benutzer zurückgeben. Um dies zu ermöglichen, setzen Sie die Eigenschaft `forbidNonWhitelisted` auf `true`, in Kombination mit `whitelist` auf `true` gesetzt.

## Transformieren von Nutzlastobjekten / Transform payload objects

Nutzlasten, die über das Netzwerk kommen, sind einfache JavaScript-Objekte

. Die `ValidationPipe` kann Nutzlasten automatisch in Objekte umwandeln, die gemäß ihren DTO-Klassen typisiert sind. Um die automatische Transformation zu aktivieren, setzen Sie `transform` auf `true`. Dies kann auf Methodenebene erfolgen:

```typescript
@Post()
@UsePipes(new ValidationPipe({ transform: true }))
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
```

Um dieses Verhalten global zu aktivieren, setzen Sie die Option auf eine globale Pipe:

```typescript
app.useGlobalPipes(
  new ValidationPipe({
    transform: true,
  }),
);
```

Mit der aktivierten Auto-Transformation-Option wird die `ValidationPipe` auch die Umwandlung primitiver Typen durchführen. Im folgenden Beispiel nimmt die `findOne()`-Methode ein Argument, das einen extrahierten `id`-Pfadparameter darstellt:

```typescript
@Get(':id')
findOne(@Param('id') id: number) {
  console.log(typeof id === 'number'); // true
  return 'This action returns a user';
}
```

Standardmäßig kommt jeder Pfadparameter und jeder Abfrageparameter als Zeichenkette über das Netzwerk. Im obigen Beispiel haben wir den `id`-Typ als `number` (in der Methodensignatur) angegeben. Daher wird die `ValidationPipe` versuchen, einen Zeichenketten-Identifikator automatisch in eine Zahl umzuwandeln.

## Explizite Umwandlung / Explicit conversion

Im obigen Abschnitt haben wir gezeigt, wie die `ValidationPipe` Abfrage- und Pfadparameter basierend auf dem erwarteten Typ implizit umwandeln kann. Diese Funktion erfordert jedoch, dass die automatische Transformation aktiviert ist.

Alternativ (mit deaktivierter automatischer Transformation) können Sie Werte explizit mithilfe der `ParseIntPipe` oder `ParseBoolPipe` umwandeln (beachten Sie, dass `ParseStringPipe` nicht erforderlich ist, da, wie bereits erwähnt, jeder Pfadparameter und Abfrageparameter standardmäßig als Zeichenkette über das Netzwerk kommt):

```typescript
@Get(':id')
findOne(
  @Param('id', ParseIntPipe) id: number,
  @Query('sort', ParseBoolPipe) sort: boolean,
) {
  console.log(typeof id === 'number'); // true
  console.log(typeof sort === 'boolean'); // true
  return 'This action returns a user';
}
```

**TIPP:** Die `ParseIntPipe` und `ParseBoolPipe` werden aus dem Paket `@nestjs/common` exportiert.

## Abgeleitete Typen / Mapped types

Beim Aufbau von Funktionen wie CRUD (Create/Read/Update/Delete) ist es oft nützlich, Varianten eines Basistypen zu erstellen. Nest bietet mehrere Dienstprogramme, die Typumwandlungen durchführen, um diese Aufgabe zu erleichtern.

**WARNUNG:** Wenn Ihre Anwendung das Paket `@nestjs/swagger` verwendet, lesen Sie dieses Kapitel für weitere Informationen zu abgeleiteten Typen. Ebenso, wenn Sie das Paket `@nestjs/graphql` verwenden, lesen Sie dieses Kapitel. Beide Pakete verlassen sich stark auf Typen und erfordern daher einen anderen Import. Wenn Sie also `@nestjs/mapped-types` verwendet haben (anstatt das entsprechende, entweder `@nestjs/swagger` oder `@nestjs/graphql` je nach Art Ihrer App), könnten Sie auf verschiedene, undokumentierte Nebeneffekte stoßen.

Beim Erstellen von Eingabevalidierungstypen (auch DTOs genannt) ist es oft nützlich, Erstellungs- und Aktualisierungsvarianten desselben Typs zu erstellen. Zum Beispiel kann die Erstellung aller Felder erfordern, während die Aktualisierung alle Felder optional machen kann.

Nest bietet die `PartialType()`-Dienstfunktion, um diese Aufgabe zu erleichtern und Boilerplate zu minimieren.

Die `PartialType()`-Funktion gibt einen Typ (Klasse) zurück, bei dem alle Eigenschaften des Eingabetypen auf optional gesetzt sind. Angenommen, wir haben einen Erstelltyp wie folgt:

```typescript
export class CreateCatDto {
  name: string;
  age: number;
  breed: string;
}
```

Standardmäßig sind alle diese Felder erforderlich. Um einen Typ zu erstellen, bei dem dieselben Felder, aber alle optional sind, verwenden Sie `PartialType()`, indem Sie die Klassenreferenz (`CreateCatDto`) als Argument übergeben:

```typescript
export class UpdateCatDto extends PartialType(CreateCatDto) {}
```

**TIPP:** Die `PartialType()`-Funktion wird aus dem Paket `@nestjs/mapped-types` importiert.

Die `PickType()`-Funktion erstellt einen neuen Typ (Klasse), indem eine Reihe von Eigenschaften aus einem Eingabetyp ausgewählt werden. Angenommen, wir beginnen mit einem Typ wie:

```typescript
export class CreateCatDto {
  name: string;
  age: number;
  breed: string;
}
```

Wir können eine Reihe von Eigenschaften aus dieser Klasse mithilfe der `PickType()`-Dienstfunktion auswählen:

```typescript
export class UpdateCatAgeDto extends PickType(CreateCatDto, ['age'] as const) {}
```

**TIPP:** Die `PickType()`-Funktion wird aus dem Paket `@nestjs/mapped-types` importiert.

Die `OmitType()`-Funktion erstellt einen Typ, indem alle Eigenschaften aus einem Eingabetyp ausgewählt und dann ein bestimmter Satz von Schlüsseln entfernt wird. Angenommen, wir beginnen mit einem Typ wie:

```typescript
export class CreateCatDto {
  name: string;
  age: number;
  breed: string;
}
```

Wir können einen abgeleiteten Typ erstellen, der jede Eigenschaft außer `name` hat, wie unten gezeigt. In dieser Konstruktion ist das zweite Argument von `OmitType` ein Array von Eigenschaftsnamen:

```typescript
export class UpdateCatDto extends OmitType(CreateCatDto, ['name'] as const) {}
```

**TIPP:** Die `OmitType()`-Funktion wird aus dem Paket `@nestjs/mapped-types` importiert.

Die `IntersectionType()`-Funktion kombiniert zwei Typen zu einem neuen Typ (Klasse). Angenommen, wir beginnen mit zwei Typen wie:

```typescript
export class CreateCatDto {
  name: string;
  breed: string;
}

export class AdditionalCatInfo {
  color: string;
}
```

Wir können einen neuen Typ erstellen, der alle Eigenschaften in beiden Typen kombiniert:

```typescript
export class UpdateCatDto extends IntersectionType(
  CreateCatDto,
  AdditionalCatInfo,
) {}
```

**TIPP:** Die `IntersectionType()`-Funktion wird aus dem Paket `@nestjs/mapped-types` importiert.

Die Dienstfunktionen zur Typabbildung sind kombinierbar. Zum Beispiel wird das Folgende einen Typ (Klasse) erzeugen, der alle Eigenschaften des `CreateCatDto`-Typs außer `name` hat, und diese Eigenschaften werden auf optional gesetzt:

```typescript
export class UpdateCatDto extends PartialType(
  OmitType(CreateCatDto, ['name'] as const),
) {}
```

## Parsen und Validieren von Arrays / Parsing and validating arrays

TypeScript speichert keine Metadaten über Generika oder Schnittstellen, daher kann `ValidationPipe` möglicherweise keine eingehenden Daten richtig validieren, wenn Sie sie in Ihren DTOs verwenden. Zum Beispiel wird `createUserDtos` im folgenden Code nicht korrekt validiert:

```typescript
@Post()
createBulk(@Body() createUserDtos: CreateUserDto[]) {
  return 'This action adds new users';
}
```

Um das Array zu validieren, erstellen Sie eine dedizierte Klasse, die eine Eigenschaft enthält, die das Array umschließt, oder verwenden Sie die `ParseArrayPipe`.

```typescript
@Post()
createBulk(
  @Body(new ParseArrayPipe({ items: CreateUserDto }))
  createUserDtos: CreateUserDto[],
) {
  return 'This action adds new users';
}
```

Darüber hinaus kann die `ParseArrayPipe` nützlich sein, wenn Abfrageparameter geparst werden. Betrachten wir eine `findByIds()`-Methode, die Benutzer basierend auf den als Abfrageparameter übergebenen Identifikatoren zurückgibt:

```typescript
@Get()
findByIds(
  @Query('ids', new ParseArrayPipe({ items: Number, separator: ',' }))
  ids: number[],
) {
  return 'This action returns users by ids';
}
```

Diese Konstruktion validiert die eingehenden Abfrageparameter von einer HTTP-GET-Anfrage wie der folgenden:

```
GET /?ids=1,2,3
```

## WebSockets und Microservices / WebSockets and Microservices

Während dieses Kapitel Beispiele mit HTTP-Stil-Anwendungen (z.B. Express oder Fastify) zeigt, funktioniert die `ValidationPipe` für WebSockets und Microservices auf die gleiche Weise, unabhängig von der verwendeten Transportmethode.

## Mehr erfahren / Learn more

Lesen Sie mehr über benutzerdefinierte Validatoren, Fehlermeldungen und verfügbare Dekoratoren, die vom `class-validator`-Paket bereitgestellt werden, [hier](https://github.com/typestack/class-validator).
