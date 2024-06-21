## Interceptors

Ein Interceptor ist eine Klasse, die mit dem `@Injectable()` Dekorator annotiert ist und die `NestInterceptor`-Schnittstelle implementiert.

Interceptors haben eine Reihe nützlicher Fähigkeiten, die von der Technik der aspektorientierten Programmierung (AOP) inspiriert sind. Sie ermöglichen es:

- Zusätzliche Logik vor/nach der Ausführung einer Methode zu binden
- Das Ergebnis, das von einer Funktion zurückgegeben wird, zu transformieren
- Die Ausnahme, die von einer Funktion geworfen wird, zu transformieren
- Das Grundverhalten einer Funktion zu erweitern
- Eine Funktion vollständig zu überschreiben, abhängig von spezifischen Bedingungen (z.B. für Caching-Zwecke)

### Grundlagen

Jeder Interceptor implementiert die `intercept()` Methode, die zwei Argumente entgegennimmt. Das erste ist die `ExecutionContext`-Instanz (genau das gleiche Objekt wie bei Guards). `ExecutionContext` erbt von `ArgumentsHost`. Wir haben `ArgumentsHost` bereits im Kapitel über Ausnahmefilter gesehen. Dort haben wir gesehen, dass es eine Wrapper-Klasse um Argumente ist, die an den ursprünglichen Handler übergeben wurden, und verschiedene Argumentarrays basierend auf dem Typ der Anwendung enthält. Mehr dazu im Kapitel über Ausnahmefilter.

### ExecutionContext

