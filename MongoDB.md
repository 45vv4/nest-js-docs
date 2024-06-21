## MongoDB-Integration mit NestJS

Nest unterstützt zwei Methoden zur Integration mit der MongoDB-Datenbank. Sie können entweder das integrierte TypeORM-Modul verwenden, das einen Connector für MongoDB hat, oder Mongoose, das beliebteste Objektmodellierungswerkzeug für MongoDB, nutzen. In diesem Kapitel beschreiben wir Letzteres, unter Verwendung des dedizierten `@nestjs/mongoose` Pakets.

### Installation der erforderlichen Abhängigkeiten

```bash
$ npm i @nestjs/mongoose mongoose
```

Sobald die Installation abgeschlossen ist, können wir das `MongooseModule` in das Hauptmodul `AppModule` importieren.

```typescript
// app.module.ts

import { Module } from '@nestjs/common';
import { MongooseModule } from '@nestjs/mongoose';

@Module({
  imports: [MongooseModule.forRoot('mongodb://localhost/nest')],
})
export class AppModule {}
```

Die `forRoot()` Methode akzeptiert dasselbe Konfigurationsobjekt wie `mongoose.connect()` aus dem Mongoose-Paket.

### Modell-Injektion

Mit Mongoose wird alles von einem Schema abgeleitet. Jedes Schema wird einer MongoDB-Sammlung zugeordnet und definiert die Form der Dokumente in dieser Sammlung. Schemas werden verwendet, um Modelle zu definieren, die für das Erstellen und Lesen von Dokumenten aus der zugrunde liegenden MongoDB-Datenbank verantwortlich sind.

Schemas können mit NestJS-Dekoratoren oder manuell mit Mongoose selbst erstellt werden. Die Verwendung von Dekoratoren zur Erstellung von Schemas reduziert den Boilerplate-Code erheblich und verbessert die allgemeine Lesbarkeit des Codes.

Definieren wir das `CatSchema`:

```typescript
// schemas/cat.schema.ts

import { Prop, Schema, SchemaFactory } from '@nestjs/mongoose';
import { HydratedDocument } from 'mongoose';

export type CatDocument = HydratedDocument<Cat>;

@Schema()
export class Cat {
  @Prop()
  name: string;

  @Prop()
  age: number;

  @Prop()
  breed: string;
}

export const CatSchema = SchemaFactory.createForClass(Cat);
```

### HINWEIS
Sie können auch eine rohe Schema-Definition mit der `DefinitionsFactory`-Klasse (aus `nestjs/mongoose`) erzeugen. Dies ermöglicht es Ihnen, die Schema-Definition manuell zu ändern, basierend auf den bereitgestellten Metadaten.

Der `@Schema()` Dekorator markiert eine Klasse als Schema-Definition. Es ordnet unsere `Cat`-Klasse einer MongoDB-Sammlung mit demselben Namen zu, aber mit einem zusätzlichen „s“ am Ende, sodass der endgültige Mongo-Sammlungsname `cats` lautet.

Der `@Prop()` Dekorator definiert eine Eigenschaft im Dokument. Zum Beispiel haben wir in der obigen Schema-Definition drei Eigenschaften definiert: `name`, `age` und `breed`. Die Schema-Typen für diese Eigenschaften werden dank der TypeScript-Metadaten (und Reflexions-) Fähigkeiten automatisch abgeleitet. In komplexeren Szenarien, in denen Typen nicht implizit abgeleitet werden können (z.B. Arrays oder verschachtelte Objektstrukturen), müssen die Typen explizit angegeben werden:

```typescript
@Prop([String])
tags: string[];
```

Alternativ akzeptiert der `@Prop()` Dekorator ein Optionsobjekt als Argument. Damit können Sie angeben, ob eine Eigenschaft erforderlich ist oder nicht, einen Standardwert festlegen oder sie als unveränderlich markieren:

```typescript
@Prop({ required: true })
name: string;
```

### Beziehung zu anderen Modellen

Wenn Sie eine Beziehung zu einem anderen Modell angeben möchten, um später zu populieren, können Sie ebenfalls den `@Prop()` Dekorator verwenden. Wenn `Cat` zum Beispiel einen `Owner` hat, der in einer anderen Sammlung namens `owners` gespeichert ist, sollte die Eigenschaft `type` und `ref` haben:

```typescript
import * as mongoose from 'mongoose';
import { Owner } from '../owners/schemas/owner.schema';

// innerhalb der Klassendefinition
@Prop({ type: mongoose.Schema.Types.ObjectId, ref: 'Owner' })
owner: Owner;
```

Wenn es mehrere Besitzer gibt, sollte Ihre Eigenschaftskonfiguration wie folgt aussehen:

```typescript
@Prop({ type: [{ type: mongoose.Schema.Types.ObjectId, ref: 'Owner' }] })
owner: Owner[];
```

