## Benutzerdefinierte Provider / Custom providers

In früheren Kapiteln haben wir verschiedene Aspekte der Dependency Injection (DI) und deren Verwendung in Nest behandelt. Ein Beispiel dafür ist die konstruktorbasierte Abhängigkeitsinjektion, die verwendet wird, um Instanzen (oft Service-Provider) in Klassen zu injizieren. Sie werden nicht überrascht sein zu erfahren, dass die Dependency Injection auf fundamentale Weise in den Kern von Nest integriert ist. Bisher haben wir nur ein Hauptmuster erkundet. Mit wachsender Komplexität Ihrer Anwendung müssen Sie möglicherweise die vollständigen Funktionen des DI-Systems nutzen. Lassen Sie uns diese daher genauer untersuchen.

### DI-Grundlagen

Dependency Injection ist eine Technik zur Inversion of Control (IoC), bei der die Instanziierung von Abhängigkeiten an den IoC-Container delegiert wird (in unserem Fall das NestJS-Laufzeitsystem), anstatt sie in Ihrem eigenen Code imperativ zu erstellen. Schauen wir uns an, was in diesem Beispiel aus dem Kapitel über Provider passiert.

Zuerst definieren wir einen Provider. Der `@Injectable()`-Dekorator kennzeichnet die `CatsService`-Klasse als Provider.

```typescript
// cats.service.ts

import { Injectable } from '@nestjs/common';
import { Cat } from './interfaces/cat.interface';

@Injectable()
export class CatsService {
  private readonly cats: Cat[] = [];

  findAll(): Cat[] {
    return this.cats;
  }
}
```

Dann fordern wir Nest auf, den Provider in unsere Controller-Klasse zu injizieren:

```typescript
// cats.controller.ts

import { Controller, Get } from '@nestjs/common';
import { CatsService } from './cats.service';
import { Cat } from './interfaces/cat.interface';

@Controller('cats')
export class CatsController {
  constructor(private catsService: CatsService) {}

  @Get()
  async findAll(): Promise<Cat[]> {
    return this.catsService.findAll();
  }
}
```

Schließlich registrieren wir den Provider beim Nest IoC-Container:

```typescript
// app.module.ts

import { Module } from '@nestjs/common';
import { CatsController } from './cats/cats.controller';
import { CatsService } from './cats/cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
export class AppModule {}
```

### Was passiert hinter den Kulissen?

Es gibt drei wichtige Schritte im Prozess:

1. In `cats.service.ts` deklariert der `@Injectable()`-Dekorator die `CatsService`-Klasse als Klasse, die vom Nest IoC-Container verwaltet werden kann.
2. In `cats.controller.ts` deklariert `CatsController` eine Abhängigkeit vom `CatsService`-Token mit konstruktorbasierter Injektion:

```typescript
constructor(private catsService: CatsService)
```

3. In `app.module.ts` assoziieren wir das `CatsService`-Token mit der `CatsService`-Klasse aus der `cats.service.ts`-Datei.

Wenn der Nest IoC-Container eine `CatsController`-Instanz instanziiert, sucht er zuerst nach Abhängigkeiten. Wenn er die `CatsService`-Abhängigkeit findet, führt er ein Lookup auf dem `CatsService`-Token durch, das die `CatsService`-Klasse zurückgibt, gemäß dem Registrierungsprozess (Schritt 3 oben). Bei SINGLETON-Scope (das Standardverhalten) erstellt Nest entweder eine Instanz von `CatsService`, cached sie und gibt sie zurück, oder gibt eine bereits gecachte Instanz zurück.

### Standard-Provider

Schauen wir uns den `@Module()`-Dekorator genauer an. In `app.module` deklarieren wir:

```typescript
@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
```

Die `providers`-Eigenschaft nimmt ein Array von Providern. Bisher haben wir diese Provider über eine Liste von Klassennamen bereitgestellt. Tatsächlich ist die Syntax `providers: [CatsService]` eine Kurzform für die vollständigere Syntax:

```typescript
providers: [
  {
    provide: CatsService,
    useClass: CatsService,
  },
];
```

Nun, da wir diese explizite Konstruktion sehen, können wir den Registrierungsprozess verstehen. Hier assoziieren wir eindeutig das `CatsService`-Token mit der `CatsService`-Klasse. Die Kurznotierung vereinfacht lediglich den häufigsten Anwendungsfall, bei dem das Token verwendet wird, um eine Instanz einer Klasse mit demselben Namen anzufordern.

### Benutzerdefinierte Provider

Was passiert, wenn Ihre Anforderungen über die von Standardprovidern hinausgehen? Hier sind einige Beispiele:

- Sie möchten eine benutzerdefinierte Instanz erstellen, anstatt Nest eine Klasse instanziieren zu lassen (oder eine gecachte Instanz zurückzugeben).
- Sie möchten eine vorhandene Klasse in einer zweiten Abhängigkeit wiederverwenden.
- Sie möchten eine Klasse durch eine Mock-Version für Tests überschreiben.

