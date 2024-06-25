## Pipes

Ein Pipe ist eine Klasse, die mit dem `@Injectable()`-Dekorator annotiert ist und das `PipeTransform`-Interface implementiert.

![Pipe](Pipe_1.png)

### Pipes haben zwei typische Anwendungsfälle:

1. **Transformation**: Eingabedaten in die gewünschte Form transformieren (z. B. von String zu Integer)
2. **Validierung**: Eingabedaten evaluieren und bei Gültigkeit unverändert durchlassen; andernfalls eine Ausnahme auslösen

In beiden Fällen arbeiten Pipes an den Argumenten, die von einem Controller-Routen-Handler verarbeitet werden. Nest setzt einen Pipe unmittelbar vor der Ausführung einer Methode ein, und der Pipe erhält die für die Methode bestimmten Argumente und arbeitet an ihnen. Jede Transformations- oder Validierungsoperation erfolgt zu diesem Zeitpunkt, danach wird der Routen-Handler mit den (möglicherweise) transformierten Argumenten aufgerufen.

Nest bietet eine Reihe von eingebauten Pipes, die sofort verwendet werden können. Es ist auch möglich, eigene benutzerdefinierte Pipes zu erstellen. In diesem Kapitel werden die eingebauten Pipes vorgestellt und gezeigt, wie man sie an Routen-Handler bindet. Anschließend betrachten wir einige benutzerdefinierte Pipes, um zu zeigen, wie man einen von Grund auf neu erstellen kann.

### Hinweis

Pipes laufen innerhalb der Ausnahmezone. Dies bedeutet, dass eine von einem Pipe ausgelöste Ausnahme von der Ausnahmeschicht (globaler Ausnahmefilter und alle auf den aktuellen Kontext angewendeten Ausnahmefilter) behandelt wird. Daraus ergibt sich, dass beim Auslösen einer Ausnahme in einem Pipe keine Controllermethode ausgeführt wird. Dies bietet eine bewährte Technik zur Validierung von Daten, die von externen Quellen an die Anwendung übermittelt werden.

### Eingebaute Pipes / Built-in pipes

Nest bietet neun sofort verwendbare Pipes:

- `ValidationPipe`
- `ParseIntPipe`
- `ParseFloatPipe`
- `ParseBoolPipe`
- `ParseArrayPipe`
- `ParseUUIDPipe`
- `ParseEnumPipe`
- `DefaultValuePipe`
- `ParseFilePipe`

Diese werden aus dem `@nestjs/common`-Paket exportiert.

Ein Beispiel für die Verwendung des `ParseIntPipe`. Dies ist ein Beispiel für den Transformationsanwendungsfall, bei dem der Pipe sicherstellt, dass ein Methodenhandler-Parameter in einen JavaScript-Integer konvertiert wird (oder eine Ausnahme auslöst, wenn die Konvertierung fehlschlägt). Später in diesem Kapitel zeigen wir eine einfache benutzerdefinierte Implementierung eines `ParseIntPipe`. Die unten gezeigten Beispieltechniken gelten auch für die anderen eingebauten Transformationspipes (`ParseBoolPipe`, `ParseFloatPipe`, `ParseEnumPipe`, `ParseArrayPipe` und `ParseUUIDPipe`, die wir in diesem Kapitel als `Parse*`-Pipes bezeichnen).

### Pipes binden / Binding pipes

Um einen Pipe zu verwenden, müssen wir eine Instanz der Pipe-Klasse an den entsprechenden Kontext binden. In unserem `ParseIntPipe`-Beispiel möchten wir den Pipe mit einer bestimmten Routen-Handler-Methode verknüpfen und sicherstellen, dass er vor dem Aufruf der Methode ausgeführt wird. Wir tun dies mit der folgenden Konstruktion, die wir als Binden des Pipes auf Methodenparameter-Ebene bezeichnen:

```typescript
@Get(':id')
async findOne(@Param('id', ParseIntPipe) id: number) {
  return this.catsService.findOne(id);
}
```

