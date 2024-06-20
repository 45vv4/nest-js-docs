### Controller

Controller sind dafür verantwortlich, eingehende Anfragen zu bearbeiten und Antworten an den Client zurückzusenden.

Ein Controller hat die Aufgabe, spezifische Anfragen für die Anwendung zu empfangen. Der Routing-Mechanismus steuert, welcher Controller welche Anfragen erhält. Häufig hat jeder Controller mehr als eine Route, und unterschiedliche Routen können verschiedene Aktionen ausführen.

Um einen grundlegenden Controller zu erstellen, verwenden wir Klassen und Dekoratoren. Dekoratoren verknüpfen Klassen mit den erforderlichen Metadaten und ermöglichen es Nest, eine Routing-Karte zu erstellen (Anfragen den entsprechenden Controllern zuzuordnen).

#### HINWEIS
Um schnell einen CRUD-Controller mit integrierter Validierung zu erstellen, können Sie den CLI-CRUD-Generator verwenden: `nest g resource [name]`.

### Routing

Im folgenden Beispiel verwenden wir den `@Controller()`-Dekorator, der erforderlich ist, um einen grundlegenden Controller zu definieren. Wir werden ein optionales Routenpräfix `cats` angeben. Durch die Verwendung eines Pfadpräfixes in einem `@Controller()`-Dekorator können wir eine Reihe verwandter Routen einfach gruppieren und redundanten Code minimieren. Zum Beispiel könnten wir eine Gruppe von Routen, die Interaktionen mit einer Katzenentität verwalten, unter der Route `/cats` zusammenfassen. In diesem Fall könnten wir das Pfadpräfix `cats` im `@Controller()`-Dekorator angeben, sodass wir diesen Teil des Pfades nicht für jede Route in der Datei wiederholen müssen.

```typescript
import { Controller, Get } from '@nestjs/common';

@Controller('cats')
export class CatsController {
  @Get()
  findAll(): string {
    return 'This action returns all cats';
  }
}
```

#### HINWEIS
Um einen Controller mit der CLI zu erstellen, führen Sie einfach den Befehl `$ nest g controller [name]` aus.

Der `@Get()`-HTTP-Anfragemethodendekorator vor der `findAll()`-Methode teilt Nest mit, einen Handler für einen bestimmten Endpunkt für HTTP-Anfragen zu erstellen. Der Endpunkt entspricht der HTTP-Anfragemethode (in diesem Fall GET) und dem Routenpfad. Was ist der Routenpfad? Der Routenpfad für einen Handler wird durch das Zusammenfügen des (optionalen) Präfixes, das für den Controller deklariert wurde, und einem beliebigen Pfad, der im Dekorator der Methode angegeben wurde, bestimmt. Da wir ein Präfix für jede Route (`cats`) deklariert haben und im Dekorator keine Pfadinformationen hinzugefügt haben, wird Nest GET `/cats`-Anfragen diesem Handler zuordnen. Wie bereits erwähnt, umfasst der Pfad sowohl das optionale Controllerpfadpräfix als auch eine beliebige Pfadzeichenfolge, die im Dekorator der Anfragemethode angegeben ist. Zum Beispiel würde ein Pfadpräfix von `cats` kombiniert mit dem Dekorator `@Get('breed')` eine Routenabbildung für Anfragen wie GET `/cats/breed` erzeugen.

In unserem obigen Beispiel, wenn eine GET-Anfrage an diesen Endpunkt gestellt wird, leitet Nest die Anfrage an unsere benutzerdefinierte `findAll()`-Methode weiter. Beachten Sie, dass der von uns gewählte Methodenname völlig willkürlich ist. Wir müssen natürlich eine Methode deklarieren, an die die Route gebunden werden soll, aber Nest misst dem gewählten Methodennamen keine Bedeutung bei.

Diese Methode wird einen 200-Statuscode und die zugehörige Antwort zurückgeben, die in diesem Fall nur eine Zeichenfolge ist. Warum passiert das? Um dies zu erklären, führen wir zunächst das Konzept ein, dass Nest zwei verschiedene Optionen zur Manipulation von Antworten verwendet:

