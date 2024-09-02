Hier ist die vollständige Implementierung einer **NestJS-Backend-REST-API** unter Verwendung von **CQRS** (Command Query Responsibility Segregation) und **MongoDB** mit **Mongoose**. Die API unterstützt die folgenden CRUD-Operationen für Benutzer: Erstellen (`POST`), Abrufen eines Benutzers (`GET /:id`), Abrufen aller Benutzer (`GET`), Aktualisieren (`PATCH /:id`) und Löschen (`DELETE /:id`). Alle Operationen sind gemäß den Prinzipien von CQRS in Lese- und Schreiboperationen getrennt. Der Code enthält zudem eine ausführliche Dokumentation und eine Schnellstart-Anleitung in der `README.md`.

### Projektstruktur

Die Projektstruktur sieht wie folgt aus:

```
nestjs-cqrs-mongodb/
│
├── src/
│   ├── user/
│   │   ├── commands/
│   │   │   ├── handlers/
│   │   │   │   ├── create-user.handler.ts
│   │   │   │   ├── update-user.handler.ts
│   │   │   │   └── delete-user.handler.ts
│   │   │   └── create-user.command.ts
│   │   │   └── update-user.command.ts
│   │   │   └── delete-user.command.ts
│   │   ├── queries/
│   │   │   ├── handlers/
│   │   │   │   ├── get-user.handler.ts
│   │   │   │   └── get-all-users.handler.ts
│   │   │   └── get-user.query.ts
│   │   │   └── get-all-users.query.ts
│   │   ├── controllers/
│   │   │   └── user.controller.ts
│   │   ├── services/
│   │   │   └── user.service.ts
│   │   ├── repositories/
│   │   │   └── user.repository.ts
│   │   ├── models/
│   │   │   └── user.schema.ts
│   │   └── user.module.ts
│   ├── app.module.ts
│   └── main.ts
├── .env
├── package.json
└── README.md
```

### Installationsanweisungen

Installiere die erforderlichen Pakete:

```bash
npm install --save @nestjs/core @nestjs/common @nestjs/mongoose @nestjs/cqrs mongoose
npm install --save-dev @types/mongoose
```

### Implementierung

#### `src/app.module.ts`

```typescript
import { Module } from '@nestjs/common';
import { MongooseModule } from '@nestjs/mongoose';
import { UserModule } from './user/user.module';

@Module({
  imports: [
    MongooseModule.forRoot(process.env.MONGODB_URI), // MongoDB-URI aus .env-Datei laden
    UserModule,
  ],
})
export class AppModule {}
```

#### `src/main.ts`

```typescript
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(3000);
}
bootstrap();
```

#### `src/user/user.module.ts`

```typescript
import { Module } from '@nestjs/common';
import { MongooseModule } from '@nestjs/mongoose';
import { CqrsModule } from '@nestjs/cqrs';
import { UserSchema } from './models/user.schema';
import { UserService } from './services/user.service';
import { UserRepository } from './repositories/user.repository';
import { CreateUserHandler } from './commands/handlers/create-user.handler';
import { UpdateUserHandler } from './commands/handlers/update-user.handler';
import { DeleteUserHandler } from './commands/handlers/delete-user.handler';
import { GetUserHandler } from './queries/handlers/get-user.handler';
import { GetAllUsersHandler } from './queries/handlers/get-all-users.handler';
import { UserController } from './controllers/user.controller';

@Module({
  imports: [
    CqrsModule,
    MongooseModule.forFeature([{ name: 'User', schema: UserSchema }]),
  ],
  controllers: [UserController],
  providers: [
    UserService,
    UserRepository,
    CreateUserHandler,
    UpdateUserHandler,
    DeleteUserHandler,
    GetUserHandler,
    GetAllUsersHandler,
  ],
})
export class UserModule {}
```

#### `src/user/models/user.schema.ts`

```typescript
import { Schema, Document } from 'mongoose';

// Definition des User-Interfaces, das das Benutzerobjekt beschreibt
export interface User extends Document {
  email: string;
  firstName: string;
  lastName: string;
}

// Mongoose-Schema für das Benutzerobjekt
export const UserSchema = new Schema({
  email: { type: String, required: true, unique: true },
  firstName: { type: String, required: true },
  lastName: { type: String, required: true },
});
```

#### `src/user/repositories/user.repository.ts`

