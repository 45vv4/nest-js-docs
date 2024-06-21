## Benutzerdefinierte Routen-Dekoratoren / Custom route decorators

Nest ist um ein Sprachfeature namens Dekoratoren aufgebaut. Dekoratoren sind ein bekanntes Konzept in vielen gängigen Programmiersprachen, aber in der JavaScript-Welt sind sie noch relativ neu. Um besser zu verstehen, wie Dekoratoren funktionieren, empfehlen wir, diesen Artikel zu lesen. Hier ist eine einfache Definition:

Ein ES2016-Dekorator ist ein Ausdruck, der eine Funktion zurückgibt und ein Ziel, einen Namen und eine Eigenschaftsbeschreibung als Argumente annehmen kann. Sie wenden ihn an, indem Sie den Dekorator mit einem `@`-Zeichen voranstellen und ihn ganz oben an dem platzieren, was Sie dekorieren möchten. Dekoratoren können entweder für eine Klasse, eine Methode oder eine Eigenschaft definiert werden.

### Param-Dekoratoren / Param decorators

Nest bietet eine Reihe nützlicher Param-Dekoratoren, die Sie zusammen mit den HTTP-Routen-Handlern verwenden können. Im Folgenden finden Sie eine Liste der bereitgestellten Dekoratoren und der entsprechenden Plain-Express (oder Fastify)-Objekte, die sie darstellen:

- `@Request()`, `@Req()`: req
- `@Response()`, `@Res()`: res
- `@Next()`: next
- `@Session()`: req.session
- `@Param(param?: string)`: req.params / req.params[param]
- `@Body(param?: string)`: req.body / req.body[param]
- `@Query(param?: string)`: req.query / req.query[param]
- `@Headers(param?: string)`: req.headers / req.headers[param]
- `@Ip()`: req.ip
- `@HostParam()`: req.hosts

Zusätzlich können Sie Ihre eigenen benutzerdefinierten Dekoratoren erstellen. Warum ist das nützlich?

In der Node.js-Welt ist es gängige Praxis, Eigenschaften an das Request-Objekt anzuhängen. Dann extrahieren Sie sie manuell in jedem Routen-Handler, indem Sie Code wie den folgenden verwenden:

```typescript
const user = req.user;
```

Um Ihren Code lesbarer und transparenter zu machen, können Sie einen `@User()` Dekorator erstellen und ihn in allen Ihren Controllern wiederverwenden.

### Benutzerdefinierte Dekoratoren erstellen

```typescript
// user.decorator.ts

import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const User = createParamDecorator(
  (data: unknown, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    return request.user;
  },
);
```

Dann können Sie ihn einfach überall dort verwenden, wo es Ihren Anforderungen entspricht.

```typescript
@Get()
async findOne(@User() user: UserEntity) {
  console.log(user);
}
```

### Daten übergeben / Passing data

Wenn das Verhalten Ihres Dekorators von bestimmten Bedingungen abhängt, können Sie den `data` Parameter verwenden, um ein Argument an die Fabrikfunktion des Dekorators zu übergeben. Ein Anwendungsfall hierfür ist ein benutzerdefinierter Dekorator, der Eigenschaften vom Request-Objekt nach Schlüssel extrahiert. Angenommen, unsere Authentifizierungsschicht validiert Anfragen und fügt eine Benutzer-Entität an das Request-Objekt an. Die Benutzer-Entität für eine authentifizierte Anfrage könnte so aussehen:

```json
{
  "id": 101,
  "firstName": "Alan",
  "lastName": "Turing",
  "email": "alan@email.com",
  "roles": ["admin"]
}
```

Definieren wir einen Dekorator, der einen Eigenschaftsnamen als Schlüssel nimmt und den zugehörigen Wert zurückgibt, wenn er existiert (oder `undefined`, wenn er nicht existiert oder wenn das Benutzerobjekt nicht erstellt wurde).

```typescript
// user.decorator.ts

import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const User = createParamDecorator(
  (data: string, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    const user = request.user;

    return data ? user?.[data] : user;
  },
);
```

So können Sie dann eine bestimmte Eigenschaft über den `@User()` Dekorator im Controller zugreifen:

```typescript
@Get()
async findOne(@User('firstName') firstName: string) {
  console.log(`Hello ${firstName}`);
}
```

Sie können diesen Dekorator mit verschiedenen Schlüsseln verwenden, um auf verschiedene Eigenschaften zuzugreifen. Wenn das Benutzerobjekt tief oder komplex ist, kann dies die Implementierung von Request-Handlern einfacher und lesbarer machen.

### HINWEIS
Für TypeScript-Benutzer: `createParamDecorator<T>()` ist generisch. Das bedeutet, dass Sie die Typsicherheit explizit erzwingen können, z.B. `createParamDecorator<string>((data, ctx) => ...)`. Alternativ können Sie einen Parametertyp in der Fabrikfunktion angeben, z.B. `createParamDecorator((data: string, ctx) => ...)`. Wenn Sie beides weglassen, wird der Typ für `data` `any` sein.

### Arbeiten mit Pipes / working pipes

Nest behandelt benutzerdefinierte Param-Dekoratoren auf die gleiche Weise wie die eingebauten (`@Body()`, `@Param()` und `@Query()`). Das bedeutet, dass Pipes auch für die benutzerdefinierten annotierten Parameter ausgeführt werden (in unseren Beispielen das `user` Argument). Darüber hinaus können Sie die Pipe direkt auf den benutzerdefinierten Dekorator anwenden:

```typescript
@Get()
async findOne(
  @User(new ValidationPipe({ validateCustomDecorators: true }))
  user: UserEntity,
) {
  console.log(user);
}
```

### HINWEIS
Beachten Sie, dass die Option `validateCustomDecorators` auf `true` gesetzt werden muss. `ValidationPipe` validiert standardmäßig keine Argumente, die mit benutzerdefinierten Dekoratoren annotiert sind.

### Dekorator-Komposition / Decorators composition

Nest bietet eine Hilfsmethode, um mehrere Dekoratoren zu kombinieren. Angenommen, Sie möchten alle Dekoratoren im Zusammenhang mit der Authentifizierung in einem einzigen Dekorator zusammenfassen. Dies könnte mit der folgenden Konstruktion erfolgen:

```typescript
// auth.decorator.ts

import { applyDecorators } from '@nestjs/common';

export function Auth(...roles: Role[]) {
  return applyDecorators(
    SetMetadata('roles', roles),
    UseGuards(AuthGuard, RolesGuard),
    ApiBearerAuth(),
    ApiUnauthorizedResponse({ description: 'Unauthorized' }),
  );
}
```

Sie können dann diesen benutzerdefinierten `@Auth()` Dekorator wie folgt verwenden:

```typescript
@Get('users')
@Auth('admin')
findAllUsers() {}
```

Dies hat den Effekt, dass alle vier Dekoratoren mit einer einzigen Deklaration angewendet werden.

### WARNUNG
Der `@ApiHideProperty()` Dekorator aus dem `@nestjs/swagger` Paket ist nicht zusammensetzbar und funktioniert nicht richtig mit der `applyDecorators` Funktion.
