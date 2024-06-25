### Anbieter / Providers

Anbieter sind ein grundlegendes Konzept in Nest. Viele der grundlegenden Nest-Klassen können als Anbieter behandelt werden – Services, Repositories, Fabriken, Helfer und so weiter. Die Hauptidee eines Anbieters ist, dass er als Abhängigkeit injiziert werden kann; das bedeutet, dass Objekte verschiedene Beziehungen zueinander aufbauen können und das "Verdrahten" dieser Objekte weitgehend dem Nest-Laufzeitsystem überlassen werden kann.

![Components](Components_1.png)

Im vorherigen Kapitel haben wir einen einfachen `CatsController` erstellt. Controller sollten HTTP-Anfragen bearbeiten und komplexere Aufgaben an Anbieter delegieren. Anbieter sind einfache JavaScript-Klassen, die in einem Modul als Anbieter deklariert werden.

**HINWEIS / HINT:** Da Nest die Möglichkeit bietet, Abhängigkeiten auf eine objektorientiertere Weise zu entwerfen und zu organisieren, empfehlen wir dringend, die SOLID-Prinzipien zu befolgen.

### Services / Services

Beginnen wir mit der Erstellung eines einfachen `CatsService`. Dieser Service wird für die Datenspeicherung und -abfrage verantwortlich sein und soll vom `CatsController` verwendet werden, weshalb er ein guter Kandidat ist, als Anbieter definiert zu werden.

```typescript
// cats.service.ts
import { Injectable } from '@nestjs/common';
import { Cat } from './interfaces/cat.interface';

@Injectable()
export class CatsService {
  private readonly cats: Cat[] = [];

  create(cat: Cat) {
    this.cats.push(cat);
  }

  findAll(): Cat[] {
    return this.cats;
  }
}
```

**HINWEIS / HINT:** Um einen Service mit der CLI zu erstellen, führen Sie einfach den Befehl `$ nest g service cats` aus.

Unser `CatsService` ist eine einfache Klasse mit einer Eigenschaft und zwei Methoden. Das einzige neue Merkmal ist, dass es den `@Injectable()` Dekorator verwendet. Der `@Injectable()` Dekorator fügt Metadaten hinzu, die erklären, dass `CatsService` eine Klasse ist, die vom Nest IoC-Container verwaltet werden kann. Übrigens verwendet dieses Beispiel auch ein `Cat`-Interface, das wahrscheinlich so aussieht:

```typescript
// interfaces/cat.interface.ts
export interface Cat {
  name: string;
  age: number;
  breed: string;
}
```

Jetzt, da wir eine Service-Klasse haben, um Katzen abzurufen, verwenden wir sie im `CatsController`:

```typescript
// cats.controller.ts
import { Controller, Get, Post, Body } from '@nestjs/common';
import { CreateCatDto } from './dto/create-cat.dto';
import { CatsService } from './cats.service';
import { Cat } from './interfaces/cat.interface';

@Controller('cats')
export class CatsController {
  constructor(private catsService: CatsService) {}

  @Post()
  async create(@Body() createCatDto: CreateCatDto) {
    this.catsService.create(createCatDto);
  }

  @Get()
  async findAll(): Promise<Cat[]> {
    return this.catsService.findAll();
  }
}
```

Der `CatsService` wird über den Klassenkonstruktor injiziert. Beachten Sie die Verwendung der `private`-Syntax. Diese Kurzform ermöglicht es uns, das `catsService`-Mitglied sofort an derselben Stelle zu deklarieren und zu initialisieren.

### Dependency Injection / Dependency injection

Nest ist um das starke Entwurfsmuster der Dependency Injection herum aufgebaut. Wir empfehlen, einen großartigen Artikel über dieses Konzept in der offiziellen Angular-Dokumentation zu lesen.

In Nest ist es dank der TypeScript-Fähigkeiten extrem einfach, Abhängigkeiten zu verwalten, da sie nur durch den Typ aufgelöst werden. Im folgenden Beispiel wird Nest den `catsService` auflösen, indem es eine Instanz von `CatsService` erstellt und zurückgibt (oder im Normalfall eines Singletons die vorhandene Instanz zurückgibt, wenn sie bereits an anderer Stelle angefordert wurde). Diese Abhängigkeit wird aufgelöst und an den Konstruktor Ihres Controllers übergeben (oder der angegebenen Eigenschaft zugewiesen):

```typescript
constructor(private catsService: CatsService) {}
```

### Scopes / Scopes

Anbieter haben normalerweise eine Lebensdauer ("Scope"), die mit dem Anwendungslebenszyklus synchronisiert ist. Wenn die Anwendung gebootet wird, muss jede Abhängigkeit aufgelöst werden, und daher muss jeder Anbieter instanziiert werden. Ebenso wird jeder Anbieter zerstört, wenn die Anwendung heruntergefahren wird. Es gibt jedoch Möglichkeiten, die Lebensdauer Ihres Anbieters auch auf Anfragenebene zu gestalten. Mehr dazu erfahren Sie hier.

