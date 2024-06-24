# Performance (Fastify) / Performance (Fastify)

Nest verwendet standardmäßig das Express-Framework. Wie bereits erwähnt, bietet Nest auch Kompatibilität mit anderen Bibliotheken, wie zum Beispiel Fastify. Nest erreicht diese Framework-Unabhängigkeit, indem es einen Framework-Adapter implementiert, dessen Hauptfunktion darin besteht, Middleware und Handler an die entsprechenden bibliotheksspezifischen Implementierungen zu übergeben.

**HINWEIS**  
Beachten Sie, dass die Zielbibliothek eine ähnliche Verarbeitungspipeline für Anfragen/Antworten bieten muss wie Express, damit ein Framework-Adapter implementiert werden kann.

Fastify ist eine gute alternative Bibliothek für Nest, da es Designprobleme auf ähnliche Weise wie Express löst. Fastify ist jedoch viel schneller als Express und erzielt fast doppelt so gute Benchmark-Ergebnisse. Eine berechtigte Frage ist, warum Nest Express als Standard-HTTP-Anbieter verwendet. Der Grund ist, dass Express weit verbreitet, bekannt und mit einer enormen Menge kompatibler Middleware ausgestattet ist, die Nest-Benutzern sofort zur Verfügung steht.

Da Nest jedoch Framework-Unabhängigkeit bietet, können Sie problemlos zwischen ihnen wechseln. Fastify kann die bessere Wahl sein, wenn Sie großen Wert auf sehr schnelle Leistung legen. Um Fastify zu verwenden, wählen Sie einfach den integrierten FastifyAdapter, wie in diesem Kapitel gezeigt.

## Installation / Installation#

Zuerst müssen wir das erforderliche Paket installieren:

```
$ npm i --save @nestjs/platform-fastify
```

## Adapter / Adapter#

Sobald die Fastify-Plattform installiert ist, können wir den FastifyAdapter verwenden.

main.ts

```typescript
import { NestFactory } from '@nestjs/core';
import {
  FastifyAdapter,
  NestFastifyApplication,
} from '@nestjs/platform-fastify';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create<NestFastifyApplication>(
    AppModule,
    new FastifyAdapter()
  );
  await app.listen(3000);
}
bootstrap();
```

Standardmäßig hört Fastify nur auf der localhost 127.0.0.1-Schnittstelle (mehr lesen). Wenn Sie Verbindungen auf anderen Hosts akzeptieren möchten, sollten Sie '0.0.0.0' im listen()-Aufruf angeben:

```typescript
async function bootstrap() {
  const app = await NestFactory.create<NestFastifyApplication>(
    AppModule,
    new FastifyAdapter(),
  );
  await app.listen(3000, '0.0.0.0');
}
```

## Plattform-spezifische Pakete / Platform specific packages#

Beachten Sie, dass Nest Fastify als HTTP-Anbieter verwendet, wenn Sie den FastifyAdapter verwenden. Dies bedeutet, dass jedes Rezept, das sich auf Express verlässt, möglicherweise nicht mehr funktioniert. Sie sollten stattdessen äquivalente Fastify-Pakete verwenden.

## Redirect-Antwort / Redirect response#

Fastify behandelt Redirect-Antworten etwas anders als Express. Um eine korrekte Weiterleitung mit Fastify durchzuführen, geben Sie sowohl den Statuscode als auch die URL zurück, wie folgt:

```typescript
@Get()
index(@Res() res) {
  res.status(302).redirect('/login');
}
```

## Fastify-Optionen / Fastify options#

Sie können Optionen durch den FastifyAdapter-Konstruktor an den Fastify-Konstruktor übergeben. Zum Beispiel:

```typescript
new FastifyAdapter({ logger: true });
```

## Middleware / Middleware#

Middleware-Funktionen erhalten die rohen req- und res-Objekte anstelle der Fastify-Wrapper. So funktioniert das unter der Haube verwendete middie-Paket und Fastify – besuchen Sie diese Seite für weitere Informationen.

logger.middleware.ts

```typescript
import { Injectable, NestMiddleware } from '@nestjs/common';
import { FastifyRequest, FastifyReply } from 'fastify';

@Injectable()
export class LoggerMiddleware implements NestMiddleware {
  use(req: FastifyRequest['raw'], res: FastifyReply['raw'], next: () => void) {
    console.log('Request...');
    next();
  }
}
```

## Routen-Konfiguration / Route Config#

Sie können die Routen-Konfigurationsfunktion von Fastify mit dem @RouteConfig() Dekorator verwenden.

```typescript
@RouteConfig({ output: 'hello world' })
@Get()
index(@Req() req) {
  return req.routeConfig.output;
}
```

## Routen-Beschränkungen / Route Constraints#

Ab v10.3.0 unterstützt @nestjs/platform-fastify die Routen-Beschränkungsfunktion von Fastify mit dem @RouteConstraints-Dekorator.

```typescript
@RouteConstraints({ version: '1.2.x' })
newFeature() {
  return 'This works only for version >= 1.2.x';
}
```

**HINWEIS**  
@RouteConfig() und @RouteConstraints werden aus @nestjs/platform-fastify importiert.

## Beispiel / Example#

Ein funktionierendes Beispiel finden Sie [hier](https://github.com/nestjs/nest/tree/master/sample/10-fastify).
