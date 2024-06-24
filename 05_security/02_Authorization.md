# Autorisierung / Authorization

Autorisierung bezieht sich auf den Prozess, der bestimmt, was ein Benutzer tun darf. Ein administrativer Benutzer darf beispielsweise Beiträge erstellen, bearbeiten und löschen. Ein nicht-administrativer Benutzer ist nur berechtigt, die Beiträge zu lesen.

Autorisierung ist orthogonal und unabhängig von Authentifizierung. Jedoch erfordert Autorisierung einen Authentifizierungsmechanismus.

Es gibt viele verschiedene Ansätze und Strategien zur Handhabung der Autorisierung. Der gewählte Ansatz hängt von den speziellen Anforderungen des jeweiligen Projekts ab. Dieses Kapitel stellt einige Ansätze zur Autorisierung vor, die an eine Vielzahl unterschiedlicher Anforderungen angepasst werden können.

## Grundlegende RBAC-Implementierung / Basic RBAC implementation

Rollenbasierte Zugriffskontrolle (RBAC) ist ein politisch neutrales Zugriffskontrollmechanismus, das um Rollen und Privilegien definiert ist. In diesem Abschnitt demonstrieren wir, wie man ein sehr einfaches RBAC-Mechanismus mithilfe von Nest Guards implementiert.

Zuerst erstellen wir ein Role-Enum, das die Rollen im System darstellt:

**role.enum.ts**

```typescript
export enum Role {
  User = 'user',
  Admin = 'admin',
}
```

**HINWEIS**
In ausgefeilteren Systemen können Sie Rollen in einer Datenbank speichern oder von einem externen Authentifizierungsanbieter beziehen.

Mit diesen Voraussetzungen können wir einen @Roles()-Dekorator erstellen. Dieser Dekorator ermöglicht die Angabe, welche Rollen erforderlich sind, um auf bestimmte Ressourcen zuzugreifen.

**roles.decorator.ts**

```typescript
import { SetMetadata } from '@nestjs/common';
import { Role } from '../enums/role.enum';

export const ROLES_KEY = 'roles';
export const Roles = (...roles: Role[]) => SetMetadata(ROLES_KEY, roles);
```

Da wir nun einen benutzerdefinierten @Roles()-Dekorator haben, können wir ihn verwenden, um jeden Routenhandler zu dekorieren.

**cats.controller.ts**

```typescript
@Post()
@Roles(Role.Admin)
create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
```

Schließlich erstellen wir eine RolesGuard-Klasse, die die dem aktuellen Benutzer zugewiesenen Rollen mit den tatsächlichen Rollen vergleicht, die von der aktuellen Route, die verarbeitet wird, erforderlich sind. Um auf die Rollen der Route zuzugreifen (benutzerdefinierte Metadaten), verwenden wir die Reflector-Hilfsklasse, die vom Framework bereitgestellt und aus dem @nestjs/core-Paket exportiert wird.

**roles.guard.ts**

```typescript
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Reflector } from '@nestjs/core';

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.getAllAndOverride<Role[]>(ROLES_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);
    if (!requiredRoles) {
      return true;
    }
    const { user } = context.switchToHttp().getRequest();
    return requiredRoles.some((role) => user.roles?.includes(role));
  }
}
```

**HINWEIS**
Siehe den Abschnitt "Reflektion und Metadaten" im Kapitel "Ausführungskontext" für weitere Details zur Verwendung von Reflector in einem kontextsensitiven Umfeld.

**HINWEIS**
Dieses Beispiel wird als "grundlegend" bezeichnet, da wir nur die Anwesenheit von Rollen auf der Routenhandler-Ebene überprüfen. In realen Anwendungen können Sie Endpunkte/Handler haben, die mehrere Operationen umfassen, von denen jede einen bestimmten Satz von Berechtigungen erfordert. In diesem Fall müssen Sie einen Mechanismus bereitstellen, um Rollen irgendwo innerhalb Ihrer Geschäftslogik zu überprüfen, was es etwas schwieriger zu warten macht, da es keinen zentralen Ort gibt, der Berechtigungen mit spezifischen Aktionen verknüpft.

