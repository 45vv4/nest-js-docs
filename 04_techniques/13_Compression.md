# Compression / Kompression

Kompression kann die Größe des Antwortkörpers erheblich verringern und dadurch die Geschwindigkeit einer Web-App erhöhen.

Für stark frequentierte Websites in der Produktion wird dringend empfohlen, die Kompression vom Anwendungsserver auszulagern - typischerweise in einem Reverse-Proxy (z.B. Nginx). In diesem Fall sollten Sie keine Kompressions-Middleware verwenden.

## Verwendung mit Express (Standard) / Use with Express (default)#

Verwenden Sie das Kompressions-Middleware-Paket, um gzip-Kompression zu aktivieren.

Zuerst das erforderliche Paket installieren:

```
$ npm i --save compression
```

Sobald die Installation abgeschlossen ist, wenden Sie die Kompressions-Middleware als globale Middleware an.

```typescript
import * as compression from 'compression';
// irgendwo in Ihrer Initialisierungsdatei
app.use(compression());
```

## Verwendung mit Fastify / Use with Fastify#

Wenn Sie den FastifyAdapter verwenden, sollten Sie fastify-compress verwenden:

```
$ npm i --save @fastify/compress
```

Sobald die Installation abgeschlossen ist, wenden Sie die @fastify/compress Middleware als globale Middleware an.

```typescript
import compression from '@fastify/compress';
// irgendwo in Ihrer Initialisierungsdatei
await app.register(compression);
```

Standardmäßig verwendet @fastify/compress die Brotli-Kompression (auf Node >= 11.7.0), wenn Browser die Unterstützung für die Codierung angeben. Während Brotli hinsichtlich des Kompressionsverhältnisses sehr effizient sein kann, kann es auch ziemlich langsam sein. Standardmäßig setzt Brotli eine maximale Kompressionsqualität von 11, obwohl diese durch Anpassung des BROTLI_PARAM_QUALITY zwischen 0 (Minimum) und 11 (Maximum) angepasst werden kann, um die Kompressionszeit zugunsten der Kompressionsqualität zu reduzieren. Ein Beispiel mit Qualität 4:

```typescript
import { constants } from 'zlib';
// irgendwo in Ihrer Initialisierungsdatei
await app.register(compression, { brotliOptions: { params: { [constants.BROTLI_PARAM_QUALITY]: 4 } } });
```

Um es zu vereinfachen, möchten Sie möglicherweise fastify-compress nur deflate und gzip verwenden lassen, um Antworten zu komprimieren; Sie erhalten möglicherweise größere Antworten, aber sie werden viel schneller geliefert.

Um Codierungen anzugeben, geben Sie ein zweites Argument an app.register:

```typescript
await app.register(compression, { encodings: ['gzip', 'deflate'] });
```

Das obige Beispiel weist fastify-compress an, nur gzip- und deflate-Codierungen zu verwenden, wobei gzip bevorzugt wird, wenn der Client beide unterstützt.
