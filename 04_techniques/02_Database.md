# Datenbank / Database

Nest ist datenbankunabhängig, sodass Sie problemlos jede SQL- oder NoSQL-Datenbank integrieren können. Je nach Ihren Vorlieben stehen Ihnen verschiedene Optionen zur Verfügung. Auf der allgemeinsten Ebene besteht die Verbindung von Nest zu einer Datenbank einfach darin, einen geeigneten Node.js-Treiber für die Datenbank zu laden, genauso wie Sie es mit Express oder Fastify tun würden.

Sie können auch direkt jede allgemeine Node.js-Datenbank-Integrationsbibliothek oder ORM verwenden, wie MikroORM (siehe MikroORM-Rezept), Sequelize (siehe Sequelize-Integration), Knex.js (siehe Knex.js-Tutorial), TypeORM und Prisma (siehe Prisma-Rezept), um auf einer höheren Abstraktionsebene zu arbeiten.

Der Einfachheit halber bietet Nest eine enge Integration mit TypeORM und Sequelize out-of-the-box mit den Paketen @nestjs/typeorm bzw. @nestjs/sequelize, die wir in diesem Kapitel behandeln werden, und Mongoose mit @nestjs/mongoose, das in diesem Kapitel behandelt wird. Diese Integrationen bieten zusätzliche NestJS-spezifische Funktionen wie Modell-/Repository-Injektion, Testbarkeit und asynchrone Konfiguration, um den Zugriff auf Ihre gewählte Datenbank noch einfacher zu gestalten.

## TypeORM-Integration / TypeORM Integration

Zur Integration mit SQL- und NoSQL-Datenbanken bietet Nest das Paket @nestjs/typeorm. TypeORM ist der ausgereifteste Object-Relational-Mapper (ORM) für TypeScript. Da es in TypeScript geschrieben ist, integriert es sich gut in das Nest-Framework.

Um es zu verwenden, installieren wir zunächst die erforderlichen Abhängigkeiten. In diesem Kapitel demonstrieren wir die Verwendung des beliebten MySQL Relational DBMS, aber TypeORM unterstützt viele relationale Datenbanken wie PostgreSQL, Oracle, Microsoft SQL Server, SQLite und sogar NoSQL-Datenbanken wie MongoDB. Das Verfahren, das wir in diesem Kapitel durchlaufen, ist für jede von TypeORM unterstützte Datenbank dasselbe. Sie müssen lediglich die zugehörigen Client-API-Bibliotheken für Ihre ausgewählte Datenbank installieren.

```bash
$ npm install --save @nestjs/typeorm typeorm mysql2
```

Sobald der Installationsprozess abgeschlossen ist, können wir das TypeOrmModule in das Stamm-AppModule importieren.

```typescript
// app.module.ts

import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';

@Module({
  imports: [
    TypeOrmModule.forRoot({
      type: 'mysql',
      host: 'localhost',
      port: 3306,
      username: 'root',
      password: 'root',
      database: 'test',
      entities: [],
      synchronize: true,
    }),
  ],
})
export class AppModule {}
```

**WARNUNG**: Das Setzen von synchronize: true sollte in der Produktion nicht verwendet werden - andernfalls können Produktionsdaten verloren gehen.

Die forRoot()-Methode unterstützt alle Konfigurationsparameter, die vom DataSource-Konstruktor aus dem TypeORM-Paket bereitgestellt werden. Darüber hinaus gibt es mehrere zusätzliche Konfigurationsparameter, die unten beschrieben sind.

| retryAttempts   | Anzahl der Versuche, eine Verbindung zur Datenbank herzustellen (Standard: 10) |
|-----------------|------------------------------------------------------------------------------|
| retryDelay      | Verzögerung zwischen den Verbindungswiederholungen (ms) (Standard: 3000)     |
| autoLoadEntities| Wenn true, werden Entitäten automatisch geladen (Standard: false)             |

**TIPP**: Erfahren Sie hier mehr über die Datenquellenoptionen.

Sobald dies erledigt ist, stehen die TypeORM DataSource- und EntityManager-Objekte im gesamten Projekt zur Injektion zur Verfügung (ohne dass Module importiert werden müssen), zum Beispiel:

```typescript
// app.module.ts

import { DataSource } from 'typeorm';

@Module({
  imports: [TypeOrmModule.forRoot(), UsersModule],
})
export class AppModule {
  constructor(private dataSource: DataSource) {}
}
```

## Repository-Muster / Repository pattern

TypeORM unterstützt das Repository-Design-Muster, sodass jede Entität ihr eigenes Repository hat. Diese Repositories können aus der Datenbank-Datenquelle abgerufen werden.

Um das Beispiel fortzusetzen, benötigen wir mindestens eine Entität. Definieren wir die User-Entität.

```typescript
// user.entity.ts

import { Entity, Column, PrimaryGeneratedColumn } from 'typeorm';

@Entity()
export class User {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  firstName: string;

  @Column()
  lastName: string;

  @Column({ default: true })
  isActive: boolean;
}
```

**TIPP**: Erfahren Sie mehr über Entitäten in der TypeORM-Dokumentation.

Die User-Entitätsdatei befindet sich im Verzeichnis users. Dieses Verzeichnis enthält alle Dateien, die zum UsersModule gehören. Sie können entscheiden, wo Sie Ihre Modella-Dateien aufbewahren möchten. Wir empfehlen jedoch, sie in der Nähe ihres Domänenbereichs, im entsprechenden Modulverzeichnis, zu erstellen.

Um die User-Entität zu verwenden, müssen wir TypeORM darüber informieren, indem wir sie in das Entitäten-Array in den Optionen der forRoot()-Methode einfügen (es sei denn, Sie verwenden einen statischen Glob-Pfad):