In diesem Beispiel gehen wir davon aus, dass request.user die Benutzerinstanz und die erlaubten Rollen (unter der Eigenschaft roles) enthält. In Ihrer App werden Sie diese Zuordnung wahrscheinlich in Ihrem benutzerdefinierten Authentifizierungsschutz vornehmen - siehe Authentifizierungskapitel für weitere Details.

Um sicherzustellen, dass dieses Beispiel funktioniert, muss Ihre Benutzerklasse wie folgt aussehen:

```typescript
class User {
  // ...andere Eigenschaften
  roles: Role[];
}
```

Stellen Sie abschließend sicher, dass Sie den RolesGuard registrieren, beispielsweise auf Controller-Ebene oder global:

```typescript
providers: [
  {
    provide: APP_GUARD,
    useClass: RolesGuard,
  },
],
```

Wenn ein Benutzer mit unzureichenden Berechtigungen einen Endpunkt anfordert, gibt Nest automatisch die folgende Antwort zurück:

```json
{
  "statusCode": 403,
  "message": "Forbidden resource",
  "error": "Forbidden"
}
```

**HINWEIS**
Wenn Sie eine andere Fehlermeldung zurückgeben möchten, sollten Sie eine eigene spezifische Ausnahme auslösen, anstatt einen booleschen Wert zurückzugeben.

## Claims-basierte Autorisierung / Claims-based authorization

Wenn eine Identität erstellt wird, können ihr eine oder mehrere Ansprüche von einer vertrauenswürdigen Partei zugewiesen werden. Ein Anspruch ist ein Name-Wert-Paar, das repräsentiert, was das Subjekt tun kann, nicht was das Subjekt ist.

Um eine Claims-basierte Autorisierung in Nest zu implementieren, können Sie die gleichen Schritte wie im RBAC-Abschnitt oben befolgen, mit einem wesentlichen Unterschied: Anstatt nach spezifischen Rollen zu suchen, sollten Sie Berechtigungen vergleichen. Jeder Benutzer hätte eine Reihe von Berechtigungen zugewiesen. Ebenso würde jede Ressource/jeder Endpunkt definieren, welche Berechtigungen erforderlich sind (zum Beispiel durch einen dedizierten @RequirePermissions()-Dekorator), um auf sie zuzugreifen.

**cats.controller.ts**

```typescript
@Post()
@RequirePermissions(Permission.CREATE_CAT)
create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
```

**HINWEIS**
Im obigen Beispiel ist Permission (ähnlich wie Role im RBAC-Abschnitt) ein TypeScript-Enum, das alle Berechtigungen enthält, die in Ihrem System verfügbar sind.

## Integration von CASL / Integrating CASL

CASL ist eine isomorphe Autorisierungsbibliothek, die einschränkt, auf welche Ressourcen ein gegebener Client zugreifen darf. Sie ist so konzipiert, dass sie schrittweise anpassbar ist und leicht zwischen einer einfachen Claim-basierten und einer vollständig ausgestatteten Subjekt- und Attribut-basierten Autorisierung skalieren kann.

Zuerst installieren wir das @casl/ability-Paket:

```sh
$ npm i @casl/ability
```

**HINWEIS**
In diesem Beispiel haben wir uns für CASL entschieden, aber Sie können auch jede andere Bibliothek wie accesscontrol oder acl verwenden, je nach Ihren Vorlieben und Projektanforderungen.

Nachdem die Installation abgeschlossen ist, definieren wir der Einfachheit halber die Mechanik von CASL, indem wir zwei Entitätsklassen erstellen: User und Article.

```typescript
class User {
  id: number;
  isAdmin: boolean;
}
```

Die User-Klasse besteht aus zwei Eigenschaften, id, die ein eindeutiger Benutzeridentifikator ist, und isAdmin, die angibt, ob ein Benutzer Administratorrechte hat.

```typescript
class Article {
  id: number;
  isPublished: boolean;
  authorId: number;
}
```

