### CSRF-Schutz / CSRF Protection

Cross-Site Request Forgery (auch bekannt als CSRF oder XSRF) ist eine Art von bösartigem Exploit einer Website, bei dem unbefugte Befehle von einem Benutzer übertragen werden, dem die Webanwendung vertraut. Um diese Art von Angriff zu verhindern, können Sie das csurf-Paket verwenden.

#### Verwendung mit Express (Standard) / Use with Express (default)#

Beginnen Sie mit der Installation des erforderlichen Pakets:

```bash
$ npm i --save csurf
```

**WARNUNG**  
Dieses Paket ist veraltet. Weitere Informationen finden Sie in den csurf-Dokumenten.

**WARNUNG**  
Wie in den csurf-Dokumenten erläutert, erfordert diese Middleware, dass entweder die Session-Middleware oder der Cookie-Parser zuerst initialisiert wird. Bitte sehen Sie in der Dokumentation nach weiteren Anweisungen.

Sobald die Installation abgeschlossen ist, wenden Sie die csurf-Middleware als globale Middleware an.

```typescript
import * as csurf from 'csurf';
// ...
// irgendwo in Ihrer Initialisierungsdatei
app.use(csurf());
```

#### Verwendung mit Fastify / Use with Fastify#

Beginnen Sie mit der Installation des erforderlichen Pakets:

```bash
$ npm i --save @fastify/csrf-protection
```

Sobald die Installation abgeschlossen ist, registrieren Sie das @fastify/csrf-protection-Plugin, wie folgt:

```typescript
import fastifyCsrf from '@fastify/csrf-protection';
// ...
// irgendwo in Ihrer Initialisierungsdatei nach der Registrierung eines Speicher-Plugins
await app.register(fastifyCsrf);
```

**WARNUNG**  
Wie in den @fastify/csrf-protection-Dokumenten [hier](https://github.com/fastify/csrf-protection#usage) erklärt, erfordert dieses Plugin, dass zuerst ein Speicher-Plugin initialisiert wird. Bitte sehen Sie in der Dokumentation nach weiteren Anweisungen.