```typescript
// app.module.ts

import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { User } from './users/user.entity';

@Module({
  imports: [
    TypeOrmModule.forRoot({
      type: 'mysql',
      host: 'localhost',
      port: 3306,
      username: 'root',
      password: 'root',
      database: 'test',
      entities: [User],
      synchronize: true,
    }),
  ],
})
export class AppModule {}
```

Schauen wir uns als nächstes das UsersModule an:

```typescript
// users.module.ts

import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { UsersService } from './users.service';
import { UsersController } from './users.controller';
import { User } from './user.entity';

@Module({
  imports: [TypeOrmModule.forFeature([User])],
  providers: [UsersService],
  controllers: [UsersController],
})
export class UsersModule {}
```

Dieses Modul verwendet die forFeature()-Methode, um zu definieren, welche Repositories im aktuellen Bereich registriert sind. Damit können wir das UsersRepository in den UsersService injizieren, indem wir den @InjectRepository()-Dekorator verwenden:

```typescript
// users.service.ts

import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { User } from './user.entity';

@Injectable()
export class UsersService {
  constructor(
    @InjectRepository(User)
    private usersRepository: Repository<User>,
  ) {}

  findAll(): Promise<User[]> {
    return this.usersRepository.find();
  }

  findOne(id: number): Promise<User | null> {
    return this.usersRepository.findOneBy({ id });
  }

  async remove(id: number): Promise<void> {
    await this.usersRepository.delete(id);
  }
}
```

**HINWEIS**: Vergessen Sie nicht, das UsersModule in das Stamm-AppModule zu importieren.

Wenn Sie das Repository außerhalb des Moduls verwenden möchten, das TypeOrmModule.forFeature importiert, müssen Sie die von ihm generierten Provider erneut exportieren. Sie können dies tun, indem Sie das gesamte Modul exportieren, wie folgt:

```typescript
// users.module.ts

import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { User } from './user.entity';

@Module({
  imports: [TypeOrmModule.forFeature([User])],
  exports: [TypeOrmModule]
})
export class UsersModule {}
```

Wenn wir jetzt UsersModule in UserHttpModule importieren, können wir @InjectRepository(User) in den Providern des letztgenannten Moduls verwenden.

```typescript
// users-http.module.ts

import { Module } from '@nestjs/common';
import { UsersModule } from './users.module';
import { UsersService } from './users.service';
import { UsersController } from './users.controller';

@Module({
  imports: [UsersModule],
  providers: [UsersService],
  controllers: [UsersController]
})
export class UserHttpModule {}
```

## Beziehungen / Relations

Beziehungen sind Assoziationen, die zwischen zwei oder mehr Tabellen hergestellt werden. Beziehungen basieren auf gemeinsamen Feldern jeder Tabelle, oft unter Einbeziehung von Primär- und Fremdschlüsseln.

Es gibt drei Arten von Beziehungen:

| One-to-one      | Jede Zeile in der Primärtabelle hat genau eine zugehörige Zeile in der Fremdtabelle. Verwenden Sie den @OneToOne()-Dekorator, um diese Art von Beziehung zu definieren. |
|-----------------|-----------------------------------------------------------------------------------------------------------|
| One-to-many / Many-to-one | Jede Zeile in der Primärtabelle hat eine oder mehrere zugehörige Zeilen in der Fremdtabelle. Verwenden Sie die Dekoratoren @OneToMany() und @ManyToOne(), um diese Art von Beziehung zu definieren. |
| Many-to-many    | Jede Zeile in der Primärtabelle hat viele zugehörige Zeilen in der Fremdtabelle, und jede Zeile in der Fremdtabelle hat viele zugehörige Zeilen in der Primärtabelle. Verwenden Sie den @ManyToMany()-Dekorator, um diese Art von Beziehung zu definieren. |

Um Beziehungen in Entitäten zu definieren, verwenden Sie die entsprechenden Dekoratoren. Zum Beispiel,

 um zu definieren, dass jeder User mehrere Fotos haben kann, verwenden Sie den @OneToMany()-Dekorator.

```typescript
// user.entity.ts

import { Entity, Column, PrimaryGeneratedColumn, OneToMany } from 'typeorm';
import { Photo } from '../photos/photo.entity';

@Entity()
export class User {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  firstName: string;

  @Column()
  lastName: string;

  @Column({ default: true })
  isActive: boolean;

  @OneToMany(type => Photo, photo => photo.user)
  photos: Photo[];
}
```

**TIPP**: Um mehr über Beziehungen in TypeORM zu erfahren, besuchen Sie die TypeORM-Dokumentation.

## Automatisches Laden von Entitäten / Auto-load entities

Das manuelle Hinzufügen von Entitäten zum entities-Array der Datenquellenoptionen kann mühsam sein. Darüber hinaus wird durch das Referenzieren von Entitäten aus dem Stamm-Modul die Anwendungsdomäne durchbrochen und Implementierungsdetails in andere Teile der Anwendung geleakt. Um dieses Problem zu lösen, wird eine alternative Lösung bereitgestellt. Um Entitäten automatisch zu laden, setzen Sie die autoLoadEntities-Eigenschaft des Konfigurationsobjekts (das in die forRoot()-Methode übergeben wird) auf true, wie unten gezeigt:

```typescript
// app.module.ts

import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';

@Module({
  imports: [
    TypeOrmModule.forRoot({
      ...
      autoLoadEntities: true,
    }),
  ],
})
export class AppModule {}
```

Mit dieser Option werden alle Entitäten, die über die forFeature()-Methode registriert wurden, automatisch zum entities-Array des Konfigurationsobjekts hinzugefügt.

**WARNUNG**: Beachten Sie, dass Entitäten, die nicht über die forFeature()-Methode registriert wurden, sondern nur von der Entität (über eine Beziehung) referenziert werden, nicht durch die autoLoadEntities-Einstellung eingeschlossen werden.

## Trennen der Entitätsdefinition / Separating entity definition

