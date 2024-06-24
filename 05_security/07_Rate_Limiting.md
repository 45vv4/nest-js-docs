### Ratenbegrenzung / Rate Limiting

Eine häufige Technik, um Anwendungen vor Brute-Force-Angriffen zu schützen, ist die Ratenbegrenzung. Um zu beginnen, müssen Sie das Paket @nestjs/throttler installieren.

```bash
$ npm i --save @nestjs/throttler
```

Sobald die Installation abgeschlossen ist, kann das ThrottlerModule wie jedes andere Nest-Paket mit den Methoden forRoot oder forRootAsync konfiguriert werden.

```typescript
// app.module.ts

@Module({
  imports: [
    ThrottlerModule.forRoot([{
      ttl: 60000,
      limit: 10,
    }]),
  ],
})
export class AppModule {}
```

Das obige Beispiel legt die globalen Optionen für die ttl, die Lebensdauer in Millisekunden, und das Limit, die maximale Anzahl von Anfragen innerhalb der ttl, für die geschützten Routen Ihrer Anwendung fest.

Sobald das Modul importiert wurde, können Sie auswählen, wie Sie den ThrottlerGuard binden möchten. Jede Art von Bindung, wie im Abschnitt Guards erwähnt, ist in Ordnung. Wenn Sie den Guard beispielsweise global binden möchten, können Sie diesen Provider zu einem beliebigen Modul hinzufügen:

```typescript
{
  provide: APP_GUARD,
  useClass: ThrottlerGuard
}
```

#### Mehrfache Throttler-Definitionen / Multiple Throttler Definitions#

Es kann vorkommen, dass Sie mehrere Throttling-Definitionen einrichten möchten, wie z.B. nicht mehr als 3 Anrufe in einer Sekunde, 20 Anrufe in 10 Sekunden und 100 Anrufe in einer Minute. Dazu können Sie Ihre Definitionen im Array mit benannten Optionen einrichten, die später in den Dekoratoren @SkipThrottle() und @Throttle() referenziert werden können, um die Optionen erneut zu ändern.

```typescript
// app.module.ts

@Module({
  imports: [
    ThrottlerModule.forRoot([
      {
        name: 'short',
        ttl: 1000,
        limit: 3,
      },
      {
        name: 'medium',
        ttl: 10000,
        limit: 20
      },
      {
        name: 'long',
        ttl: 60000,
        limit: 100
      }
    ]),
  ],
})
export class AppModule {}
```

#### Anpassung / Customization#

Es kann Zeiten geben, in denen Sie den Guard an einen Controller oder global binden möchten, aber die Ratenbegrenzung für ein oder mehrere Ihrer Endpunkte deaktivieren möchten. Dafür können Sie den @SkipThrottle() Dekorator verwenden, um den Throttler für eine ganze Klasse oder eine einzelne Route zu negieren. Der @SkipThrottle() Dekorator kann auch ein Objekt mit String-Schlüsseln und booleschen Werten annehmen, falls Sie die meisten einer Klasse ausschließen möchten, aber nicht jede Route, und es pro Throttler-Set konfigurieren möchten, falls Sie mehr als eines haben. Wenn Sie kein Objekt übergeben, wird standardmäßig { default: true } verwendet.

```typescript
@SkipThrottle()
@Controller('users')
export class UsersController {}
```

Dieser @SkipThrottle() Dekorator kann verwendet werden, um eine Route oder eine Klasse zu überspringen oder das Überspringen einer Route in einer Klasse, die übersprungen wird, zu negieren.

```typescript
@SkipThrottle()
@Controller('users')
export class UsersController {
  // Ratenbegrenzung wird auf diese Route angewendet.
  @SkipThrottle({ default: false })
  dontSkip() {
    return 'List users work with Rate limiting.';
  }
  // Diese Route wird die Ratenbegrenzung überspringen.
  doSkip() {
    return 'List users work without Rate limiting.';
  }
}
```

Es gibt auch den @Throttle() Dekorator, der verwendet werden kann, um das in dem globalen Modul festgelegte Limit und die ttl zu überschreiben, um strengere oder lockerere Sicherheitsoptionen zu bieten. Dieser Dekorator kann ebenfalls auf eine Klasse oder eine Funktion angewendet werden. Ab Version 5 und höher nimmt der Dekorator ein Objekt mit dem String in Bezug auf den Namen des Throttler-Sets und ein Objekt mit den Schlüsseln limit und ttl sowie Ganzzahlwerten, ähnlich den Optionen, die an das Root-Modul übergeben werden. Wenn Sie in Ihren ursprünglichen Optionen keinen Namen festgelegt haben, verwenden Sie den String default. Sie müssen es so konfigurieren:

```typescript
// Override default configuration for Rate limiting and duration.
@Throttle({ default: { limit: 3, ttl: 60000 } })
@Get()
findAll() {
  return "List users works with custom rate limiting.";
}
```

#### Proxies#

Wenn Ihre Anwendung hinter einem Proxy-Server läuft, überprüfen Sie die spezifischen HTTP-Adapter-Optionen (express und fastify) für die trust proxy Option und aktivieren Sie diese. Dadurch können Sie die ursprüngliche IP-Adresse aus dem X-Forwarded-For-Header abrufen und die getTracker() Methode überschreiben, um den Wert aus dem Header statt aus req.ip zu holen. Das folgende Beispiel funktioniert sowohl mit express als auch fastify:

```typescript
// throttler-behind-proxy.guard.ts
import { ThrottlerGuard } from '@nestjs/throttler';
import { Injectable } from '@nestjs/common';

@Injectable()
export class ThrottlerBehindProxyGuard extends ThrottlerGuard {
  protected async getTracker(req: Record<string, any>): Promise<string> {
    return req.ips.length ? req.ips[0] : req.ip; // individuelle IP-Extraktion, um Ihren eigenen Anforderungen zu entsprechen
  }
}

// app.controller.ts
import { ThrottlerBehindProxyGuard } from './throttler-behind-proxy.guard';

@UseGuards(ThrottlerBehindProxyGuard)
```

