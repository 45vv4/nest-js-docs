## Guards / Guards

Ein Guard ist eine Klasse, die mit dem `@Injectable()` Dekorator annotiert ist und die `CanActivate`-Schnittstelle implementiert.

Guards haben eine einzige Verantwortung: Sie entscheiden, ob eine gegebene Anfrage vom Routen-Handler bearbeitet wird oder nicht, abhängig von bestimmten Bedingungen (wie Berechtigungen, Rollen, ACLs usw.), die zur Laufzeit vorliegen. Dies wird oft als Autorisierung bezeichnet. Autorisierung (und die damit oft zusammenarbeitende Authentifizierung) wurde in traditionellen Express-Anwendungen typischerweise durch Middleware gehandhabt. Middleware ist eine gute Wahl für die Authentifizierung, da Dinge wie Token-Validierung und das Anhängen von Eigenschaften an das Request-Objekt nicht stark mit einem bestimmten Routen-Kontext (und dessen Metadaten) verbunden sind.

Aber Middleware ist von Natur aus "dumm". Sie weiß nicht, welcher Handler nach dem Aufruf der `next()` Funktion ausgeführt wird. Guards hingegen haben Zugriff auf die `ExecutionContext`-Instanz und wissen daher genau, was als Nächstes ausgeführt wird. Sie sind so konzipiert, dass sie, ähnlich wie Ausnahmefilter, Pipes und Interceptors, es ermöglichen, Verarbeitungsschritte genau an der richtigen Stelle im Anfrage-/Antwort-Zyklus einzufügen und dies deklarativ zu tun. Dies hilft, den Code DRY (Don't Repeat Yourself) und deklarativ zu halten.

### HINWEIS
Guards werden nach aller Middleware, aber vor jedem Interceptor oder Pipe ausgeführt.

### Autorisierungs-Guard / Authorization guard

Wie erwähnt, ist die Autorisierung ein großartiger Anwendungsfall für Guards, da bestimmte Routen nur verfügbar sein sollten, wenn der Anrufer (normalerweise ein spezifisch authentifizierter Benutzer) ausreichende Berechtigungen hat. Der AuthGuard, den wir jetzt erstellen, geht von einem authentifizierten Benutzer aus (und dass daher ein Token an die Anfrage-Header angehängt ist). Er wird das Token extrahieren und validieren und die extrahierten Informationen verwenden, um zu bestimmen, ob die Anfrage fortgesetzt werden kann oder nicht.

```typescript
// auth.guard.ts

import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Observable } from 'rxjs';

@Injectable()
export class AuthGuard implements CanActivate {
  canActivate(
    context: ExecutionContext,
  ): boolean | Promise<boolean> | Observable<boolean> {
    const request = context.switchToHttp().getRequest();
    return validateRequest(request);
  }
}
```

### HINWEIS
Wenn Sie nach einem realen Beispiel suchen, wie Sie einen Authentifizierungsmechanismus in Ihrer Anwendung implementieren, besuchen Sie dieses Kapitel. Ebenso finden Sie hier ein ausgefeilteres Beispiel für die Autorisierung.

Die Logik innerhalb der `validateRequest()` Funktion kann so einfach oder ausgefeilt sein, wie nötig. Der Hauptpunkt dieses Beispiels ist zu zeigen, wie Guards in den Anfrage-/Antwort-Zyklus passen.

Jeder Guard muss eine `canActivate()` Funktion implementieren. Diese Funktion sollte einen Boolean zurückgeben, der anzeigt, ob die aktuelle Anfrage erlaubt ist oder nicht. Sie kann die Antwort entweder synchron oder asynchron (über ein Promise oder Observable) zurückgeben. Nest verwendet den Rückgabewert, um die nächste Aktion zu steuern:

- Wenn `true` zurückgegeben wird, wird die Anfrage verarbeitet.
- Wenn `false` zurückgegeben wird, verweigert Nest die Anfrage.

### ExecutionContext / Execution context

Die `canActivate()` Funktion nimmt ein einziges Argument, die `ExecutionContext`-Instanz. `ExecutionContext` erbt von `ArgumentsHost`. Wir haben `ArgumentsHost` bereits im Kapitel über Ausnahmefilter gesehen. Im obigen Beispiel verwenden wir die gleichen Hilfsmethoden, die in `ArgumentsHost` definiert sind, um eine Referenz auf das Request-Objekt zu erhalten. Sie können auf den Abschnitt über `ArgumentsHost` im Kapitel über Ausnahmefilter zurückgreifen, um mehr zu diesem Thema zu erfahren.

Durch die Erweiterung von `ArgumentsHost` fügt `ExecutionContext` auch mehrere neue Hilfsmethoden hinzu, die zusätzliche Details über den aktuellen Ausführungsprozess bereitstellen. Diese Details können hilfreich sein, um allgemeinere Guards zu erstellen, die über eine breite Palette von Controllern, Methoden und Ausführungskontexten hinweg arbeiten können. Erfahren Sie mehr über `ExecutionContext` [hier](https://docs.nestjs.com).

### Rollenbasierte Authentifizierung / Role-based authentication

Lassen Sie uns einen funktionaleren Guard erstellen, der den Zugriff nur Benutzern mit einer bestimmten Rolle erlaubt. Wir beginnen mit einer einfachen Guard-Vorlage und bauen darauf in den folgenden Abschnitten auf. Für den Moment erlaubt er allen Anfragen, fortzufahren:

```typescript
// roles.guard.ts

import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Observable } from 'rxjs';

@Injectable()
export class RolesGuard implements CanActivate {
  canActivate(
    context: ExecutionContext,
  ): boolean | Promise<boolean> | Observable<boolean> {
    return true;
  }
}
```

### Guards binden / Binding guards

Wie Pipes und Ausnahmefilter können Guards controller-gebunden, methoden-gebunden oder global-gebunden sein. Im Folgenden richten wir einen controller-gebundenen Guard ein, indem wir den `@UseGuards()` Dekorator verwenden. Dieser Dekorator kann ein einzelnes Argument oder eine durch Kommas getrennte Liste von Argumenten annehmen. Dies ermöglicht es, die geeignete Menge an Guards mit einer Deklaration einfach anzuwenden.

```typescript
@Controller('cats')
@UseGuards(RolesGuard)
export class CatsController {}
```

### HINWEIS
Der `@UseGuards()` Dekorator wird aus dem `@nestjs/common` Paket importiert.

Oben haben wir die `RolesGuard` Klasse (statt einer Instanz) übergeben, wodurch die Verantwortung für die Instanziierung dem Framework überlassen wird und die Abhängigkeitsinjektion ermöglicht wird. Wie bei Pipes und Ausnahmefiltern können wir auch eine Instanz vor Ort übergeben:

```typescript
@Controller('cats')
@UseGuards(new RolesGuard())
export class CatsController {}
```

Die obige Konstruktion befestigt den Guard an jeden Handler, der von diesem Controller deklariert wird. Wenn wir möchten, dass der Guard nur für eine einzelne Methode gilt, wenden wir den `@UseGuards()` Dekorator auf Methodenebene an.

Um einen globalen Guard einzurichten, verwenden Sie die `useGlobalGuards()` Methode der Nest-Anwendungsinstanz:

```typescript
const app = await NestFactory.create(AppModule);
app.useGlobalGuards(new RolesGuard());
```

### HINWEIS
Im Fall von hybriden Apps richtet die `useGlobalGuards()` Methode standardmäßig keine Guards für Gateways und Microservices ein (siehe [Hybridanwendung](https://docs.nestjs.com) für Informationen, wie dieses Verhalten geändert werden kann). Für "standard" (nicht-hybride) Microservice-Apps montiert `useGlobalGuards()` die Guards global.

Globale Guards werden über die gesamte Anwendung hinweg verwendet, für jeden Controller und jeden Routen-Handler. In Bezug auf die Abhängigkeitsinjektion können globale Guards, die von außerhalb eines Moduls registriert wurden (mit `useGlobalGuards()` wie im obigen Beispiel), keine Abhängigkeiten injizieren, da dies außerhalb des Kontexts eines Moduls geschieht. Um dieses Problem zu lösen, können Sie einen Guard direkt aus einem Modul heraus einrichten, indem Sie die folgende Konstruktion verwenden:

```typescript
// app.module.ts

import { Module } from '@nestjs/common';
import { APP_GUARD } from '@nestjs/core';

@Module({
  providers: [
    {
      provide: APP_GUARD,
      useClass: RolesGuard,
    },
  ],
})
export class AppModule {}
```

### HINWEIS
Wenn Sie diesen Ansatz verwenden, um Abhängigkeitsinjektion für den Guard durchzuführen, beachten Sie, dass der Guard, unabhängig von dem Modul, in dem diese Konstruktion verwendet wird, tatsächlich global ist. Wo sollte dies getan werden? Wählen Sie das Modul, in dem der Guard (in diesem Beispiel `RolesGuard`) definiert ist. Auch `useClass` ist nicht der einzige Weg, um benutzerdefinierte Provider-Registrierungen zu behandeln. Erfahren Sie mehr [hier](https://docs.nestjs.com).

### Rollen pro Handler setzen / Setting roles per handler

Unser `RolesGuard` funktioniert, ist aber noch nicht sehr smart. Wir nutzen noch nicht die wichtigste Funktion eines Guards - den Ausführungskontext. Er weiß noch nichts über Rollen oder welche Rollen für jeden Handler erlaubt sind. Der `CatsController` könnte zum Beispiel unterschiedliche Berechtigungsschemata für verschiedene Routen haben. Einige könnten nur für einen Admin-Benutzer verfügbar sein, andere könnten für alle offen sein. Wie können wir Rollen flexibel und wiederverwendbar auf Routen abstimmen?

Hier kommt benutzerdefinierte Metadaten ins Spiel (mehr dazu [hier](https://docs.nestjs.com)). Nest bietet die Möglichkeit, benutzerdefinierte Metadaten an Routen-Handler anzuhängen, entweder durch Dekoratoren, die über die statische Methode `Reflector#createDecorator` erstellt wurden, oder durch den eingebauten `@SetMetadata()` Dekorator.

Zum Beispiel erstellen wir einen `@Roles()` Dekorator, der die Metadaten an den Handler anhängt:

```typescript
// roles.decorator.ts

import { Reflector } from '@nestjs/core';

export const Roles = Reflector.createDecorator<string[]>();
```

Der `Roles` Dekorator hier ist eine Funktion, die ein einziges Argument vom Typ `string[]` annimmt.

Nun, um diesen Dekorator zu verwenden, annot

ieren wir einfach den Handler damit:

```typescript
// cats.controller.ts

@Post()
@Roles(['admin'])
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
```

Hier haben wir die `Roles` Dekorator-Metadaten an die `create()` Methode angehängt, was darauf hinweist, dass nur Benutzer mit der Rolle `admin` auf diese Route zugreifen dürfen.

Alternativ könnten wir statt der Methode `Reflector#createDecorator` den eingebauten `@SetMetadata()` Dekorator verwenden. Erfahren Sie mehr dazu [hier](https://docs.nestjs.com).

### Alles zusammenfügen / Putting it all together

Lassen Sie uns nun zurückgehen und dies mit unserem `RolesGuard` verbinden. Derzeit gibt er in allen Fällen einfach `true` zurück und erlaubt jeder Anfrage fortzufahren. Wir möchten den Rückgabewert bedingt basierend auf dem Vergleich der Rollen, die dem aktuellen Benutzer zugewiesen sind, mit den tatsächlichen Rollen machen, die für die aktuelle Route erforderlich sind. Um auf die Rollen der Route (benutzerdefinierte Metadaten) zuzugreifen, verwenden wir erneut die `Reflector` Hilfsklasse, wie folgt:

```typescript
// roles.guard.ts

import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { Roles } from './roles.decorator';

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const roles = this.reflector.get<string[]>('roles', context.getHandler());
    if (!roles) {
      return true;
    }
    const request = context.switchToHttp().getRequest();
    const user = request.user;
    return matchRoles(roles, user.roles);
  }
}
```

### HINWEIS
In der Node.js-Welt ist es gängige Praxis, den autorisierten Benutzer an das Request-Objekt anzuhängen. In unserem Beispielcode oben gehen wir davon aus, dass `request.user` die Benutzerinstanz und die erlaubten Rollen enthält. In Ihrer App werden Sie diese Zuordnung wahrscheinlich in Ihrem benutzerdefinierten Authentifizierungs-Guard (oder Middleware) vornehmen. Weitere Informationen zu diesem Thema finden Sie in diesem Kapitel.

### WARNUNG
Die Logik innerhalb der `matchRoles()` Funktion kann so einfach oder ausgefeilt sein, wie nötig. Der Hauptpunkt dieses Beispiels ist zu zeigen, wie Guards in den Anfrage-/Antwort-Zyklus passen. Weitere Details zur Nutzung von `Reflector` in einem kontextsensitiven Ansatz finden Sie im Abschnitt [Reflexion und Metadaten](https://docs.nestjs.com).

Wenn ein Benutzer mit unzureichenden Berechtigungen eine Endpunktanfrage stellt, gibt Nest automatisch die folgende Antwort zurück:

```json
{
  "statusCode": 403,
  "message": "Forbidden resource",
  "error": "Forbidden"
}
```

Beachten Sie, dass im Hintergrund, wenn ein Guard `false` zurückgibt, das Framework eine `ForbiddenException` wirft. Wenn Sie eine andere Fehlerantwort zurückgeben möchten, sollten Sie Ihre eigene spezifische Ausnahme werfen. Zum Beispiel:

```typescript
throw new UnauthorizedException();
```

Jede von einem Guard geworfene Ausnahme wird von der Ausnahmeschicht (globaler Ausnahmefilter und alle Ausnahmefilter, die auf den aktuellen Kontext angewendet werden) behandelt.

### HINWEIS
Wenn Sie nach einem realen Beispiel suchen, wie Sie die Autorisierung implementieren, schauen Sie sich dieses Kapitel an.