Sie können eine Entität und ihre Spalten direkt im Modell mithilfe von Dekoratoren definieren. Aber einige Leute bevorzugen es, Entitäten und ihre Spalten in separaten Dateien zu definieren, indem sie die "entity schemas" verwenden.

```typescript
import { EntitySchema } from 'typeorm';
import { User } from './user.entity';

export const UserSchema = new EntitySchema<User>({
  name: 'User',
  target: User,
  columns: {
    id: {
      type: Number,
      primary: true,
      generated: true,
    },
    firstName: {
      type: String,
    },
    lastName: {
      type: String,
    },
    isActive: {
      type: Boolean,
      default: true,
    },
  },
  relations: {
    photos: {
      type: 'one-to-many',
      target: 'Photo', // der Name des PhotoSchema
    },
  },
});
```

**WARNUNG**: Wenn Sie die target-Option angeben, muss der Wert der name-Option derselbe wie der Name der Zielklasse sein. Wenn Sie das target nicht angeben, können Sie jeden Namen verwenden.

Nest ermöglicht es Ihnen, eine EntitySchema-Instanz überall dort zu verwenden, wo eine Entität erwartet wird, zum Beispiel:

```typescript
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { UserSchema } from './user.schema';
import { UsersController } from './users.controller';
import { UsersService } from './users.service';

@Module({
  imports: [TypeOrmModule.forFeature([UserSchema])],
  providers: [UsersService],
  controllers: [UsersController],
})
export class UsersModule {}
```

## TypeORM-Transaktionen / TypeORM Transactions

Eine Datenbanktransaktion symbolisiert eine Arbeitseinheit, die innerhalb eines Datenbankverwaltungssystems gegen eine Datenbank ausgeführt wird und auf kohärente und zuverlässige Weise unabhängig von anderen Transaktionen behandelt wird. Eine Transaktion stellt im Allgemeinen jede Änderung in einer Datenbank dar (mehr erfahren).

Es gibt viele verschiedene Strategien zur Handhabung von TypeORM-Transaktionen. Wir empfehlen die Verwendung der QueryRunner-Klasse, da sie die volle Kontrolle über die Transaktion gibt.

Zuerst müssen wir das DataSource-Objekt auf normale Weise in eine Klasse injizieren:

```typescript
@Injectable()
export class UsersService {
  constructor(private dataSource: DataSource) {}
}
```

**TIPP**: Die DataSource-Klasse wird aus dem TypeORM-Paket importiert.

Nun können wir dieses Objekt verwenden, um eine Transaktion zu erstellen:

```typescript
async createMany(users: User[]) {
  const queryRunner = this.dataSource.createQueryRunner();

  await queryRunner.connect();
  await queryRunner.startTransaction();
  try {
    await queryRunner.manager.save(users[0]);
    await queryRunner.manager.save(users[1]);

    await queryRunner.commitTransaction();
  } catch (err) {
    // da wir Fehler haben, lassen Sie uns die Änderungen zurückrollen
    await queryRunner.rollbackTransaction();
  } finally {
    // Sie müssen einen QueryRunner freigeben, der manuell instanziiert wurde
    await queryRunner.release();
  }
}
```

**TIPP**: Beachten Sie, dass die dataSource nur verwendet wird, um den QueryRunner zu erstellen. Um diese Klasse zu testen, müsste jedoch das gesamte DataSource-Objekt gemockt werden (das mehrere Methoden bereitstellt). Daher empfehlen wir die Verwendung einer Hilfsfabrikklasse (z.B. QueryRunnerFactory) und die Definition einer Schnittstelle mit einem begrenzten Satz von Methoden, die erforderlich sind, um Transaktionen aufrechtzuerhalten. Diese Technik macht das Mocken dieser Methoden ziemlich einfach.

Alternativ können Sie den Callback-Stil-Ansatz mit der transaction-Methode des DataSource-Objekts verwenden (mehr lesen).

```typescript
async createMany(users: User[]) {
  await this.dataSource.transaction(async manager => {
    await manager.save(users[0]);
    await manager.save(users[1]);
  });
}
```

## Subscriber / Subscribers

Mit TypeORM-Subscribers können Sie auf bestimmte Entitätsereignisse hören.

```typescript
import {
  DataSource,
  EntitySubscriberInterface,
  EventSubscriber,
  InsertEvent,
} from 'typeorm';
import { User } from './user.entity';

@EventSubscriber()
export class UserSubscriber implements EntitySubscriberInterface<User> {
  constructor(dataSource: DataSource) {
    dataSource.subscribers.push(this);
  }

  listenTo() {
    return User;
  }

  beforeInsert(event: InsertEvent<User>) {
    console.log(`BEFORE USER INSERTED: `, event.entity);
  }
}
```

**WARNUNG**: Event-Subscribers können nicht request-scoped sein.

Fügen Sie nun die UserSubscriber-Klasse zum Provider-Array hinzu:

```typescript
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { User } from './user.entity';
import { UsersController } from './users.controller';
import { UsersService } from './users.service';
import { UserSubscriber } from './user.subscriber';

@Module({
  imports: [TypeOrmModule.forFeature([User])],
  providers: [UsersService, UserSubscriber],
  controllers: [UsersController],
})
export class UsersModule {}
```

**TIPP**: Erfahren Sie mehr über Entitätssubscriber hier.

## Migrationen / Migrations

Migrationen bieten eine Möglichkeit, das Datenbankschema schrittweise zu aktualisieren, um es mit dem Datenmodell der Anwendung synchron zu halten und gleichzeitig vorhandene Daten in der Datenbank zu erhalten. Um Migrationen zu generieren, auszuführen und zurückzusetzen, bietet TypeORM eine eigene CLI.

Migrationsklassen sind von den Quellcodes der Nest-Anwendung getrennt. Ihr Lebenszyklus wird von der TypeORM CLI verwaltet. Daher können Sie Abhängigkeitsinjektion und andere Nest-spezifische Funktionen nicht mit Migrationen nutzen. Um mehr über Migrationen zu erfahren, folgen Sie dem Leitfaden in der TypeORM-Dokumentation.