Nest ermöglicht es Ihnen, benutzerdefinierte Provider zu definieren, um diese Fälle zu behandeln. Es bietet mehrere Möglichkeiten, benutzerdefinierte Provider zu definieren. Lassen Sie uns diese durchgehen.

### Hinweis

Wenn Sie Probleme mit der Auflösung von Abhängigkeiten haben, können Sie die `NEST_DEBUG`-Umgebungsvariable setzen und während des Startvorgangs zusätzliche Abhängigkeitsauflösungsprotokolle erhalten.

### Wert-Provider: useValue

Die `useValue`-Syntax ist nützlich, um einen konstanten Wert zu injizieren, eine externe Bibliothek in den Nest-Container zu bringen oder eine reale Implementierung durch ein Mock-Objekt zu ersetzen. Angenommen, Sie möchten Nest zwingen, einen Mock-CatsService für Testzwecke zu verwenden.

```typescript
import { CatsService } from './cats.service';

const mockCatsService = {
  // mock implementation
};

@Module({
  imports: [CatsModule],
  providers: [
    {
      provide: CatsService,
      useValue: mockCatsService,
    },
  ],
})
export class AppModule {}
```

In diesem Beispiel wird das `CatsService`-Token auf das Mock-Objekt `mockCatsService` aufgelöst. `useValue` erfordert einen Wert - in diesem Fall ein Literalobjekt, das dieselbe Schnittstelle wie die `CatsService`-Klasse hat, die es ersetzt. Aufgrund der strukturellen Typisierung von TypeScript können Sie jedes Objekt verwenden, das eine kompatible Schnittstelle hat, einschließlich eines Literalobjekts oder einer mit `new` instanziierten Klasseninstanz.

### Nicht-klassenbasierte Provider-Tokens

Bisher haben wir Klassennamen als unsere Provider-Tokens verwendet (den Wert der `provide`-Eigenschaft in einem Provider, der im `providers`-Array aufgelistet ist). Dies wird durch das Standardmuster der konstruktorbasierten Injektion unterstützt, bei dem das Token ebenfalls ein Klassenname ist. Manchmal möchten wir die Flexibilität haben, Strings oder Symbole als DI-Token zu verwenden. Zum Beispiel:

```typescript
import { connection } from './connection';

@Module({
  providers: [
    {
      provide: 'CONNECTION',
      useValue: connection,
    },
  ],
})
export class AppModule {}
```

In diesem Beispiel assoziieren wir ein string-wertiges Token (`'CONNECTION'`) mit einem vorhandenen Verbindungsobjekt, das wir aus einer externen Datei importiert haben.

### Hinweis

Zusätzlich zur Verwendung von Strings als Tokenwerte können Sie auch JavaScript-Symbole oder TypeScript-Enums verwenden.

Wir haben bereits gesehen, wie man einen Provider mit dem Standardmuster der konstruktorbasierten Injektion injiziert. Dieses Muster erfordert, dass die Abhängigkeit mit einem Klassennamen deklariert wird. Der benutzerdefinierte Provider `CONNECTION` verwendet ein string-wertiges Token. Lassen Sie uns sehen, wie man einen solchen Provider injiziert. Dazu verwenden wir den `@Inject()`-Dekorator. Dieser Dekorator nimmt ein einziges Argument - das Token.

```typescript
@Injectable()
export class CatsRepository {
  constructor(@Inject('CONNECTION') connection: Connection) {}
}
```

### Hinweis

Der `@Inject()`-Dekorator wird aus dem `@nestjs/common`-Paket importiert.

Während wir den String `'CONNECTION'` in den obigen Beispielen direkt verwenden, ist es aus Gründen der sauberen Code-Organisation eine gute Praxis, Tokens in einer separaten Datei wie `constants.ts` zu definieren. Behandeln Sie sie ähnlich wie Symbole oder Enums, die in einer eigenen Datei definiert und bei Bedarf importiert werden.

### Klassen-Provider: useClass

Die `useClass`-Syntax ermöglicht es Ihnen, dynamisch eine Klasse zu bestimmen, die ein Token auflösen soll. Angenommen, wir haben eine abstrakte (oder Standard-) `ConfigService`-Klasse. Abhängig von der aktuellen Umgebung möchten wir, dass Nest eine andere Implementierung des Konfigurationsdienstes bereitstellt. Der folgende Code implementiert eine solche Strategie.

```typescript
const configServiceProvider = {
  provide: ConfigService,
  useClass:
    process.env.NODE_ENV === 'development'
      ? DevelopmentConfigService
      : ProductionConfigService,
};

@Module({
  providers: [configServiceProvider],
})
export class AppModule {}
```

Schauen wir uns einige Details in diesem Codebeispiel an. Sie werden feststellen, dass wir `configServiceProvider` mit einem Literalobjekt definieren und es dann in der `providers`-Eigenschaft des Modul-Dekorators übergeben. Dies

 ist nur eine Codeorganisation, aber funktional äquivalent zu den Beispielen, die wir bisher in diesem Kapitel verwendet haben.

