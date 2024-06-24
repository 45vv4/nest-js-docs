# Model-View-Controller / Model-View-Controller

Nest verwendet standardmäßig die Express-Bibliothek im Hintergrund. Daher gelten alle Techniken zur Verwendung des MVC (Model-View-Controller) Musters in Express auch für Nest.

Zunächst erstellen wir eine einfache Nest-Anwendung mit dem CLI-Tool:

```
$ npm i -g @nestjs/cli
$ nest new project
```

Um eine MVC-App zu erstellen, benötigen wir auch eine Template-Engine, um unsere HTML-Ansichten zu rendern:

```
$ npm install --save hbs
```

Wir haben die hbs (Handlebars) Engine verwendet, Sie können jedoch jede verwenden, die Ihren Anforderungen entspricht. Sobald der Installationsprozess abgeschlossen ist, müssen wir die Express-Instanz mit dem folgenden Code konfigurieren:

main.ts

```typescript
import { NestFactory } from '@nestjs/core';
import { NestExpressApplication } from '@nestjs/platform-express';
import { join } from 'path';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create<NestExpressApplication>(
    AppModule,
  );

  app.useStaticAssets(join(__dirname, '..', 'public'));
  app.setBaseViewsDir(join(__dirname, '..', 'views'));
  app.setViewEngine('hbs');

  await app.listen(3000);
}
bootstrap();
```

Wir haben Express mitgeteilt, dass das Verzeichnis public für das Speichern von statischen Assets verwendet wird, views Templates enthält und die hbs Template-Engine zum Rendern der HTML-Ausgabe verwendet werden soll.

## Template-Rendering / Template rendering#

Erstellen wir nun ein views-Verzeichnis und eine index.hbs Vorlage darin. In der Vorlage werden wir eine Nachricht anzeigen, die vom Controller übergeben wird:

```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8" />
    <title>App</title>
  </head>
  <body>
    {{ message }}
  </body>
</html>
```

Öffnen Sie als nächstes die app.controller Datei und ersetzen Sie die root() Methode durch den folgenden Code:

app.controller.ts

```typescript
import { Get, Controller, Render } from '@nestjs/common';

@Controller()
export class AppController {
  @Get()
  @Render('index')
  root() {
    return { message: 'Hello world!' };
  }
}
```

In diesem Code geben wir das Template an, das im @Render() Dekorator verwendet werden soll, und der Rückgabewert der Routen-Handler-Methode wird zur Vorlage zum Rendern übergeben. Beachten Sie, dass der Rückgabewert ein Objekt mit einer Eigenschaft message ist, die dem von uns im Template erstellten Platzhalter message entspricht.

Während die Anwendung läuft, öffnen Sie Ihren Browser und navigieren Sie zu http://localhost:3000. Sie sollten die Nachricht "Hello world!" sehen.

## Dynamisches Template-Rendering / Dynamic template rendering#

Wenn die Anwendungslogik dynamisch entscheiden muss, welches Template gerendert werden soll, sollten wir den @Res() Dekorator verwenden und den Ansichtsname im Routen-Handler angeben, anstatt im @Render() Dekorator:

**HINWEIS**  
Wenn Nest den @Res() Dekorator erkennt, injiziert es das bibliotheksspezifische Antwortobjekt. Wir können dieses Objekt verwenden, um das Template dynamisch zu rendern. Erfahren Sie mehr über die API des Antwortobjekts [hier](https://docs.nestjs.com/techniques/mvc).

app.controller.ts

```typescript
import { Get, Controller, Res, Render } from '@nestjs/common';
import { Response } from 'express';
import { AppService } from './app.service';

@Controller()
export class AppController {
  constructor(private appService: AppService) {}

  @Get()
  root(@Res() res: Response) {
    return res.render(
      this.appService.getViewName(),
      { message: 'Hello world!' },
    );
  }
}
```

## Beispiel / Example#

Ein funktionierendes Beispiel finden Sie [hier](https://github.com/nestjs/nest/tree/master/sample/15-mvc).

## Fastify / Fastify#

Wie in diesem Kapitel erwähnt, können wir jeden kompatiblen HTTP-Anbieter zusammen mit Nest verwenden. Eine solche Bibliothek ist Fastify. Um eine MVC-Anwendung mit Fastify zu erstellen, müssen wir die folgenden Pakete installieren:

```
$ npm i --save @fastify/static @fastify/view handlebars
```

Die nächsten Schritte umfassen fast denselben Prozess wie bei Express, mit geringfügigen Unterschieden, die spezifisch für die Plattform sind. Sobald der Installationsprozess abgeschlossen ist, öffnen Sie die main.ts Datei und aktualisieren Sie deren Inhalt:

main.ts

```typescript
import { NestFactory } from '@nestjs/core';
import { NestFastifyApplication, FastifyAdapter } from '@nestjs/platform-fastify';
import { AppModule } from './app.module';
import { join } from 'path';

async function bootstrap() {
  const app = await NestFactory.create<NestFastifyApplication>(
    AppModule,
    new FastifyAdapter(),
  );
  app.useStaticAssets({
    root: join(__dirname, '..', 'public'),
    prefix: '/public/',
  });
  app.setViewEngine({
    engine: {
      handlebars: require('handlebars'),
    },
    templates: join(__dirname, '..', 'views'),
  });
  await app.listen(3000);
}
bootstrap();
```

Die Fastify-API ist leicht unterschiedlich, aber das Endergebnis dieser Methodenaufrufe bleibt dasselbe. Ein Unterschied bei Fastify ist, dass der im @Render() Dekorator übergebene Template-Name eine Dateierweiterung enthalten muss.

app.controller.ts

```typescript
import { Get, Controller, Render } from '@nestjs/common';

@Controller()
export class AppController {
  @Get()
  @Render('index.hbs')
  root() {
    return { message: 'Hello world!' };
  }
}
```

Während die Anwendung läuft, öffnen Sie Ihren Browser und navigieren Sie zu http://localhost:3000. Sie sollten die Nachricht "Hello world!" sehen.

## Beispiel / Example#

Ein funktionierendes Beispiel finden Sie [hier](https://github.com/nestjs/nest/tree/master/sample/17-mvc-fastify).