Dies stellt sicher, dass eine der folgenden zwei Bedingungen zutrifft: Entweder ist der Parameter, den wir in der `findOne()`-Methode erhalten, eine Zahl (wie erwartet in unserem Aufruf von `this.catsService.findOne()`), oder eine Ausnahme wird ausgelöst, bevor der Routen-Handler aufgerufen wird.

Zum Beispiel, wenn die Route wie folgt aufgerufen wird:

```
GET localhost:3000/abc
```

wird Nest eine Ausnahme auslösen wie diese:

```json
{
  "statusCode": 400,
  "message": "Validation failed (numeric string is expected)",
  "error": "Bad Request"
}
```

Die Ausnahme verhindert, dass der Rumpf der `findOne()`-Methode ausgeführt wird.

Im obigen Beispiel übergeben wir eine Klasse (`ParseIntPipe`), nicht eine Instanz, wodurch die Verantwortung für die Instanziierung dem Framework überlassen wird und die Abhängigkeitsinjektion ermöglicht wird. Wie bei Pipes und Guards können wir stattdessen auch eine sofortige Instanz übergeben. Das Übergeben einer sofortigen Instanz ist nützlich, wenn wir das Verhalten des eingebauten Pipes durch Übergeben von Optionen anpassen möchten:

```typescript
@Get(':id')
async findOne(
  @Param('id', new ParseIntPipe({ errorHttpStatusCode: HttpStatus.NOT_ACCEPTABLE }))
  id: number,
) {
  return this.catsService.findOne(id);
}
```

Das Binden der anderen Transformationspipes (alle `Parse*`-Pipes) funktioniert ähnlich. Diese Pipes arbeiten im Kontext der Validierung von Routenparametern, Abfragezeichenfolgenparametern und Anforderungskörperwerten.

Ein Beispiel mit einem Abfragezeichenfolgenparameter:

```typescript
@Get()
async findOne(@Query('id', ParseIntPipe) id: number) {
  return this.catsService.findOne(id);
}
```

Hier ein Beispiel für die Verwendung des `ParseUUIDPipe`, um einen String-Parameter zu parsen und zu validieren, ob es sich um eine UUID handelt:

```typescript
@Get(':uuid')
async findOne(@Param('uuid', new ParseUUIDPipe()) uuid: string) {
  return this.catsService.findOne(uuid);
}
```

### Hinweis

Beim Verwenden des `ParseUUIDPipe()` parsen Sie UUID in Version 3, 4 oder 5. Wenn Sie nur eine bestimmte Version von UUID benötigen, können Sie eine Version in den Pipe-Optionen angeben.

Oben haben wir Beispiele für das Binden der verschiedenen `Parse*`-Familien von eingebauten Pipes gesehen. Das Binden von Validierungspipes ist etwas anders; wir werden das im folgenden Abschnitt besprechen.

### Hinweis

Siehe auch Validierungstechniken für umfangreiche Beispiele von Validierungspipes.

### Benutzerdefinierte Pipes / Custom pipes

Wie bereits erwähnt, können Sie Ihre eigenen benutzerdefinierten Pipes erstellen. Obwohl Nest einen robusten eingebauten `ParseIntPipe` und `ValidationPipe` bietet, lassen Sie uns einfache benutzerdefinierte Versionen von jedem von Grund auf neu erstellen, um zu sehen, wie benutzerdefinierte Pipes konstruiert werden.

Wir beginnen mit einem einfachen `ValidationPipe`. Zunächst soll er einfach einen Eingabewert nehmen und sofort denselben Wert zurückgeben, was wie eine Identitätsfunktion funktioniert.

```typescript
// validation.pipe.ts

import { PipeTransform, Injectable, ArgumentMetadata } from '@nestjs/common';

@Injectable()
export class ValidationPipe implements PipeTransform {
  transform(value: any, metadata: ArgumentMetadata) {
    return value;
  }
}
```

### Hinweis