**HINWEIS**  
Sie können die API des req Request-Objekts für express [hier](https://expressjs.com/en/api.html#req.ips) und für fastify [hier](https://fastify.dev/docs/latest/Reference/Request/) finden.

#### Websockets#

Dieses Modul kann mit Websockets arbeiten, erfordert jedoch eine Klassenerweiterung. Sie können den ThrottlerGuard erweitern und die handleRequest Methode wie folgt überschreiben:

```typescript
@Injectable()
export class WsThrottlerGuard extends ThrottlerGuard {
  async handleRequest(context: ExecutionContext, limit: number, ttl: number, throttler: ThrottlerOptions): Promise<boolean> {
    const client = context.switchToWs().getClient();
    const ip = client._socket.remoteAddress;
    const key = this.generateKey(context, ip, throttler.name);
    const { totalHits } = await this.storageService.increment(key, ttl);

    if (totalHits > limit) {
      throw new ThrottlerException();
    }

    return true;
  }
}
```

**HINWEIS**  
Wenn Sie ws verwenden, ist es notwendig, _socket durch conn zu ersetzen.

Einige Dinge sind zu beachten, wenn Sie mit WebSockets arbeiten:

- Der Guard kann nicht mit dem APP_GUARD oder app.useGlobalGuards() registriert werden.
- Wenn ein Limit erreicht ist, wird Nest ein exception Event auslösen, stellen Sie also sicher, dass ein Listener dafür bereitsteht.

**HINWEIS**  
Wenn Sie das @nestjs/platform-ws Paket verwenden, können Sie client._socket.remoteAddress verwenden.

#### GraphQL#

Der ThrottlerGuard kann auch verwendet werden, um mit GraphQL-Anfragen zu arbeiten. Erweitern Sie den Guard erneut, aber dieses Mal wird die getRequestResponse Methode überschrieben:

```typescript
@Injectable()
export class GqlThrottlerGuard extends ThrottlerGuard {
  getRequestResponse(context: ExecutionContext) {
    const gqlCtx = GqlExecutionContext.create(context);
    const ctx = gqlCtx.getContext();
    return { req: ctx.req, res: ctx.res };
  }
}
```

#### Konfiguration / Configuration#

Die folgenden Optionen sind gültig für das Objekt, das an das Array der ThrottlerModule-Optionen übergeben wird:

- `name`: der Name zur internen Verfolgung, welcher Throttler-Satz verwendet wird. Standardmäßig `default`, wenn nicht übergeben.
- `ttl`: die Anzahl der Millisekunden, die jede Anfrage im Speicher verbleibt.
- `limit`: die maximale Anzahl von Anfragen innerhalb der TTL-Grenze.
- `ignoreUserAgents`: ein Array von regulären Ausdrücken von User-Agents, die beim Drosseln von Anfragen ignoriert werden sollen.
- `skipIf`: eine Funktion, die den ExecutionContext übernimmt und einen booleschen Wert zurückgibt, um die Throttler-Logik zu umgehen. Ähnlich wie @SkipThrottler(), aber basierend auf der Anfrage.

Wenn Sie den Speicher stattdessen einrichten müssen oder einige der oben genannten Optionen globaler verwenden möchten, die für jeden Throttler-Satz gelten, können Sie die Optionen oben über den Schlüssel `throttlers` übergeben und die folgende Tabelle verwenden:

- `storage`: ein benutzerdefinierter Speicherdienst, in dem das Throttling verfolgt werden soll. Siehe [hier](https://docs.nestjs.com/security/rate-limiting#storages).
- `ignoreUserAgents`: ein Array von regulären Ausdrücken von User-Agents, die beim Drosseln von Anfragen ignoriert werden sollen.
- `skipIf`: eine Funktion, die den ExecutionContext übernimmt und einen booleschen Wert zurückgibt, um die Throttler-Logik zu umgehen. Ähnlich wie @SkipThrottler(), aber basierend auf der Anfrage.
-

 `throttlers`: ein Array von Throttler-Sätzen, definiert mit der obigen Tabelle.

#### Asynchrone Konfiguration / Async Configuration#

Sie möchten Ihre Ratenbegrenzungskonfiguration möglicherweise asynchron anstelle von synchron abrufen. Sie können die forRootAsync() Methode verwenden, die Dependency Injection und asynchrone Methoden ermöglicht.

Ein Ansatz wäre, eine Factory-Funktion zu verwenden:

```typescript
@Module({
  imports: [
    ThrottlerModule.forRootAsync({
      imports: [ConfigModule],
      inject: [ConfigService],
      useFactory: (config: ConfigService) => [
        {
          ttl: config.get('THROTTLE_TTL'),
          limit: config.get('THROTTLE_LIMIT'),
        },
      ],
    }),
  ],
})
export class AppModule {}
```

Sie können auch die useClass-Syntax verwenden:

```typescript
@Module({
  imports: [
    ThrottlerModule.forRootAsync({
      imports: [ConfigModule],
      useClass: ThrottlerConfigService,
    }),
  ],
})
export class AppModule {}
```

Dies ist machbar, solange ThrottlerConfigService die Schnittstelle ThrottlerOptionsFactory implementiert.

#### Speicher / Storages#

Der eingebaute Speicher ist ein In-Memory-Cache, der die getätigten Anfragen bis zum Ablauf der TTL verfolgt, die durch die globalen Optionen festgelegt wird. Sie können Ihre eigene Speicheroption in die storage-Option des ThrottlerModule einfügen, solange die Klasse die ThrottlerStorage-Schnittstelle implementiert.

Für verteilte Server könnten Sie den Community-Storage-Provider für Redis verwenden, um eine einzige Quelle der Wahrheit zu haben.

**HINWEIS**  
ThrottlerStorage kann von @nestjs/throttler importiert werden.

#### Zeit-Helfer / Time Helpers#

Es gibt ein paar Hilfsmethoden, um die Timings lesbarer zu machen, wenn Sie diese bevorzugen. @nestjs/throttler exportiert fünf verschiedene Helfer: seconds, minutes, hours, days und weeks. Um sie zu verwenden, rufen Sie einfach seconds(5) oder einen der anderen Helfer auf, und die korrekte Anzahl von Millisekunden wird zurückgegeben.

#### Migrationsanleitung / Migration Guide#

Für die meisten Benutzer wird es ausreichen, Ihre Optionen in ein Array zu packen.

Wenn Sie einen benutzerdefinierten Speicher verwenden, sollten Sie Ihre ttl und limit in ein Array packen und der throttlers-Eigenschaft des Optionsobjekts zuweisen.

Jede @ThrottleSkip() sollte jetzt ein Objekt mit String-Booleschen Props übernehmen. Die Strings sind die Namen der Throttler. Wenn Sie keinen Namen haben, übergeben Sie den String 'default', da dies ansonsten unter der Haube verwendet wird.

Jede @Throttle() Dekoratoren sollten jetzt auch ein Objekt mit String-Schlüsseln übernehmen, die sich auf die Namen der Throttler-Kontexte beziehen (wiederum 'default', wenn kein Name) und Werte von Objekten, die limit und ttl-Schlüssel haben.

**WICHTIG**  
Die ttl ist jetzt in Millisekunden. Wenn Sie Ihre ttl in Sekunden zur Lesbarkeit beibehalten möchten, verwenden Sie den seconds-Helfer aus diesem Paket. Es multipliziert die ttl einfach mit 1000, um sie in Millisekunden umzuwandeln.

Weitere Informationen finden Sie im Changelog.

