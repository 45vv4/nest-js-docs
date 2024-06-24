# Authentifizierung / Authentication

Authentifizierung ist ein wesentlicher Bestandteil der meisten Anwendungen. Es gibt viele verschiedene Ansätze und Strategien zur Handhabung der Authentifizierung. Der gewählte Ansatz hängt von den speziellen Anforderungen des jeweiligen Projekts ab. Dieses Kapitel stellt mehrere Authentifizierungsansätze vor, die an eine Vielzahl unterschiedlicher Anforderungen angepasst werden können.

Lassen Sie uns unsere Anforderungen konkretisieren. In diesem Anwendungsfall beginnen die Clients mit der Authentifizierung mittels Benutzername und Passwort. Nach der Authentifizierung stellt der Server ein JWT aus, das als Bearer-Token in einem Autorisierungsheader bei nachfolgenden Anfragen gesendet werden kann, um die Authentifizierung nachzuweisen. Wir erstellen außerdem eine geschützte Route, die nur für Anfragen zugänglich ist, die ein gültiges JWT enthalten.

Wir beginnen mit der ersten Anforderung: der Authentifizierung eines Benutzers. Anschließend erweitern wir dies durch das Ausstellen eines JWT. Schließlich erstellen wir eine geschützte Route, die bei der Anfrage nach einem gültigen JWT sucht.

## Erstellen eines Authentifizierungsmoduls / Creating an authentication module

Wir beginnen mit der Generierung eines AuthModule und darin eines AuthService und eines AuthController. Wir verwenden den AuthService, um die Authentifizierungslogik zu implementieren, und den AuthController, um die Authentifizierungspunkte offenzulegen.

```sh
$ nest g module auth
$ nest g controller auth
$ nest g service auth
```

Während wir den AuthService implementieren, werden wir feststellen, dass es nützlich ist, Benutzeroperationen in einem UsersService zu kapseln. Lassen Sie uns also jetzt dieses Modul und diesen Dienst generieren:

```sh
$ nest g module users
$ nest g service users
```

Ersetzen Sie den Standardinhalt dieser generierten Dateien wie unten gezeigt. Für unsere Beispiel-App verwaltet der UsersService einfach eine hartcodierte In-Memory-Liste von Benutzern und eine find-Methode, um einen Benutzer nach Benutzernamen abzurufen. In einer realen App würden Sie hier Ihr Benutzermodell und Ihre Persistenzschicht aufbauen und die Bibliothek Ihrer Wahl verwenden (z.B. TypeORM, Sequelize, Mongoose usw.).

**users/users.service.ts**

```typescript
import { Injectable } from '@nestjs/common';

// Dies sollte eine echte Klasse/Schnittstelle sein, die eine Benutzerentität darstellt
export type User = any;

@Injectable()
export class UsersService {
  private readonly users = [
    {
      userId: 1,
      username: 'john',
      password: 'changeme',
    },
    {
      userId: 2,
      username: 'maria',
      password: 'guess',
    },
  ];

  async findOne(username: string): Promise<User | undefined> {
    return this.users.find(user => user.username === username);
  }
}
```

Im UsersModule ist die einzige erforderliche Änderung, den UsersService zum Exports-Array des @Module-Dekorators hinzuzufügen, damit er außerhalb dieses Moduls sichtbar ist (wir werden ihn bald in unserem AuthService verwenden).

**users/users.module.ts**

```typescript
import { Module } from '@nestjs/common';
import { UsersService } from './users.service';

@Module({
  providers: [UsersService],
  exports: [UsersService],
})
export class UsersModule {}
```

## Implementierung des "Sign in"-Endpunkts / Implementing the "Sign in" endpoint

Unser AuthService hat die Aufgabe, einen Benutzer abzurufen und das Passwort zu überprüfen. Wir erstellen eine signIn() Methode zu diesem Zweck. Im untenstehenden Code verwenden wir einen praktischen ES6-Spread-Operator, um die Passwort-Eigenschaft aus dem Benutzerobjekt zu entfernen, bevor wir es zurückgeben. Dies ist eine gängige Praxis beim Zurückgeben von Benutzerobjekten, da Sie sensible Felder wie Passwörter oder andere Sicherheitsschlüssel nicht preisgeben möchten.