`PipeTransform<T, R>` ist ein generisches Interface, das von jedem Pipe implementiert werden muss. Das generische Interface verwendet `T`, um den Typ des Eingabewerts anzugeben, und `R`, um den Rückgabetyp der `transform()`-Methode anzugeben.

Jeder Pipe muss die `transform()`-Methode implementieren, um den `PipeTransform`-Interfacevertrag zu erfüllen. Diese Methode hat zwei Parameter:

- `value`
- `metadata`

Der `value`-Parameter ist das aktuell verarbeitete Methodenargument (bevor es von der Routen-Handling-Methode empfangen wird), und `metadata` sind die Metadaten des aktuell verarbeiteten Methodenarguments. Das Metadatenobjekt hat diese Eigenschaften:

```typescript
export interface ArgumentMetadata {
  type: 'body' | 'query' | 'param' | 'custom';
  metatype?: Type<unknown>;
  data?: string;
}
```

Diese Eigenschaften beschreiben das aktuell verarbeitete Argument:

- `type`: Gibt an, ob es sich bei dem Argument um einen Body `@Body()`, eine Abfrage `@Query()`, ein Parameter `@Param()` oder ein benutzerdefiniertes Parameter (mehr dazu hier) handelt.
- `metatype`: Bietet den Metatyp des Arguments, zum Beispiel `String`. Hinweis: Der Wert ist `undefined`, wenn Sie entweder eine Typdeklaration in der Signatur der Routen-Handler-Methode weglassen oder normales JavaScript verwenden.
- `data`: Der an den Dekorator übergebene String, zum Beispiel `@Body('string')`. Es ist `undefined`, wenn Sie die Klammern des Dekorators leer lassen.

### Warnung

TypeScript-Interfaces verschwinden während der Transpilation. Wenn der Typ eines Methodenparameters als Interface statt als Klasse deklariert wird, ist der `metatype`-Wert `Object`.

### Schema-basierte Validierung / Schema based validation

Machen wir unseren `ValidationPipe` etwas nützlicher. Betrachten Sie die `create()`-Methode des `CatsController` genauer, bei der wir wahrscheinlich sicherstellen möchten, dass das Post-Body-Objekt gültig ist, bevor wir versuchen, unsere Servicemethode auszuführen.

```typescript
@Post()
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
```

Konzentrieren wir uns auf den `createCatDto`-Body-Parameter. Sein Typ ist `CreateCatDto`:

```typescript
// create-cat.dto.ts

export class CreateCatDto {
  name: string;
  age: number;
  breed: string;
}
```

Wir

 möchten sicherstellen, dass jede eingehende Anfrage an die `create`-Methode einen gültigen Body enthält. Also müssen wir die drei Mitglieder des `createCatDto`-Objekts validieren. Wir könnten dies innerhalb der Routen-Handler-Methode tun, aber das wäre nicht ideal, da es das Single Responsibility Principle (SRP) verletzen würde.

Eine andere Möglichkeit wäre, eine Validator-Klasse zu erstellen und die Aufgabe dorthin zu delegieren. Dies hat den Nachteil, dass wir daran denken müssten, diesen Validator am Anfang jeder Methode aufzurufen.

Wie wäre es mit der Erstellung von Validierungsmiddleware? Dies könnte funktionieren, aber leider ist es nicht möglich, generische Middleware zu erstellen, die in allen Kontexten in der gesamten Anwendung verwendet werden kann. Dies liegt daran, dass Middleware den Ausführungskontext, einschließlich des aufgerufenen Handlers und aller seiner Parameter, nicht kennt.

Dies ist natürlich genau der Anwendungsfall, für den Pipes entwickelt wurden. Lassen Sie uns also fortfahren und unseren `ValidationPipe` verfeinern.

### Objekt-Schema-Validierung / Object schema validation

Es gibt mehrere Ansätze zur Validierung von Objekten auf eine saubere, DRY-Art. Ein üblicher Ansatz ist die Verwendung von schema-basierter Validierung. Lassen Sie uns diesen Ansatz ausprobieren.

