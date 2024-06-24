# Zwischenspeicherung / Caching

Zwischenspeicherung ist eine großartige und einfache Technik, die dazu beiträgt, die Leistung Ihrer App zu verbessern. Sie fungiert als temporärer Datenspeicher und ermöglicht einen schnellen Datenzugriff.

## Installation / Installation

Zuerst die erforderlichen Pakete installieren:

```bash
$ npm install @nestjs/cache-manager cache-manager
```

**WARNUNG**

Version 4 von cache-manager verwendet Sekunden für TTL (Time-To-Live). Die aktuelle Version von cache-manager (v5) hat auf Millisekunden umgestellt. NestJS konvertiert den Wert nicht und leitet den von Ihnen bereitgestellten TTL einfach an die Bibliothek weiter. Mit anderen Worten:
- Wenn Sie cache-manager v4 verwenden, geben Sie TTL in Sekunden an.
- Wenn Sie cache-manager v5 verwenden, geben Sie TTL in Millisekunden an.

Die Dokumentation bezieht sich auf Sekunden, da NestJS für Version 4 von cache-manager veröffentlicht wurde.

## In-Memory-Cache / In-memory cache

Nest bietet eine einheitliche API für verschiedene Cache-Speicheranbieter. Der eingebaute ist ein In-Memory-Datenspeicher. Sie können jedoch problemlos auf eine umfassendere Lösung wie Redis umsteigen.

Um die Zwischenspeicherung zu aktivieren, importieren Sie das CacheModule und rufen Sie dessen register()-Methode auf.

```typescript
import { Module } from '@nestjs/common';
import { CacheModule } from '@nestjs/cache-manager';
import { AppController } from './app.controller';

@Module({
  imports: [CacheModule.register()],
  controllers: [AppController],
})
export class AppModule {}
```

## Interaktion mit dem Cache-Speicher / Interacting with the Cache store

Um mit der Cache-Manager-Instanz zu interagieren, injizieren Sie sie in Ihre Klasse mithilfe des CACHE_MANAGER-Tokens, wie folgt:

```typescript
constructor(@Inject(CACHE_MANAGER) private cacheManager: Cache) {}
```

**HINWEIS**

Die Cache-Klasse wird aus dem cache-manager importiert, während das CACHE_MANAGER-Token aus dem @nestjs/cache-manager-Paket stammt.

Die get-Methode der Cache-Instanz (aus dem cache-manager-Paket) wird verwendet, um Elemente aus dem Cache abzurufen. Wenn das Element nicht im Cache vorhanden ist, wird null zurückgegeben.

```typescript
const value = await this.cacheManager.get('key');
```

Um ein Element zum Cache hinzuzufügen, verwenden Sie die set-Methode:

```typescript
await this.cacheManager.set('key', 'value');
```

Die standardmäßige Ablaufzeit des Caches beträgt 5 Sekunden.

Sie können manuell eine TTL (Ablaufzeit in Sekunden) für diesen speziellen Schlüssel angeben, wie folgt:

```typescript
await this.cacheManager.set('key', 'value', 1000);
```

Um das Ablaufen des Caches zu deaktivieren, setzen Sie die TTL-Konfigurationseigenschaft auf 0:

```typescript
await this.cacheManager.set('key', 'value', 0);
```

Um ein Element aus dem Cache zu entfernen, verwenden Sie die del-Methode:

```typescript
await this.cacheManager.del('key');
```

Um den gesamten Cache zu leeren, verwenden Sie die reset-Methode:

```typescript
await this.cacheManager.reset();
```

## Automatisches Zwischenspeichern von Antworten / Auto-caching responses

**WARNUNG**

In GraphQL-Anwendungen werden Interceptoren separat für jeden Feld-Resolver ausgeführt. Daher funktioniert CacheModule (das Interceptoren verwendet, um Antworten zwischenspeichern) nicht ordnungsgemäß.

Um das automatische Zwischenspeichern von Antworten zu aktivieren, binden Sie einfach den CacheInterceptor an den Stellen, an denen Sie Daten zwischenspeichern möchten.

```typescript
@Controller()
@UseInterceptors(CacheInterceptor)
export class AppController {
  @Get()
  findAll(): string[] {
    return [];
  }
}
```

**WARNUNG**

Nur GET-Endpunkte werden zwischengespeichert. Außerdem können HTTP-Server-Routen, die das native Response-Objekt (@Res()) injizieren, den Cache-Interceptor nicht verwenden. Siehe Response-Mapping für weitere Details.

