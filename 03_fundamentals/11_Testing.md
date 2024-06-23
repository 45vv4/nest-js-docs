```markdown
# Testen / Testing

Automatisierte Tests gelten als wesentlicher Bestandteil jeder ernsthaften Softwareentwicklung. Automatisierung macht es einfach, einzelne Tests oder Test-Suiten während der Entwicklung schnell und einfach zu wiederholen. Dies hilft sicherzustellen, dass Veröffentlichungen die Qualitäts- und Leistungsziele erfüllen. Automatisierung hilft, die Abdeckung zu erhöhen und bietet eine schnellere Feedback-Schleife für Entwickler. Automatisierung steigert sowohl die Produktivität einzelner Entwickler als auch die Sicherstellung, dass Tests zu kritischen Zeitpunkten im Entwicklungslebenszyklus durchgeführt werden, wie z.B. bei der Quellcode-Kontrolle, der Feature-Integration und der Versionserstellung.

Solche Tests umfassen oft eine Vielzahl von Typen, einschließlich Unit-Tests, End-to-End-Tests (e2e), Integrationstests und so weiter. Obwohl die Vorteile unbestreitbar sind, kann es mühsam sein, sie einzurichten. Nest bemüht sich, Entwicklungs-Best-Practices zu fördern, einschließlich effektiven Testens, und enthält daher Funktionen wie die folgenden, um Entwicklern und Teams beim Erstellen und Automatisieren von Tests zu helfen. Nest:

- erstellt automatisch Standard-Unit-Tests für Komponenten und e2e-Tests für Anwendungen
- bietet Standard-Tools (wie einen Test-Runner, der einen isolierten Modul-/Anwendungs-Loader erstellt)
- bietet Integration mit Jest und Supertest out-of-the-box, bleibt dabei aber unabhängig von Test-Tools
- stellt das Nest-Abhängigkeitsinjektionssystem in der Testumgebung zur Verfügung, um Komponenten einfach zu mocken

Wie erwähnt, können Sie jedes Test-Framework verwenden, das Ihnen gefällt, da Nest keine spezifischen Tools erzwingt. Ersetzen Sie einfach die benötigten Elemente (wie den Test-Runner) und Sie genießen dennoch die Vorteile der fertiggestellten Testfunktionen von Nest.

## Installation / Installation

Um zu beginnen, installieren Sie zunächst das erforderliche Paket:

```bash
$ npm i --save-dev @nestjs/testing
```

## Unit-Tests / Unit testing

Im folgenden Beispiel testen wir zwei Klassen: CatsController und CatsService. Wie erwähnt, wird Jest als Standard-Test-Framework bereitgestellt. Es dient als Test-Runner und bietet auch Assert-Funktionen und Test-Double-Utilities, die beim Mocken, Spying usw. helfen. Im folgenden grundlegenden Test instanziieren wir diese Klassen manuell und stellen sicher, dass der Controller und der Service ihre API-Verträge erfüllen.

```typescript
// cats.controller.spec.ts

import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

describe('CatsController', () => {
  let catsController: CatsController;
  let catsService: CatsService;

  beforeEach(() => {
    catsService = new CatsService();
    catsController = new CatsController(catsService);
  });

  describe('findAll', () => {
    it('sollte ein Array von Katzen zurückgeben', async () => {
      const result = ['test'];
      jest.spyOn(catsService, 'findAll').mockImplementation(() => result);

      expect(await catsController.findAll()).toBe(result);
    });
  });
});
```

**TIPP**: Halten Sie Ihre Testdateien in der Nähe der Klassen, die sie testen. Testdateien sollten ein .spec- oder .test-Suffix haben.

Da das obige Beispiel trivial ist, testen wir eigentlich nichts Spezifisches für Nest. Tatsächlich verwenden wir nicht einmal die Abhängigkeitsinjektion (beachten Sie, dass wir eine Instanz von CatsService an unseren CatsController übergeben). Diese Form des Testens - bei der wir die zu testenden Klassen manuell instanziieren - wird oft als isoliertes Testen bezeichnet, da es unabhängig vom Framework ist. Lassen Sie uns einige fortgeschrittenere Fähigkeiten einführen, die Ihnen helfen, Anwendungen zu testen, die umfangreicher Nest-Funktionen nutzen.

## Test-Utilities / Testing utilities

Das @nestjs/testing-Paket bietet eine Reihe von Utilities, die einen robusteren Testprozess ermöglichen. Lassen Sie uns das vorherige Beispiel mit der eingebauten Test-Klasse neu schreiben:

```typescript
// cats.controller.spec.ts