Die Artikel-Klasse hat drei Eigenschaften, nämlich id, isPublished und authorId. id ist ein eindeutiger Artikelidentifikator, isPublished gibt an, ob ein Artikel bereits veröffentlicht wurde oder nicht, und authorId ist die ID eines Benutzers, der den Artikel geschrieben hat.

Nun lassen Sie uns unsere Anforderungen für dieses Beispiel überprüfen und verfeinern:

- Administratoren können alle Entitäten verwalten (erstellen/lesen/aktualisieren/löschen).
- Benutzer haben Lesezugriff auf alles.
- Benutzer können ihre Artikel aktualisieren (article.authorId === userId).
- Bereits veröffentlichte Artikel können nicht entfernt werden (article.isPublished === true).

Mit diesen Anforderungen im Hinterkopf können wir ein Action-Enum erstellen, das alle möglichen Aktionen darstellt, die Benutzer mit Entitäten durchführen können:

```typescript
export enum Action {
  Manage = 'manage',
  Create = 'create',
  Read = 'read',
  Update = 'update',
  Delete = 'delete',
}
```

**HINWEIS**
manage ist ein spezielles Schlüsselwort in CASL, das "jede" Aktion repräsentiert.

Um die CASL-Bibliothek zu kapseln, generieren wir nun das CaslModule und die CaslAbilityFactory.

```sh
$ nest g module casl
$ nest g class casl/casl-ability.factory
```

Damit können wir die createForUser()-Methode in der CaslAbilityFactory definieren. Diese Methode erstellt das Ability-Objekt für einen gegebenen Benutzer:

```typescript
type Subjects = InferSubjects<typeof Article | typeof User> | 'all';

export type AppAbility = Ability<[Action, Subjects]>;

@Injectable()
export class CaslAbilityFactory {
  createForUser(user: User) {
    const { can, cannot, build } = new AbilityBuilder<
      Ability<[Action, Subjects]>
    >(Ability as AbilityClass<AppAbility>);

    if (user.isAdmin) {
      can(Action.Manage, 'all

'); // Lese- und Schreibzugriff auf alles
    } else {
      can(Action.Read, 'all'); // Nur Lesezugriff auf alles
    }

    can(Action.Update, Article, { authorId: user.id });
    cannot(Action.Delete, Article, { isPublished: true });

    return build({
      // Lesen Sie https://casl.js.org/v6/en/guide/subject-type-detection#use-classes-as-subject-types für Details
      detectSubjectType: (item) =>
        item.constructor as ExtractSubjectType<Subjects>,
    });
  }
}
```

**HINWEIS**
all ist ein spezielles Schlüsselwort in CASL, das "jedes Subjekt" repräsentiert.

**HINWEIS**
Die Ability, AbilityBuilder, AbilityClass und ExtractSubjectType Klassen werden aus dem @casl/ability-Paket exportiert.

**HINWEIS**
Die detectSubjectType-Option lässt CASL verstehen, wie der Subjekttyp aus einem Objekt ermittelt wird. Weitere Informationen finden Sie in der CASL-Dokumentation.

Im obigen Beispiel haben wir die Ability-Instanz mithilfe der AbilityBuilder-Klasse erstellt. Wie Sie wahrscheinlich vermutet haben, akzeptieren can und cannot die gleichen Argumente, haben aber unterschiedliche Bedeutungen. can erlaubt das Ausführen einer Aktion auf dem angegebenen Subjekt und cannot verbietet dies. Beide können bis zu vier Argumente akzeptieren. Um mehr über diese Funktionen zu erfahren, besuchen Sie die offizielle CASL-Dokumentation.

Stellen Sie abschließend sicher, dass Sie die CaslAbilityFactory zu den Anbietern und Exporten im CaslModule-Modul hinzufügen:

```typescript
import { Module } from '@nestjs/common';
import { CaslAbilityFactory } from './casl-ability.factory';

@Module({
  providers: [CaslAbilityFactory],
  exports: [CaslAbilityFactory],
})
export class CaslModule {}
```

Damit können wir die CaslAbilityFactory in jeder Klasse mithilfe der Standardkonstruktorinjektion injizieren, solange das CaslModule im Hostkontext importiert ist:

```typescript
constructor(private caslAbilityFactory: CaslAbilityFactory) {}
```

Dann verwenden wir es in einer Klasse wie folgt:

```typescript
const ability = this.caslAbilityFactory.createForUser(user);
if (ability.can(Action.Read, 'all')) {
  // "user" hat Lesezugriff auf alles
}
```

**HINWEIS**
Erfahren Sie mehr über die Ability-Klasse in der offiziellen CASL-Dokumentation.

Angenommen, wir haben einen Benutzer, der kein Administrator ist. In diesem Fall sollte der Benutzer Artikel lesen können, aber das Erstellen neuer Artikel oder das Entfernen bestehender Artikel sollte verboten sein:

```typescript
const user = new User();
user.isAdmin = false;

const ability = this.caslAbilityFactory.createForUser(user);
ability.can(Action.Read, Article); // true
ability.can(Action.Delete, Article); // false
ability.can(Action.Create, Article); // false
```

**HINWEIS**
Obwohl sowohl die Ability- als auch die AbilityBuilder-Klasse can- und cannot-Methoden bereitstellen, haben sie unterschiedliche Zwecke und akzeptieren leicht unterschiedliche Argumente.

Wie in unseren Anforderungen angegeben, sollte der Benutzer in der Lage sein, seine Artikel zu aktualisieren:

```typescript
const user = new User();
user.id = 1;

const article = new Article();
article.authorId = user.id;

const ability = this.caslAbilityFactory.createForUser(user);
ability.can(Action.Update, article); // true

article.authorId = 2;
ability.can(Action.Update, article); // false
```

Wie Sie sehen können, ermöglicht uns die Ability-Instanz das Überprüfen von Berechtigungen auf eine ziemlich lesbare Weise. Ebenso ermöglicht es uns AbilityBuilder, Berechtigungen (und verschiedene Bedingungen) auf ähnliche Weise zu definieren. Weitere Beispiele finden Sie in der offiziellen Dokumentation.

## Fortgeschritten: Implementierung eines PoliciesGuard / Advanced: Implementing a PoliciesGuard

In diesem Abschnitt demonstrieren wir, wie man einen etwas ausgefeilteren Guard erstellt, der überprüft, ob ein Benutzer bestimmte Autorisierungsrichtlinien erfüllt, die auf Methodenebene konfiguriert werden können (Sie können ihn erweitern, um auch Richtlinien zu respektieren, die auf Klassenebene konfiguriert sind). In diesem Beispiel verwenden wir das CASL-Paket nur zu Illustrationszwecken, aber die Verwendung dieser Bibliothek ist nicht erforderlich. Außerdem verwenden wir den CaslAbilityFactory-Anbieter, den wir im vorherigen Abschnitt erstellt haben.

Zuerst lassen Sie uns die Anforderungen konkretisieren. Das Ziel ist es, einen Mechanismus bereitzustellen, der es ermöglicht, Richtlinienprüfungen pro Routenhandler anzugeben. Wir unterstützen sowohl Objekte als auch Funktionen (für einfachere Überprüfungen und für diejenigen, die einen funktionaleren Code bevorzugen).

Beginnen wir mit der Definition von Schnittstellen für Richtlinienhandler:

```typescript
import { AppAbility } from '../casl/casl-ability.factory';

interface IPolicyHandler {
  handle(ability: AppAbility): boolean;
}

type PolicyHandlerCallback = (ability: AppAbility) => boolean;

export type PolicyHandler = IPolicyHandler | PolicyHandlerCallback;
```

Wie oben erwähnt, haben wir zwei mögliche Arten der Definition eines Richtlinienhandlers bereitgestellt: ein Objekt (Instanz einer Klasse, die die IPolicyHandler-Schnittstelle implementiert) und eine Funktion (die den PolicyHandlerCallback-Typ erfüllt).

Damit können wir einen @CheckPolicies()-Dekorator erstellen. Dieser Dekorator ermöglicht die Angabe, welche Richtlinien erfüllt sein müssen, um auf bestimmte Ressourcen zuzugreifen.

