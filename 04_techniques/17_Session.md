# Session / Session

HTTP-Sitzungen bieten eine Möglichkeit, Informationen über den Benutzer über mehrere Anfragen hinweg zu speichern, was besonders nützlich für MVC-Anwendungen ist.

## Verwendung mit Express (Standard) / Use with Express (default)#

Zuerst das erforderliche Paket (und seine Typen für TypeScript-Benutzer) installieren:

```
$ npm i express-session
$ npm i -D @types/express-session
```

Sobald die Installation abgeschlossen ist, wenden Sie die express-session Middleware als globale Middleware an (zum Beispiel in Ihrer main.ts Datei).

```typescript
import * as session from 'express-session';
// irgendwo in Ihrer Initialisierungsdatei
app.use(
  session({
    secret: 'my-secret',
    resave: false,
    saveUninitialized: false,
  }),
);
```

**HINWEIS**  
Der standardmäßige serverseitige Sitzungsspeicher ist absichtlich nicht für eine Produktionsumgebung ausgelegt. Er wird unter den meisten Bedingungen Speicher lecken, skaliert nicht über einen einzigen Prozess hinaus und ist für Debugging und Entwicklung gedacht. Lesen Sie mehr im offiziellen Repository.

Das Geheimnis wird verwendet, um das Session-ID-Cookie zu signieren. Dies kann entweder ein String für ein einzelnes Geheimnis oder ein Array mit mehreren Geheimnissen sein. Wenn ein Array von Geheimnissen bereitgestellt wird, wird nur das erste Element verwendet, um das Session-ID-Cookie zu signieren, während alle Elemente bei der Überprüfung der Signatur in Anfragen berücksichtigt werden. Das Geheimnis selbst sollte nicht leicht von einem Menschen gelesen werden können und am besten eine zufällige Zeichenfolge sein.

Das Aktivieren der resave-Option erzwingt, dass die Sitzung zurück in den Sitzungsspeicher gespeichert wird, selbst wenn die Sitzung während der Anfrage nie geändert wurde. Der Standardwert ist true, aber die Verwendung des Standards wurde abgelehnt, da sich der Standard in Zukunft ändern wird.

Ebenso erzwingt das Aktivieren der saveUninitialized-Option, dass eine "uninitialisierte" Sitzung im Speicher gespeichert wird. Eine Sitzung ist uninitialisiert, wenn sie neu, aber nicht geändert ist. Das Wählen von false ist nützlich für die Implementierung von Anmelde-Sitzungen, die Reduzierung des Server-Speicherverbrauchs oder die Einhaltung von Gesetzen, die eine Erlaubnis vor dem Setzen eines Cookies erfordern. Das Wählen von false hilft auch bei Rennbedingungen, bei denen ein Client mehrere parallele Anfragen ohne Sitzung stellt (Quelle).

Sie können mehrere andere Optionen an die session-Middleware übergeben, lesen Sie mehr darüber in der API-Dokumentation.

**HINWEIS**  
Bitte beachten Sie, dass secure: true eine empfohlene Option ist. Es erfordert jedoch eine HTTPS-aktivierte Website, d.h., HTTPS ist notwendig für sichere Cookies. Wenn secure gesetzt ist und Sie auf Ihre Seite über HTTP zugreifen, wird das Cookie nicht gesetzt. Wenn Sie Ihren Node.js-Server hinter einem Proxy haben und secure: true verwenden, müssen Sie "trust proxy" in express setzen.

Mit dieser Einrichtung können Sie nun Sitzungswerte innerhalb der Routen-Handler wie folgt setzen und lesen:

```typescript
@Get()
findAll(@Req() request: Request) {
  request.session.visits = request.session.visits ? request.session.visits + 1 : 1;
}
```

**HINWEIS**  
Der @Req() Dekorator wird aus @nestjs/common importiert, während Request aus dem express Paket stammt.

Alternativ können Sie den @Session() Dekorator verwenden, um ein Sitzungsobjekt aus der Anfrage zu extrahieren, wie folgt:

```typescript
@Get()
findAll(@Session() session: Record<string, any>) {
  session.visits = session.visits ? session.visits + 1 : 1;
}
```

**HINWEIS**  
Der @Session() Dekorator wird aus dem @nestjs/common Paket importiert.

## Verwendung mit Fastify / Use with Fastify#

Zuerst das erforderliche Paket installieren:

```
$ npm i @fastify/secure-session
```

Sobald die Installation abgeschlossen ist, registrieren Sie das fastify-secure-session Plugin:

```typescript
import secureSession from '@fastify/secure-session';

// irgendwo in Ihrer Initialisierungsdatei
const app = await NestFactory.create<NestFastifyApplication>(
  AppModule,
  new FastifyAdapter(),
);
await app.register(secureSession, {
  secret: 'averylogphrasebiggerthanthirtytwochars',
  salt: 'mq9hDxBVDbspDR6n',
});
```

**HINWEIS**  
Sie können auch einen Schlüssel vorab generieren (siehe Anweisungen) oder die Schlüsselrotation verwenden. Lesen Sie mehr über die verfügbaren Optionen im offiziellen Repository.

Mit dieser Einrichtung können Sie nun Sitzungswerte innerhalb der Routen-Handler wie folgt setzen und lesen:

```typescript
@Get()
findAll(@Req() request: FastifyRequest) {
  const visits = request.session.get('visits');
  request.session.set('visits', visits ? visits + 1 : 1);
}
```

Alternativ können Sie den @Session() Dekorator verwenden, um ein Sitzungsobjekt aus der Anfrage zu extrahieren, wie folgt:

```typescript
@Get()
findAll(@Session() session: secureSession.Session) {
  const visits = session.get('visits');
  session.set('visits', visits ? visits + 1 : 1);
}
```

**HINWEIS**  
Der @Session() Dekorator wird aus @nestjs/common importiert, während secureSession.Session aus dem @fastify/secure-session Paket stammt (Import-Anweisung: `import * as secureSession from '@fastify/secure-session'`).