### Benutzerdefinierte Anbieter / Custom providers

Nest verfügt über einen eingebauten Inversion-of-Control ("IoC")-Container, der die Beziehungen zwischen Anbietern auflöst. Dieses Feature liegt der oben beschriebenen Dependency Injection zugrunde, ist aber in der Tat weitaus leistungsfähiger als bisher beschrieben. Es gibt mehrere Möglichkeiten, einen Anbieter zu definieren: Sie können einfache Werte, Klassen und entweder asynchrone oder synchrone Fabriken verwenden. Weitere Beispiele finden Sie hier.

### Optionale Anbieter / Optional providers

Gelegentlich haben Sie möglicherweise Abhängigkeiten, die nicht unbedingt aufgelöst werden müssen. Zum Beispiel könnte Ihre Klasse von einem Konfigurationsobjekt abhängen, aber wenn keines übergeben wird, sollten die Standardwerte verwendet werden. In einem solchen Fall wird die Abhängigkeit optional, da das Fehlen des Konfigurationsanbieters nicht zu Fehlern führen würde.

Um anzugeben, dass ein Anbieter optional ist, verwenden Sie den `@Optional()` Dekorator in der Signatur des Konstruktors:

```typescript
import { Injectable, Optional, Inject } from '@nestjs/common';

@Injectable()
export class HttpService<T> {
  constructor(@Optional() @Inject('HTTP_OPTIONS') private httpClient: T) {}
}
```

Beachten Sie, dass wir im obigen Beispiel einen benutzerdefinierten Anbieter verwenden, weshalb wir das benutzerdefinierte Token `HTTP_OPTIONS` einschließen. Frühere Beispiele zeigten eine Konstruktor-basierte Injektion, die eine Abhängigkeit durch eine Klasse im Konstruktor angibt. Lesen Sie mehr über benutzerdefinierte Anbieter und ihre zugehörigen Tokens hier.

### Eigenschaftsbasierte Injektion / Property-based injection

Die bisher verwendete Technik wird als Konstruktor-basierte Injektion bezeichnet, da Anbieter über die Konstruktormethode injiziert werden. In einigen sehr spezifischen Fällen kann die eigenschaftsbasierte Injektion nützlich sein. Zum Beispiel, wenn Ihre oberste Klasse von einem oder mehreren Anbietern abhängt, das Weitergeben all dieser Anbieter durch Aufruf von `super()` in Unterklassen aus dem Konstruktor heraus jedoch sehr mühsam sein kann. Um dies zu vermeiden, können Sie den `@Inject()` Dekorator auf der Eigenschaftsebene verwenden:

```typescript
import { Injectable, Inject } from '@nestjs/common';

@Injectable()
export class HttpService<T> {
  @Inject('HTTP_OPTIONS')
  private readonly httpClient: T;
}
```

**WARNUNG / WARNING:** Wenn Ihre Klasse keine andere Klasse erweitert, sollten Sie immer die Konstruktor-basierte Injektion bevorzugen. Der Konstruktor gibt explizit an, welche Abhängigkeiten erforderlich sind und bietet bessere Sichtbarkeit als Klassenattribute, die mit `@Inject` annotiert sind.

### Registrierung von Anbietern / Provider registration

Jetzt, da wir einen Anbieter (`CatsService`) definiert haben und einen Verbraucher dieses Dienstes (`CatsController`) haben, müssen wir den Dienst bei Nest registrieren, damit er die Injektion durchführen kann. Dies tun wir, indem wir unsere Moduldaratei (`app.module.ts`) bearbeiten und den Dienst dem `providers` Array des `@Module()` Dekorators hinzufügen:

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

Nest kann nun die Abhängigkeiten der `CatsController`-Klasse auflösen.

So sollte unsere Verzeichnisstruktur jetzt aussehen:

```
src
|-- cats
|   |-- dto
|   |   `-- create-cat.dto.ts
|   |-- interfaces
|   |   `-- cat.interface.ts
|   |-- cats.controller.ts
|   |-- cats.service.ts
|-- app.module.ts
|-- main.ts
```

### Manuelle Instanziierung / Manual instantiation

Bisher haben wir besprochen, wie Nest automatisch die meisten Details der Auflösung von Abhängigkeiten behandelt. Unter bestimmten Umständen müssen Sie das eingebaute Dependency Injection System verlassen und Anbieter manuell abrufen oder instanziieren. Wir diskutieren kurz zwei solche Themen unten.

Um vorhandene Instanzen zu erhalten oder Anbieter dynamisch zu instanziieren, können Sie die Modulreferenz verwenden.

Um Anbieter innerhalb der `bootstrap()`-Funktion zu erhalten (z. B. für eigenständige Anwendungen ohne Controller oder um einen Konfigurationsdienst während des Bootstrappings zu nutzen), siehe Eigenständige Anwendungen.