Durch die Erweiterung von `ArgumentsHost` fügt `ExecutionContext` auch mehrere neue Hilfsmethoden hinzu, die zusätzliche Details über den aktuellen Ausführungsprozess bereitstellen. Diese Details können hilfreich sein, um allgemeinere Interceptors zu erstellen, die über eine breite Palette von Controllern, Methoden und Ausführungskontexten hinweg arbeiten können. Erfahren Sie mehr über `ExecutionContext` [hier](https://docs.nestjs.com).

### CallHandler

Das zweite Argument ist ein `CallHandler`. Die `CallHandler`-Schnittstelle implementiert die `handle()` Methode, die Sie verwenden können, um die Routen-Handler-Methode zu einem bestimmten Zeitpunkt in Ihrem Interceptor aufzurufen. Wenn Sie die `handle()` Methode in Ihrer Implementierung der `intercept()` Methode nicht aufrufen, wird die Routen-Handler-Methode überhaupt nicht ausgeführt.

Dieser Ansatz bedeutet, dass die `intercept()` Methode den Anfrage-/Antwort-Stream effektiv umschließt. Infolgedessen können Sie benutzerdefinierte Logik sowohl vor als auch nach der Ausführung des endgültigen Routen-Handlers implementieren. Es ist klar, dass Sie Code in Ihrer `intercept()` Methode schreiben können, der vor dem Aufruf von `handle()` ausgeführt wird, aber wie beeinflussen Sie, was danach passiert? Da die `handle()` Methode ein Observable zurückgibt, können wir leistungsstarke RxJS-Operatoren verwenden, um die Antwort weiter zu manipulieren. In der Terminologie der aspektorientierten Programmierung wird der Aufruf des Routen-Handlers (d.h. der Aufruf von `handle()`) als Pointcut bezeichnet, was darauf hinweist, dass es der Punkt ist, an dem unsere zusätzliche Logik eingefügt wird.

Betrachten Sie beispielsweise eine eingehende `POST /cats`-Anfrage. Diese Anfrage ist für den `create()`-Handler im `CatsController` bestimmt. Wenn ein Interceptor, der die `handle()` Methode nicht aufruft, irgendwo entlang des Weges aufgerufen wird, wird die `create()` Methode nicht ausgeführt. Sobald `handle()` aufgerufen wird (und sein Observable zurückgegeben wurde), wird der `create()` Handler ausgelöst. Und sobald der Antwortstream über das Observable empfangen wird, können zusätzliche Operationen am Stream durchgeführt und ein endgültiges Ergebnis an den Anrufer zurückgegeben werden.

### Aspektinterzeption / Aspect interception 

Der erste Anwendungsfall, den wir uns ansehen, ist die Verwendung eines Interceptors, um Benutzerinteraktionen zu protokollieren (z.B. das Speichern von Benutzeraufrufen, das asynchrone Senden von Ereignissen oder das Berechnen eines Zeitstempels). Wir zeigen unten einen einfachen `LoggingInterceptor`:

```typescript
// logging.interceptor.ts

import { Injectable } from '@nestjs/common';
import { Observable } from 'rxjs';
import { tap } from 'rxjs/operators';

@Injectable()
export class LoggingInterceptor {
  intercept(context, next) {
    console.log('Before...');

    const now = Date.now();
    return next
      .handle()
      .pipe(
        tap(() => console.log(`After... ${Date.now() - now}ms`)),
      );
  }
}
```

### HINWEIS
Das `NestInterceptor<T, R>` ist eine generische Schnittstelle, bei der T den Typ eines `Observable<T>` angibt (unterstützt den Antwortstream), und R den Typ des von `Observable<R>` umschlossenen Werts angibt.

### BINDUNG VON INTERCEPTORS / Binding interceptors

Um den Interceptor einzurichten, verwenden wir den `@UseInterceptors()` Dekorator, der aus dem `@nestjs/common` Paket importiert wird. Wie Pipes und Guards können Interceptors controller-gebunden, methoden-gebunden oder global-gebunden sein.

```typescript
// cats.controller.ts

@UseInterceptors(LoggingInterceptor)
export class CatsController {}
```

### HINWEIS
Der `@UseInterceptors()` Dekorator wird aus dem `@nestjs/common` Paket importiert.

Mit der obigen Konstruktion wird jeder Routen-Handler, der im `CatsController` definiert ist, den `LoggingInterceptor` verwenden. Wenn jemand den `GET /cats`-Endpunkt aufruft, sehen Sie die folgende Ausgabe in Ihrer Standardausgabe:

```
Before...
After... 1ms
```

Beachten Sie, dass wir die `LoggingInterceptor` Klasse (statt einer Instanz) übergeben haben, wodurch die Verantwortung für die Instanziierung dem Framework überlassen wird und die Abhängigkeitsinjektion ermöglicht wird. Wie bei Pipes, Guards und Ausnahmefiltern können wir auch eine Instanz vor Ort übergeben:

```typescript
// cats.controller.ts

@UseInterceptors(new LoggingInterceptor())
export class CatsController {}
```

Wie erwähnt, befestigt die obige Konstruktion den Interceptor an jeden Handler, der von diesem Controller deklariert wird. Wenn wir den Gültigkeitsbereich des Interceptors auf eine einzelne Methode beschränken möchten, wenden wir den Dekorator einfach auf Methodenebene an.

Um einen globalen Interceptor einzurichten, verwenden wir die `useGlobalInterceptors()` Methode der Nest-Anwendungsinstanz:

```typescript
const app = await NestFactory.create(AppModule);
app.useGlobalInterceptors(new LoggingInterceptor());
```

Globale Interceptors werden über die gesamte Anwendung hinweg verwendet, für jeden Controller und jeden Routen-Handler. In Bezug auf die Abhängigkeitsinjektion können globale Interceptors, die von außerhalb eines Moduls registriert wurden (mit `useGlobalInterceptors()` wie im obigen Beispiel), keine Abhängigkeiten injizieren, da dies außerhalb des Kontexts eines Moduls geschieht. Um dieses Problem zu lösen, können Sie einen Interceptor direkt aus einem Modul heraus einrichten, indem Sie die folgende Konstruktion verwenden:

```typescript
// app.module.ts

import { Module } from '@nestjs/common';
import { APP_INTERCEPTOR } from '@nestjs/core';

@Module({
  providers: [
    {
      provide: APP_INTERCEPTOR,
      useClass: LoggingInterceptor,
    },
  ],
})
export class AppModule {}
```

### HINWEIS
Wenn Sie diesen Ansatz verwenden, um Abhängigkeitsinjektion für den Interceptor durchzuführen, beachten Sie, dass der Interceptor, unabhängig von dem Modul, in dem diese Konstruktion verwendet wird, tatsächlich global ist. Wo sollte dies getan werden? Wählen Sie das Modul, in dem der Interceptor (in diesem Beispiel `LoggingInterceptor`) definiert ist. Auch `useClass` ist nicht der einzige Weg, um benutzerdefinierte Provider-Registrierungen zu behandeln. Erfahren Sie mehr [hier](https://docs.nestjs.com).

### Antwort-Mapping / Response mapping

Wir wissen bereits, dass `handle()` ein Observable zurückgibt. Der Stream enthält den Wert, der vom Routen-Handler zurückgegeben wird, und daher können wir ihn mit dem `map()` Operator von RxJS leicht ändern.

### WARNUNG
Die Antwort-Mapping-Funktion funktioniert nicht mit der bibliotheksspezifischen Antwortstrategie (die direkte Verwendung des `@Res()` Objekts ist verboten).

Lassen Sie uns den `TransformInterceptor` erstellen, der jede Antwort auf eine triviale Weise modifizieren wird, um den Prozess zu demonstrieren. Er wird den `map()` Operator von RxJS verwenden, um das Antwortobjekt der `data` Eigenschaft eines neu erstellten Objekts zuzuweisen und das neue Objekt an den Client zurückzugeben.

```typescript
// transform.interceptor.ts

import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

export interface Response<T> {
  data: T;
}

@Injectable()
export class TransformInterceptor<T> implements NestInterceptor<T, Response<T>> {
  intercept(context: ExecutionContext, next: CallHandler): Observable<Response<T>> {
    return next.handle().pipe(map(data => ({ data })));
  }
}
```

### HINWEIS
Nest-Interceptors arbeiten sowohl mit synchronen als auch mit asynchronen `intercept()`-Methoden. Sie können die Methode bei Bedarf einfach auf `async` umstellen.

Mit der obigen Konstruktion sieht die Antwort, wenn jemand den `GET /cats`-Endpunkt aufruft, wie folgt aus (vorausgesetzt, der Routen-Handler gibt ein leeres Array `[]` zurück):

```json
{
 

 "data": []
}
```

Interceptors sind von großem Wert, um wiederverwendbare Lösungen für Anforderungen zu erstellen, die in einer gesamten Anwendung auftreten. Stellen Sie sich beispielsweise vor, wir müssten jede Vorkommen eines `null` Werts in eine leere Zeichenkette `''` umwandeln. Wir können dies mit einer einzigen Codezeile tun und den Interceptor global binden, sodass er automatisch von jedem registrierten Handler verwendet wird.

```typescript
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

@Injectable()
export class ExcludeNullInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next
      .handle()
      .pipe(map(value => value === null ? '' : value ));
  }
}
```

### Ausnahme-Mapping / Exception mapping

Ein weiterer interessanter Anwendungsfall ist die Nutzung des `catchError()` Operators von RxJS, um geworfene Ausnahmen zu überschreiben:

```typescript
// errors.interceptor.ts

import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  BadGatewayException,
  CallHandler,
} from '@nestjs/common';
import { Observable, throwError } from 'rxjs';
import { catchError } from 'rxjs/operators';

@Injectable()
export class ErrorsInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next
      .handle()
      .pipe(
        catchError(err => throwError(() => new BadGatewayException())),
      );
  }
}
```

### Stream-Überschreibung / Stream overriding

Es gibt mehrere Gründe, warum wir manchmal das Aufrufen des Handlers vollständig verhindern und stattdessen einen anderen Wert zurückgeben möchten. Ein offensichtliches Beispiel ist die Implementierung eines Caches zur Verbesserung der Antwortzeit. Werfen wir einen Blick auf einen einfachen Cache-Interceptor, der seine Antwort aus einem Cache zurückgibt. In einem realistischen Beispiel würden wir andere Faktoren wie TTL, Cache-Invalidierung, Cache-Größe usw. berücksichtigen, aber das geht über den Rahmen dieser Diskussion hinaus. Hier geben wir ein einfaches Beispiel, das das Hauptkonzept demonstriert.

```typescript
// cache.interceptor.ts

import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable, of } from 'rxjs';

@Injectable()
export class CacheInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const isCached = true;
    if (isCached) {
      return of([]);
    }
    return next.handle();
  }
}
```

Unser `CacheInterceptor` hat eine fest codierte `isCached` Variable und eine fest codierte Antwort `[]`. Der entscheidende Punkt ist, dass wir hier einen neuen Stream zurückgeben, der vom `of()` Operator von RxJS erstellt wird, wodurch der Routen-Handler überhaupt nicht aufgerufen wird. Wenn jemand einen Endpunkt aufruft, der den `CacheInterceptor` verwendet, wird die Antwort (ein fest codiertes, leeres Array) sofort zurückgegeben. Um eine generische Lösung zu erstellen, können Sie den `Reflector` verwenden und einen benutzerdefinierten Dekorator erstellen. Der `Reflector` wird im Guards-Kapitel ausführlich beschrieben.

### Weitere Operatoren / More operators

Die Möglichkeit, den Stream mit RxJS-Operatoren zu manipulieren, bietet uns viele Fähigkeiten. Betrachten wir einen weiteren häufigen Anwendungsfall. Stellen Sie sich vor, Sie möchten Zeitüberschreitungen bei Routenanfragen behandeln. Wenn Ihr Endpunkt nach einer bestimmten Zeitspanne nichts zurückgibt, möchten Sie mit einer Fehlerantwort abbrechen. Die folgende Konstruktion ermöglicht dies:

```typescript
// timeout.interceptor.ts

import { Injectable, NestInterceptor, ExecutionContext, CallHandler, RequestTimeoutException } from '@nestjs/common';
import { Observable, throwError, TimeoutError } from 'rxjs';
import { catchError, timeout } from 'rxjs/operators';

@Injectable()
export class TimeoutInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(
      timeout(5000),
      catchError(err => {
        if (err instanceof TimeoutError) {
          return throwError(() => new RequestTimeoutException());
        }
        return throwError(() => err);
      }),
    );
  };
};
```

Nach 5 Sekunden wird die Anfrageverarbeitung abgebrochen. Sie können auch benutzerdefinierte Logik hinzufügen, bevor Sie `RequestTimeoutException` werfen (z.B. Ressourcen freigeben).