import { Test } from '@nestjs/testing';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

describe('CatsController', () => {
  let catsController: CatsController;
  let catsService: CatsService;

  beforeEach(async () => {
    const moduleRef = await Test.createTestingModule({
        controllers: [CatsController],
        providers: [CatsService],
      }).compile();

    catsService = moduleRef.get<CatsService>(CatsService);
    catsController = moduleRef.get<CatsController>(CatsController);
  });

  describe('findAll', () => {
    it('sollte ein Array von Katzen zurückgeben', async () => {
      const result = ['test'];
      jest.spyOn(catsService, 'findAll').mockImplementation(() => result);

      expect(await catsController.findAll()).toBe(result);
    });
  });
});
```

Die Test-Klasse ist nützlich, um einen Anwendungsausführungskontext bereitzustellen, der im Wesentlichen die vollständige Nest-Laufzeit simuliert, aber Ihnen Hooks bietet, die das Verwalten von Klasseninstanzen erleichtern, einschließlich Mocking und Overriding. Die Test-Klasse hat eine createTestingModule()-Methode, die ein Modul-Metadaten-Objekt als Argument nimmt (dasselbe Objekt, das Sie an den @Module()-Dekorator übergeben). Diese Methode gibt eine TestingModule-Instanz zurück, die wiederum einige Methoden bereitstellt. Für Unit-Tests ist die compile()-Methode wichtig. Diese Methode bootstrapt ein Modul mit seinen Abhängigkeiten (ähnlich wie eine Anwendung im konventionellen main.ts-Datei mit NestFactory.create() gebootstrapt wird) und gibt ein Modul zurück, das zum Testen bereit ist.

**TIPP**: Die compile()-Methode ist asynchron und muss daher abgewartet werden. Sobald das Modul kompiliert ist, können Sie jede deklarierte statische Instanz (Controller und Provider) mit der get()-Methode abrufen.

TestingModule erbt von der Modulreferenz-Klasse und daher die Fähigkeit, scoped Provider (transient oder request-scoped) dynamisch aufzulösen. Dies erfolgt mit der resolve()-Methode (die get()-Methode kann nur statische Instanzen abrufen).

```typescript
const moduleRef = await Test.createTestingModule({
  controllers: [CatsController],
  providers: [CatsService],
}).compile();

catsService = await moduleRef.resolve(CatsService);
```

**WARNUNG**: Die resolve()-Methode gibt eine eindeutige Instanz des Providers aus dessen eigenem DI-Container-Unterbaum zurück. Jeder Unterbaum hat einen eindeutigen Kontextbezeichner. Wenn Sie diese Methode mehrmals aufrufen und Instanzreferenzen vergleichen, werden Sie sehen, dass sie nicht gleich sind.

**TIPP**: Erfahren Sie hier mehr über die Funktionen der Modulreferenz.

Anstelle der Verwendung der Produktionsversion eines Providers können Sie ihn für Testzwecke mit einem benutzerdefinierten Provider überschreiben. Zum Beispiel können Sie einen Datenbankdienst mocken, anstatt eine Verbindung zu einer Live-Datenbank herzustellen. Wir werden im nächsten Abschnitt Überschreibungen behandeln, aber sie sind auch für Unit-Tests verfügbar.

## Automatisches Mocking / Auto mocking

Nest ermöglicht es Ihnen auch, eine Mock-Factory zu definieren, die auf alle Ihre fehlenden Abhängigkeiten angewendet wird. Dies ist nützlich für Fälle, in denen Sie eine große Anzahl von Abhängigkeiten in einer Klasse haben und deren Mocking viel Zeit und Einrichtung in Anspruch nehmen würde. Um diese Funktion zu nutzen, muss die createTestingModule()-Methode mit der useMocker()-Methode verkettet werden, wobei eine Factory für Ihre Abhängigkeits-Mocks übergeben wird. Diese Factory kann ein optionales Token annehmen, das ein Instanz-Token ist, jedes Token, das für einen Nest-Provider gültig ist, und gibt eine Mock-Implementierung zurück. Das folgende Beispiel zeigt die Erstellung eines generischen Mockers mit jest-mock und eines spezifischen Mocks für CatsService mit jest.fn().

```typescript
// ...
import { ModuleMocker, MockFunctionMetadata } from 'jest-mock';

