# File upload / Datei-Upload

Um Datei-Uploads zu handhaben, bietet Nest ein eingebautes Modul basierend auf dem Multer-Middleware-Paket für Express. Multer verarbeitet Daten, die im multipart/form-data-Format gepostet werden, das hauptsächlich für das Hochladen von Dateien über eine HTTP-POST-Anfrage verwendet wird. Dieses Modul ist vollständig konfigurierbar und Sie können sein Verhalten an die Anforderungen Ihrer Anwendung anpassen.

**WARNUNG**  
Multer kann keine Daten verarbeiten, die nicht im unterstützten multipart-Format (multipart/form-data) vorliegen. Beachten Sie auch, dass dieses Paket nicht mit dem FastifyAdapter kompatibel ist.

Für eine bessere Typensicherheit installieren wir das Multer-Typisierungspaket:

```
$ npm i -D @types/multer
```

Mit diesem installierten Paket können wir nun den Typ Express.Multer.File verwenden (Sie können diesen Typ wie folgt importieren: `import { Express } from 'express'`).

## Grundlegendes Beispiel / Basic example#

Um eine einzelne Datei hochzuladen, binden Sie einfach den FileInterceptor() an den Routen-Handler und extrahieren Sie die Datei aus der Anfrage mithilfe des @UploadedFile() Dekorators.

```typescript
@Post('upload')
@UseInterceptors(FileInterceptor('file'))
uploadFile(@UploadedFile() file: Express.Multer.File) {
  console.log(file);
}
```

**HINWEIS**  
Der FileInterceptor() Dekorator wird aus dem @nestjs/platform-express Paket exportiert. Der @UploadedFile() Dekorator wird aus @nestjs/common exportiert.

Der FileInterceptor() Dekorator nimmt zwei Argumente entgegen:

- `fieldName`: String, der den Namen des Feldes aus dem HTML-Formular angibt, das eine Datei enthält.
- `options`: Optionales Objekt vom Typ MulterOptions. Dies ist dasselbe Objekt, das vom Multer-Konstruktor verwendet wird (mehr Details [hier](https://github.com/expressjs/multer#multeropts)).

**WARNUNG**  
FileInterceptor() ist möglicherweise nicht mit Drittanbietern von Cloud-Diensten wie Google Firebase oder anderen kompatibel.

## Dateivalidierung / File validation#

Oftmals kann es nützlich sein, eingehende Datei-Metadaten wie Dateigröße oder Dateityp zu validieren. Dafür können Sie Ihre eigene Pipe erstellen und sie an den Parameter binden, der mit dem UploadedFile-Dekorator annotiert ist. Das untenstehende Beispiel zeigt, wie eine grundlegende Datei-Größen-Validator-Pipe implementiert werden könnte:

```typescript
import { PipeTransform, Injectable, ArgumentMetadata } from '@nestjs/common';

@Injectable()
export class FileSizeValidationPipe implements PipeTransform {
  transform(value: any, metadata: ArgumentMetadata) {
    // "value" ist ein Objekt, das die Attribute und Metadaten der Datei enthält
    const oneKb = 1000;
    return value.size < oneKb;
  }
}
```

Nest bietet eine eingebaute Pipe zur Handhabung häufiger Anwendungsfälle und zur Erleichterung/Standardisierung der Hinzufügung neuer Fälle. Diese Pipe wird ParseFilePipe genannt und kann wie folgt verwendet werden:

```typescript
@Post('file')
uploadFileAndPassValidation(
  @Body() body: SampleDto,
  @UploadedFile(
    new ParseFilePipe({
      validators: [
        // ... Satz von Datei-Validator-Instanzen hier
      ]
    })
  )
  file: Express.Multer.File,
) {
  return {
    body,
    file: file.buffer.toString(),
  };
}
```

Wie Sie sehen können, ist es erforderlich, ein Array von Datei-Validatoren anzugeben, die von der ParseFilePipe ausgeführt werden. Wir werden die Schnittstelle eines Validators besprechen, aber es ist erwähnenswert, dass diese Pipe auch zwei zusätzliche optionale Optionen hat:

- `errorHttpStatusCode`: Der HTTP-Statuscode, der geworfen wird, wenn ein Validator fehlschlägt. Standard ist 400 (BAD REQUEST).
- `exceptionFactory`: Eine Fabrik, die die Fehlermeldung empfängt und einen Fehler zurückgibt.

Nun zurück zur FileValidator-Schnittstelle. Um Validatoren mit dieser Pipe zu integrieren, müssen Sie entweder die eingebauten Implementierungen verwenden oder Ihren eigenen benutzerdefinierten FileValidator bereitstellen. Siehe Beispiel unten:

```typescript
export abstract class FileValidator<TValidationOptions = Record<string, any>> {
  constructor(protected readonly validationOptions: TValidationOptions) {}

  /**
   * Gibt an, ob diese Datei als gültig betrachtet werden sollte, gemäß den im Konstruktor übergebenen Optionen.
   * @param file die Datei aus dem Anfrageobjekt
   */
  abstract isValid(file?: any): boolean | Promise<boolean>;

  /**
   * Baut eine Fehlermeldung, falls die Validierung fehlschlägt.
   * @param file die Datei aus dem Anfrageobjekt
   */
  abstract buildErrorMessage(file: any): string;
}
```

**HINWEIS**  
Die FileValidator-Schnittstelle unterstützt asynchrone Validierung über ihre isValid-Funktion. Um die Typensicherheit zu nutzen, können Sie den file-Parameter auch als Express.Multer.File typisieren, wenn Sie express (Standard) als Treiber verwenden.

FileValidator ist eine reguläre Klasse, die Zugriff auf das Dateiobjekt hat und es gemäß den vom Kunden bereitgestellten Optionen validiert. Nest hat zwei eingebaute FileValidator-Implementierungen, die Sie in Ihrem Projekt verwenden können:

- `MaxFileSizeValidator`: Überprüft, ob die Größe einer gegebenen Datei kleiner als der angegebene Wert ist (gemessen in Bytes).
- `FileTypeValidator`: Überprüft, ob der MIME-Typ einer gegebenen Datei dem angegebenen Wert entspricht.

**WARNUNG**  
Um den Dateityp zu überprüfen, verwendet die FileTypeValidator-Klasse den von multer erkannten Typ. Standardmäßig leitet multer den Dateityp aus der Dateierweiterung auf dem Gerät des Benutzers ab. Es überprüft jedoch nicht den tatsächlichen Dateiinhalt. Da Dateien in beliebige Erweiterungen umbenannt werden können, sollten Sie eine benutzerdefinierte Implementierung (z.B. Überprüfung der Magic Number der Datei) in Betracht ziehen, wenn Ihre App eine sicherere Lösung erfordert.

Um zu verstehen, wie diese in Verbindung mit der oben genannten ParseFilePipe verwendet werden können, verwenden wir einen geänderten Ausschnitt des zuletzt präsentierten Beispiels:

```typescript
@UploadedFile(
  new ParseFilePipe({
    validators: [
      new MaxFileSizeValidator({ maxSize: 1000 }),
      new FileTypeValidator({ fileType: 'image/jpeg' }),
    ],
  }),
)
file: Express.Multer.File,
```

**HINWEIS**  
Wenn die Anzahl der Validatoren stark zunimmt oder ihre Optionen die Datei überladen, können Sie dieses Array in einer separaten Datei definieren und es hier als benannte Konstante wie fileValidators importieren.

Schließlich können Sie die spezielle ParseFilePipeBuilder-Klasse verwenden, die es Ihnen ermöglicht, Ihre Validatoren zu kombinieren und zu konstruieren. Durch die Verwendung wie unten gezeigt, können Sie die manuelle Instanziierung jedes Validators vermeiden und deren Optionen direkt übergeben:

```typescript
@UploadedFile(
  new ParseFilePipeBuilder()
    .addFileTypeValidator({
      fileType: 'jpeg',
    })
    .addMaxSizeValidator({
      maxSize: 1000
    })
    .build({
      errorHttpStatusCode: HttpStatus.UNPROCESSABLE_ENTITY
    }),
)
file: Express.Multer.File,
```

**HINWEIS**  
Das Vorhandensein einer Datei ist standardmäßig erforderlich, aber Sie können dies optional machen, indem Sie die Datei `fileIsRequired: false` in die Optionen der build-Funktion (auf derselben Ebene wie `errorHttpStatusCode`) hinzufügen.

## Array von Dateien / Array of files#

Um ein Array von Dateien (identifiziert durch einen einzelnen Feldnamen) hochzuladen, verwenden Sie den FilesInterceptor() Dekorator (beachten Sie das Plural Files im Dekoratornamen). Dieser Dekorator nimmt drei Argumente entgegen:

- `fieldName`: wie oben beschrieben
- `maxCount`: Optionale Zahl, die die maximale Anzahl von zu akzeptierenden Dateien definiert
- `options`: Optionales MulterOptions-Objekt, wie oben beschrieben

Beim Verwenden von FilesInterceptor() extrahieren Sie Dateien aus der Anfrage mit dem @UploadedFiles() Dekorator.

```typescript
@Post('upload')
@UseInterceptors(FilesInterceptor('files'))
uploadFile(@UploadedFiles() files: Array<Express.Multer.File>) {
  console.log(files);
}
```

**HINWEIS**  
Der FilesInterceptor() Dekorator wird aus dem @nestjs/platform-express Paket exportiert. Der @UploadedFiles() Dekorator wird aus @nestjs/common exportiert.

## Mehrere Dateien / Multiple files#

Um mehrere Dateien (alle mit unterschiedlichen Feldnamen-Schlüsseln) hochzuladen, verwenden Sie den FileFieldsInterceptor() Dekorator. Dieser Dekorator nimmt zwei Argumente entgegen:

- `uploadedFields`: Ein Array von Objekten, wobei jedes Objekt eine erforderliche name-Eigenschaft mit einem String-Wert, der einen Feldnamen angibt, wie oben beschrieben, und eine optionale maxCount-Eigenschaft, wie oben beschrieben, spezifiziert.
- `options`: Optionales MulterOptions-Objekt, wie oben beschrieben

Beim Verwenden von FileFieldsInterceptor() extrahieren Sie Dateien aus der Anfrage mit dem @UploadedFiles() Dekorator.

```typescript
@Post('upload')
@UseInterceptors(FileFieldsInterceptor([
  { name: 'avatar', maxCount: 1 },
  { name: '

background', maxCount: 1 },
]))
uploadFile(@UploadedFiles() files: { avatar?: Express.Multer.File[], background?: Express.Multer.File[] }) {
  console.log(files);
}
```

## Beliebige Dateien / Any files#

Um alle Felder mit beliebigen Feldnamen-Schlüsseln hochzuladen, verwenden Sie den AnyFilesInterceptor() Dekorator. Dieser Dekorator kann ein optionales Optionsobjekt wie oben beschrieben akzeptieren.

Beim Verwenden von AnyFilesInterceptor() extrahieren Sie Dateien aus der Anfrage mit dem @UploadedFiles() Dekorator.

```typescript
@Post('upload')
@UseInterceptors(AnyFilesInterceptor())
uploadFile(@UploadedFiles() files: Array<Express.Multer.File>) {
  console.log(files);
}
```

## Keine Dateien / No files#

Um multipart/form-data zu akzeptieren, aber keine Dateien hochladen zu lassen, verwenden Sie den NoFilesInterceptor. Dies setzt Multipart-Daten als Attribute im Anfragekörper. Dateien, die mit der Anfrage gesendet werden, lösen eine BadRequestException aus.

```typescript
@Post('upload')
@UseInterceptors(NoFilesInterceptor())
handleMultiPartData(@Body() body) {
  console.log(body)
}
```

## Standardoptionen / Default options#

Sie können Multer-Optionen in den Datei-Interceptors wie oben beschrieben angeben. Um Standardoptionen festzulegen, können Sie die statische Methode register() aufrufen, wenn Sie das MulterModule importieren, und unterstützte Optionen übergeben. Sie können alle [hier](https://github.com/expressjs/multer#multeropts) aufgelisteten Optionen verwenden.

```typescript
MulterModule.register({
  dest: './upload',
});
```

**HINWEIS**  
Die MulterModule-Klasse wird aus dem @nestjs/platform-express Paket exportiert.

## Asynchrone Konfiguration / Async configuration#

Wenn Sie MulterModule-Optionen asynchron statt statisch festlegen müssen, verwenden Sie die Methode registerAsync(). Wie bei den meisten dynamischen Modulen bietet Nest mehrere Techniken zur Handhabung asynchroner Konfigurationen.

Eine Technik besteht darin, eine Factory-Funktion zu verwenden:

```typescript
MulterModule.registerAsync({
  useFactory: () => ({
    dest: './upload',
  }),
});
```

Wie andere Factory-Anbieter kann unsere Factory-Funktion asynchron sein und Abhängigkeiten durch Inject injizieren.

```typescript
MulterModule.registerAsync({
  imports: [ConfigModule],
  useFactory: async (configService: ConfigService) => ({
    dest: configService.get<string>('MULTER_DEST'),
  }),
  inject: [ConfigService],
});
```

Alternativ können Sie das MulterModule mit einer Klasse anstelle einer Factory konfigurieren, wie unten gezeigt:

```typescript
MulterModule.registerAsync({
  useClass: MulterConfigService,
});
```

Die obenstehende Konstruktion instanziiert MulterConfigService innerhalb des MulterModule und verwendet es zur Erstellung des erforderlichen Optionsobjekts. Beachten Sie, dass in diesem Beispiel die MulterConfigService die MulterOptionsFactory-Schnittstelle implementieren muss, wie unten gezeigt. Das MulterModule ruft die Methode createMulterOptions() auf dem instanziierten Objekt der angegebenen Klasse auf.

```typescript
@Injectable()
class MulterConfigService implements MulterOptionsFactory {
  createMulterOptions(): MulterModuleOptions {
    return {
      dest: './upload',
    };
  }
}
```

Wenn Sie einen vorhandenen Optionsanbieter wiederverwenden möchten, anstatt eine private Kopie innerhalb des MulterModule zu erstellen, verwenden Sie die useExisting-Syntax.

```typescript
MulterModule.registerAsync({
  imports: [ConfigModule],
  useExisting: ConfigService,
});
```

## Beispiel / Example#

Ein funktionierendes Beispiel finden Sie [hier](https://github.com/nestjs/nest/tree/master/sample/29-file-upload).
