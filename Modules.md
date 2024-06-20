### Module / Modules

Ein Modul ist eine Klasse, die mit einem `@Module()` Dekorator annotiert ist. Der `@Module()` Dekorator stellt Metadaten bereit, die Nest verwendet, um die Anwendungsstruktur zu organisieren.

Jede Anwendung hat mindestens ein Modul, ein Stamm-Modul. Das Stamm-Modul ist der Ausgangspunkt, den Nest verwendet, um den Anwendungsgraphen zu erstellen – die interne Datenstruktur, die Nest verwendet, um Modul- und Anbieterbeziehungen und -abhängigkeiten aufzulösen. Während sehr kleine Anwendungen theoretisch nur das Stamm-Modul haben können, ist dies nicht der typische Fall. Wir möchten betonen, dass Module als effektive Methode zur Organisation Ihrer Komponenten dringend empfohlen werden. Daher wird die resultierende Architektur für die meisten Anwendungen mehrere Module verwenden, von denen jedes einen eng verwandten Satz von Fähigkeiten kapselt.

Der `@Module()` Dekorator nimmt ein einzelnes Objekt, dessen Eigenschaften das Modul beschreiben:

- **providers**: die Anbieter, die vom Nest-Injektor instanziiert werden und die mindestens in diesem Modul gemeinsam genutzt werden können
- **controllers**: die Menge der in diesem Modul definierten Controller, die instanziiert werden müssen
- **imports**: die Liste der importierten Module, die die in diesem Modul erforderlichen Anbieter exportieren
- **exports**: die Teilmenge der Anbieter, die von diesem Modul bereitgestellt werden und in anderen Modulen verfügbar sein sollten, die dieses Modul importieren. Sie können entweder den Anbieter selbst oder nur dessen Token (provide value) verwenden

Das Modul kapselt standardmäßig Anbieter. Das bedeutet, dass es unmöglich ist, Anbieter zu injizieren, die weder direkt Teil des aktuellen Moduls sind noch aus den importierten Modulen exportiert wurden. Daher können Sie die exportierten Anbieter eines Moduls als die öffentliche Schnittstelle oder API des Moduls betrachten.

### Feature-Module / Feature modules

Der `CatsController` und der `CatsService` gehören zur selben Anwendungsdomäne. Da sie eng miteinander verwandt sind, macht es Sinn, sie in ein Feature-Modul zu verschieben. Ein Feature-Modul organisiert einfach den Code, der für ein bestimmtes Feature relevant ist, hält den Code organisiert und etabliert klare Grenzen. Dies hilft uns, die Komplexität zu verwalten und nach den SOLID-Prinzipien zu entwickeln, insbesondere wenn die Größe der Anwendung und/oder des Teams wächst.

Um dies zu demonstrieren, erstellen wir das `CatsModule`.

```typescript
// cats/cats.module.ts
import { Module } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
export class CatsModule {}
```

**HINWEIS:** Um ein Modul mit der CLI zu erstellen, führen Sie einfach den Befehl `$ nest g module cats` aus.

Oben haben wir das `CatsModule` in der Datei `cats.module.ts` definiert und alles, was zu diesem Modul gehört, in das Verzeichnis `cats` verschoben. Das Letzte, was wir tun müssen, ist, dieses Modul in das Stamm-Modul (das `AppModule`, definiert in der Datei `app.module.ts`) zu importieren.

```typescript
// app.module.ts
import { Module } from '@nestjs/common';
import { CatsModule } from './cats/cats.module';

@Module({
  imports: [CatsModule],
})
export class AppModule {}
```

So sieht unsere Verzeichnisstruktur jetzt aus:

```
src
|-- cats
|   |-- dto
|   |   `-- create-cat.dto.ts
|   |-- interfaces
|   |   `-- cat.interface.ts
|   |-- cats.controller.ts
|   |-- cats.module.ts
|   |-- cats.service.ts
|-- app.module.ts
|-- main.ts
```

### Gemeinsame Module / Shared modules

In Nest sind Module standardmäßig Singletons, und daher können Sie mühelos dieselbe Instanz eines Anbieters zwischen mehreren Modulen teilen.

Jedes Modul ist automatisch ein gemeinsames Modul. Sobald es erstellt ist, kann es von jedem Modul wiederverwendet werden. Angenommen, wir möchten eine Instanz des `CatsService` zwischen mehreren anderen Modulen teilen. Um dies zu tun, müssen wir den `CatsService`-Anbieter zuerst exportieren, indem wir ihn dem `exports`-Array des Moduls hinzufügen, wie unten gezeigt:

```typescript
// cats.module.ts
import { Module } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
  exports: [CatsService]
})
export class CatsModule {}
```

Jetzt hat jedes Modul, das das `CatsModule` importiert, Zugriff auf den `CatsService` und wird dieselbe Instanz mit allen anderen Modulen teilen, die es ebenfalls importieren.

### Module-Wiederexport / Module re-exporting

Wie oben gezeigt, können Module ihre internen Anbieter exportieren. Darüber hinaus können sie auch Module wiederexportieren, die sie importieren. Im folgenden Beispiel wird das `CommonModule` sowohl in das `CoreModule` importiert als auch aus diesem exportiert, sodass es für andere Module, die dieses importieren, verfügbar ist.

```typescript
@Module({
  imports: [CommonModule],
  exports: [CommonModule],
})
export class CoreModule {}
```