const moduleMocker = new ModuleMocker(global);

describe('CatsController', () => {
  let controller: CatsController;

  beforeEach(async () => {
    const moduleRef = await Test.createTestingModule({
      controllers: [CatsController],
    })
      .useMocker((token) => {
        const results = ['test1', 'test2'];
        if (token === CatsService) {
          return { findAll: jest.fn().mockResolvedValue(results) };
        }
        if (typeof token === 'function') {
          const mockMetadata = moduleMocker.getMetadata(token) as MockFunctionMetadata<any, any>;
          const Mock = moduleMocker.generateFromMetadata(mockMetadata);
          return new Mock();
        }
      })
      .compile();

    controller = moduleRef.get(CatsController);
  });
});
```

Sie können diese Mocks auch aus dem Test-Container abrufen, wie Sie es normalerweise mit benutzerdefinierten Providern tun würden, moduleRef.get(CatsService).

**TIPP**: Eine allgemeine Mock-Factory wie

 createMock aus @golevelup/ts-jest kann ebenfalls direkt übergeben werden.

**TIPP**: REQUEST- und INQUIRER-Provider können nicht automatisch gemockt werden, da sie bereits im Kontext vordefiniert sind. Sie können jedoch mit der benutzerdefinierten Provider-Syntax oder durch Verwendung der .overrideProvider-Methode überschrieben werden.

## End-to-End-Tests / End-to-end testing

Im Gegensatz zu Unit-Tests, die sich auf einzelne Module und Klassen konzentrieren, decken End-to-End-Tests (e2e) die Interaktion von Klassen und Modulen auf einer aggregierteren Ebene ab - näher an der Art der Interaktion, die Endbenutzer mit dem Produktionssystem haben werden. Mit dem Wachstum einer Anwendung wird es schwierig, das End-to-End-Verhalten jedes API-Endpunkts manuell zu testen. Automatisierte End-to-End-Tests helfen uns sicherzustellen, dass das Gesamtverhalten des Systems korrekt ist und die Projektanforderungen erfüllt. Um e2e-Tests durchzuführen, verwenden wir eine ähnliche Konfiguration wie bei den Unit-Tests. Darüber hinaus macht es Nest einfach, die Supertest-Bibliothek zu verwenden, um HTTP-Anfragen zu simulieren.

```typescript
// cats.e2e-spec.ts

import * as request from 'supertest';
import { Test } from '@nestjs/testing';
import { CatsModule } from '../../src/cats/cats.module';
import { CatsService } from '../../src/cats/cats.service';
import { INestApplication } from '@nestjs/common';

describe('Cats', () => {
  let app: INestApplication;
  let catsService = { findAll: () => ['test'] };

  beforeAll(async () => {
    const moduleRef = await Test.createTestingModule({
      imports: [CatsModule],
    })
      .overrideProvider(CatsService)
      .useValue(catsService)
      .compile();

    app = moduleRef.createNestApplication();
    await app.init();
  });

  it(`/GET cats`, () => {
    return request(app.getHttpServer())
      .get('/cats')
      .expect(200)
      .expect({
        data: catsService.findAll(),
      });
  });

  afterAll(async () => {
    await app.close();
  });
});
```

**TIPP**: Wenn Sie Fastify als Ihren HTTP-Adapter verwenden, erfordert dies eine etwas andere Konfiguration und hat eingebaute Testfunktionen:

```typescript
let app: NestFastifyApplication;

beforeAll(async () => {
  app = moduleRef.createNestApplication<NestFastifyApplication>(new FastifyAdapter());

  await app.init();
  await app.getHttpAdapter().getInstance().ready();
});

it(`/GET cats`, () => {
  return app
    .inject({
      method: 'GET',
      url: '/cats',
    })
    .then((result) => {
      expect(result.statusCode).toEqual(200);
      expect(result.payload).toEqual(/* erwartetesPayload */);
    });
});

