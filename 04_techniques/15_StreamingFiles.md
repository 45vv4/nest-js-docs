# Streaming files / Dateien streamen

**HINWEIS**  
Dieses Kapitel zeigt, wie Sie Dateien aus Ihrer HTTP-Anwendung streamen können. Die unten dargestellten Beispiele gelten nicht für GraphQL- oder Microservice-Anwendungen.

Es kann Zeiten geben, in denen Sie eine Datei von Ihrer REST-API an den Client zurücksenden möchten. Um dies mit Nest zu tun, würden Sie normalerweise Folgendes tun:

```typescript
@Controller('file')
export class FileController {
  @Get()
  getFile(@Res() res: Response) {
    const file = createReadStream(join(process.cwd(), 'package.json'));
    file.pipe(res);
  }
}
```

Dabei verlieren Sie jedoch den Zugriff auf Ihre Post-Controller-Interceptor-Logik. Um dies zu handhaben, können Sie eine StreamableFile-Instanz zurückgeben und im Hintergrund kümmert sich das Framework um das Piping der Antwort.

## StreamableFile-Klasse / Streamable File class#

Eine StreamableFile ist eine Klasse, die den Stream hält, der zurückgegeben werden soll. Um eine neue StreamableFile zu erstellen, können Sie entweder einen Buffer oder einen Stream an den StreamableFile-Konstruktor übergeben.

**HINWEIS**  
Die StreamableFile-Klasse kann aus @nestjs/common importiert werden.

## Plattformübergreifende Unterstützung / Cross-platform support#

Fastify kann standardmäßig Dateien senden, ohne dass stream.pipe(res) aufgerufen werden muss, sodass Sie die StreamableFile-Klasse überhaupt nicht verwenden müssen. Nest unterstützt jedoch die Verwendung von StreamableFile in beiden Plattformtypen, sodass Sie sich keine Sorgen um die Kompatibilität zwischen den beiden Engines machen müssen, wenn Sie zwischen Express und Fastify wechseln.

## Beispiel / Example#

Im Folgenden finden Sie ein einfaches Beispiel, bei dem die package.json als Datei anstelle eines JSON zurückgegeben wird, aber die Idee erstreckt sich natürlich auf Bilder, Dokumente und alle anderen Dateitypen.

```typescript
import { Controller, Get, StreamableFile } from '@nestjs/common';
import { createReadStream } from 'fs';
import { join } from 'path';

@Controller('file')
export class FileController {
  @Get()
  getFile(): StreamableFile {
    const file = createReadStream(join(process.cwd(), 'package.json'));
    return new StreamableFile(file);
  }
}
```

Der Standardinhaltstyp ist application/octet-stream. Wenn Sie die Antwort anpassen müssen, können Sie die Methode res.set oder den @Header() Dekorator verwenden, wie folgt:

```typescript
import { Controller, Get, StreamableFile, Res } from '@nestjs/common';
import { createReadStream } from 'fs';
import { join } from 'path';
import type { Response } from 'express';

@Controller('file')
export class FileController {
  @Get()
  getFile(@Res({ passthrough: true }) res: Response): StreamableFile {
    const file = createReadStream(join(process.cwd(), 'package.json'));
    res.set({
      'Content-Type': 'application/json',
      'Content-Disposition': 'attachment; filename="package.json"',
    });
    return new StreamableFile(file);
  }

  // Oder sogar:
  @Get()
  @Header('Content-Type', 'application/json')
  @Header('Content-Disposition', 'attachment; filename="package.json"')
  getStaticFile(): StreamableFile {
    const file = createReadStream(join(process.cwd(), 'package.json'));
    return new StreamableFile(file);
  }
}
```