Die Zod-Bibliothek ermöglicht es Ihnen, Schemata auf einfache Weise zu erstellen, mit einer lesbaren API. Lassen Sie uns einen `ValidationPipe` erstellen, der Zod-basierte Schemata verwendet.

Beginnen Sie mit der Installation des erforderlichen Pakets:

```bash
$ npm install --save zod
```

Im folgenden Codebeispiel erstellen wir eine einfache Klasse, die ein Schema als Konstruktorargument annimmt. Wir wenden dann die Methode `schema.parse()` an, die unser eingehendes Argument gegen das bereitgestellte Schema validiert.

Wie bereits erwähnt, gibt ein `ValidationPipe` entweder den Wert unverändert zurück oder löst eine Ausnahme aus.

Im nächsten Abschnitt sehen Sie, wie wir das entsprechende Schema für eine bestimmte Controllermethode mithilfe des `@UsePipes()`-Dekorators bereitstellen. Auf diese Weise wird unser `ValidationPipe` wiederverwendbar über Kontexte hinweg, genau wie wir es uns vorgenommen haben.

```typescript
import { PipeTransform, ArgumentMetadata, BadRequestException } from '@nestjs/common';
import { ZodSchema  } from 'zod';

export class ZodValidationPipe implements PipeTransform {
  constructor(private schema: ZodSchema) {}

  transform(value: unknown, metadata: ArgumentMetadata) {
    try {
      const parsedValue = this.schema.parse(value);
      return parsedValue;
    } catch (error) {
      throw new BadRequestException('Validation failed');
    }
  }
}
```

### Binden von Validierungspipes / Binding validation pipes

Früher haben wir gesehen, wie man Transformationspipes (wie `ParseIntPipe` und den Rest der `Parse*`-Pipes) bindet.

Das Binden von Validierungspipes ist ebenfalls sehr einfach.

In diesem Fall möchten wir den Pipe auf der Methodenaufrufebene binden. In unserem aktuellen Beispiel müssen wir Folgendes tun, um den `ZodValidationPipe` zu verwenden:

1. Erstellen Sie eine Instanz des `ZodValidationPipe`
2. Übergeben Sie das kontextspezifische Zod-Schema im Klassenkonstruktor des Pipes
3. Binden Sie den Pipe an die Methode

Beispiel eines Zod-Schemas:

```typescript
import { z } from 'zod';

export const createCatSchema = z
  .object({
    name: z.string(),
    age: z.number(),
    breed: z.string(),
  })
  .required();

export type CreateCatDto = z.infer<typeof createCatSchema>;
```

Wir tun dies mit dem `@UsePipes()`-Dekorator, wie unten gezeigt:

```typescript
// cats.controller.ts

@Post()
@UsePipes(new ZodValidationPipe(createCatSchema))
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
```

### Hinweis

Der `@UsePipes()`-Dekorator wird aus dem `@nestjs/common`-Paket importiert.

### Warnung

Die Zod-Bibliothek erfordert die Aktivierung der `strictNullChecks`-Konfiguration in Ihrer `tsconfig.json`-Datei.

### Klassenvalidierung / Class validator

### Warnung

Die Techniken in diesem Abschnitt erfordern TypeScript und sind nicht verfügbar, wenn Ihre App mit normalem JavaScript geschrieben ist.

Betrachten wir eine alternative Implementierung für unsere Validierungstechnik.

Nest funktioniert gut mit der `class-validator`-Bibliothek. Diese leistungsstarke Bibliothek ermöglicht es Ihnen, dekoratorbasierte Validierung zu verwenden. Dekoratorbasierte Validierung ist besonders mächtig, wenn sie mit den Pipe-Fähigkeiten von Nest kombiniert wird, da wir Zugriff auf den Metatyp der verarbeiteten Eigenschaft haben. Bevor wir beginnen, müssen wir die erforderlichen Pakete installieren:

```bash
$ npm i --save class-validator class-transformer
```

Sobald diese installiert sind, können wir einige Dekoratoren zur `CreateCatDto`-Klasse hinzufügen. Hier sehen wir einen wesentlichen Vorteil dieser Technik: Die `CreateCatDto`-Klasse bleibt die einzige Quelle der Wahrheit für unser Post-Body-Objekt (anstatt eine separate Validierungsklasse erstellen zu müssen).

```typescript
// create-cat.dto.ts

import { IsString, IsInt } from 'class-validator';

export class CreateCatDto {
  @IsString()
  name: string;

  @IsInt()
  age: number;

  @IsString()
  breed: string;
}
```

### Hinweis

Lesen Sie hier mehr über die `class-validator`-Dekoratoren.

Jetzt können wir eine `ValidationPipe`-Klasse erstellen, die diese Anmerkungen verwendet.

```typescript
// validation.pipe.ts

import { PipeTransform, Injectable, ArgumentMetadata, BadRequestException } from '@nestjs/common';
import { validate } from 'class-validator';
import { plainToInstance } from 'class-transformer';

@Injectable()
export class ValidationPipe implements PipeTransform<any> {
  async transform(value: any, { metatype }: ArgumentMetadata) {
    if (!metatype || !this.toValidate(metatype)) {
      return value;
    }
    const object = plainToInstance(metatype, value);
    const errors = await validate(object);
    if (errors.length > 0) {
      throw new BadRequestException('Validation failed');
    }
    return value;
  }

  private toValidate(metatype: Function): boolean {
    const types: Function[] = [String, Boolean, Number, Array, Object];
    return !types.includes(metatype);
  }
}
```

### Hinweis