### Manuelle Schema-Definition

Alternativ können Sie ein Schema manuell definieren:

```typescript
export const CatSchema = new mongoose.Schema({
  name: String,
  age: Number,
  breed: String,
});
```

Die `cat.schema` Datei befindet sich in einem Ordner im `cats` Verzeichnis, wo wir auch das `CatsModule` definieren. Während Sie Schema-Dateien überall speichern können, empfehlen wir, sie in der Nähe ihrer verwandten Domänenobjekte im entsprechenden Modulverzeichnis zu speichern.

### CatsModule

```typescript
// cats.module.ts

import { Module } from '@nestjs/common';
import { MongooseModule } from '@nestjs/mongoose';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';
import { Cat, CatSchema } from './schemas/cat.schema';

@Module({
  imports: [MongooseModule.forFeature([{ name: Cat.name, schema: CatSchema }])],
  controllers: [CatsController],
  providers: [CatsService],
})
export class CatsModule {}
```

Das `MongooseModule` bietet die `forFeature()` Methode zur Konfiguration des Moduls, einschließlich der Definition, welche Modelle im aktuellen Bereich registriert werden sollen. Wenn Sie die Modelle auch in einem anderen Modul verwenden möchten, fügen Sie `MongooseModule` zum `exports` Abschnitt des `CatsModule` hinzu und importieren Sie das `CatsModule` im anderen Modul.

### Injektion eines Modells in den CatsService

Nachdem Sie das Schema registriert haben, können Sie ein `Cat` Modell in den `CatsService` mit dem `@InjectModel()` Dekorator injizieren:

```typescript
// cats.service.ts

import { Model } from 'mongoose';
import { Injectable } from '@nestjs/common';
import { InjectModel } from '@nestjs/mongoose';
import { Cat } from './schemas/cat.schema';
import { CreateCatDto } from './dto/create-cat.dto';

@Injectable()
export class CatsService {
  constructor(@InjectModel(Cat.name) private catModel: Model<Cat>) {}

  async create(createCatDto: CreateCatDto): Promise<Cat> {
    const createdCat = new this.catModel(createCatDto);
    return createdCat.save();
  }

  async findAll(): Promise<Cat[]> {
    return this.catModel.find().exec();
  }
}
```

### Verbindung

Manchmal müssen Sie auf das native Mongoose-Verbindungsobjekt zugreifen, z.B. um native API-Aufrufe auf dem Verbindungsobjekt zu tätigen. Sie können die Mongoose-Verbindung mit dem `@InjectConnection()` Dekorator injizieren:

```typescript
import { Injectable } from '@nestjs/common';
import { InjectConnection } from '@nestjs/mongoose';
import { Connection } from 'mongoose';

@Injectable()
export class CatsService {
  constructor(@InjectConnection() private connection: Connection) {}
}
```

### Mehrere Datenbanken

Einige Projekte erfordern mehrere Datenbankverbindungen. Dies kann auch mit diesem Modul erreicht werden. Um mit mehreren Verbindungen zu arbeiten, erstellen Sie zuerst die Verbindungen. In diesem Fall wird die Namensgebung der Verbindung obligatorisch.

```typescript
// app.module.ts

import { Module } from '@nestjs/common';
import { MongooseModule } from '@nestjs/mongoose';

@Module({
  imports: [
    MongooseModule.forRoot('mongodb://localhost/test', {
      connectionName: 'cats',
    }),
    MongooseModule.forRoot('mongodb://localhost/users', {
      connectionName: 'users',
    }),
  ],
})
export class AppModule {}
```

### HINWEIS
Bitte beachten Sie, dass Sie keine mehreren Verbindungen ohne Namen oder mit demselben Namen haben sollten, da sie sonst überschrieben werden.

Mit diesem Setup müssen Sie der `MongooseModule.forFeature()` Funktion mitteilen, welche Verbindung verwendet werden soll:

```typescript
@Module({
  imports: [
    MongooseModule.forFeature([{ name: Cat.name, schema: CatSchema }], 'cats'),
  ],
})
export class CatsModule {}
```

Sie können auch die Verbindung für eine gegebene Verbindung injizieren:

```typescript
import { Injectable } from '@nestjs/common';
import { InjectConnection } from '@nestjs/mongoose';
import { Connection } from 'mongoose';

@Injectable()
export class CatsService {
  constructor(@InjectConnection('cats') private connection: Connection) {}
}
```

Um eine gegebene Verbindung zu einem benutzerdefinierten Provider zu injizieren, verwenden Sie die `getConnectionToken()` Funktion und geben den Namen der Verbindung als Argument an:

```typescript
{
  provide: CatsService,
  useFactory: (catsConnection: Connection) => {
    return new CatsService(catsConnection);
  },
  inject: [getConnectionToken('cats')],
}
```

