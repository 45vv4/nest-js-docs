### Helm / Helmet

Helmet kann helfen, Ihre App vor einigen bekannten Web-Sicherheitslücken zu schützen, indem es HTTP-Header entsprechend setzt. Im Allgemeinen ist Helmet nur eine Sammlung kleinerer Middleware-Funktionen, die sicherheitsrelevante HTTP-Header setzen (mehr erfahren).

**HINWEIS**
Beachten Sie, dass das Anwenden von Helmet als globales Middleware oder das Registrieren davon vor anderen Aufrufen von `app.use()` oder Setup-Funktionen, die `app.use()` aufrufen könnten, erfolgen muss. Dies liegt an der Funktionsweise der zugrunde liegenden Plattform (d.h. Express oder Fastify), bei der die Reihenfolge, in der Middleware/Routes definiert sind, wichtig ist. Wenn Sie Middleware wie Helmet oder Cors nach der Definition einer Route verwenden, wird diese Middleware nicht auf diese Route angewendet, sondern nur auf Routen, die nach der Middleware definiert sind.

#### Verwendung mit Express (Standard) / Use with Express (default)#

Beginnen Sie mit der Installation des erforderlichen Pakets.

```bash
$ npm i --save helmet
```

Sobald die Installation abgeschlossen ist, wenden Sie es als globale Middleware an.

```typescript
import helmet from 'helmet';
// irgendwo in Ihrer Initialisierungsdatei
app.use(helmet());
```

**WARNUNG**
Bei der Verwendung von Helmet, @apollo/server (4.x) und dem Apollo Sandbox kann es zu einem Problem mit CSP im Apollo Sandbox kommen. Um dieses Problem zu lösen, konfigurieren Sie das CSP wie unten gezeigt:

```typescript
app.use(helmet({
  crossOriginEmbedderPolicy: false,
  contentSecurityPolicy: {
    directives: {
      imgSrc: [`'self'`, 'data:', 'apollo-server-landing-page.cdn.apollographql.com'],
      scriptSrc: [`'self'`, `https: 'unsafe-inline'`],
      manifestSrc: [`'self'`, 'apollo-server-landing-page.cdn.apollographql.com'],
      frameSrc: [`'self'`, 'sandbox.embed.apollographql.com'],
    },
  },
}));
```

#### Verwendung mit Fastify / Use with Fastify#

Wenn Sie den FastifyAdapter verwenden, installieren Sie das @fastify/helmet Paket:

```bash
$ npm i --save @fastify/helmet
```

`fastify-helmet` sollte nicht als Middleware, sondern als Fastify-Plugin verwendet werden, d.h. durch Verwendung von `app.register()`:

```typescript
import helmet from '@fastify/helmet'
// irgendwo in Ihrer Initialisierungsdatei
await app.register(helmet);
```

**WARNUNG**
Bei der Verwendung von apollo-server-fastify und @fastify/helmet kann es zu einem Problem mit CSP im GraphQL-Playground kommen. Um diesen Konflikt zu lösen, konfigurieren Sie das CSP wie unten gezeigt:

```typescript
await app.register(fastifyHelmet, {
   contentSecurityPolicy: {
     directives: {
       defaultSrc: [`'self'`, 'unpkg.com'],
       styleSrc: [
         `'self'`,
         `'unsafe-inline'`,
         'cdn.jsdelivr.net',
         'fonts.googleapis.com',
         'unpkg.com',
       ],
       fontSrc: [`'self'`, 'fonts.gstatic.com', 'data:'],
       imgSrc: [`'self'`, 'data:', 'cdn.jsdelivr.net'],
       scriptSrc: [
         `'self'`,
         `https: 'unsafe-inline'`,
         `cdn.jsdelivr.net`,
         `'unsafe-eval'`,
       ],
     },
   },
 });
```

Wenn Sie CSP überhaupt nicht verwenden möchten, können Sie dies so konfigurieren:

```typescript
await app.register(fastifyHelmet, {
  contentSecurityPolicy: false,
});
```