#### Standard (empfohlen)
Bei Verwendung dieser eingebauten Methode wird, wenn ein Anfragen-Handler ein JavaScript-Objekt oder -Array zurückgibt, es automatisch in JSON serialisiert. Wenn er jedoch einen JavaScript-Primitive-Typ (z. B. Zeichenfolge, Zahl, boolean) zurückgibt, sendet Nest nur den Wert, ohne zu versuchen, ihn zu serialisieren. Dies macht die Antwortverarbeitung einfach: Geben Sie einfach den Wert zurück, und Nest kümmert sich um den Rest.

Darüber hinaus ist der Statuscode der Antwort immer standardmäßig 200, außer bei POST-Anfragen, die 201 verwenden. Wir können dieses Verhalten leicht ändern, indem wir den `@HttpCode(...)`-Dekorator auf Handler-Ebene hinzufügen (siehe Statuscodes).

#### Bibliotheksspezifisch
Wir können das bibliotheksspezifische (z. B. Express) Antwortobjekt verwenden, das mithilfe des `@Res()`-Dekorators in die Methodensignatur des Handlers injiziert werden kann (z. B. `findAll(@Res() response)`). Mit diesem Ansatz haben Sie die Möglichkeit, die nativen Antwortverarbeitungsmethoden zu verwenden, die von diesem Objekt bereitgestellt werden. Beispielsweise können Sie mit Express Antworten unter Verwendung von Code wie `response.status(200).send()` konstruieren.

#### WARNUNG
Nest erkennt, wenn der Handler entweder `@Res()` oder `@Next()` verwendet, was darauf hinweist, dass Sie die bibliotheksspezifische Option gewählt haben. Wenn beide Ansätze gleichzeitig verwendet werden, wird der Standardansatz für diese einzelne Route automatisch deaktiviert und funktioniert nicht mehr wie erwartet. Um beide Ansätze gleichzeitig zu verwenden (zum Beispiel, indem das Antwortobjekt nur zum Setzen von Cookies/Headers injiziert wird, der Rest jedoch dem Framework überlassen wird), müssen Sie die Passthrough-Option auf `true` setzen im `@Res({ passthrough: true })` Dekorator.

### Request-Objekt

Handler benötigen häufig Zugriff auf die Details der Client-Anfrage. Nest bietet Zugriff auf das Anforderungsobjekt der zugrunde liegenden Plattform (standardmäßig Express). Wir können auf das Anforderungsobjekt zugreifen, indem wir Nest anweisen, es zu injizieren, indem wir den `@Req()`-Dekorator zur Signatur des Handlers hinzufügen.

```typescript
import { Controller, Get, Req } from '@nestjs/common';
import { Request } from 'express';

@Controller('cats')
export class CatsController {
  @Get()
  findAll(@Req() request: Request): string {
    return 'This action returns all cats';
  }
}
```

#### HINWEIS
Um von den Typisierungen von Express zu profitieren (wie im obigen Beispiel des `request: Request`-Parameters), installieren Sie das Paket `@types/express`.

Das Anforderungsobjekt repräsentiert die HTTP-Anfrage und verfügt über Eigenschaften für die Abfragezeichenfolge, Parameter, HTTP-Header und den Body der Anfrage (lesen Sie hier mehr). In den meisten Fällen ist es nicht notwendig, diese Eigenschaften manuell zu erfassen. Wir können stattdessen dedizierte Dekoratoren verwenden, wie zum Beispiel `@Body()` oder `@Query()`, die von Haus aus verfügbar sind. Unten ist eine Liste der bereitgestellten Dekoratoren und der einfachen, plattformspezifischen Objekte, die sie darstellen.

```markdown
@Request(), @Req()	req
@Response(), @Res()*	res
@Next()	next
@Session()	req.session
@Param(key?: string)	req.params / req.params[key]
@Body(key?: string)	req.body / req.body[key]
@Query(key?: string)	req.query / req.query[key]
@Headers(name?: string)	req.headers / req.headers[name]
@Ip()	req.ip
@HostParam()	req.hosts
```