```typescript
import { Model } from 'mongoose';
import { InjectModel } from '@nestjs/mongoose';
import { Injectable } from '@nestjs/common';
import { User } from '../models/user.schema';

// Repository zur Verwaltung der Benutzer in der Datenbank
@Injectable()
export class UserRepository {
  constructor(@InjectModel('User') private readonly userModel: Model<User>) {}

  async create(user: User): Promise<User> {
    const newUser = new this.userModel(user);
    return newUser.save();
  }

  async findById(id: string): Promise<User | null> {
    return this.userModel.findById(id).exec();
  }

  async findAll(): Promise<User[]> {
    return this.userModel.find().exec();
  }

  async update(id: string, user: Partial<User>): Promise<User | null> {
    return this.userModel.findByIdAndUpdate(id, user, { new: true }).exec();
  }

  async delete(id: string): Promise<User | null> {
    return this.userModel.findByIdAndDelete(id).exec();
  }
}
```

#### `src/user/commands/create-user.command.ts`

```typescript
// Kommando zur Erstellung eines neuen Benutzers
export class CreateUserCommand {
  constructor(
    public readonly email: string,
    public readonly firstName: string,
    public readonly lastName: string,
  ) {}
}
```

#### `src/user/commands/handlers/create-user.handler.ts`

```typescript
import { CommandHandler, ICommandHandler } from '@nestjs/cqrs';
import { CreateUserCommand } from '../create-user.command';
import { UserRepository } from '../../repositories/user.repository';

// Handler für das CreateUser-Kommando
@CommandHandler(CreateUserCommand)
export class CreateUserHandler implements ICommandHandler<CreateUserCommand> {
  constructor(private readonly userRepository: UserRepository) {}

  async execute(command: CreateUserCommand) {
    const { email, firstName, lastName } = command;
    return this.userRepository.create({ email, firstName, lastName });
  }
}
```

#### `src/user/commands/update-user.command.ts`

```typescript
// Kommando zum Aktualisieren eines Benutzers
export class UpdateUserCommand {
  constructor(
    public readonly id: string,
    public readonly email?: string,
    public readonly firstName?: string,
    public readonly lastName?: string,
  ) {}
}
```

#### `src/user/commands/handlers/update-user.handler.ts`

```typescript
import { CommandHandler, ICommandHandler } from '@nestjs/cqrs';
import { UpdateUserCommand } from '../update-user.command';
import { UserRepository } from '../../repositories/user.repository';

// Handler für das UpdateUser-Kommando
@CommandHandler(UpdateUserCommand)
export class UpdateUserHandler implements ICommandHandler<UpdateUserCommand> {
  constructor(private readonly userRepository: UserRepository) {}

  async execute(command: UpdateUserCommand) {
    const { id, email, firstName, lastName } = command;
    return this.userRepository.update(id, { email, firstName, lastName });
  }
}
```

#### `src/user/commands/delete-user.command.ts`

```typescript
// Kommando zum Löschen eines Benutzers
export class DeleteUserCommand {
  constructor(public readonly id: string) {}
}
```

#### `src/user/commands/handlers/delete-user.handler.ts`

```typescript
import { CommandHandler, ICommandHandler } from '@nestjs/cqrs';
import { DeleteUserCommand } from '../delete-user.command';
import { UserRepository } from '../../repositories/user.repository';

// Handler für das DeleteUser-Kommando
@CommandHandler(DeleteUserCommand)
export class DeleteUserHandler implements ICommandHandler<DeleteUserCommand> {
  constructor(private readonly userRepository: UserRepository) {}

  async execute(command: DeleteUserCommand) {
    const { id } = command;
    return this.userRepository.delete(id);
  }
}
```

#### `src/user/queries/get-user.query.ts`

```typescript
// Query zum Abrufen eines Benutzers anhand seiner ID
export class GetUserQuery {
  constructor(public readonly id: string) {}
}
```

#### `src/user/queries/handlers/get-user.handler.ts`

```typescript
import { IQueryHandler, QueryHandler } from '@nestjs/cqrs';
import { GetUserQuery } from '../get-user.query';
import { UserRepository } from '../../repositories/user.repository';

// Handler für das GetUser-Query
@QueryHandler(GetUserQuery)
export class GetUserHandler implements IQueryHandler<GetUserQuery> {
  constructor(private readonly userRepository: UserRepository) {}

  async execute(query: GetUserQuery) {
    const { id } = query;
    return this.userRepository.findById(id);
  }
}
```

#### `src/user/queries/get-all-users.query.ts`

```typescript
// Query zum Abrufen aller Benutzer
export class GetAllUsersQuery {}
```

#### `src/user/queries/handlers/get-all-users.handler.ts`