## Mehrere Datenbanken / Multiple databases

Einige Projekte erfordern mehrere Datenbankverbindungen. Dies kann auch mit diesem Modul erreicht werden. Um mit mehreren Verbindungen zu arbeiten, erstellen Sie zuerst die Verbindungen. In diesem Fall wird das Benennen von Datenquellen obligatorisch.

Angenommen, Sie haben eine Album-Entität, die in einer eigenen Datenbank gespeichert ist.

```typescript
const defaultOptions = {
  type: 'postgres',
  port: 5432,
  username: 'user',
  password: 'password',
  database: 'db',
  synchronize: true,
};

@Module({
  imports: [
    TypeOrmModule.forRoot({
      ...defaultOptions,
      host: 'user_db_host',
      entities: [User],
    }),
    TypeOrmModule.forRoot({
      ...defaultOptions,
      name: 'albumsConnection',
      host: 'album_db_host',
      entities: [Album],
    }),
  ],
})
export class AppModule {}
```

**HINWEIS**: Wenn Sie keinen Namen für eine Datenquelle festlegen, wird ihr Name auf default gesetzt. Beachten Sie bitte, dass Sie nicht mehrere Verbindungen ohne Namen oder mit demselben Namen haben sollten, da sie sonst überschrieben werden.

**HINWEIS**: Wenn Sie TypeOrmModule.forRootAsync verwenden, müssen Sie den Namen der Datenquelle auch außerhalb von useFactory festlegen. Zum Beispiel:

```typescript
TypeOrmModule.forRootAsync({
  name: 'albumsConnection',
  useFactory: ...,
 

 inject: ...,
}),
```

Weitere Details finden Sie in diesem Issue.

Zu diesem Zeitpunkt haben Sie die User- und Album-Entitäten mit ihrer eigenen Datenquelle registriert. Mit dieser Konfiguration müssen Sie der forFeature()-Methode des TypeOrmModule und dem @InjectRepository()-Dekorator mitteilen, welche Datenquelle verwendet werden soll. Wenn Sie keinen Datenquellennamen angeben, wird die Standarddatenquelle verwendet.

```typescript
@Module({
  imports: [
    TypeOrmModule.forFeature([User]),
    TypeOrmModule.forFeature([Album], 'albumsConnection'),
  ],
})
export class AppModule {}
```

Sie können auch die DataSource oder den EntityManager für eine gegebene Datenquelle injizieren:

```typescript
@Injectable()
export class AlbumsService {
  constructor(
    @InjectDataSource('albumsConnection')
    private dataSource: DataSource,
    @InjectEntityManager('albumsConnection')
    private entityManager: EntityManager,
  ) {}
}
```

Es ist auch möglich, eine beliebige DataSource in die Provider zu injizieren:

```typescript
@Module({
  providers: [
    {
      provide: AlbumsService,
      useFactory: (albumsConnection: DataSource) => {
        return new AlbumsService(albumsConnection);
      },
      inject: [getDataSourceToken('albumsConnection')],
    },
  ],
})
export class AlbumsModule {}
```

## Tests / Testing

Wenn es um das Unit-Testing einer Anwendung geht, möchten wir normalerweise vermeiden, eine Datenbankverbindung herzustellen, um unsere Test-Suites unabhängig zu halten und deren Ausführungsprozess so schnell wie möglich zu gestalten. Aber unsere Klassen könnten von Repositories abhängen, die aus der Datenquelle (Verbindung) instanziiert werden. Wie gehen wir damit um? Die Lösung besteht darin, Mock-Repositories zu erstellen. Um dies zu erreichen, richten wir benutzerdefinierte Provider ein. Jedes registrierte Repository wird automatisch durch ein <EntityName>Repository-Token dargestellt, wobei EntityName der Name Ihrer Entitätsklasse ist.

Das @nestjs/typeorm-Paket stellt die Funktion getRepositoryToken() bereit, die basierend auf einer gegebenen Entität ein vorbereitetes Token zurückgibt.

```typescript
@Module({
  providers: [
    UsersService,
    {
      provide: getRepositoryToken(User),
      useValue: mockRepository,
    },
  ],
})
export class UsersModule {}
```

Jetzt wird ein Ersatz-mockRepository als UsersRepository verwendet. Wann immer eine Klasse nach UsersRepository mit einem @InjectRepository()-Dekorator fragt, wird Nest das registrierte mockRepository-Objekt verwenden.

## Asynchrone Konfiguration / Async configuration

Möglicherweise möchten Sie Ihre Repository-Moduloptionen asynchron anstelle von statisch übergeben. In diesem Fall verwenden Sie die forRootAsync()-Methode, die mehrere Möglichkeiten zur Handhabung der asynchronen Konfiguration bietet.

Eine Möglichkeit besteht darin, eine Factory-Funktion zu verwenden:

```typescript
TypeOrmModule.forRootAsync({
  useFactory: () => ({
    type: 'mysql',
    host: 'localhost',
    port: 3306,
    username: 'root',
    password: 'root',
    database: 'test',
    entities: [],
    synchronize: true,
  }),
});
```

Unsere Factory verhält sich wie jeder andere asynchrone Provider (z.B. kann sie asynchron sein und es ist möglich, Abhängigkeiten über inject zu injizieren).

```typescript
TypeOrmModule.forRootAsync({
  imports: [ConfigModule],
  useFactory: (configService: ConfigService) => ({
    type: 'mysql',
    host: configService.get('HOST'),
    port: +configService.get('PORT'),
    username: configService.get('USERNAME'),
    password: configService.get('PASSWORD'),
    database: configService.get('DATABASE'),
    entities: [],
    synchronize: true,
  }),
  inject: [ConfigService],
});
```