* Zur Kompatibilität mit Typisierungen über zugrunde liegende HTTP-Plattformen hinweg (z. B. Express und Fastify) bietet Nest die Dekoratoren `@Res()` und `@Response()` an. `@Res()` ist einfach ein Alias für `@Response()`. Beide setzen direkt die Schnittstelle des nativen plattformspezifischen Antwortobjekts frei. Wenn Sie sie verwenden, sollten Sie auch die Typisierungen für die zugrunde liegende Bibliothek importieren (z. B. `@types/express`), um den vollen Nutzen daraus zu ziehen. Beachten Sie, dass, wenn Sie entweder `@Res()` oder `@Response()` in einen Methoden-Handler injizieren, Nest in den bibliotheksspezifischen Modus für diesen Handler wechselt und Sie für die Verwaltung der Antwort verantwortlich werden. In diesem Fall müssen Sie eine Art von Antwort ausgeben, indem Sie einen Aufruf des Antwortobjekts machen (z. B. `res.json(...)` oder `res.send(...)`), oder der HTTP-Server hängt sich auf.

#### HINWEIS
Um zu erfahren, wie Sie Ihre eigenen benutzerdefinierten Dekoratoren erstellen, besuchen Sie dieses Kapitel.

### Ressourcen

Zuvor haben wir einen Endpunkt definiert, um die Katzenressource (GET-Route) abzurufen. Normalerweise möchten wir auch einen Endpunkt bereitstellen, der neue Datensätze erstellt. Lassen Sie uns den POST-Handler erstellen:

```typescript
import { Controller, Get, Post } from '@nestjs/common';

@Controller('cats')
export class CatsController {
  @Post()
  create(): string {
    return 'This action adds a new cat';
  }

  @Get()
  findAll(): string {
    return 'This action returns all cats';
  }
}
```

So einfach ist das. Nest bietet Dekoratoren für alle Standard-HTTP-Methoden: `@Get()

`, `@Post()`, `@Put()`, `@Delete()`, `@Patch()`, `@Options()` und `@Head()`. Darüber hinaus definiert `@All()` einen Endpunkt, der alle von ihnen behandelt.

### Route-Wildcards

Musterbasierte Routen werden ebenfalls unterstützt. Zum Beispiel wird das Sternchen als Wildcard verwendet und passt zu jeder Kombination von Zeichen.

```typescript
@Get('ab*cd')
findAll() {
  return 'This route uses a wildcard';
}
```

Der `ab*cd`-Routenpfad passt zu `abcd`, `ab_cd`, `abecd` und so weiter. Die Zeichen `?`, `+`, `*` und `()` können in einem Routenpfad verwendet werden und sind Teilmengen ihrer regulären Ausdrucksgegenstücke. Der Bindestrich (`-`) und der Punkt (`.`) werden in stringbasierten Pfaden wörtlich interpretiert.

#### WARNUNG
Eine Wildcard in der Mitte der Route wird nur von Express unterstützt.

### Statuscode

Wie bereits erwähnt, ist der Statuscode der Antwort immer standardmäßig 200, außer bei POST-Anfragen, die 201 verwenden. Wir können dieses Verhalten leicht ändern, indem wir den `@HttpCode(...)`-Dekorator auf Handler-Ebene hinzufügen.

```typescript
@Post()
@HttpCode(204)
create() {
  return 'This action adds a new cat';
}
```

#### HINWEIS
Importieren Sie `HttpCode` aus dem `@nestjs/common`-Paket.

Häufig ist Ihr Statuscode nicht statisch, sondern hängt von verschiedenen Faktoren ab. In diesem Fall können Sie ein bibliotheksspezifisches Antwortobjekt (injiziert mit `@Res()`) verwenden (oder im Fehlerfall eine Ausnahme werfen).

### Header

Um einen benutzerdefinierten Antwort-Header anzugeben, können Sie entweder einen `@Header()`-Dekorator oder ein bibliotheksspezifisches Antwortobjekt verwenden (und direkt `res.header()` aufrufen).

```typescript
@Post()
@Header('Cache-Control', 'none')
create() {
  return 'This action adds a new cat';
}
```

#### HINWEIS
Importieren Sie `Header` aus dem `@nestjs/common`-Paket.

### Weiterleitung

Um eine Antwort zu einer bestimmten URL umzuleiten, können Sie entweder einen `@Redirect()`-Dekorator oder ein bibliotheksspezifisches Antwortobjekt verwenden (und direkt `res.redirect()` aufrufen).

`@Redirect()` nimmt zwei Argumente an, `url` und `statusCode`, beide sind optional. Der Standardwert von `statusCode` ist 302 (Found), wenn weggelassen.