**auth/auth.service.ts**

```typescript
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { UsersService } from '../users/users.service';

@Injectable()
export class AuthService {
  constructor(private usersService: UsersService) {}

  async signIn(username: string, pass: string): Promise<any> {
    const user = await this.usersService.findOne(username);
    if (user?.password !== pass) {
      throw new UnauthorizedException();
    }
    const { password, ...result } = user;
    // TODO: Ein JWT generieren und hier zurückgeben
    // anstelle des Benutzerobjekts
    return result;
  }
}
```

**WARNUNG**
Natürlich würden Sie in einer realen Anwendung kein Passwort im Klartext speichern. Stattdessen würden Sie eine Bibliothek wie bcrypt verwenden, mit einem gesalzenen Einweg-Hash-Algorithmus. Mit diesem Ansatz würden Sie nur gehashte Passwörter speichern und dann das gespeicherte Passwort mit einer gehashten Version des eingehenden Passworts vergleichen, sodass Sie niemals Benutzerpasswörter im Klartext speichern oder preisgeben. Um unsere Beispiel-App einfach zu halten, verletzen wir dieses absolute Gebot und verwenden Klartext. Tun Sie dies nicht in Ihrer realen App!

Nun aktualisieren wir unser AuthModule, um das UsersModule zu importieren.

**auth/auth.module.ts**

```typescript
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { AuthController } from './auth.controller';
import { UsersModule } from '../users/users.module';

@Module({
  imports: [UsersModule],
  providers: [AuthService],
  controllers: [AuthController],
})
export class AuthModule {}
```

Damit dies erledigt ist, öffnen wir den AuthController und fügen ihm eine signIn() Methode hinzu. Diese Methode wird vom Client aufgerufen, um einen Benutzer zu authentifizieren. Sie erhält den Benutzernamen und das Passwort im Anfragekörper und gibt ein JWT-Token zurück, wenn der Benutzer authentifiziert ist.

**auth/auth.controller.ts**

```typescript
import { Body, Controller, Post, HttpCode, HttpStatus } from '@nestjs/common';
import { AuthService } from './auth.service';

@Controller('auth')
export class AuthController {
  constructor(private authService: AuthService) {}

  @HttpCode(HttpStatus.OK)
  @Post('login')
  signIn(@Body() signInDto: Record<string, any>) {
    return this.authService.signIn(signInDto.username, signInDto.password);
  }
}
```

**HINWEIS**
Idealerweise sollten wir anstelle der Verwendung des Record<string, any> Typs eine DTO-Klasse verwenden, um die Form des Anfragekörpers zu definieren. Weitere Informationen finden Sie im Validierungskapitel.

## JWT-Token / JWT token

Wir sind bereit, zum JWT-Teil unseres Authentifizierungssystems überzugehen. Lassen Sie uns unsere Anforderungen überprüfen und verfeinern:

- Benutzern ermöglichen, sich mit Benutzername/Passwort zu authentifizieren und ein JWT für die Verwendung in nachfolgenden Anrufen zu geschützten API-Endpunkten zurückzugeben. Wir sind auf dem besten Weg, diese Anforderung zu erfüllen. Um sie abzuschließen, müssen wir den Code schreiben, der ein JWT ausstellt.
- API-Routen erstellen, die basierend auf dem Vorhandensein eines gültigen JWT als Bearer-Token geschützt sind.

Wir müssen ein zusätzliches Paket installieren, um unsere JWT-Anforderungen zu unterstützen:

```sh
$ npm install --save @nestjs/jwt
```