Alternativ können Sie die useClass-Syntax verwenden:

```typescript
TypeOrmModule.forRootAsync({
  useClass: TypeOrmConfigService,
});
```

Die obige Konstruktion wird die TypeOrmConfigService innerhalb des TypeOrmModule instanziieren und verwenden, um ein Optionsobjekt bereitzustellen, indem createTypeOrmOptions() aufgerufen wird. Beachten Sie, dass dies bedeutet, dass die TypeOrmConfigService die Schnittstelle TypeOrmOptionsFactory implementieren muss, wie unten gezeigt:

```typescript
@Injectable()
export class TypeOrmConfigService implements TypeOrmOptionsFactory {
  createTypeOrmOptions(): TypeOrmModuleOptions {
    return {
      type: 'mysql',
      host: 'localhost',
      port: 3306,
      username: 'root',
      password: 'root',
      database: 'test',
      entities: [],
      synchronize: true,
    };
  }
}
```

Um die Erstellung von TypeOrmConfigService innerhalb des TypeOrmModule zu verhindern und einen Provider zu verwenden, der aus einem anderen Modul importiert wurde, können Sie die useExisting-Syntax verwenden:

```typescript
TypeOrmModule.forRootAsync({
  imports: [ConfigModule],
  useExisting: ConfigService,
});
```

Diese Konstruktion funktioniert genauso wie useClass mit einem entscheidenden Unterschied - TypeOrmModule wird importierte Module durchsuchen, um einen bestehenden ConfigService wiederzuverwenden, anstatt einen neuen zu instanziieren.

**TIPP**: Stellen Sie sicher, dass die name-Eigenschaft auf derselben Ebene wie die useFactory, useClass oder useValue-Eigenschaft definiert ist. Dies ermöglicht es Nest, die Datenquelle unter dem entsprechenden Injektionstoken richtig zu registrieren.

## Benutzerdefinierte DataSource-Fabrik / Custom DataSource Factory

In Verbindung mit der asynchronen Konfiguration mit useFactory, useClass oder useExisting können Sie optional eine dataSourceFactory-Funktion angeben, die es Ihnen ermöglicht, Ihre eigene TypeORM-Datenquelle bereitzustellen, anstatt TypeOrmModule die Datenquelle erstellen zu lassen.

dataSourceFactory empfängt die während der asynchronen Konfiguration mit useFactory, useClass oder useExisting konfigurierte TypeORM DataSourceOptions und gibt ein Promise zurück, das eine TypeORM DataSource auflöst.

```typescript
TypeOrmModule.forRootAsync({
  imports: [ConfigModule],
  inject: [ConfigService],
  // Verwenden Sie useFactory, useClass oder useExisting,
  // um die DataSourceOptions zu konfigurieren.
  useFactory: (configService: ConfigService) => ({
    type: 'mysql',
    host: configService.get('HOST'),
    port: +configService.get('PORT'),
    username: configService.get('USERNAME'),
    password: configService.get('PASSWORD'),
    database: configService.get('DATABASE'),
    entities: [],
    synchronize: true,
  }),
  // dataSource empfängt die konfigurierten DataSourceOptions
  // und gibt ein Promise<DataSource> zurück.
  dataSourceFactory: async (options) => {
    const dataSource = await new DataSource(options).initialize();
    return dataSource;
  },
});
```

**TIPP**: Die DataSource-Klasse wird aus dem TypeORM-Paket importiert.

## Beispiel / Example