```typescript
@Get()
@Redirect('https://nestjs.com', 301)
```

#### HINWEIS
Manchmal möchten Sie den HTTP-Statuscode oder die Weiterleitungs-URL dynamisch bestimmen. Dies können Sie tun, indem Sie ein Objekt zurückgeben, das der `HttpRedirectResponse`-Schnittstelle (aus `@nestjs/common`) folgt.

Zurückgegebene Werte überschreiben alle Argumente, die dem `@Redirect()`-Dekorator übergeben wurden. Zum Beispiel:

```typescript
@Get('docs')
@Redirect('https://docs.nestjs.com', 302)
getDocs(@Query('version') version) {
  if (version && version === '5') {
    return { url: 'https://docs.nestjs.com/v5/' };
  }
}
```

### Routenparameter

Routen mit statischen Pfaden funktionieren nicht, wenn Sie dynamische Daten als Teil der Anfrage akzeptieren müssen (z. B. GET `/cats/1` um Katze mit der ID 1 zu bekommen). Um Routen mit Parametern zu definieren, können wir Routentoken im Pfad der Route hinzufügen, um den dynamischen Wert an dieser Position in der Anforderungs-URL zu erfassen. Das Routentoken im `@Get()`-Dekorator-Beispiel unten zeigt diese Verwendung. Routenparameter, die auf diese Weise deklariert werden, können mit dem `@Param()`-Dekorator erfasst werden, der der Methodensignatur hinzugefügt werden sollte.

#### HINWEIS
Routen mit Parametern sollten nach allen statischen Pfaden deklariert werden. Dies verhindert, dass die parameterisierten Pfade den Datenverkehr abfangen, der für die statischen Pfade bestimmt ist.

