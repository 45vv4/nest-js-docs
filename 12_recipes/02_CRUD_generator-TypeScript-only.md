# CRUD-Generator (nur TypeScript) / CRUD generator (TypeScript only)

Während der Lebensdauer eines Projekts, wenn wir neue Funktionen entwickeln, müssen wir oft neue Ressourcen zu unserer Anwendung hinzufügen. Diese Ressourcen erfordern typischerweise mehrere, sich wiederholende Operationen, die wir jedes Mal wiederholen müssen, wenn wir eine neue Ressource definieren.

## Einführung / Introduction

Stellen wir uns ein Szenario aus der realen Welt vor, in dem wir CRUD-Endpunkte für 2 Entitäten, sagen wir Benutzer- und Produktentitäten, bereitstellen müssen. Nach den Best Practices müssten wir für jede Entität mehrere Operationen durchführen, wie folgt:

1. Generieren eines Moduls (`nest g mo`), um den Code organisiert zu halten und klare Grenzen zu schaffen (Gruppierung verwandter Komponenten).
2. Generieren eines Controllers (`nest g co`), um CRUD-Routen (oder Abfragen/Mutationen für GraphQL-Anwendungen) zu definieren.
3. Generieren eines Services (`nest g s`), um die Geschäftslogik zu implementieren und zu isolieren.
4. Generieren einer Entitätsklasse/-schnittstelle, um die Datenstruktur der Ressource darzustellen.
5. Generieren von Data Transfer Objects (oder Eingaben für GraphQL-Anwendungen), um zu definieren, wie die Daten über das Netzwerk gesendet werden.

Das sind viele Schritte!

Um diesen sich wiederholenden Prozess zu beschleunigen, bietet die Nest-CLI einen Generator (Schematic), der automatisch den gesamten Boilerplate-Code generiert, um uns all diese Arbeit zu ersparen und die Entwicklererfahrung wesentlich einfacher zu gestalten.

**HINWEIS / NOTE**
Der Schematic unterstützt das Generieren von HTTP-Controllern, Microservice-Controllern, GraphQL-Resolv-ern (sowohl Code-First als auch Schema-First) und WebSocket-Gateways.

## Generieren einer neuen Ressource / Generating a new resource

Um eine neue Ressource zu erstellen, führen Sie einfach den folgenden Befehl im Stammverzeichnis Ihres Projekts aus:

```shell
$ nest g resource
```

Der Befehl `nest g resource` generiert nicht nur alle NestJS-Bausteine (Modul-, Service-, Controller-Klassen), sondern auch eine Entitätsklasse, DTO-Klassen sowie die Testdateien (.spec).

Unten sehen Sie die generierte Controller-Datei (für die REST-API):

```typescript
@Controller('users')
export class UsersController {
  constructor(private readonly usersService: UsersService) {}

  @Post()
  create(@Body() createUserDto: CreateUserDto) {
    return this.usersService.create(createUserDto);
  }

  @Get()
  findAll() {
    return this.usersService.findAll();
  }

  @Get(':id')
  findOne(@Param('id') id: string) {
    return this.usersService.findOne(+id);
  }

  @Patch(':id')
  update(@Param('id') id: string, @Body() updateUserDto: UpdateUserDto) {
    return this.usersService.update(+id, updateUserDto);
  }

  @Delete(':id')
  remove(@Param('id') id: string) {
    return this.usersService.remove(+id);
  }
}
```

Es werden auch automatisch Platzhalter für alle CRUD-Endpunkte (Routen für REST-APIs, Abfragen und Mutationen für GraphQL, Nachrichten-Abonnements für sowohl Microservices als auch WebSocket-Gateways) erstellt - alles ohne einen Finger zu rühren.

**HINWEIS / NOTE**
Generierte Serviceklassen sind nicht an ein spezifisches ORM (oder Datenquelle) gebunden. Dies macht den Generator generisch genug, um den Anforderungen jedes Projekts gerecht zu werden. Standardmäßig enthalten alle Methoden Platzhalter, die Sie mit den für Ihr Projekt spezifischen Datenquellen füllen können.

Ebenso, wenn Sie Resolver für eine GraphQL-Anwendung generieren möchten, wählen Sie einfach GraphQL (Code-First) (oder GraphQL (Schema-First)) als Ihre Transportschicht aus.

In diesem Fall wird NestJS eine Resolver-Klasse anstelle eines REST-API-Controllers generieren:

```shell
$ nest g resource users

> ? What transport layer do you use? GraphQL (code first)
> ? Would you like to generate CRUD entry points? Yes
> CREATE src/users/users.module.ts (224 bytes)
> CREATE src/users/users.resolver.spec.ts (525 bytes)
> CREATE src/users/users.resolver.ts (1109 bytes)
> CREATE src/users/users.service.spec.ts (453 bytes)
> CREATE src/users/users.service.ts (625 bytes)
> CREATE src/users/dto/create-user.input.ts (195 bytes)
> CREATE src/users/dto/update-user.input.ts (281 bytes)
> CREATE src/users/entities/user.entity.ts (187 bytes)
> UPDATE src/app.module.ts (312 bytes)
```

**TIPP / HINT**
Um zu vermeiden, dass Testdateien generiert werden, können Sie das `--no-spec`-Flag wie folgt setzen: `nest g resource users --no-spec`

Wie wir unten sehen können, wurden nicht nur alle Boilerplate-Mutationen und Abfragen erstellt, sondern alles ist auch miteinander verknüpft. Wir verwenden den UsersService, die User-Entität und unsere DTOs.

```typescript
import { Resolver, Query, Mutation, Args, Int } from '@nestjs/graphql';
import { UsersService } from './users.service';
import { User } from './entities/user.entity';
import { CreateUserInput } from './dto/create-user.input';
import { UpdateUserInput } from './dto/update-user.input';

@Resolver(() => User)
export class UsersResolver {
  constructor(private readonly usersService: UsersService) {}

  @Mutation(() => User)
  createUser(@Args('createUserInput') createUserInput: CreateUserInput) {
    return this.usersService.create(createUserInput);
  }

  @Query(() => [User], { name: 'users' })
  findAll() {
    return this.usersService.findAll();
  }

  @Query(() => User, { name: 'user' })
  findOne(@Args('id', { type: () => Int }) id: number) {
    return this.usersService.findOne(id);
  }

  @Mutation(() => User)
  updateUser(@Args('updateUserInput') updateUserInput: UpdateUserInput) {
    return this.usersService.update(updateUserInput.id, updateUserInput);
  }

  @Mutation(() => User)
  removeUser(@Args('id', { type: () => Int }) id: number) {
    return this.usersService.remove(id);
  }
}
```

## Unterstützen Sie uns / Support us

Nest ist ein MIT-lizenziertes Open-Source-Projekt. Es kann dank der Unterstützung dieser großartigen Menschen wachsen. Wenn Sie sich ihnen anschließen möchten, lesen Sie hier mehr.