Um die Menge des erforderlichen Boilerplates zu reduzieren, können Sie den CacheInterceptor global für alle Endpunkte binden:

```typescript
import { Module } from '@nestjs/common';
import { CacheModule, CacheInterceptor } from '@nestjs/cache-manager';
import { AppController } from './app.controller';
import { APP_INTERCEPTOR } from '@nestjs/core';

@Module({
  imports: [CacheModule.register()],
  controllers: [AppController],
  providers: [
    {
      provide: APP_INTERCEPTOR,
      useClass: CacheInterceptor,
    },
  ],
})
export class AppModule {}
```

## Zwischenspeicherung anpassen / Customize caching

Alle zwischengespeicherten Daten haben ihre eigene Ablaufzeit (TTL). Um Standardwerte anzupassen, übergeben Sie das options-Objekt an die register()-Methode.

```typescript
CacheModule.register({
  ttl: 5, // Sekunden
  max: 10, // maximale Anzahl von Elementen im Cache
});
```

## Modul global verwenden / Use module globally

Wenn Sie CacheModule in anderen Modulen verwenden möchten, müssen Sie es importieren (wie es bei jedem Nest-Modul üblich ist). Alternativ können Sie es als globales Modul deklarieren, indem Sie die isGlobal-Eigenschaft des options-Objekts auf true setzen, wie unten gezeigt. In diesem Fall müssen Sie CacheModule nicht in anderen Modulen importieren, sobald es im Root-Modul (z. B. AppModule) geladen wurde.

```typescript
CacheModule.register({
  isGlobal: true,
});
```

## Globale Cache-Überschreibungen / Global cache overrides

Während der globale Cache aktiviert ist, werden Cache-Einträge unter einem CacheKey gespeichert, der automatisch basierend auf dem Routenpfad generiert wird. Sie können bestimmte Cache-Einstellungen (@CacheKey() und @CacheTTL()) auf Basis von Methoden überschreiben, um angepasste Caching-Strategien für einzelne Controller-Methoden zu ermöglichen. Dies kann besonders relevant sein, wenn verschiedene Cache-Speicher verwendet werden.

```typescript
@Controller()
export class AppController {
  @CacheKey('custom_key')
  @CacheTTL(20)
  findAll(): string[] {
    return [];
  }
}
```

**HINWEIS**

Die @CacheKey()- und @CacheTTL()-Dekoratoren werden aus dem @nestjs/cache-manager-Paket importiert.

Der @CacheKey()-Dekorator kann mit oder ohne entsprechenden @CacheTTL()-Dekorator verwendet werden und umgekehrt. Man kann wählen, nur den @CacheKey() oder nur den @CacheTTL() zu überschreiben. Einstellungen, die nicht mit einem Dekorator überschrieben werden, verwenden die global registrierten Standardwerte (siehe Zwischenspeicherung anpassen).

## WebSockets und Microservices / WebSockets and Microservices

Sie können den CacheInterceptor auch auf WebSocket-Abonnenten sowie Microservice-Muster anwenden (unabhängig von der verwendeten Transportmethode).

```typescript
@CacheKey('events')
@UseInterceptors(CacheInterceptor)
@SubscribeMessage('events')
handleEvent(client: Client, data: string[]): Observable<string[]> {
  return [];
}
```

Jedoch ist der zusätzliche @CacheKey()-Dekorator erforderlich, um einen Schlüssel anzugeben, der anschließend verwendet wird, um zwischengespeicherte Daten zu speichern und abzurufen. Bitte beachten Sie auch, dass nicht alles zwischengespeichert werden sollte. Aktionen, die Geschäftsoperationen ausführen, anstatt einfach nur Daten abzufragen, sollten niemals zwischengespeichert werden.

Zusätzlich können Sie eine Cache-Ablaufzeit (TTL) mit dem @CacheTTL()-Dekorator angeben, der den globalen Standard-TTL-Wert überschreibt.

```typescript
@CacheTTL(10)
@UseInterceptors(CacheInterceptor)
@SubscribeMessage('events')
handleEvent(client: Client, data: string[]): Observable<string[]> {
  return [];
}
```

**HINWEIS**

Der @CacheTTL()-Dekorator kann mit oder ohne entsprechenden @CacheKey()-Dekorator verwendet werden.

## Nachverfolgung anpassen / Adjust tracking