```typescript
@Get(':id')
findOne(@Param() params: any): string {
  console.log(params.id);
  return `This action returns a #${params.id} cat`;
}
```

`@Param()` wird verwendet, um einen Methodenparameter zu dekorieren (params im obigen Beispiel) und macht die Routenparameter als Eigenschaften dieses dekorierten Methodenparameters im Methodenbody verfügbar. Wie im obigen Code zu sehen ist, können wir auf den `id`-Parameter zugreifen, indem wir `params.id` referenzieren. Sie können auch einen bestimmten Parametertoken an den Dekorator übergeben und dann den Routenparameter direkt nach Namen im Methodenbody referenzieren.

#### HINWEIS
Importieren Sie `Param` aus dem `@nestjs/common`-Paket.

```typescript
@Get(':id')
findOne(@Param('id') id: string): string {
  return `This action returns a #${id} cat`;
}
```

### Sub-Domain-Routing

Der `@Controller`-Dekorator kann eine Host-Option übernehmen, um sicherzustellen, dass der HTTP-Host der eingehenden Anfragen einem bestimmten Wert entspricht.

```typescript
@Controller({ host: 'admin.example.com' })
export class AdminController {
  @Get()
  index(): string {
    return 'Admin page';
  }
}
```

#### WARNUNG
Da Fastify keine Unterstützung für verschachtelte Router bietet, sollte bei der Verwendung von Sub-Domain-Routing der (Standard-)Express-Adapter verwendet werden.

Ähnlich wie bei einem Routenpfad kann die Host-Option Tokens verwenden, um den dynamischen Wert an dieser Position im Hostnamen zu erfassen. Das Host-Parametertoken im `@Controller()`-Dekorator-Beispiel unten zeigt diese Verwendung. Host-Parameter, die auf diese Weise deklariert werden, können mit dem `@HostParam()`-Dekorator erfasst werden, der der Methodensignatur hinzugefügt werden sollte.

```typescript
@Controller({ host: ':account.example.com' })
export class AccountController {
  @Get()
  getInfo(@HostParam('account') account: string) {
    return account;
  }
}
```

### Scopes

Für Leute, die aus verschiedenen Programmiersprachen-Hintergründen kommen, mag es unerwartet sein zu erfahren, dass in Nest fast alles über eingehende Anfragen hinweg geteilt wird. Wir haben einen Verbindungspool zur Datenbank, Singleton-Services mit globalem Zustand usw. Denken Sie daran, dass Node.js nicht dem Request/Response Multi-Threaded Stateless Model folgt, bei dem jede Anfrage von einem separaten Thread verarbeitet wird. Daher ist die Verwendung von Singleton-Instanzen für unsere Anwendungen völlig sicher.

Es gibt jedoch Grenzfälle, in denen die lebenszyklusbasierte Anfrage des Controllers das gewünschte Verhalten sein kann, beispielsweise bei anfragebasiertem Caching in GraphQL-Anwendungen, Anfragenverfolgung oder Multi-Tenancy. Erfahren Sie hier, wie Sie die Scopes steuern können.

### Asynchronität

Wir lieben modernes JavaScript und wissen, dass die Datenextraktion meist asynchron ist. Deshalb unterstützt und funktioniert Nest gut mit asynchronen Funktionen.

#### HINWEIS
Erfahren Sie mehr über die Async/Await-Funktion hier

Jede asynchrone Funktion muss ein Promise zurückgeben. Das bedeutet, dass Sie einen verzögerten Wert zurückgeben können, den Nest selbst auflösen kann. Schauen wir uns ein Beispiel an:

```typescript
@Get()
async findAll(): Promise<any[]> {
  return [];
}
```

Der obige Code ist völlig gültig. Darüber hinaus sind Nest-Routen-Handler sogar noch leistungsfähiger, da sie in der Lage sind, RxJS-observable Streams zurückzugeben. Nest wird sich unter der Haube automatisch auf die Quelle abonnieren und den zuletzt emittierten Wert nehmen (sobald der Stream abgeschlossen ist).

```typescript
@Get()
findAll(): Observable<any[]> {
  return of([]);
}
```

Beide der obigen Ansätze funktionieren und Sie können verwenden, was Ihren Anforderungen entspricht.

### Anfragelasten

Unser vorheriges Beispiel für den POST-Routen-Handler akzeptierte keine Client-Parameter. Lassen Sie uns dies beheben, indem wir den `@Body()`-Dekorator hier hinzufügen.

Aber zuerst (wenn Sie TypeScript verwenden), müssen wir das DTO (Data Transfer Object)-Schema bestimmen. Ein DTO ist ein Objekt, das definiert, wie die Daten über das Netzwerk gesendet werden. Wir könnten das DTO-Schema mithilfe von TypeScript-Schnittstellen oder einfachen Klassen bestimmen. Interessanterweise empfehlen wir hier die Verwendung von Klassen. Warum? Klassen sind Teil des JavaScript-ES6-Standards und daher werden sie als reale Entitäten im kompilierten JavaScript beibehalten. Andererseits werden TypeScript-Schnittstellen während der Transpilation entfernt, sodass Nest nicht zur Laufzeit auf sie verweisen kann. Dies ist wichtig, weil Funktionen wie Pipes zusätzliche Möglichkeiten eröffnen, wenn sie zur Laufzeit Zugriff auf den Metatyp der Variable haben.

Lassen Sie uns die `CreateCatDto`-Klasse erstellen:

```typescript
export class CreateCatDto {
  name: string;
  age: number;
  breed: string;
}
```

Sie hat nur drei

 grundlegende Eigenschaften. Danach können wir das neu erstellte DTO innerhalb des `CatsController` verwenden:

```typescript
@Post()
async create(@Body() createCatDto: CreateCatDto) {
  return 'This action adds a new cat';
}
```

#### HINWEIS
Unsere `ValidationPipe` kann Eigenschaften herausfiltern, die nicht vom Methoden-Handler empfangen werden sollten. In diesem Fall können wir die akzeptablen Eigenschaften auf die Whitelist setzen und jede Eigenschaft, die nicht in der Whitelist enthalten ist, wird automatisch aus dem resultierenden Objekt entfernt. Im `CreateCatDto`-Beispiel ist unsere Whitelist die Eigenschaften `name`, `age` und `breed`. Erfahren Sie hier mehr.

### Fehlerbehandlung

Es gibt ein separates Kapitel über die Fehlerbehandlung (z. B. Arbeiten mit Ausnahmen) hier.

### Vollständiges Ressourcenbeispiel

Im Folgenden ist ein Beispiel, das mehrere der verfügbaren Dekoratoren verwendet, um einen grundlegenden Controller zu erstellen. Dieser Controller stellt einige Methoden zum Zugriff und zur Manipulation interner Daten bereit.

```typescript
import { Controller, Get, Query, Post, Body, Put, Param, Delete } from '@nestjs/common';
import { CreateCatDto, UpdateCatDto, ListAllEntities } from './dto';