Wenn Sie nur das Modell aus einer benannten Datenbank injizieren möchten, können Sie den Verbindungsnamen als zweites Argument zum `@InjectModel()` Dekorator verwenden:

```typescript
// cats.service.ts

@Injectable()
export class CatsService {
  constructor(@InjectModel(Cat.name, 'cats') private catModel: Model<Cat>) {}
}
```

### Hooks (Middleware)

Middleware (auch Pre- und Post-Hooks genannt) sind Funktionen, die während der Ausführung asynchroner Funktionen die Kontrolle übernehmen. Middleware wird auf Schema-Ebene

 angegeben und ist nützlich für das Schreiben von Plugins. Das Aufrufen von `pre()` oder `post()` nach der Kompilierung eines Modells funktioniert in Mongoose nicht. Um einen Hook vor der Modellregistrierung zu registrieren, verwenden Sie die `forFeatureAsync()` Methode des `MongooseModule` zusammen mit einem Factory-Provider. Mit dieser Technik können Sie auf ein Schema-Objekt zugreifen und dann die `pre()` oder `post()` Methode verwenden, um einen Hook für dieses Schema zu registrieren:

```typescript
@Module({
  imports: [
    MongooseModule.forFeatureAsync([
      {
        name: Cat.name,
        useFactory: () => {
          const schema = CatsSchema;
          schema.pre('save', function () {
            console.log('Hello from pre save');
          });
          return schema;
        },
      },
    ]),
  ],
})
export class AppModule {}
```

Wie andere Factory-Provider kann unsere Factory-Funktion asynchron sein und Abhängigkeiten durch `inject` injizieren:

```typescript
@Module({
  imports: [
    MongooseModule.forFeatureAsync([
      {
        name: Cat.name,
        imports: [ConfigModule],
        useFactory: (configService: ConfigService) => {
          const schema = CatsSchema;
          schema.pre('save', function() {
            console.log(
              `${configService.get('APP_NAME')}: Hello from pre save`,
            ),
          });
          return schema;
        },
        inject: [ConfigService],
      },
    ]),
  ],
})
export class AppModule {}
```

### Plugins

Um ein Plugin für ein bestimmtes Schema zu registrieren, verwenden Sie die `forFeatureAsync()` Methode:

```typescript
@Module({
  imports: [
    MongooseModule.forFeatureAsync([
      {
        name: Cat.name,
        useFactory: () => {
          const schema = CatsSchema;
          schema.plugin(require('mongoose-autopopulate'));
          return schema;
        },
      },
    ]),
  ],
})
export class AppModule {}
```

Um ein Plugin für alle Schemas auf einmal zu registrieren, rufen Sie die `.plugin()` Methode des Verbindungsobjekts auf. Sie sollten auf die Verbindung zugreifen, bevor Modelle erstellt werden; dazu verwenden Sie die `connectionFactory`:

```typescript
// app.module.ts

import { Module } from '@nestjs/common';
import { MongooseModule } from '@nestjs/mongoose';

@Module({
  imports: [
    MongooseModule.forRoot('mongodb://localhost/test', {
      connectionFactory: (connection) => {
        connection.plugin(require('mongoose-autopopulate'));
        return connection;
      }
    }),
  ],
})
export class AppModule {}
```

### Discriminators

Discriminators sind ein Mechanismus zur Schema-Vererbung. Sie ermöglichen es Ihnen, mehrere Modelle mit sich überschneidenden Schemas auf derselben zugrunde liegenden MongoDB-Sammlung zu haben.

Angenommen, Sie möchten verschiedene Arten von Ereignissen in einer einzigen Sammlung verfolgen. Jedes Ereignis hat einen Zeitstempel.

```typescript
// event.schema.ts

@Schema({ discriminatorKey: 'kind' })
export class Event {
  @Prop({
    type: String,
    required: true,
    enum: [ClickedLinkEvent.name, SignUpEvent.name],
  })
  kind: string;

  @Prop({ type: Date, required: true })
  time: Date;
}

export const EventSchema = SchemaFactory.createForClass(Event);
```

### HINWEIS
Die Art und Weise, wie Mongoose zwischen den verschiedenen Discriminator-Modellen unterscheidet, erfolgt über den "Discriminator Key", der standardmäßig `__t` ist. Mongoose fügt Ihren Schemas einen String-Pfad namens `__t` hinzu, den es verwendet, um zu verfolgen, welcher Discriminator dieses Dokument eine Instanz ist. Sie können auch die `discriminatorKey` Option verwenden, um den Pfad für die Diskriminierung zu definieren.

Instanzen von `SignedUpEvent` und `ClickedLinkEvent` werden in derselben Sammlung wie generische Ereignisse gespeichert.

Definieren wir nun die `ClickedLinkEvent` Klasse:

```typescript
// click-link-event.schema.ts

@Schema()
export class ClickedLinkEvent {
  kind: string;
  time: Date;

  @Prop({ type: String, required: true })
  url: string;
}

export const ClickedLinkEventSchema = SchemaFactory.createForClass(ClickedLinkEvent);
```

Und die `SignUpEvent` Klasse:

```typescript
// sign-up-event.schema.ts

@Schema()
export class SignUpEvent {
  kind: string;
  time: Date;

  @Prop({ type: String, required: true })
  user: string;
}

export const SignUpEventSchema = SchemaFactory.createForClass(SignUpEvent);
```

Mit diesem Setup verwenden Sie die `discriminators` Option, um einen Discriminator für ein bestimmtes Schema zu registrieren. Es funktioniert sowohl mit `MongooseModule.forFeature` als auch mit `MongooseModule.forFeatureAsync`:

```typescript
// event.module.ts

import { Module } from '@nestjs/common';
import { MongooseModule } from '@nestjs/mongoose';

@Module({
  imports: [
    MongooseModule.forFeature([
      {
        name: Event.name,
        schema: EventSchema,
        discriminators: [
          { name: ClickedLinkEvent.name, schema: ClickedLinkEventSchema },
          { name: SignUpEvent.name, schema: SignUpEventSchema },
        ],
      },
    ]),
  ]
})
export class EventsModule {}
```

### Testen

Beim Unit-Testen einer Anwendung möchten wir normalerweise jede Datenbankverbindung vermeiden, um unsere Test-Suites einfacher einzurichten und schneller auszuführen. Aber unsere Klassen könnten von Modellen abhängen, die aus der Verbindungsinstanz gezogen werden. Wie lösen wir diese Klassen? Die Lösung besteht darin, Mock-Modelle zu erstellen.

Um dies zu erleichtern, stellt das `@nestjs/mongoose` Paket eine `getModelToken()` Funktion zur Verfügung, die ein vorbereitetes Injektionstoken basierend auf einem Token-Namen zurückgibt. Mit diesem Token können Sie leicht eine Mock-Implementierung mit einer der Standard-Custom-Provider-Techniken bereitstellen, einschließlich `useClass`, `useValue` und `useFactory`:

```typescript
@Module({
  providers: [
    CatsService,
    {
      provide: getModelToken(Cat.name),
      useValue: catModel,
    },
  ],
})
export class CatsModule {}
```

In diesem Beispiel wird ein fest codiertes `catModel` (Objektinstanz) bereitgestellt, wann immer ein Consumer ein `Model<Cat>` mit einem `@InjectModel()` Dekorator injiziert.

### Asynchrone Konfiguration

Wenn Sie Modulkoptionswerte asynchron statt statisch übergeben müssen, verwenden Sie die `forRootAsync()` Methode. Wie bei den meisten dynamischen Modulen bietet Nest mehrere Techniken zur asynchronen Konfiguration.

Eine Technik besteht darin, eine Factory-Funktion zu verwenden:

```typescript
MongooseModule.forRootAsync({
  useFactory: () => ({
    uri: 'mongodb://localhost/nest',
  }),
});
```

Wie andere Factory-Provider kann unsere Factory-Funktion asynchron sein und Abhängigkeiten durch `inject` injizieren:

```typescript
MongooseModule.forRootAsync({
  imports: [ConfigModule],
  useFactory: async (configService: ConfigService) => ({
    uri: configService.get<string>('MONGODB_URI'),
  }),
  inject: [ConfigService],
});
```

Alternativ können Sie das `MongooseModule` mithilfe einer Klasse statt einer Factory konfigurieren:

```typescript
MongooseModule.forRootAsync({
  useClass: MongooseConfigService,
});
```

Die obige Konstruktion instanziiert `MongooseConfigService` innerhalb des `MongooseModule` und verwendet es, um das erforderliche Optionsobjekt zu erstellen. Beachten Sie, dass in diesem Beispiel die `MongooseConfigService`-Klasse die `MongooseOptionsFactory` Schnittstelle implementieren muss:

```typescript
@Injectable()
export class MongooseConfigService implements MongooseOptionsFactory {
  createMongooseOptions(): MongooseModuleOptions {
    return {
      uri: 'mongodb://localhost/nest',
    };
  }
}
```

Wenn Sie einen bestehenden Options-Provider wiederverwenden möchten, anstatt eine private Kopie innerhalb des `MongooseModule` zu erstellen, verwenden Sie die `useExisting` Syntax:

```typescript
MongooseModule.forRootAsync({
  imports: [ConfigModule],
  useExisting: ConfigService,
});
```

### Beispiel

Ein funktionierendes Beispiel ist [hier](https://github.com/nestjs/nest/tree/master/sample/10-mongoose) verfügbar.
