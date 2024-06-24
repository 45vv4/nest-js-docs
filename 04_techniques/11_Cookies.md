# Cookies / Cookies

Ein HTTP-Cookie ist ein kleines Datenstück, das vom Browser des Benutzers gespeichert wird. Cookies wurden entwickelt, um eine zuverlässige Mechanismus für Websites zu bieten, zustandsbehaftete Informationen zu speichern. Wenn der Benutzer die Website erneut besucht, wird das Cookie automatisch mit der Anfrage gesendet.

## Verwendung mit Express (Standard) / Use with Express (default)#

Zuerst das erforderliche Paket (und seine Typen für TypeScript-Benutzer) installieren:

```
$ npm i cookie-parser
$ npm i -D @types/cookie-parser
```

Sobald die Installation abgeschlossen ist, wenden Sie die cookie-parser Middleware als globale Middleware an (zum Beispiel in Ihrer main.ts Datei).

```typescript
import * as cookieParser from 'cookie-parser';
// irgendwo in Ihrer Initialisierungsdatei
app.use(cookieParser());
```

Sie können mehrere Optionen an die cookieParser Middleware übergeben:

- `secret`: Ein String oder Array, das zum Signieren von Cookies verwendet wird. Dies ist optional und wenn nicht angegeben, werden keine signierten Cookies analysiert. Wenn ein String angegeben wird, wird dieser als Geheimnis verwendet. Wenn ein Array angegeben wird, wird versucht, das Cookie mit jedem Geheimnis der Reihe nach zu entschlüsseln.
- `options`: Ein Objekt, das als zweite Option an `cookie.parse` übergeben wird. Weitere Informationen finden Sie unter `cookie`.

Die Middleware analysiert den Cookie-Header in der Anfrage und stellt die Cookie-Daten als Eigenschaft `req.cookies` und, wenn ein Geheimnis angegeben wurde, als Eigenschaft `req.signedCookies` zur Verfügung. Diese Eigenschaften sind Name-Wert-Paare des Cookie-Namens zum Cookie-Wert.

Wenn ein Geheimnis angegeben wird, wird dieses Modul alle signierten Cookie-Werte entschlüsseln und validieren und diese Name-Wert-Paare von `req.cookies` in `req.signedCookies` verschieben. Ein signiertes Cookie ist ein Cookie, dessen Wert mit `s:` beginnt. Signierte Cookies, die die Signaturvalidierung nicht bestehen, haben den Wert `false` anstelle des manipulierten Werts.

Mit diesem Mechanismus können Sie jetzt Cookies innerhalb der Routen-Handler wie folgt lesen:

```typescript
@Get()
findAll(@Req() request: Request) {
  console.log(request.cookies); // oder "request.cookies['cookieKey']"
  // oder console.log(request.signedCookies);
}
```

**HINWEIS**  
Der @Req() Dekorator wird aus @nestjs/common importiert, während `Request` aus dem express Paket stammt.

Um ein Cookie an eine ausgehende Antwort anzuhängen, verwenden Sie die Methode `Response#cookie()`:

```typescript
@Get()
findAll(@Res({ passthrough: true }) response: Response) {
  response.cookie('key', 'value')
}
```

**WARNUNG**  
Wenn Sie die Antwortverarbeitung der Logik dem Framework überlassen möchten, denken Sie daran, die `passthrough` Option auf `true` zu setzen, wie oben gezeigt. Lesen Sie [hier](https://docs.nestjs.com/controllers#library-specific-approach) mehr darüber.

**HINWEIS**  
Der @Res() Dekorator wird aus @nestjs/common importiert, während `Response` aus dem express Paket stammt.

## Verwendung mit Fastify / Use with Fastify#

Zuerst das erforderliche Paket installieren:

```
$ npm i @fastify/cookie
```

Sobald die Installation abgeschlossen ist, registrieren Sie das @fastify/cookie Plugin:

```typescript
import fastifyCookie from '@fastify/cookie';

// irgendwo in Ihrer Initialisierungsdatei
const app = await NestFactory.create<NestFastifyApplication>(AppModule, new FastifyAdapter());
await app.register(fastifyCookie, {
  secret: 'my-secret', // zum Signieren von Cookies
});
```

Mit diesem Mechanismus können Sie jetzt Cookies innerhalb der Routen-Handler wie folgt lesen:

```typescript
@Get()
findAll(@Req() request: FastifyRequest) {
  console.log(request.cookies); // oder "request.cookies['cookieKey']"
}
```

**HINWEIS**  
Der @Req() Dekorator wird aus @nestjs/common importiert, während `FastifyRequest` aus dem fastify Paket stammt.

Um ein Cookie an eine ausgehende Antwort anzuhängen, verwenden Sie die Methode `FastifyReply#setCookie()`:

```typescript
@Get()
findAll(@Res({ passthrough: true }) response: FastifyReply) {
  response.setCookie('key', 'value')
}
```

Um mehr über die Methode `FastifyReply#setCookie()` zu erfahren, besuchen Sie diese Seite.

**WARNUNG**  
Wenn Sie die Antwortverarbeitung der Logik dem Framework überlassen möchten, denken Sie daran, die `passthrough` Option auf `true` zu setzen, wie oben gezeigt. Lesen Sie [hier](https://docs.nestjs.com/controllers#library-specific-approach) mehr darüber.

**HINWEIS**  
Der @Res() Dekorator wird aus @nestjs/common importiert, während `FastifyReply` aus dem fastify Paket stammt.

## Erstellen eines benutzerdefinierten Dekorators (plattformübergreifend) / Creating a custom decorator (cross-platform)#

Um eine bequeme, deklarative Methode zum Zugriff auf eingehende Cookies bereitzustellen, können wir einen benutzerdefinierten Dekorator erstellen.

```typescript
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const Cookies = createParamDecorator((data: string, ctx: ExecutionContext) => {
  const request = ctx.switchToHttp().getRequest();
  return data ? request.cookies?.[data] : request.cookies;
});
```

Der @Cookies() Dekorator extrahiert alle Cookies oder ein benanntes Cookie aus dem `req.cookies` Objekt und belegt den dekorierten Parameter mit diesem Wert.

Mit diesem Mechanismus können wir jetzt den Dekorator in einer Routen-Handler-Signatur wie folgt verwenden:

```typescript
@Get()
findAll(@Cookies('name') name: string) {}
```