@Controller('cats')
export class CatsController {
  @Post()
  create(@Body() createCatDto: CreateCatDto) {
    return 'This action adds a new cat';
  }

  @Get()
  findAll(@Query() query: ListAllEntities) {
    return `This action returns all cats (limit: ${query.limit} items)`;
  }

  @Get(':id')
  findOne(@Param('id') id: string) {
    return `This action returns a #${id} cat`;
  }

  @Put(':id')
  update(@Param('id') id: string, @Body() updateCatDto: UpdateCatDto) {
    return `This action updates a #${id} cat`;
  }

  @Delete(':id')
  remove(@Param('id') id: string) {
    return `This action removes a #${id} cat`;
  }
}
```

#### HINWEIS
Die Nest-CLI bietet einen Generator (Schematic), der automatisch den gesamten Boilerplate-Code generiert, um uns zu helfen, all dies zu vermeiden und das Entwicklererlebnis viel einfacher zu machen. Lesen Sie mehr über diese Funktion hier.

### In Betrieb nehmen

Mit dem oben vollständig definierten Controller weiß Nest immer noch nicht, dass der `CatsController` existiert, und wird daher keine Instanz dieser Klasse erstellen.

Controller gehören immer zu einem Modul, weshalb wir das `controllers`-Array innerhalb des `@Module()`-Dekorators einfügen. Da wir bisher keine anderen Module außer dem Root `AppModule` definiert haben, werden wir dieses verwenden, um den `CatsController` einzuführen:

```typescript
import { Module } from '@nestjs/common';
import { CatsController } from './cats/cats.controller';

@Module({
  controllers: [CatsController],
})
export class AppModule {}
```

Wir haben die Metadaten an die Modulkasse mithilfe des `@Module()`-Dekorators angehängt, und Nest kann nun leicht erkennen, welche Controller eingebunden werden müssen.

### Bibliotheksspezifischer Ansatz

Bisher haben wir den Nest-Standardweg zur Manipulation von Antworten besprochen. Der zweite Weg zur Manipulation der Antwort ist die Verwendung eines bibliotheksspezifischen Antwortobjekts. Um ein bestimmtes Antwortobjekt zu injizieren, müssen wir den `@Res()`-Dekorator verwenden. Um die Unterschiede zu zeigen, lassen Sie uns den `CatsController` wie folgt umschreiben:

```typescript
import { Controller, Get, Post, Res, HttpStatus } from '@nestjs/common';
import { Response } from 'express';

@Controller('cats')
export class CatsController {
  @Post()
  create(@Res() res: Response) {
    res.status(HttpStatus.CREATED).send();
  }

  @Get()
  findAll(@Res() res: Response) {
     res.status(HttpStatus.OK).json([]);
  }
}
```

Obwohl dieser Ansatz funktioniert und tatsächlich in gewisser Weise mehr Flexibilität bietet, indem er die volle Kontrolle über das Antwortobjekt ermöglicht (Header-Manipulation, bibliotheksspezifische Funktionen usw.), sollte er mit Vorsicht verwendet werden. Im Allgemeinen ist der Ansatz viel weniger klar und hat einige Nachteile. Der Hauptnachteil ist, dass Ihr Code plattformabhängig wird (da zugrunde liegende Bibliotheken unterschiedliche APIs auf dem Antwortobjekt haben können) und schwerer zu testen (Sie müssen das Antwortobjekt verspotten usw.).

Außerdem verlieren Sie in dem obigen Beispiel die Kompatibilität mit Nest-Funktionen, die von der Nest-Standardantwortverarbeitung abhängen, wie Interceptoren und `@HttpCode()` / `@Header()`-Dekoratoren. Um dies zu beheben, können Sie die Passthrough-Option auf `true` setzen, wie folgt:

```typescript
@Get()
findAll(@Res({ passthrough: true }) res: Response) {
  res.status(HttpStatus.OK);
  return [];
}
```

Nun können Sie mit dem nativen Antwortobjekt interagieren (z. B. Cookies oder Header je nach bestimmten Bedingungen setzen), aber den Rest dem Framework überlassen.