Standardmäßig verwendet Nest die Anfrage-URL (in einer HTTP-App) oder den Cache-Schlüssel (in WebSocket- und Microservices-Apps, festgelegt durch den @CacheKey()-Dekorator), um Cache-Einträge mit Ihren Endpunkten zu verknüpfen. Manchmal möchten Sie jedoch die Nachverfolgung basierend auf verschiedenen Faktoren einrichten, beispielsweise mithilfe von HTTP-Headern (z. B. Authorization, um Profilendpunkte korrekt zu identifizieren).

Um dies zu erreichen, erstellen Sie eine Unterklasse von CacheInterceptor und überschreiben die trackBy()-Methode.

```typescript
@Injectable()
class HttpCacheInterceptor extends CacheInterceptor {
  trackBy(context: ExecutionContext): string | undefined {
    return 'key';
  }
}
```

## Verschiedene Speicher / Different stores

Dieser Dienst nutzt cache-manager im Hintergrund. Das cache-manager-Paket unterstützt eine Vielzahl nützlicher Speicher, beispielsweise Redis. Eine vollständige Liste der unterstützten Speicher finden Sie [hier](https://github.com/jaredwray/cache-manager#store-engines). Um den Redis-Speicher einzurichten, übergeben Sie einfach das Paket zusammen mit den entsprechenden Optionen an die register()-Methode.

```typescript
import type { RedisClientOptions } from 'redis';
import * as redisStore from 'cache-manager-redis-store';
import { Module } from '@nestjs/common';
import { CacheModule } from '@nestjs/cache-manager';
import { AppController } from './app.controller';

@Module({
  imports: [
    Cache

Module.register<RedisClientOptions>({
      store: redisStore,

      // Spezifische Konfiguration des Speichers:
      host: 'localhost',
      port: 6379,
    }),
  ],
  controllers: [AppController],
})
export class AppModule {}
```

**WARNUNG**

cache-manager-redis-store unterstützt Redis v4 nicht. Damit die ClientOpts-Schnittstelle existiert und korrekt funktioniert, müssen Sie die neueste Redis 3.x.x-Hauptversion installieren. Siehe dieses Problem, um den Fortschritt dieses Upgrades zu verfolgen.

## Asynchrone Konfiguration / Async configuration

Möglicherweise möchten Sie Moduloptionen asynchron übergeben, anstatt sie statisch zur Kompilierzeit zu übergeben. Verwenden Sie in diesem Fall die registerAsync()-Methode, die verschiedene Möglichkeiten zur asynchronen Konfiguration bietet.

Ein Ansatz besteht darin, eine Fabrikfunktion zu verwenden:

```typescript
CacheModule.registerAsync({
  useFactory: () => ({
    ttl: 5,
  }),
});
```

Unsere Fabrik verhält sich wie alle anderen asynchronen Modulfabriken (sie kann asynchron sein und in der Lage sein, Abhängigkeiten über inject zu injizieren).

```typescript
CacheModule.registerAsync({
  imports: [ConfigModule],
  useFactory: async (configService: ConfigService) => ({
    ttl: configService.get('CACHE_TTL'),
  }),
  inject: [ConfigService],
});
```

Alternativ können Sie die useClass-Methode verwenden:

```typescript
CacheModule.registerAsync({
  useClass: CacheConfigService,
});
```

Die obige Konstruktion instanziiert CacheConfigService innerhalb von CacheModule und verwendet es, um das options-Objekt zu erhalten. CacheConfigService muss das CacheOptionsFactory-Interface implementieren, um die Konfigurationsoptionen bereitzustellen:

```typescript
@Injectable()
class CacheConfigService implements CacheOptionsFactory {
  createCacheOptions(): CacheModuleOptions {
    return {
      ttl: 5,
    };
  }
}
```

Wenn Sie einen vorhandenen Konfigurationsanbieter verwenden möchten, der aus einem anderen Modul importiert wurde, verwenden Sie die useExisting-Syntax:

```typescript
CacheModule.registerAsync({
  imports: [ConfigModule],
  useExisting: ConfigService,
});
```

Dies funktioniert genauso wie useClass mit einem entscheidenden Unterschied - CacheModule durchsucht importierte Module, um einen bereits erstellten ConfigService wiederzuverwenden, anstatt einen eigenen zu instanziieren.

**HINWEIS**

CacheModule#register und CacheModule#registerAsync und CacheOptionsFactory haben ein optionales Generikum (Typ-Argument), um spezifische Konfigurationsoptionen für den Speicher einzugrenzen, wodurch es typsicher wird.

## Beispiel / Example

Ein funktionierendes Beispiel ist [hier](https://github.com/nestjs/nest/tree/master/sample/20-cache) verfügbar.
