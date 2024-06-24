# HTTP module / HTTP-Modul

Axios ist ein umfangreich ausgestattetes HTTP-Client-Paket, das weit verbreitet ist. Nest umschließt Axios und stellt es über das integrierte HttpModule bereit. Das HttpModule exportiert die HttpService-Klasse, die auf Axios basierende Methoden bereitstellt, um HTTP-Anfragen auszuführen. Die Bibliothek verwandelt die resultierenden HTTP-Antworten auch in Observables.

**HINWEIS**  
Sie können auch jede allgemeine Node.js-HTTP-Client-Bibliothek direkt verwenden, einschließlich got oder undici.

## Installation / Installation#

Um es zu verwenden, installieren wir zunächst die erforderlichen Abhängigkeiten:

```
$ npm i --save @nestjs/axios axios
```

## Erste Schritte / Getting started#

Sobald der Installationsprozess abgeschlossen ist, importieren Sie das HttpModule, um den HttpService zu verwenden.

```typescript
@Module({
  imports: [HttpModule],
  providers: [CatsService],
})
export class CatsModule {}
```

Als nächstes injizieren Sie den HttpService unter Verwendung der normalen Konstruktorinjektion.

**HINWEIS**  
HttpModule und HttpService werden aus dem @nestjs/axios Paket importiert.

```typescript
@Injectable()
export class CatsService {
  constructor(private readonly httpService: HttpService) {}

  findAll(): Observable<AxiosResponse<Cat[]>> {
    return this.httpService.get('http://localhost:3000/cats');
  }
}
```

**HINWEIS**  
AxiosResponse ist eine Schnittstelle, die aus dem axios Paket exportiert wird ($ npm i axios). Alle HttpService-Methoden geben eine AxiosResponse zurück, die in ein Observable-Objekt eingebettet ist.

## Konfiguration / Configuration#

Axios kann mit einer Vielzahl von Optionen konfiguriert werden, um das Verhalten des HttpService anzupassen. Lesen Sie [hier](https://github.com/axios/axios#request-config) mehr darüber. Um die zugrunde liegende Axios-Instanz zu konfigurieren, übergeben Sie ein optionales Optionsobjekt an die register() Methode des HttpModule, wenn Sie es importieren. Dieses Optionsobjekt wird direkt an den zugrunde liegenden Axios-Konstruktor übergeben.

```typescript
@Module({
  imports: [
    HttpModule.register({
      timeout: 5000,
      maxRedirects: 5,
    }),
  ],
  providers: [CatsService],
})
export class CatsModule {}
```

## Asynchrone Konfiguration / Async configuration#

Wenn Sie die Moduloptionen asynchron anstatt statisch übergeben müssen, verwenden Sie die registerAsync() Methode. Wie bei den meisten dynamischen Modulen bietet Nest mehrere Techniken zur Handhabung asynchroner Konfiguration.

Eine Technik besteht darin, eine Factory-Funktion zu verwenden:

```typescript
HttpModule.registerAsync({
  useFactory: () => ({
    timeout: 5000,
    maxRedirects: 5,
  }),
});
```

Wie andere Factory-Anbieter kann unsere Factory-Funktion asynchron sein und Abhängigkeiten durch Inject injizieren.

```typescript
HttpModule.registerAsync({
  imports: [ConfigModule],
  useFactory: async (configService: ConfigService) => ({
    timeout: configService.get('HTTP_TIMEOUT'),
    maxRedirects: configService.get('HTTP_MAX_REDIRECTS'),
  }),
  inject: [ConfigService],
});
```

Alternativ können Sie das HttpModule mit einer Klasse anstelle einer Factory konfigurieren, wie unten gezeigt.

```typescript
HttpModule.registerAsync({
  useClass: HttpConfigService,
});
```

Die obige Konstruktion instanziiert HttpConfigService innerhalb des HttpModule und verwendet es zur Erstellung eines Optionsobjekts. Beachten Sie, dass in diesem Beispiel die HttpConfigService die HttpModuleOptionsFactory-Schnittstelle implementieren muss, wie unten gezeigt. Das HttpModule ruft die Methode createHttpOptions() auf dem instanziierten Objekt der angegebenen Klasse auf.

```typescript
@Injectable()
class HttpConfigService implements HttpModuleOptionsFactory {
  createHttpOptions(): HttpModuleOptions {
    return {
      timeout: 5000,
      maxRedirects: 5,
    };
  }
}
```

Wenn Sie einen vorhandenen Optionsanbieter wiederverwenden möchten, anstatt eine private Kopie innerhalb des HttpModule zu erstellen, verwenden Sie die useExisting-Syntax.

```typescript
HttpModule.registerAsync({
  imports: [ConfigModule],
  useExisting: HttpConfigService,
});
```

## Verwendung von Axios direkt / Using Axios directly#

Wenn Sie denken, dass die Optionen von HttpModule.register nicht ausreichend für Sie sind oder wenn Sie einfach nur auf die zugrunde liegende Axios-Instanz zugreifen möchten, die von @nestjs/axios erstellt wird, können Sie dies über HttpService#axiosRef wie folgt tun:

```typescript
@Injectable()
export class CatsService {
  constructor(private readonly httpService: HttpService) {}

  findAll(): Promise<AxiosResponse<Cat[]>> {
    return this.httpService.axiosRef.get('http://localhost:3000/cats');
    //                      ^ AxiosInstance-Schnittstelle
  }
}
```

## Vollständiges Beispiel / Full example#

Da der Rückgabewert der HttpService-Methoden ein Observable ist, können wir rxjs - firstValueFrom oder lastValueFrom verwenden, um die Daten der Anfrage in Form eines Versprechens abzurufen.

```typescript
import { catchError, firstValueFrom } from 'rxjs';

@Injectable()
export class CatsService {
  private readonly logger = new Logger(CatsService.name);
  constructor(private readonly httpService: HttpService) {}

  async findAll(): Promise<Cat[]> {
    const { data } = await firstValueFrom(
      this.httpService.get<Cat[]>('http://localhost:3000/cats').pipe(
        catchError((error: AxiosError) => {
          this.logger.error(error.response.data);
          throw 'An error happened!';
        }),
      ),
    );
    return data;
  }
}
```

**HINWEIS**  
Besuchen Sie die RxJS-Dokumentation zu firstValueFrom und lastValueFrom, um die Unterschiede zwischen ihnen zu erfahren.