**HINWEIS**
Das @nestjs/jwt Paket (siehe mehr [hier](https://github.com/nestjs/jwt)) ist ein Dienstprogramm-Paket, das bei der JWT-Manipulation hilft. Dies umfasst das Generieren und Verifizieren von JWT-Token.

Um unsere Dienste sauber modularisiert zu halten, werden wir das Generieren des JWT im authService behandeln. Öffnen Sie die Datei auth.service.ts im auth-Ordner, injizieren Sie den JwtService und aktualisieren Sie die signIn-Methode, um ein JWT-Token zu generieren, wie unten gezeigt:

**auth/auth.service.ts**

```typescript
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { UsersService } from '../users/users.service';
import { JwtService } from '@nestjs/jwt';

@Injectable()
export class AuthService {
  constructor(
    private usersService: UsersService,
    private jwtService: JwtService
  ) {}

  async signIn(
    username: string,
    pass: string,
  ): Promise<{ access_token: string }> {
    const user = await this.usersService.findOne(username);
    if (user?.password !== pass) {
      throw new UnauthorizedException();
    }
    const payload = { sub: user.userId, username: user.username };
    return {
      access_token: await this.jwtService.signAsync(payload),
    };
  }
}
```

Wir verwenden die @nestjs/jwt-Bibliothek, die eine signAsync() Funktion bereitstellt, um unser JWT aus einem Teil der Benutzereigenschaften zu generieren, die wir dann als einfaches Objekt mit einer einzigen access_token-Eigenschaft zurückgeben. Hinweis: Wir wählen eine Eigenschaftsbezeichnung von sub, um unseren userId-Wert zu halten, um mit den JWT-Standards übereinzustimmen.

Wir müssen nun das AuthModule aktualisieren, um die neuen Abhängigkeiten zu importieren und das JwtModule zu konfigurieren.

Zuerst erstellen Sie constants.ts im auth-Ordner und fügen den folgenden Code hinzu:

**auth/constants.ts**

```typescript
export const jwtConstants = {
  secret: 'VERWENDEN SIE DIESE WERT NICHT. ERSTELLEN SIE STATTESSEN EIN GEHEIMNISVOLLEN SCHLÜSSEL UND

 BEWAHREN SIE IHN SICHER AUßERHALB DES QUELLCODES AUF.',
};
```

Wir verwenden dies, um unseren Schlüssel zwischen den JWT-Signierungs- und Verifizierungsschritten zu teilen.

**WARNUNG**
Geben Sie diesen Schlüssel nicht öffentlich preis. Wir haben dies hier getan, um klar zu machen, was der Code tut, aber in einem Produktionssystem müssen Sie diesen Schlüssel durch geeignete Maßnahmen wie einen Geheimnisspeicher, eine Umgebungsvariable oder einen Konfigurationsdienst schützen.

Nun öffnen Sie auth.module.ts im auth-Ordner und aktualisieren es, damit es wie folgt aussieht:

**auth/auth.module.ts**

```typescript
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { UsersModule } from '../users/users.module';
import { JwtModule } from '@nestjs/jwt';
import { AuthController } from './auth.controller';
import { jwtConstants } from './constants';

@Module({
  imports: [
    UsersModule,
    JwtModule.register({
      global: true,
      secret: jwtConstants.secret,
      signOptions: { expiresIn: '60s' },
    }),
  ],
  providers: [AuthService],
  controllers: [AuthController],
  exports: [AuthService],
})
export class AuthModule {}
```

**HINWEIS**
Wir registrieren das JwtModule als global, um uns die Arbeit zu erleichtern. Das bedeutet, dass wir das JwtModule nirgendwo sonst in unserer Anwendung importieren müssen.

Wir konfigurieren das JwtModule mit register() und übergeben ein Konfigurationsobjekt. Weitere Informationen zum Nest JwtModule finden Sie [hier](https://github.com/nestjs/jwt/blob/master/README.md) und zu den verfügbaren Konfigurationsoptionen [hier](https://github.com/auth0/node-jsonwebtoken#usage).

Lassen Sie uns unsere Routen erneut mit cURL testen. Sie können mit einem der Benutzerobjekte testen, die im UsersService hartcodiert sind.

```sh
$ # POST zu /auth/login
$ curl -X POST http://localhost:3000/auth/login -d '{"username": "john", "password": "changeme"}' -H "Content-Type: application/json"
{"access_token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."}
$ # Hinweis: Oben JWT abgeschnitten
```

## Implementierung des Authentifizierungsschutzes / Implementing the authentication guard

Wir können nun unsere letzte Anforderung angehen: Endpunkte zu schützen, indem wir verlangen, dass ein gültiges JWT in der Anfrage vorhanden ist. Wir werden dies tun, indem wir einen AuthGuard erstellen, den wir zum Schutz unserer Routen verwenden können.

**auth/auth.guard.ts**

```typescript
import {
  CanActivate,
  ExecutionContext,
  Injectable,
  UnauthorizedException,
} from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';
import { jwtConstants } from './constants';
import { Request } from 'express';

@Injectable()
export class AuthGuard implements CanActivate {
  constructor(private jwtService: JwtService) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest();
    const token = this.extractTokenFromHeader(request);
    if (!token) {
      throw new UnauthorizedException();
    }
    try {
      const payload = await this.jwtService.verifyAsync(
        token,
        {
          secret: jwtConstants.secret
        }
      );
      // 💡 Wir weisen das Payload dem Anforderungsobjekt hier zu
      // sodass wir darauf in unseren Routenhandlern zugreifen können
      request['user'] = payload;
    } catch {
      throw new UnauthorizedException();
    }
    return true;
  }

  private extractTokenFromHeader(request: Request): string | undefined {
    const [type, token] = request.headers.authorization?.split(' ') ?? [];
    return type === 'Bearer' ? token : undefined;
  }
}
```

Wir können nun unsere geschützte Route implementieren und unseren AuthGuard registrieren, um sie zu schützen.

Öffnen Sie die Datei auth.controller.ts und aktualisieren Sie sie wie unten gezeigt:

**auth.controller.ts**

```typescript
import {
  Body,
  Controller,
  Get,
  HttpCode,
  HttpStatus,
  Post,
  Request,
  UseGuards
} from '@nestjs/common';
import { AuthGuard } from './auth.guard';
import { AuthService } from './auth.service';

@Controller('auth')
export class AuthController {
  constructor(private authService: AuthService) {}

  @HttpCode(HttpStatus.OK)
  @Post('login')
  signIn(@Body() signInDto: Record<string, any>) {
    return this.authService.signIn(signInDto.username, signInDto.password);
  }

  @UseGuards(AuthGuard)
  @Get('profile')
  getProfile(@Request() req) {
    return req.user;
  }
}
```

Wir wenden den AuthGuard, den wir gerade erstellt haben, auf die Route GET /profile an, sodass sie geschützt wird.

Stellen Sie sicher, dass die App läuft, und testen Sie die Routen erneut mit cURL.

```sh
$ # GET /profile
$ curl http://localhost:3000/auth/profile
{"statusCode":401,"message":"Unauthorized"}

$ # POST /auth/login
$ curl -X POST http://localhost:3000/auth/login -d '{"username": "john", "password": "changeme"}' -H "Content-Type: application/json"
{"access_token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2Vybm..."}

$ # GET /profile mit dem access_token aus dem vorherigen Schritt als Bearer-Code
$ curl http://localhost:3000/auth/profile -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2Vybm..."
{"sub":1,"username":"john","iat":...,"exp":...}
```

Beachten Sie, dass wir im AuthModule das JWT so konfiguriert haben, dass es nach 60 Sekunden abläuft. Dies ist eine zu kurze Ablaufzeit, und die Behandlung der Details der Token-Ablauf- und Erneuerung liegt außerhalb des Rahmens dieses Artikels. Wir haben dies jedoch gewählt, um eine wichtige Eigenschaft von JWTs zu demonstrieren. Wenn Sie 60 Sekunden nach der Authentifizierung warten, bevor Sie einen GET /auth/profile-Anfrage versuchen, erhalten Sie eine 401 Unauthorized-Antwort. Dies liegt daran, dass @nestjs/jwt das JWT automatisch auf seine Ablaufzeit überprüft und Ihnen somit die Mühe erspart, dies in Ihrer Anwendung zu tun.

Wir haben nun unsere JWT-Authentifizierungsimplementierung abgeschlossen. JavaScript-Clients (wie Angular/React/Vue) und andere JavaScript-Apps können sich jetzt authentifizieren und sicher mit unserem API-Server kommunizieren.

## Authentifizierung global aktivieren / Enable authentication globally

Wenn der Großteil Ihrer Endpunkte standardmäßig geschützt sein soll, können Sie den Authentifizierungsschutz als globalen Schutz registrieren und anstelle der Verwendung des @UseGuards()-Dekorators auf jedem Controller einfach kennzeichnen, welche Routen öffentlich sein sollen.

Registrieren Sie zunächst den AuthGuard als globalen Schutz mithilfe der folgenden Konstruktion (in einem beliebigen Modul, z.B. im AuthModule):

```typescript
providers: [
  {
    provide: APP_GUARD,
    useClass: AuthGuard,
  },
],
```

Nest bindet den AuthGuard automatisch an alle Endpunkte.

Nun müssen wir einen Mechanismus bereitstellen, um Routen als öffentlich zu deklarieren. Dafür erstellen wir einen benutzerdefinierten Dekorator mithilfe der SetMetadata-Dekorator-Fabrikfunktion.

```typescript
import { SetMetadata } from '@nestjs/common';

export const IS_PUBLIC_KEY = 'isPublic';
export const Public = () => SetMetadata(IS_PUBLIC_KEY, true);
```

In der obigen Datei haben wir zwei Konstanten exportiert. Eine ist unser Metadaten-Schlüssel namens IS_PUBLIC_KEY, und die andere ist unser neuer Dekorator selbst, den wir Public nennen (Sie können ihn alternativ SkipAuth oder AllowAnon nennen, was immer zu Ihrem Projekt passt).

Da wir nun einen benutzerdefinierten @Public()-Dekorator haben, können wir ihn verwenden, um jede Methode wie folgt zu dekorieren:

```typescript
@Public()
@Get()
findAll() {
  return [];
}
```

Zuletzt müssen wir den AuthGuard so ändern, dass er true zurückgibt, wenn die "isPublic"-Metadaten gefunden werden. Dafür verwenden wir die Reflector-Klasse (lesen Sie mehr [hier](https://docs.nestjs.com/guards#putting-it-all-together)).

```typescript
@Injectable()
export class AuthGuard implements CanActivate {
  constructor(private jwtService: JwtService, private reflector: Reflector) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);
    if (isPublic) {
      // 💡 Siehe diese Bedingung
      return true;
    }

    const request = context.switchToHttp().getRequest();
    const token = this.extractTokenFromHeader(request);
    if (!token) {
      throw new UnauthorizedException();
    }
    try {
      const payload = await this.jwtService.verifyAsync(token, {
        secret: jwtConstants.secret,
      });
      // 💡 Wir weisen das Payload dem Anforderungsobjekt hier zu
      // sodass wir darauf in unseren Routenhandlern zugreifen können
      request['user'] = payload;
    } catch {
      throw new UnauthorizedException();
    }
    return true;
  }



  private extractTokenFromHeader(request: Request): string | undefined {
    const [type, token] = request.headers.authorization?.split(' ') ?? [];
    return type === 'Bearer' ? token : undefined;
  }
}
```

## Passport-Integration / Passport integration

Passport ist die beliebteste node.js-Authentifizierungsbibliothek, bekannt in der Community und erfolgreich in vielen Produktionsanwendungen verwendet. Es ist einfach, diese Bibliothek mit einer Nest-Anwendung unter Verwendung des @nestjs/passport-Moduls zu integrieren.

Um zu erfahren, wie Sie Passport mit NestJS integrieren können, sehen Sie sich dieses Kapitel an.

### Beispiel / Example

Sie finden eine vollständige Version des Codes in diesem Kapitel [hier](https://github.com/nestjs/nest/tree/master/sample/19-auth-jwt).