### Dependency Injection / Dependency injection

Eine Modulklasse kann auch Anbieter injizieren (z.B. für Konfigurationszwecke):

```typescript
// cats.module.ts
import { Module } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
export class CatsModule {
  constructor(private catsService: CatsService) {}
}
```

Modulklassen selbst können jedoch nicht als Anbieter injiziert werden, da dies zu zirkulären Abhängigkeiten führen würde.

### Globale Module / Global modules

Wenn Sie dasselbe Set von Modulen überall importieren müssen, kann das mühsam werden. Im Gegensatz zu Angular sind Anbieter in Nest im globalen Scope registriert. Sobald sie definiert sind, sind sie überall verfügbar. Nest kapselt Anbieter jedoch innerhalb des Modul-Scopes. Sie können die Anbieter eines Moduls nicht woanders verwenden, ohne das kapselnde Modul zuerst zu importieren.

Wenn Sie einen Satz von Anbietern bereitstellen möchten, die überall standardmäßig verfügbar sein sollen (z.B. Helfer, Datenbankverbindungen, etc.), machen Sie das Modul mit dem `@Global()` Dekorator global.

```typescript
import { Module, Global } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Global()
@Module({
  controllers: [CatsController],
  providers: [CatsService],
  exports: [CatsService],
})
export class CatsModule {}
```

Der `@Global()` Dekorator macht das Modul global. Globale Module sollten nur einmal registriert werden, in der Regel vom Stamm- oder Kernmodul. Im obigen Beispiel wird der `CatsService`-Anbieter allgegenwärtig sein, und Module, die den Service injizieren möchten, müssen das `CatsModule` nicht in ihrem `imports`-Array importieren.

**HINWEIS:** Alles global zu machen, ist keine gute Designentscheidung. Globale Module sollen die Menge des notwendigen Boilerplates reduzieren. Das `imports`-Array ist in der Regel der bevorzugte Weg, um die API eines Moduls für Verbraucher verfügbar zu machen.

### Dynamische Module / Dynamic modules

Das Nest-Modulsystem enthält ein leistungsstarkes Feature namens dynamische Module. Dieses Feature ermöglicht es Ihnen, leicht anpassbare Module zu erstellen, die Anbieter dynamisch registrieren und konfigurieren können. Dynamische Module werden hier ausführlich behandelt. In diesem Kapitel geben wir einen kurzen Überblick, um die Einführung in Module zu vervollständigen.

Im Folgenden finden Sie ein Beispiel für eine dynamische Moduldefinition für ein `DatabaseModule`:

```typescript
import { Module, DynamicModule } from '@nestjs/common';
import { createDatabaseProviders } from './database.providers';
import { Connection } from './connection.provider';

@Module({
  providers: [Connection],
  exports: [Connection],
})
export class DatabaseModule {
  static forRoot(entities = [], options?): DynamicModule {
    const providers = createDatabaseProviders(options, entities);
    return {
      module: DatabaseModule,
      providers: providers,
      exports: providers,
    };
  }
}
```

**HINWEIS:** Die `forRoot()` Methode kann ein dynamisches Modul entweder synchron oder asynchron (d.h. über ein Promise) zurückgeben.

Dieses Modul definiert den `Connection`-Anbieter standardmäßig (in den Metadaten des `@Module()` Dekorators), aber zusätzlich – abhängig von den in die `forRoot()` Methode übergebenen Objekten `entities` und `options` – stellt es eine Sammlung von Anbietern, beispielsweise Repositories, bereit. Beachten Sie, dass die von dem dynamischen Modul zurückgegebenen Eigenschaften die Basis-Modulmetadaten erweitern (anstatt sie zu überschreiben), die im `@Module()` Dekorator definiert sind. Auf diese Weise werden sowohl der statisch deklarierte `Connection`-Anbieter als auch die dynamisch generierten Repository-Anbieter aus dem Modul exportiert.

Wenn Sie ein dynamisches Modul im globalen Scope registrieren möchten, setzen Sie die `global`-Eigenschaft auf `true`.

```typescript
{
 

 global: true,
  module: DatabaseModule,
  providers: providers,
  exports: providers,
}
```

**WARNUNG:** Wie oben erwähnt, ist es keine gute Designentscheidung, alles global zu machen.

Das `DatabaseModule` kann wie folgt importiert und konfiguriert werden:

```typescript
import { Module } from '@nestjs/common';
import { DatabaseModule } from './database/database.module';
import { User } from './users/entities/user.entity';

@Module({
  imports: [DatabaseModule.forRoot([User])],
})
export class AppModule {}
```

Wenn Sie ein dynamisches Modul wiederum re-exportieren möchten, können Sie den Aufruf der `forRoot()` Methode im `exports`-Array weglassen:

```typescript
import { Module } from '@nestjs/common';
import { DatabaseModule } from './database/database.module';
import { User } from './users/entities/user.entity';

@Module({
  imports: [DatabaseModule.forRoot([User])],
  exports: [DatabaseModule],
})
export class AppModule {}
```

Das Kapitel über dynamische Module behandelt dieses Thema ausführlicher und enthält ein funktionierendes Beispiel.

**HINWEIS:** Erfahren Sie, wie Sie hochgradig anpassbare dynamische Module mit dem `ConfigurableModuleBuilder` erstellen können, hier in diesem Kapitel.