```typescript
export const CHECK_POLICIES_KEY = 'check_policy';
export const CheckPolicies = (...handlers: PolicyHandler[]) =>
  SetMetadata(CHECK_POLICIES_KEY, handlers);
```

Nun erstellen wir einen PoliciesGuard, der alle an einen Routenhandler gebundenen Richtlinienhandler extrahiert und ausführt.

```typescript
@Injectable()
export class PoliciesGuard implements CanActivate {
  constructor(
    private reflector: Reflector,
    private caslAbilityFactory: CaslAbilityFactory,
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const policyHandlers =
      this.reflector.get<PolicyHandler[]>(
        CHECK_POLICIES_KEY,
        context.getHandler(),
      ) || [];

    const { user } = context.switchToHttp().getRequest();
    const ability = this.caslAbilityFactory.createForUser(user);

    return policyHandlers.every((handler) =>
      this.execPolicyHandler(handler, ability),
    );
  }

  private execPolicyHandler(handler: PolicyHandler, ability: AppAbility) {
    if (typeof handler === 'function') {
      return handler(ability);
    }
    return handler.handle(ability);
  }
}
```

**HINWEIS**
In diesem Beispiel gehen wir davon aus, dass request.user die Benutzerinstanz enthält. In Ihrer App werden Sie diese Zuordnung wahrscheinlich in Ihrem benutzerdefinierten Authentifizierungsschutz vornehmen - siehe Authentifizierungskapitel für weitere Details.

Lassen Sie uns dieses Beispiel aufschlüsseln. Die policyHandlers sind ein Array von Handlers, die der Methode durch den @CheckPolicies()-Dekorator zugewiesen sind. Als Nächstes verwenden wir die Methode CaslAbilityFactory#create, die das Ability-Objekt erstellt, mit dem wir überprüfen können, ob ein Benutzer über ausreichende Berechtigungen verfügt, um bestimmte Aktionen durchzuführen. Wir übergeben dieses Objekt dem Richtlinienhandler, der entweder eine Funktion oder eine Instanz einer Klasse ist, die die IPolicyHandler-Schnittstelle implementiert und die handle()-Methode bereitstellt, die einen booleschen Wert zurückgibt. Schließlich verwenden wir die Array#every-Methode, um sicherzustellen, dass jeder Handler einen true-Wert zurückgegeben hat.

Um diesen Guard zu testen, binden Sie ihn an einen beliebigen Routenhandler und registrieren Sie einen Inline-Richtlinienhandler (funktionaler Ansatz), wie folgt:

```typescript
@Get()
@UseGuards(PoliciesGuard)
@CheckPolicies((ability: AppAbility) => ability.can(Action.Read, Article))
findAll() {
  return this.articlesService.findAll();
}
```

Alternativ können wir eine Klasse definieren, die die IPolicyHandler-Schnittstelle implementiert:

```typescript
export class ReadArticlePolicyHandler implements IPolicyHandler {
  handle(ability: AppAbility) {
    return ability.can(Action.Read, Article);
  }
}
```

Und verwenden Sie es wie folgt:

```typescript
@Get()
@UseGuards(PoliciesGuard)
@CheckPolicies(new ReadArticlePolicyHandler())
findAll() {
  return this.articlesService.findAll();
}
```

**HINWEIS**
Da wir den Richtlinienhandler an Ort und Stelle mit dem new-Schlüsselwort instanziieren müssen, kann die ReadArticlePolicyHandler-Klasse keine Abhängigkeitsinjektion verwenden. Dies kann mit der ModuleRef#get-Methode (lesen Sie [hier](https://docs.nestjs.com/fundamentals/module-ref) mehr) behoben werden. Grundsätzlich müssen Sie anstelle der Registrierung von Funktionen und Instanzen über den @CheckPolicies()-Dekorator die Übergabe eines Type<IPolicyHandler> zulassen. Dann könnten Sie in Ihrem Guard eine Instanz mit einem Typverweis abrufen: moduleRef.get(IHREN_HANDLER_TYP) oder sie sogar dynamisch mit der Methode ModuleRef#create instanziieren.