Ein funktionierendes Beispiel ist [hier](https://github.com/nestjs/nest/tree/master/sample/24-typeorm-multi-db) verfügbar.

## Sequelize-Integration / Sequelize Integration

Eine Alternative zur Verwendung von TypeORM ist die Verwendung des Sequelize ORM mit dem Paket @nestjs/sequelize. Zusätzlich nutzen wir das Paket sequelize-typescript, das eine Reihe zusätzlicher Dekoratoren bereitstellt, um Entitäten deklarativ zu definieren.

Um es zu verwenden, installieren wir zunächst die erforderlichen Abhängigkeiten. In diesem Kapitel demonstrieren wir die Verwendung des beliebten MySQL Relational DBMS, aber Sequelize unterstützt viele relationale Datenbanken wie PostgreSQL, MySQL, Microsoft SQL Server, SQLite und MariaDB. Das Verfahren, das wir in diesem Kapitel durchlaufen, ist für jede von Sequelize unterstützte Datenbank dasselbe. Sie müssen lediglich die zugehörigen Client-API-Bibliotheken für Ihre ausgewählte Datenbank installieren.

```bash
$ npm install --save @nestjs/sequelize sequelize sequelize-typescript mysql2
$ npm install --save-dev @types/sequelize
```

Sobald der Installationsprozess abgeschlossen ist, können wir das SequelizeModule in das Stamm-AppModule importieren.

```typescript
// app.module.ts

import { Module } from '@nestjs/common';
import { SequelizeModule } from '@nestjs/sequelize';

@Module({
  imports: [
    SequelizeModule.forRoot({
      dialect: 'mysql',
      host: 'localhost',
      port: 3306,
      username: 'root',
      password: 'root',
      database: 'test',
      models: [],
    }),
  ],
})
export class AppModule {}
```

Die forRoot()-Methode unterstützt alle Konfigurationsparameter, die vom Sequelize-Konstruktor bereitgestellt werden (mehr lesen). Darüber hinaus gibt es mehrere zusätzliche Konfigurationsparameter, die unten beschrieben sind.

| retryAttempts         | Anzahl der Versuche, eine Verbindung zur Datenbank herzustellen (Standard: 10) |
|-----------------------|------------------------------------------------------------------------------|
| retryDelay            | Verzögerung zwischen den Verbindungswiederholungen (ms) (Standard: 3000)     |
| autoLoadModels        | Wenn true, werden Modelle automatisch geladen (Standard: false)              |
| keepConnectionAlive   | Wenn true, wird die Verbindung beim Herunterfahren der Anwendung nicht geschlossen (Standard: false) |
| synchronize           | Wenn true, werden automatisch gelad

ene Modelle synchronisiert (Standard: true) |

Sobald dies erledigt ist, steht das Sequelize-Objekt im gesamten Projekt zur Injektion zur Verfügung (ohne dass Module importiert werden müssen), zum Beispiel:

```typescript
// app.service.ts

import { Injectable } from '@nestjs/common';
import { Sequelize } from 'sequelize-typescript';

@Injectable()
export class AppService {
  constructor(private sequelize: Sequelize) {}
}
```

## Modelle / Models

Sequelize implementiert das Active Record-Muster. Mit diesem Muster verwenden Sie Modellklassen direkt, um mit der Datenbank zu interagieren. Um das Beispiel fortzusetzen, benötigen wir mindestens ein Modell. Definieren wir das User-Modell.

```typescript
// user.model.ts

import { Column, Model, Table } from 'sequelize-typescript';

@Table
export class User extends Model {
  @Column
  firstName: string;

  @Column
  lastName: string;

  @Column({ defaultValue: true })
  isActive: boolean;
}
```

**TIPP**: Erfahren Sie mehr über die verfügbaren Dekoratoren [hier](https://github.com/RobinBuschmann/sequelize-typescript#models).

Die User-Modell-Datei befindet sich im Verzeichnis users. Dieses Verzeichnis enthält alle Dateien, die zum UsersModule gehören. Sie können entscheiden, wo Sie Ihre Modell-Dateien aufbewahren möchten. Wir empfehlen jedoch, sie in der Nähe ihres Domänenbereichs, im entsprechenden Modulverzeichnis, zu erstellen.

Um das User-Modell zu verwenden, müssen wir Sequelize darüber informieren, indem wir es in das models-Array in den Optionen der forRoot()-Methode einfügen:

```typescript
// app.module.ts

import { Module } from '@nestjs/common';
import { SequelizeModule } from '@nestjs/sequelize';
import { User } from './users/user.model';

@Module({
  imports: [
    SequelizeModule.forRoot({
      dialect: 'mysql',
      host: 'localhost',
      port: 3306,
      username: 'root',
      password: 'root',
      database: 'test',
      models: [User],
    }),
  ],
})
export class AppModule {}
```

Schauen wir uns als nächstes das UsersModule an:

```typescript
// users.module.ts

import { Module } from '@nestjs/common';
import { SequelizeModule } from '@nestjs/sequelize';
import { User } from './user.model';
import { UsersController } from './users.controller';
import { UsersService } from './users.service';

@Module({
  imports: [SequelizeModule.forFeature([User])],
  providers: [UsersService],
  controllers: [UsersController],
})
export class UsersModule {}
```

Dieses Modul verwendet die forFeature()-Methode, um zu definieren, welche Modelle im aktuellen Bereich registriert sind. Damit können wir das UserModel in den UsersService injizieren, indem wir den @InjectModel()-Dekorator verwenden:

```typescript
// users.service.ts

import { Injectable } from '@nestjs/common';
import { InjectModel } from '@nestjs/sequelize';
import { User } from './user.model';

@Injectable()
export class UsersService {
  constructor(
    @InjectModel(User)
    private userModel: typeof User,
  ) {}

  async findAll(): Promise<User[]> {
    return this.userModel.findAll();
  }

  findOne(id: string): Promise<User> {
    return this.userModel.findOne({
      where: {
        id,
      },
    });
  }

  async remove(id: string): Promise<void> {
    const user = await this.findOne(id);
    await user.destroy();
  }
}
```

**HINWEIS**: Vergessen Sie nicht, das UsersModule in das Stamm-AppModule zu importieren.

Wenn Sie das Repository außerhalb des Moduls verwenden möchten, das SequelizeModule.forFeature importiert, müssen Sie die von ihm generierten Provider erneut exportieren. Sie können dies tun, indem Sie das gesamte Modul exportieren, wie folgt:

```typescript
// users.module.ts

import { Module } from '@nestjs/common';
import { SequelizeModule } from '@nestjs/sequelize';
import { User } from './user.entity';

@Module({
  imports: [SequelizeModule.forFeature([User])],
  exports: [SequelizeModule]
})
export class UsersModule {}
```

Wenn wir jetzt UsersModule in UserHttpModule importieren, können wir @InjectModel(User) in den Providern des letztgenannten Moduls verwenden.

```typescript
// users-http.module.ts

import { Module } from '@nestjs/common';
import { UsersModule } from './users.module';
import { UsersService } from './users.service';
import { UsersController } from './users.controller';

@Module({
  imports: [UsersModule],
  providers: [UsersService],
  controllers: [UsersController]
})
export class UserHttpModule {}
```

## Beziehungen / Relations

Beziehungen sind Assoziationen, die zwischen zwei oder mehr Tabellen hergestellt werden. Beziehungen basieren auf gemeinsamen Feldern jeder Tabelle, oft unter Einbeziehung von Primär- und Fremdschlüsseln.

Es gibt drei Arten von Beziehungen:

| One-to-one      | Jede Zeile in der Primärtabelle hat genau eine zugehörige Zeile in der Fremdtabelle |
|-----------------|-----------------------------------------------------------------------------------------------------------|
| One-to-many / Many-to-one | Jede Zeile in der Primärtabelle hat eine oder mehrere zugehörige Zeilen in der Fremdtabelle |
| Many-to-many    | Jede Zeile in der Primärtabelle hat viele zugehörige Zeilen in der Fremdtabelle, und jede Zeile in der Fremdtabelle hat viele zugehörige Zeilen in der Primärtabelle |

Um Beziehungen in Modellen zu definieren, verwenden Sie die entsprechenden Dekoratoren. Zum Beispiel, um zu definieren, dass jeder User mehrere Fotos haben kann, verwenden Sie den @HasMany()-Dekorator.

```typescript
// user.model.ts

import { Column, Model, Table, HasMany } from 'sequelize-typescript';
import { Photo } from '../photos/photo.model';

@Table
export class User extends Model {
  @Column
  firstName: string;

  @Column
  lastName: string;

  @Column({ defaultValue: true })
  isActive: boolean;

  @HasMany(() => Photo)
  photos: Photo[];
}
```

**TIPP**: Um mehr über Assoziationen in Sequelize zu erfahren, lesen Sie dieses Kapitel.

## Automatisches Laden von Modellen / Auto-load models

Das manuelle Hinzufügen von Modellen zum models-Array der Verbindungsoptionen kann mühsam sein. Darüber hinaus wird durch das Referenzieren von Modellen aus dem Stamm-Modul die Anwendungsdomäne durchbrochen und Implementierungsdetails in andere Teile der Anwendung geleakt. Um dieses Problem zu lösen, laden Sie Modelle automatisch, indem Sie sowohl die autoLoadModels- als auch die synchronize-Eigenschaften des Konfigurationsobjekts (das in die forRoot()-Methode übergeben wird) auf true setzen, wie unten gezeigt:

```typescript
// app.module.ts

import { Module } from '@nestjs/common';
import { SequelizeModule } from '@nestjs/sequelize';

@Module({
  imports: [
    SequelizeModule.forRoot({
      ...
      autoLoadModels: true,
      synchronize: true,
    }),
  ],
})
export class AppModule {}
```

Mit dieser Option wird jedes Modell, das über die forFeature()-Methode registriert wurde, automatisch zum models-Array des Konfigurationsobjekts hinzugefügt.

**WARNUNG**: Beachten Sie, dass Modelle, die nicht über die forFeature()-Methode registriert wurden, sondern nur vom Modell (über eine Assoziation) referenziert werden, nicht eingeschlossen werden.

## Sequelize-Transaktionen / Sequelize Transactions

Eine Datenbanktransaktion symbolisiert eine Arbeitseinheit, die innerhalb eines Datenbankverwaltungssystems gegen eine Datenbank ausgeführt wird und auf kohärente und zuverlässige Weise unabhängig von anderen Transaktionen behandelt wird. Eine Transaktion stellt im Allgemeinen jede Änderung in einer Datenbank dar (mehr erfahren).

Es gibt viele verschiedene Strategien zur Handhabung von Sequelize-Transaktionen. Unten finden Sie eine Beispielimplementierung einer verwalteten Transaktion (Auto-Callback).

Zuerst müssen wir das Sequelize-Objekt auf normale Weise in eine Klasse injizieren:

```typescript
@Injectable()
export class UsersService {
  constructor(private sequelize: Sequelize) {}
}
```

**TIPP**: Die Sequelize-Klasse wird aus dem Paket sequelize-typescript importiert.

Nun können wir dieses Objekt verwenden, um eine Transaktion zu erstellen:

```typescript
async createMany() {
  try {
    await this.sequelize.transaction(async t => {
      const transactionHost = { transaction: t };

      await this.userModel.create(
          { firstName: 'Abraham', lastName: 'Lincoln' },
          transactionHost,
      );
      await this.userModel.create(
          { firstName: 'John', lastName: 'Boothe' },
          transactionHost,
      );
    });
  } catch (err) {
    // Die Transaktion wurde zurückgesetzt
    // err ist das, was die Promise-Kette zurückgesetzt hat, die an den Transaktions-Callback zurückgegeben wurde
  }
}
```

**TIPP**: Beachten Sie, dass die Sequelize-Instanz nur verwendet wird, um die Transaktion zu starten. Um diese Klasse zu testen, müsste jedoch das gesamte Sequelize-Objekt gemockt werden (das mehrere Methoden bereitstellt). Daher empfehlen wir die Verwendung einer Hilfsfabrikklasse (z.B. TransactionRunner) und die Definition einer Schnittstelle mit einem begrenzten Satz von Methoden, die erforderlich sind, um Transaktionen aufrechtzuerhalten. Diese Technik macht das Mocken dieser Methoden ziemlich einfach.

## Migrationen / Migrations

Migrationen bieten eine Möglichkeit, das Daten

bankschema schrittweise zu aktualisieren, um es mit dem Datenmodell der Anwendung synchron zu halten und gleichzeitig vorhandene Daten in der Datenbank zu erhalten. Um Migrationen zu generieren, auszuführen und zurückzusetzen, bietet Sequelize eine eigene CLI.

Migrationsklassen sind von den Quellcodes der Nest-Anwendung getrennt. Ihr Lebenszyklus wird von der Sequelize CLI verwaltet. Daher können Sie Abhängigkeitsinjektion und andere Nest-spezifische Funktionen nicht mit Migrationen nutzen. Um mehr über Migrationen zu erfahren, folgen Sie dem Leitfaden in der Sequelize-Dokumentation.

## Mehrere Datenbanken / Multiple databases

Einige Projekte erfordern mehrere Datenbankverbindungen. Dies kann auch mit diesem Modul erreicht werden. Um mit mehreren Verbindungen zu arbeiten, erstellen Sie zuerst die Verbindungen. In diesem Fall wird das Benennen von Verbindungen obligatorisch.

Angenommen, Sie haben eine Album-Entität, die in einer eigenen Datenbank gespeichert ist.

```typescript
const defaultOptions = {
  dialect: 'postgres',
  port: 5432,
  username: 'user',
  password: 'password',
  database: 'db',
  synchronize: true,
};

@Module({
  imports: [
    SequelizeModule.forRoot({
      ...defaultOptions,
      host: 'user_db_host',
      models: [User],
    }),
    SequelizeModule.forRoot({
      ...defaultOptions,
      name: 'albumsConnection',
      host: 'album_db_host',
      models: [Album],
    }),
  ],
})
export class AppModule {}
```

**HINWEIS**: Wenn Sie keinen Namen für eine Verbindung festlegen, wird ihr Name auf default gesetzt. Beachten Sie bitte, dass Sie nicht mehrere Verbindungen ohne Namen oder mit demselben Namen haben sollten, da sie sonst überschrieben werden.

Zu diesem Zeitpunkt haben Sie die User- und Album-Modelle mit ihrer eigenen Verbindung registriert. Mit dieser Konfiguration müssen Sie der forFeature()-Methode des SequelizeModule und dem @InjectModel()-Dekorator mitteilen, welche Verbindung verwendet werden soll. Wenn Sie keinen Verbindungsnamen angeben, wird die Standardverbindung verwendet.

```typescript
@Module({
  imports: [
    SequelizeModule.forFeature([User]),
    SequelizeModule.forFeature([Album], 'albumsConnection'),
  ],
})
export class AppModule {}
```

Sie können auch die Sequelize-Instanz für eine gegebene Verbindung injizieren:

```typescript
@Injectable()
export class AlbumsService {
  constructor(
    @InjectConnection('albumsConnection')
    private sequelize: Sequelize,
  ) {}
}
```

Es ist auch möglich, eine beliebige Sequelize-Instanz in die Provider zu injizieren:

```typescript
@Module({
  providers: [
    {
      provide: AlbumsService,
      useFactory: (albumsSequelize: Sequelize) => {
        return new AlbumsService(albumsSequelize);
      },
      inject: [getDataSourceToken('albumsConnection')],
    },
  ],
})
export class AlbumsModule {}
```

## Tests / Testing

Wenn es um das Unit-Testing einer Anwendung geht, möchten wir normalerweise vermeiden, eine Datenbankverbindung herzustellen, um unsere Test-Suites unabhängig zu halten und deren Ausführungsprozess so schnell wie möglich zu gestalten. Aber unsere Klassen könnten von Modellen abhängen, die aus der Verbindungsinstanz instanziiert werden. Wie gehen wir damit um? Die Lösung besteht darin, Mock-Modelle zu erstellen. Um dies zu erreichen, richten wir benutzerdefinierte Provider ein. Jedes registrierte Modell wird automatisch durch ein <ModelName>Model-Token dargestellt, wobei ModelName der Name Ihrer Modellklasse ist.

Das @nestjs/sequelize-Paket stellt die Funktion getModelToken() bereit, die basierend auf einem gegebenen Modell ein vorbereitetes Token zurückgibt.

```typescript
@Module({
  providers: [
    UsersService,
    {
      provide: getModelToken(User),
      useValue: mockModel,
    },
  ],
})
export class UsersModule {}
```

Jetzt wird ein Ersatz-mockModel als UserModel verwendet. Wann immer eine Klasse nach UserModel mit einem @InjectModel()-Dekorator fragt, wird Nest das registrierte mockModel-Objekt verwenden.

## Asynchrone Konfiguration / Async configuration

Möglicherweise möchten Sie Ihre SequelizeModule-Optionen asynchron anstelle von statisch übergeben. In diesem Fall verwenden Sie die forRootAsync()-Methode, die mehrere Möglichkeiten zur Handhabung der asynchronen Konfiguration bietet.

Eine Möglichkeit besteht darin, eine Factory-Funktion zu verwenden:

```typescript
SequelizeModule.forRootAsync({
  useFactory: () => ({
    dialect: 'mysql',
    host: 'localhost',
    port: 3306,
    username: 'root',
    password: 'root',
    database: 'test',
    models: [],
  }),
});
```

Unsere Factory verhält sich wie jeder andere asynchrone Provider (z.B. kann sie asynchron sein und es ist möglich, Abhängigkeiten über inject zu injizieren).

```typescript
SequelizeModule.forRootAsync({
  imports: [ConfigModule],
  useFactory: (configService: ConfigService) => ({
    dialect: 'mysql',
    host: configService.get('HOST'),
    port: +configService.get('PORT'),
    username: configService.get('USERNAME'),
    password: configService.get('PASSWORD'),
    database: configService.get('DATABASE'),
    models: [],
  }),
  inject: [ConfigService],
});
```

Alternativ können Sie die useClass-Syntax verwenden:

```typescript
SequelizeModule.forRootAsync({
  useClass: SequelizeConfigService,
});
```

Die obige Konstruktion wird die SequelizeConfigService innerhalb des SequelizeModule instanziieren und verwenden, um ein Optionsobjekt bereitzustellen, indem createSequelizeOptions() aufgerufen wird. Beachten Sie, dass dies bedeutet, dass die SequelizeConfigService die Schnittstelle SequelizeOptionsFactory implementieren muss, wie unten gezeigt:

```typescript
@Injectable()
class SequelizeConfigService implements SequelizeOptionsFactory {
  createSequelizeOptions(): SequelizeModuleOptions {
    return {
      dialect: 'mysql',
      host: 'localhost',
      port: 3306,
      username: 'root',
      password: 'root',
      database: 'test',
      models: [],
    };
  }
}
```

Um die Erstellung von SequelizeConfigService innerhalb des SequelizeModule zu verhindern und einen Provider zu verwenden, der aus einem anderen Modul importiert wurde, können Sie die useExisting-Syntax verwenden:

```typescript
SequelizeModule.forRootAsync({
  imports: [ConfigModule],
  useExisting: ConfigService,
});
```

Diese Konstruktion funktioniert genauso wie useClass mit einem entscheidenden Unterschied - SequelizeModule wird importierte Module durchsuchen, um einen bestehenden ConfigService wiederzuverwenden, anstatt einen neuen zu instanziieren.

## Beispiel / Example

Ein funktionierendes Beispiel ist [hier](https://github.com/nestjs/nest/tree/master/sample/24-typeorm-multi-db) verfügbar.