```typescript
import { IQueryHandler, QueryHandler } from '@nestjs/cqrs';
import { GetAllUsersQuery } from '../get-all-users.query';
import { UserRepository } from '../../repositories/user.repository';

// Handler für das GetAllUsers-Query
@QueryHandler(GetAllUsersQuery)
export class GetAllUsersHandler implements IQueryHandler<GetAllUsersQuery> {
  constructor(private readonly userRepository: UserRepository) {}

  async execute(query: GetAllUsersQuery) {
    return this.userRepository.findAll();
  }
}
```

#### `src/user/controllers/user.controller.ts`

```typescript
import { Controller, Post, Body, Get, Param, Patch, Delete } from '@nestjs/common';
import { CommandBus, QueryBus } from '@nestjs/cqrs';
import { CreateUserCommand } from '../commands/create-user.command';
import { UpdateUserCommand } from '../commands/update-user.command';
import { DeleteUserCommand } from '../commands/delete-user.command';
import { GetUserQuery } from '../queries/get-user.query';
import { GetAllUsersQuery } from '../queries/get-all-users.query';

// Controller zur Verwaltung der Benutzerendpunkte
@Controller('users')
export class UserController {
  constructor(
    private readonly commandBus: CommandBus,
    private readonly queryBus: QueryBus,
  ) {}

  /**
   * Erstelle einen neuen Benutzer
   * @param email E-Mail-Adresse des Benutzers
   * @param firstName Vorname des Benutzers
   * @param lastName Nachname des Benutzers
   * @returns Der erstellte Benutzer
   */
  @Post()
  async createUser(
    @Body('email') email: string,
    @Body('firstName') firstName: string,
    @Body('lastName') lastName: string,
  ) {
    return this.commandBus.execute(new CreateUserCommand(email, firstName, lastName));
  }

  /**
   * Hole einen Benutzer anhand seiner ID
   * @param id ID des Benutzers
   * @returns Der Benutzer mit der angegebenen ID
   */
  @Get(':id')
  async getUser(@Param('id') id: string) {
    return this.queryBus.execute(new GetUserQuery(id));
  }

  /**
   * Hole alle Benutzer
   * @returns Eine Liste aller Benutzer
   */
  @Get()
  async getAllUsers() {
    return this.queryBus.execute(new GetAllUsersQuery());
  }

  /**
   * Aktualisiere die Eigenschaften eines Benutzers
   * @param id ID des Benutzers
   * @param email Neue E-Mail-Adresse des Benutzers (optional)
   * @param firstName Neuer Vorname des Benutzers (optional)
   * @param lastName Neuer Nachname des Benutzers (optional)
   * @returns Der aktualisierte Benutzer
   */
  @Patch(':id')
  async updateUser(
    @Param('id') id: string,
    @Body('email') email?: string,
    @Body('firstName') firstName?: string,
    @Body('lastName') lastName?: string,
  ) {
    return this.commandBus.execute(new UpdateUserCommand(id, email, firstName, lastName));
  }

  /**
   * Lösche einen Benutzer anhand seiner ID
   * @param id ID des Benutzers
   * @returns Bestätigung des Löschvorgangs
   */
  @Delete(':id')
  async deleteUser(@Param('id') id: string) {
    return this.commandBus.execute(new DeleteUserCommand(id));
  }
}
```

### Schnellstart-Anleitung in `README.md`

```markdown
# NestJS CQRS MongoDB REST API

Eine REST-API für die Verwaltung von Benutzern mit NestJS, CQRS und MongoDB.

## Voraussetzungen

- Node.js (>= 14.x)
- npm (>= 6.x)
- MongoDB (lokal oder in der Cloud)

## Installation

```bash
git clone <repository-url>
cd nestjs-cqrs-mongodb
npm install
```

## Konfiguration

Erstelle eine `.env` Datei im Stammverzeichnis und füge die MongoDB-Verbindung hinzu:

```bash
MONGODB_URI=mongodb://localhost:27017/nestjs-cqrs
```

## Starten des Servers

```bash
npm run start
```

Der Server läuft nun unter [http://localhost:3000](http://localhost:3000).

## Endpunkte

- `POST /users` - Erstelle einen neuen Benutzer
- `GET /users/:id` - Hole einen Benutzer anhand seiner ID
- `GET /users` - Hole alle Benutzer
- `PATCH /users/:id` - Aktualisiere die Eigenschaften eines Benutzers
- `DELETE /users/:id` - Lösche einen Benutzer

## Lizenz

MIT
```

### Fazit

Die oben genannte Implementierung umfasst alle CRUD-Operationen für eine Benutzerverwaltung und ist gemäß den Prinzipien von CQRS in Lese- und Schreiboperationen getrennt. Der Code ist vollständig dokumentiert und bietet eine klare Struktur, die die Wartbarkeit und Erweiterbarkeit der Anwendung gewährleistet.