afterAll(async () => {
  await app.close();
});
```

In diesem Beispiel bauen wir auf einigen der zuvor beschriebenen Konzepte auf. Zusätzlich zur compile()-Methode, die wir zuvor verwendet haben, verwenden wir jetzt die createNestApplication()-Methode, um eine vollständige Nest-Laufzeitumgebung zu instanziieren. Wir speichern eine Referenz zur laufenden App in unserer app-Variable, damit wir sie verwenden können, um HTTP-Anfragen zu simulieren.

Wir simulieren HTTP-Tests mit der request()-Funktion von Supertest. Wir möchten, dass diese HTTP-Anfragen an unsere laufende Nest-App weitergeleitet werden, daher übergeben wir der request()-Funktion eine Referenz auf den HTTP-Listener, der Nest zugrunde liegt (der wiederum von der Express-Plattform bereitgestellt werden kann). Daher die Konstruktion request(app.getHttpServer()). Der Aufruf von request() gibt uns einen umwickelten HTTP-Server, der jetzt mit der Nest-App verbunden ist und Methoden zum Simulieren einer tatsächlichen HTTP-Anfrage bereitstellt. Zum Beispiel wird durch die Verwendung von request(...).get('/cats') eine Anfrage an die Nest-App initiiert, die identisch ist mit einer tatsächlichen HTTP-Anfrage wie get '/cats', die über das Netzwerk eingeht.

In diesem Beispiel stellen wir auch eine alternative (Test-Double) Implementierung des CatsService bereit, die einfach einen fest codierten Wert zurückgibt, den wir testen können. Verwenden Sie overrideProvider(), um eine solche alternative Implementierung bereitzustellen. Ebenso bietet Nest Methoden zum Überschreiben von Modulen, Guards, Interceptors, Filtern und Pipes mit den Methoden overrideModule(), overrideGuard(), overrideInterceptor(), overrideFilter() und overridePipe() an.

Jede der Überschreibungsmethoden (außer overrideModule()) gibt ein Objekt mit 3 verschiedenen Methoden zurück, die denen ähneln, die für benutzerdefinierte Provider beschrieben wurden:

- useClass: Sie geben eine Klasse an, die instanziiert wird, um die Instanz zu liefern, die das Objekt (Provider, Guard usw.) überschreibt.
- useValue: Sie geben eine Instanz an, die das Objekt überschreibt.
- useFactory: Sie geben eine Funktion an, die eine Instanz zurückgibt, die das Objekt überschreibt.

Andererseits gibt overrideModule() ein Objekt mit der Methode useModule() zurück, mit der Sie ein Modul angeben können, das das Originalmodul überschreibt, wie folgt:

```typescript
const moduleRef = await Test.createTestingModule({
  imports: [AppModule],
})
  .overrideModule(CatsModule)
  .useModule(AlternateCatsModule)
  .compile();