Außerdem haben wir den Klassennamen `ConfigService` als unser Token verwendet. Für jede Klasse, die von `ConfigService` abhängt, wird Nest eine Instanz der bereitgestellten Klasse (`DevelopmentConfigService` oder `ProductionConfigService`) injizieren und jede Standardimplementierung überschreiben, die möglicherweise anderswo deklariert wurde (z.B. ein `ConfigService`, der mit einem `@Injectable()`-Dekorator deklariert wurde).

### Factory-Provider: useFactory

Die `useFactory`-Syntax ermöglicht es, Provider dynamisch zu erstellen. Der tatsächliche Provider wird durch den von einer Factory-Funktion zurückgegebenen Wert bereitgestellt. Die Factory-Funktion kann so einfach oder komplex sein, wie es nötig ist. Eine einfache Factory benötigt möglicherweise keine anderen Provider. Eine komplexere Factory kann selbst andere Provider injizieren, die sie zur Berechnung ihres Ergebnisses benötigt. Für letzteren Fall hat die Factory-Provider-Syntax ein Paar verwandter Mechanismen:

- Die Factory-Funktion kann (optionale) Argumente akzeptieren.
- Die (optionale) `inject`-Eigenschaft akzeptiert ein Array von Providern, die Nest auflösen und als Argumente an die Factory-Funktion während des Instanziierungsprozesses übergeben wird. Diese Provider können auch als optional markiert werden. Die beiden Listen sollten korreliert sein: Nest wird Instanzen aus der `inject`-Liste als Argumente in der gleichen Reihenfolge an die Factory-Funktion übergeben. Das folgende Beispiel demonstriert dies.

```typescript
const connectionProvider = {
  provide: 'CONNECTION',
  useFactory: (optionsProvider: OptionsProvider, optionalProvider?: string) => {
    const options = optionsProvider.get();
    return new DatabaseConnection(options);
  },
  inject: [OptionsProvider, { token: 'SomeOptionalProvider', optional: true }],
  //       \_____________/            \__________________/
  //        This provider              The provider with this
  //        is mandatory.              token can resolve to `undefined`.
};

@Module({
  providers: [
    connectionProvider,
    OptionsProvider,
    // { provide: 'SomeOptionalProvider', useValue: 'anything' },
  ],
})
export class AppModule {}
```

### Alias-Provider: useExisting

Die `useExisting`-Syntax ermöglicht es Ihnen, Aliase für vorhandene Provider zu erstellen. Dies schafft zwei Möglichkeiten, auf denselben Provider zuzugreifen. Im folgenden Beispiel ist das (string-basierte) Token `AliasedLoggerService` ein Alias für das (klassenbasierte) Token `LoggerService`. Angenommen, wir haben zwei verschiedene Abhängigkeiten, eine für `AliasedLoggerService` und eine für `LoggerService`. Wenn beide Abhängigkeiten mit SINGLETON-Scope angegeben sind, werden sie beide auf dieselbe Instanz aufgelöst.

```typescript
@Injectable()
class LoggerService {
  // Implementierungsdetails
}

const loggerAliasProvider = {
  provide: 'AliasedLoggerService',
  useExisting: LoggerService,
};

@Module({
  providers: [LoggerService, loggerAliasProvider],
})
export class AppModule {}
```

### Nicht-dienstbasierte Provider

Während Provider oft Dienste bereitstellen, sind sie nicht darauf beschränkt. Ein Provider kann jeden Wert liefern. Zum Beispiel kann ein Provider ein Array von Konfigurationsobjekten basierend auf der aktuellen Umgebung bereitstellen, wie unten gezeigt:

```typescript
const configFactory = {
  provide: 'CONFIG',
  useFactory: () => {
    return process.env.NODE_ENV === 'development' ? devConfig : prodConfig;
  },
};

@Module({
  providers: [configFactory],
})
export class AppModule {}
```

### Benutzerdefinierten Provider exportieren

Wie jeder Provider ist ein benutzerdefinierter Provider auf sein deklarierendes Modul beschränkt. Um ihn für andere Module sichtbar zu machen, muss er exportiert werden. Um einen benutzerdefinierten Provider zu exportieren, können wir entweder sein Token oder das vollständige Provider-Objekt verwenden.

Das folgende Beispiel zeigt den Export mit dem Token:

```typescript
const connectionFactory = {
  provide: 'CONNECTION',
  useFactory: (optionsProvider: OptionsProvider) => {
    const options = optionsProvider.get();
    return new DatabaseConnection(options);
  },
  inject: [OptionsProvider],
};

@Module({
  providers: [connectionFactory],
  exports: ['CONNECTION'],
})
export class AppModule {}
```

Alternativ exportieren Sie mit dem vollständigen Provider-Objekt:

```typescript
const connectionFactory = {
  provide: 'CONNECTION',
  useFactory: (optionsProvider: OptionsProvider) => {
    const options = optionsProvider get();
    return new DatabaseConnection(options);
  },
  inject: [OptionsProvider],
};

@Module({
  providers: [connectionFactory],
  exports: [connectionFactory],
})
export class AppModule {}
```