Zur Erinnerung: Sie müssen keine generische Validierungspipe selbst erstellen, da die `ValidationPipe` von Nest sofort verfügbar ist. Die eingebaute `ValidationPipe` bietet mehr Optionen als das Beispiel, das wir in diesem Kapitel erstellt haben, das aus Gründen der Veranschaulichung der Mechanik eines benutzerdefinierten Pipes einfach gehalten wurde. Sie finden alle Details sowie viele Beispiele [hier](https://docs.nestjs.com/pipes).

### Hinweis

Wir haben oben die Bibliothek `class-transformer` verwendet, die vom selben Autor wie die Bibliothek `class-validator` erstellt wurde, und daher spielen sie sehr gut zusammen.

Gehen wir den Code durch. Beachten Sie zunächst, dass die `transform()`-Methode als `async` markiert ist. Dies ist möglich, da Nest sowohl synchrone als auch asynchrone Pipes unterstützt. Wir machen diese Methode asynchron, da einige der `class-validator`-Validierungen asynchron sein können (nutzen Promises).

Beachten Sie als Nächstes, dass wir Destructuring verwenden, um das `metatype`-Feld (nur dieses Mitglied aus einem `ArgumentMetadata` extrahierend) in unseren `metatype`-Parameter zu extrahieren. Dies ist nur eine Abkürzung, um die vollständigen `ArgumentMetadata` zu erhalten und dann eine zusätzliche Anweisung zu haben, um die `metatype`-Variable zuzuweisen.

Beachten Sie als Nächstes die Hilfsfunktion `toValidate()`. Sie ist dafür verantwortlich, den Validierungsschritt zu umgehen, wenn das aktuell verarbeitete Argument ein nativer JavaScript-Typ ist (diese können keine Validierungsdekorationen haben, daher gibt es keinen Grund, sie durch den Validierungsschritt zu führen).

Als Nächstes verwenden wir die `class-transformer`-Funktion `plainToInstance()`, um unser einfaches JavaScript-Argumentobjekt in ein typisiertes Objekt zu transformieren, damit wir die Validierung anwenden können. Der Grund, warum wir dies tun müssen, ist, dass das eingehende Post-Body-Objekt beim Deserialisieren aus der Netzwerkanfrage keine Typinformationen hat (so funktioniert die zugrunde liegende Plattform, wie z. B. Express). `class-validator` muss die Validierungsdekorationen verwenden, die wir zuvor für unser DTO definiert haben, daher müssen wir diese Transformation durchführen, um den eingehenden Body als ein entsprechend dekoriertes Objekt zu behandeln, nicht nur als einfaches Vanilla-Objekt.

Schließlich, wie bereits erwähnt, gibt dieser Validierungspipe entweder den Wert unverändert zurück oder löst eine Ausnahme aus.

Der letzte Schritt ist das Binden des `ValidationPipe`. Pipes können parameter-, methoden-, controller- oder global-gebunden sein. Früher haben wir mit unserem Zod-basierten Validierungspipe ein Beispiel für das Binden des

 Pipes auf Methodenebene gesehen. Im folgenden Beispiel werden wir die Pipe-Instanz an den Routen-Handler-`@Body()`-Dekorator binden, damit unser Pipe aufgerufen wird, um den Post-Body zu validieren.

```typescript
// cats.controller.ts

@Post()
async create(
  @Body(new ValidationPipe()) createCatDto: CreateCatDto,
) {
  this.catsService.create(createCatDto);
}
```

Parametergebundene Pipes sind nützlich, wenn die Validierungslogik nur ein bestimmtes Parameter betrifft.

### Global gebundene Pipes / Global scoped pipes

Da die `ValidationPipe` so generisch wie möglich erstellt wurde, können wir ihre volle Nützlichkeit realisieren, indem wir sie als global-gebundenen Pipe einrichten, sodass sie auf jeden Routen-Handler in der gesamten Anwendung angewendet wird.

```typescript
// main.ts

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalPipes(new ValidationPipe());
  await app.listen(3000);
}
bootstrap();
```

### Hinweis

Bei hybriden Apps richtet die Methode `useGlobalPipes()` keine Pipes für Gateways und Microservices ein. Für "normale" (nicht-hybride) Microservice-Apps montiert `useGlobalPipes()` die Pipes global.

Globale Pipes werden in der gesamten Anwendung verwendet, für jeden Controller und jeden Routen-Handler.

Beachten Sie, dass globale Pipes, die von außerhalb eines Moduls (mit `useGlobalPipes()` wie im obigen Beispiel) registriert werden, keine Abhängigkeiten injizieren können, da die Bindung außerhalb des Kontexts eines Moduls erfolgt. Um dieses Problem zu lösen, können Sie einen globalen Pipe direkt aus einem Modul heraus einrichten, indem Sie die folgende Konstruktion verwenden:

```typescript
// app.module.ts

import { Module } from '@nestjs/common';
import { APP_PIPE } from '@nestjs/core';

@Module({
  providers: [
    {
      provide: APP_PIPE,
      useClass: ValidationPipe,
    },
  ],
})
export class AppModule {}
```

### Hinweis

Wenn Sie diesen Ansatz zur Durchführung der Abhängigkeitsinjektion für den Pipe verwenden, beachten Sie, dass der Pipe unabhängig vom Modul, in dem diese Konstruktion verwendet wird, global ist. Wo sollte dies gemacht werden? Wählen Sie das Modul, in dem der Pipe (z. B. `ValidationPipe` im obigen Beispiel) definiert ist. `useClass` ist auch nicht die einzige Möglichkeit, benutzerdefinierte Anbieter zu registrieren. Erfahren Sie mehr darüber [hier](https://docs.nestjs.com/techniques/validation).

### Der eingebaute `ValidationPipe` / The built-in ValidationPipe

Zur Erinnerung: Sie müssen keine generische Validierungspipe selbst erstellen, da die `ValidationPipe` von Nest sofort verfügbar ist. Die eingebaute `ValidationPipe` bietet mehr Optionen als das Beispiel, das wir in diesem Kapitel erstellt haben, das aus Gründen der Veranschaulichung der Mechanik eines benutzerdefinierten Pipes einfach gehalten wurde. Sie finden alle Details sowie viele Beispiele [hier](https://docs.nestjs.com/techniques/validation).

### Transformationsanwendungsfall / Transformation use case

Validierung ist nicht der einzige Anwendungsfall für benutzerdefinierte Pipes. Zu Beginn dieses Kapitels haben wir erwähnt, dass ein Pipe auch die Eingabedaten in das gewünschte Format transformieren kann. Dies ist möglich, da der von der `transform()`-Funktion zurückgegebene Wert den vorherigen Wert des Arguments vollständig überschreibt.

Wann ist dies nützlich? Betrachten Sie, dass die vom Client übermittelten Daten manchmal eine Änderung durchlaufen müssen - zum Beispiel das Konvertieren eines Strings in einen Integer - bevor sie ordnungsgemäß vom Routen-Handler verarbeitet werden können. Darüber hinaus können einige erforderliche Datenfelder fehlen, und wir möchten Standardwerte anwenden. Transformationspipes können diese Funktionen ausführen, indem sie eine Verarbeitungsfunktion zwischen die Clientanforderung und den Anforderungsverarbeitungshandler einfügen.

Hier ist ein einfacher `ParseIntPipe`, der dafür verantwortlich ist, einen String in einen Integerwert zu parsen. (Wie bereits erwähnt, hat Nest einen eingebauten `ParseIntPipe`, der ausgefeilter ist; wir fügen dies als einfaches Beispiel für einen benutzerdefinierten Transformationspipe hinzu).

```typescript
// parse-int.pipe.ts

import { PipeTransform, Injectable, ArgumentMetadata, BadRequestException } from '@nestjs/common';

@Injectable()
export class ParseIntPipe implements PipeTransform<string, number> {
  transform(value: string, metadata: ArgumentMetadata): number {
    const val = parseInt(value, 10);
    if (isNaN(val)) {
      throw new BadRequestException('Validation failed');
    }
    return val;
  }
}
```

Wir können diesen Pipe dann an den ausgewählten Parameter binden, wie unten gezeigt:

```typescript
@Get(':id')
async findOne(@Param('id', new ParseIntPipe()) id) {
  return this.catsService.findOne(id);
}
```

Ein weiterer nützlicher Transformationsfall wäre es, eine bestehende Benutzerentität aus der Datenbank mithilfe einer in der Anfrage übermittelten ID auszuwählen:

```typescript
@Get(':id')
findOne(@Param('id', UserByIdPipe) userEntity: UserEntity) {
  return userEntity;
}
```

Wir überlassen die Implementierung dieses Pipes dem Leser, aber beachten Sie, dass er wie alle anderen Transformationspipes einen Eingabewert (eine ID) erhält und einen Ausgabewert (ein `UserEntity`-Objekt) zurückgibt. Dies kann Ihren Code deklarativer und DRY machen, indem Boilerplate-Code aus Ihrem Handler abstrahiert und in einen gemeinsamen Pipe verschoben wird.

### Bereitstellen von Standardwerten / Providing defaults

`Parse*`-Pipes erwarten, dass ein Parameterwert definiert ist. Sie lösen eine Ausnahme aus, wenn sie `null` oder `undefined`-Werte erhalten. Um einer Endpunktanforderung zu ermöglichen, fehlende Abfragezeichenfolgenparameterwerte zu behandeln, müssen wir einen Standardwert bereitstellen, der vor dem Betrieb der `Parse*`-Pipes injiziert wird. Der `DefaultValuePipe` dient diesem Zweck. Einfach eine `DefaultValuePipe` im `@Query()`-Dekorator vor dem relevanten `Parse*`-Pipe instanziieren, wie unten gezeigt:

```typescript
@Get()
async findAll(
  @Query('activeOnly', new DefaultValuePipe(false), ParseBoolPipe) activeOnly: boolean,
  @Query('page', new DefaultValuePipe(0), ParseIntPipe) page: number,
) {
  return this.catsService.findAll({ activeOnly, page });
}
```