```

Jede der Überschreibungsmethoden gibt wiederum die TestingModule-Instanz zurück und kann daher im Fluent-Stil mit anderen Methoden verkettet werden. Sie sollten compile() am Ende einer solchen Kette verwenden, um Nest zu veranlassen, das Modul zu instanziieren und zu initialisieren.

Manchmal möchten Sie möglicherweise auch einen benutzerdefinierten Logger bereitstellen, z.B. wenn die Tests ausgeführt werden (zum Beispiel auf einem CI-Server). Verwenden Sie die setLogger()-Methode und übergeben Sie ein Objekt, das die LoggerService-Schnittstelle erfüllt, um dem TestModuleBuilder mitzuteilen, wie während der Tests protokolliert werden soll (standardmäßig werden nur "Fehler"-Protokolle in die Konsole protokolliert).

Das kompilierte Modul hat mehrere nützliche Methoden, wie in der folgenden Tabelle beschrieben:

| Methode                          | Beschreibung                                                                                       |
|----------------------------------|---------------------------------------------------------------------------------------------------|
| createNestApplication()          | Erstellt und gibt eine Nest-Anwendung (INestApplication-Instanz) basierend auf dem gegebenen Modul zurück. Beachten Sie, dass Sie die Anwendung manuell mit der init()-Methode initialisieren müssen. |
| createNestMicroservice()         | Erstellt und gibt eine Nest-Microservice (INestMicroservice-Instanz) basierend auf dem gegebenen Modul zurück. |
| get()                            | Ruft eine statische Instanz eines Controllers oder Providers (einschließlich Guards, Filter usw.) ab, die im Anwendungskontext verfügbar ist. Geerbt von der Modulreferenz-Klasse. |
| resolve()                        | Ruft eine dynamisch erstellte scoped Instanz (Request oder Transient) eines Controllers oder Providers (einschließlich Guards, Filter usw.) ab, die im Anwendungskontext verfügbar ist. Geerbt von der Modulreferenz-Klasse. |
| select()                         | Navigiert durch den Abhängigkeitsgraphen des Moduls; kann verwendet werden, um eine bestimmte Instanz aus dem ausgewählten Modul abzurufen (in Verbindung mit strict mode (strict: true) in der get()-Methode verwendet). |

**TIPP**: Halten Sie Ihre e2e-Testdateien im Testverzeichnis. Die Testdateien sollten ein .e2e-spec-Suffix haben.

## Global registrierte Enhancer überschreiben / Overriding globally registered enhancers

Wenn Sie einen global registrierten Guard (oder Pipe, Interceptor oder Filter) haben, müssen Sie einige weitere Schritte unternehmen, um diesen Enhancer zu überschreiben. Zur Erinnerung, die ursprüngliche Registrierung sieht folgendermaßen aus:

```typescript
providers: [
  {
    provide: APP_GUARD,
    useClass: JwtAuthGuard,
  },
],
```

Dies registriert den Guard als "Multi"-Provider über das APP_* Token. Um den JwtAuthGuard hier zu ersetzen, muss die Registrierung einen vorhandenen Provider in diesem Slot verwenden:

```typescript
providers: [
  {
    provide: APP_GUARD,
    useExisting: JwtAuthGuard,
    // ^^^^^^^^ beachten Sie die Verwendung von 'useExisting' anstelle von 'useClass'
  },
  JwtAuthGuard,
],
```

**TIPP**: Ändern Sie useClass zu useExisting, um einen registrierten Provider anstelle von Nest zu referenzieren, der ihn hinter dem Token instanziiert.

Jetzt ist der JwtAuthGuard für Nest als regulärer Provider sichtbar, der beim Erstellen des TestingModule überschrieben werden kann:

```typescript
const moduleRef = await Test.createTestingModule({
  imports: [AppModule],
})
  .overrideProvider(JwtAuthGuard)
  .useClass(MockAuthGuard)
  .compile();
```

Jetzt werden alle Ihre Tests den MockAuthGuard bei jeder Anfrage verwenden.

## Testen von request-scoped Instanzen / Testing request-scoped instances

Request-scoped Provider werden für jede eingehende Anfrage eindeutig erstellt. Die Instanz wird nach Abschluss der Anfrage vom Garbage Collector entfernt. Dies stellt ein Problem dar, da wir nicht auf einen DI-Unterbaum zugreifen können, der speziell für eine getestete Anfrage generiert wurde.

Wir wissen

 (basierend auf den obigen Abschnitten), dass die resolve()-Methode verwendet werden kann, um eine dynamisch instanziierte Klasse abzurufen. Auch, wie hier beschrieben, wissen wir, dass wir einen eindeutigen Kontextbezeichner übergeben können, um den Lebenszyklus eines DI-Container-Unterbaums zu steuern. Wie nutzen wir dies im Testkontext?

Die Strategie besteht darin, einen Kontextbezeichner im Voraus zu generieren und Nest zu zwingen, diese bestimmte ID zu verwenden, um einen Unterbaum für alle eingehenden Anfragen zu erstellen. Auf diese Weise können wir auf Instanzen zugreifen, die für eine getestete Anfrage erstellt wurden.

Um dies zu erreichen, verwenden Sie jest.spyOn() auf der ContextIdFactory:

```typescript
const contextId = ContextIdFactory.create();
jest.spyOn(ContextIdFactory, 'getByRequest').mockImplementation(() => contextId);
```

Jetzt können wir den contextId verwenden, um auf einen einzelnen generierten DI-Container-Unterbaum für jede nachfolgende Anfrage zuzugreifen.

```typescript
catsService = await moduleRef.resolve(CatsService, contextId);
```
