## Passport (Authentifizierung) / Passport (authentication)

**Passport** ist die bekannteste Authentifizierungsbibliothek für Node.js. Sie ist in der Community weit verbreitet und wird erfolgreich in vielen produktiven Anwendungen eingesetzt. Es ist einfach, diese Bibliothek mithilfe des `@nestjs/passport`-Moduls in eine NestJS-Anwendung zu integrieren. Auf einer höheren Ebene führt Passport eine Reihe von Schritten aus, um:

- Einen Benutzer zu authentifizieren, indem seine "Zugangsdaten" überprüft werden (wie z. B. Benutzername/Passwort, JSON Web Token (JWT) oder Identitätstoken von einem Identitätsanbieter).
- Den authentifizierten Zustand zu verwalten (durch Ausgabe eines tragbaren Tokens, wie eines JWT, oder durch Erstellung einer Express-Sitzung).
- Informationen über den authentifizierten Benutzer an das Anforderungsobjekt anzuhängen, um sie später in Routenhandlern zu verwenden.

Passport bietet ein umfangreiches Ökosystem an Strategien, die verschiedene Authentifizierungsmechanismen implementieren. Obwohl das Konzept einfach ist, gibt es eine große Auswahl an Passport-Strategien, die man wählen kann. Passport abstrahiert diese verschiedenen Schritte in ein standardisiertes Muster, und das `@nestjs/passport`-Modul umhüllt und standardisiert dieses Muster in vertraute NestJS-Konstrukte.

In diesem Kapitel implementieren wir eine vollständige End-to-End-Authentifizierungslösung für einen RESTful-API-Server unter Verwendung dieser leistungsstarken und flexiblen Module. Die hier beschriebenen Konzepte können verwendet werden, um jede Passport-Strategie zu implementieren und das Authentifizierungsschema anzupassen. Du kannst die Schritte in diesem Kapitel befolgen, um dieses vollständige Beispiel zu erstellen.

### Authentifizierungsanforderungen / Authentication requirements

Lass uns unsere Anforderungen konkretisieren. Für diesen Anwendungsfall werden Clients zunächst mit einem Benutzernamen und einem Passwort authentifiziert. Sobald die Authentifizierung erfolgt ist, stellt der Server ein JWT aus, das als Trägertoken in einem Autorisierungsheader bei nachfolgenden Anfragen gesendet werden kann, um die Authentifizierung nachzuweisen. Wir erstellen auch eine geschützte Route, die nur für Anfragen zugänglich ist, die ein gültiges JWT enthalten.

Wir beginnen mit der ersten Anforderung: der Authentifizierung eines Benutzers. Anschließend erweitern wir dies durch die Ausgabe eines JWT. Schließlich erstellen wir eine geschützte Route, die auf ein gültiges JWT in der Anfrage überprüft.

Zuerst müssen wir die erforderlichen Pakete installieren. Passport bietet eine Strategie namens `passport-local`, die einen Benutzernamen-/Passwort-Authentifizierungsmechanismus implementiert, der unseren Anforderungen für diesen Teil des Anwendungsfalls entspricht.

```bash
$ npm install --save @nestjs/passport passport passport-local
$ npm install --save-dev @types/passport-local
```

**Hinweis** / **Notice**  
Für jede Passport-Strategie, die du auswählst, benötigst du immer die Pakete `@nestjs/passport` und `passport`. Dann musst du das strategie-spezifische Paket installieren (z. B. `passport-jwt` oder `passport-local`), das die jeweilige Authentifizierungsstrategie implementiert, die du aufbaust. Zusätzlich kannst du, wie oben mit `@types/passport-local` gezeigt, auch die Typdefinitionen für jede Passport-Strategie installieren, die dir beim Schreiben von TypeScript-Code hilft.

### Implementierung von Passport-Strategien / Implementing Passport strategies

Jetzt sind wir bereit, die Authentifizierungsfunktion zu implementieren. Wir beginnen mit einer Übersicht über den Prozess, der für jede Passport-Strategie verwendet wird. Es ist hilfreich, Passport als ein kleines Framework für sich zu betrachten. Die Eleganz des Frameworks liegt darin, dass es den Authentifizierungsprozess in ein paar grundlegende Schritte abstrahiert, die du basierend auf der Strategie, die du implementierst, anpassen kannst. Es ist wie ein Framework, weil du es durch die Bereitstellung von Anpassungsparametern (als einfache JSON-Objekte) und benutzerdefiniertem Code in Form von Rückruffunktionen konfigurierst, die Passport zur richtigen Zeit aufruft. Das `@nestjs/passport`-Modul verpackt dieses Framework in ein Nest-Stil-Paket, wodurch es einfach in eine NestJS-Anwendung integriert werden kann. Wir verwenden `@nestjs/passport` unten, aber zuerst schauen wir uns an, wie Vanilla Passport funktioniert.

In Vanilla Passport konfigurierst du eine Strategie, indem du zwei Dinge bereitstellst:

1. Einen Satz von Optionen, die spezifisch für diese Strategie sind. Zum Beispiel könntest du bei einer JWT-Strategie ein Geheimnis zur Signierung von Tokens angeben.
2. Einen "Verify Callback", bei dem du Passport mitteilst, wie es mit deinem Benutzerverzeichnis (wo du Benutzerkonten verwaltest) interagieren soll. Hier prüfst du, ob ein Benutzer existiert (und/oder einen neuen Benutzer erstellst) und ob seine Zugangsdaten gültig sind. Die Passport-Bibliothek erwartet, dass dieser Rückruf einen vollständigen Benutzer zurückgibt, wenn die Validierung erfolgreich ist, oder `null`, wenn sie fehlschlägt (Misserfolg ist definiert als entweder der Benutzer wird nicht gefunden oder im Fall von `passport-local`, das Passwort stimmt nicht überein).

Mit `@nestjs/passport` konfigurierst du eine Passport-Strategie, indem du die `PassportStrategy`-Klasse erweiterst. Du übergibst die Strategieoptionen (Punkt 1 oben), indem du die `super()`-Methode in deiner Unterklasse aufrufst und optional ein Optionsobjekt übergibst. Du stellst den "Verify Callback" (Punkt 2 oben) bereit, indem du eine `validate()`-Methode in deiner Unterklasse implementierst.

Wir beginnen mit der Generierung eines `AuthModule` und darin eines `AuthService`:

```bash
$ nest g module auth
$ nest g service auth
```

Während wir den `AuthService` implementieren, finden wir es nützlich, Benutzeroperationen in einem `UsersService` zu kapseln. Lass uns dieses Modul und den Service jetzt generieren:

```bash
$ nest g module users
$ nest g service users
```

Ersetze den Standardinhalt dieser generierten Dateien wie unten gezeigt. Für unsere Beispielanwendung verwaltet der `UsersService` einfach eine fest codierte In-Memory-Liste von Benutzern und eine `find`-Methode, um einen Benutzer anhand des Benutzernamens abzurufen. In einer echten Anwendung würdest du hier dein Benutzermodell und die Persistenzschicht mit der Bibliothek deiner Wahl (z. B. TypeORM, Sequelize, Mongoose usw.) aufbauen.

`users/users.service.ts`

```typescript
import { Injectable } from '@nestjs/common';

// Dies sollte eine echte Klasse/Schnittstelle darstellen, die eine Benutzerentität repräsentiert
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

Im `UsersModule` ist die einzige notwendige Änderung, den `UsersService` zum `exports`-Array des `@Module`-Dekorators hinzuzufügen, damit er außerhalb dieses Moduls sichtbar ist (wir werden ihn bald in unserem `AuthService` verwenden).

`users/users.module.ts`

```typescript
import { Module } from '@nestjs/common';
import { UsersService } from './users.service';

@Module({
  providers: [UsersService],
  exports: [UsersService],
})
export class UsersModule {}
```

Unser `AuthService` hat die Aufgabe, einen Benutzer abzurufen und das Passwort zu verifizieren. Wir erstellen eine `validateUser()`-Methode für diesen Zweck. Im folgenden Code verwenden wir den praktischen ES6-Spread-Operator, um die Passwort-Eigenschaft aus dem Benutzerobjekt zu entfernen, bevor wir es zurückgeben. Wir werden die `validateUser()`-Methode gleich in unserer Passport-Local-Strategie aufrufen.

`auth/auth.service.ts`

```typescript
import { Injectable } from '@nestjs/common';
import { UsersService } from '../users/users.service';

@Injectable()
export class AuthService {
  constructor(private usersService: UsersService) {}

  async validateUser(username: string, pass: string): Promise<any> {
    const user = await this.usersService.findOne(username);
    if (user && user.password === pass) {
      const { password, ...result } = user;
      return result;
    }
    return null;
  }
}
```

**Warnung** / **Warning**  
Natürlich würdest du in einer echten Anwendung kein Passwort im Klartext speichern. Stattdessen würdest du eine Bibliothek wie `bcrypt` mit einem gesalzenen One-Way-Hash-Algorithmus verwenden. Mit diesem Ansatz würdest du nur gehashte Passwörter speichern und das gespeicherte Passwort mit einer gehashten Version des eingehenden Passworts vergleichen, sodass Benutzerpasswörter niemals im Klartext gespeichert oder offengelegt werden. Um unsere Beispielanwendung einfach zu halten, verletzen wir dieses absolute Gebot und verwenden Klartext. Mach das nicht in deiner echten Anwendung!

Nun aktualisieren wir unser `AuthModule`, um

 das `UsersModule` zu importieren.

`auth/auth.module.ts`

```typescript
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { UsersModule } from '../users/users.module';

@Module({
  imports: [UsersModule],
  providers: [AuthService],
})
export class AuthModule {}
```

### Implementierung von Passport Local / Implementing Passport local

Jetzt können wir unsere Passport-Local-Authentifizierungsstrategie implementieren. Erstelle eine Datei namens `local.strategy.ts` im `auth`-Ordner und füge den folgenden Code hinzu:

`auth/local.strategy.ts`

```typescript
import { Strategy } from 'passport-local';
import { PassportStrategy } from '@nestjs/passport';
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { AuthService } from './auth.service';

@Injectable()
export class LocalStrategy extends PassportStrategy(Strategy) {
  constructor(private authService: AuthService) {
    super();
  }

  async validate(username: string, password: string): Promise<any> {
    const user = await this.authService.validateUser(username, password);
    if (!user) {
      throw new UnauthorizedException();
    }
    return user;
  }
}
```

Wir haben das zuvor beschriebene Rezept für alle Passport-Strategien befolgt. In unserem Anwendungsfall mit `passport-local` gibt es keine Konfigurationsoptionen, sodass unser Konstruktor einfach `super()` ohne ein Optionsobjekt aufruft.

**Hinweis** / **Hint**  
Wir können ein Optionsobjekt im Aufruf von `super()` übergeben, um das Verhalten der Passport-Strategie anzupassen. In diesem Beispiel erwartet die `passport-local`-Strategie standardmäßig Eigenschaften namens `username` und `password` im Anforderungskörper. Übergebe ein Optionsobjekt, um unterschiedliche Eigenschaftsnamen anzugeben, z. B.: `super({ usernameField: 'email' })`. Weitere Informationen findest du in der Passport-Dokumentation.

Wir haben auch die `validate()`-Methode implementiert. Für jede Strategie ruft Passport die Verify-Funktion auf (implementiert mit der `validate()`-Methode in `@nestjs/passport`) unter Verwendung eines entsprechenden, strategie-spezifischen Satzes von Parametern. Für die `local-strategy` erwartet Passport eine `validate()`-Methode mit der folgenden Signatur: `validate(username: string, password:string): any`.

Der Großteil der Validierungsarbeit wird in unserem `AuthService` erledigt (mit Hilfe unseres `UsersService`), sodass diese Methode ziemlich einfach ist. Die `validate()`-Methode für jede Passport-Strategie folgt einem ähnlichen Muster, das sich nur in den Details unterscheidet, wie Zugangsdaten dargestellt werden. Wenn ein Benutzer gefunden wird und die Zugangsdaten gültig sind, wird der Benutzer zurückgegeben, sodass Passport seine Aufgaben (z. B. Erstellen der Benutzer-Eigenschaft am Anforderungsobjekt) abschließen und die Anforderungsverarbeitungspipeline fortgesetzt werden kann. Wenn der Benutzer nicht gefunden wird, werfen wir eine Ausnahme und lassen unsere Ausnahmeebene dies handhaben.

Normalerweise besteht der einzige wesentliche Unterschied in der `validate()`-Methode jeder Strategie darin, wie du feststellst, ob ein Benutzer existiert und gültig ist. Zum Beispiel könnten wir in einer JWT-Strategie, abhängig von den Anforderungen, bewerten, ob die in dem dekodierten Token enthaltene `userId` mit einem Datensatz in unserer Benutzerdatenbank übereinstimmt oder einer Liste widerrufener Tokens entspricht. Daher ist dieses Muster des Subclassing und der Implementierung einer strategie-spezifischen Validierung konsistent, elegant und erweiterbar.

Wir müssen unser `AuthModule` konfigurieren, um die gerade definierten Passport-Funktionen zu verwenden. Aktualisiere `auth.module.ts` wie folgt:

`auth/auth.module.ts`

```typescript
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { UsersModule } from '../users/users.module';
import { PassportModule } from '@nestjs/passport';
import { LocalStrategy } from './local.strategy';

@Module({
  imports: [UsersModule, PassportModule],
  providers: [AuthService, LocalStrategy],
})
export class AuthModule {}
```

### Eingebaute Passport Guards / Built-in Passport Guards

Das Kapitel Guards beschreibt die Hauptfunktion von Guards: Zu bestimmen, ob eine Anfrage vom Routenhandler bearbeitet wird oder nicht. Das bleibt wahr, und wir werden diese Standardfunktionalität bald nutzen. Im Zusammenhang mit der Verwendung des `@nestjs/passport`-Moduls führen wir jedoch auch eine kleine neue Falte ein, die anfangs verwirrend sein kann, also lass uns das jetzt besprechen. Beachte, dass deine App aus Authentifizierungssicht in zwei Zuständen existieren kann:

1. Der Benutzer/Client ist nicht eingeloggt (nicht authentifiziert).
2. Der Benutzer/Client ist eingeloggt (authentifiziert).

Im ersten Fall (Benutzer ist nicht eingeloggt) müssen wir zwei unterschiedliche Funktionen ausführen:

1. Beschränke die Routen, auf die ein nicht authentifizierter Benutzer zugreifen kann (d. h. verweigere den Zugriff auf geschützte Routen). Wir verwenden Guards in ihrer vertrauten Kapazität, um diese Funktion zu handhaben, indem wir einen Guard auf den geschützten Routen platzieren. Wie du vielleicht vermutest, werden wir in diesem Guard nach dem Vorhandensein eines gültigen JWT suchen, also arbeiten wir später an diesem Guard, sobald wir JWTs erfolgreich ausgeben.

2. Leite den Authentifizierungsschritt selbst ein, wenn ein zuvor nicht authentifizierter Benutzer versucht, sich anzumelden. Dies ist der Schritt, bei dem wir einem gültigen Benutzer ein JWT ausstellen. Wenn wir darüber nachdenken, wissen wir, dass wir Benutzername/Passwort-Zugangsdaten posten müssen, um die Authentifizierung zu initiieren. Daher richten wir eine `POST /auth/login`-Route ein, um dies zu handhaben. Dies wirft die Frage auf: Wie genau rufen wir die `passport-local`-Strategie in dieser Route auf?

Die Antwort ist einfach: durch die Verwendung eines anderen, leicht abweichenden Typs von Guard. Das `@nestjs/passport`-Modul stellt uns einen eingebauten Guard zur Verfügung, der dies für uns erledigt. Dieser Guard ruft die Passport-Strategie auf und startet die oben beschriebenen Schritte (Abrufen von Zugangsdaten, Ausführen der Verify-Funktion, Erstellen der Benutzer-Eigenschaft usw.).

Der zweite oben genannte Fall (eingeloggter Benutzer) verlässt sich einfach auf den standardmäßigen Guard-Typ, den wir bereits besprochen haben, um den Zugriff auf geschützte Routen für eingeloggte Benutzer zu ermöglichen.

### Login-Route / Login route

Nachdem die Strategie vorhanden ist, können wir nun eine grundlegende `/auth/login`-Route implementieren und den eingebauten Guard anwenden, um den `passport-local`-Fluss zu initiieren.

Öffne die Datei `app.controller.ts` und ersetze deren Inhalt durch Folgendes:

`app.controller.ts`

```typescript
import { Controller, Request, Post, UseGuards } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Controller()
export class AppController {
  @UseGuards(AuthGuard('local'))
  @Post('auth/login')
  async login(@Request() req) {
    return req.user;
  }
}
```

Mit `@UseGuards(AuthGuard('local'))` verwenden wir einen AuthGuard, den `@nestjs/passport` automatisch für uns bereitgestellt hat, als wir die `passport-local`-Strategie erweitert haben. Lass uns das genauer betrachten. Unsere Passport-Local-Strategie hat einen Standardnamen 'local'. Wir verweisen auf diesen Namen im `@UseGuards()`-Dekorator, um ihn mit dem von der `passport-local`-Paket bereitgestellten Code zu verknüpfen. Dies wird verwendet, um zu klären, welche Strategie aufgerufen werden soll, falls wir mehrere Passport-Strategien in unserer App haben (von denen jede möglicherweise einen strategie-spezifischen AuthGuard bereitstellt). Obwohl wir bisher nur eine solche Strategie haben, fügen wir bald eine zweite hinzu, daher ist diese Klarstellung erforderlich.

Um unsere Route zu testen, lassen wir unsere `/auth/login`-Route einfach den Benutzer zurückgeben. Dies ermöglicht es uns auch, eine weitere Passport-Funktion zu demonstrieren: Passport erstellt automatisch ein Benutzerobjekt, basierend auf dem Wert, den wir von der `validate()`-Methode zurückgeben, und ordnet es dem Anforderungsobjekt als `req.user` zu. Später werden wir dies durch Code ersetzen, um ein JWT zu erstellen und zurückzugeben.

Da dies API-Routen sind, testen wir sie mit der weit verbreiteten cURL-Bibliothek. Du kannst mit einem der Benutzerobjekte testen, die im `UsersService` fest codiert sind.

```bash
$ # POST to /auth/login
$ curl -X POST http://localhost:3000/auth/login -d '{"username": "john", "password": "changeme"}' -H "Content-Type: application/json"
$ # result -> {"userId":1,"username":"john"}
```

Obwohl dies funktioniert, führt das direkte Übergeben des Strategienamens an den `AuthGuard()` magische Zeichenfolgen in den Code ein. Stattdessen empfehlen wir, eine eigene Klasse zu erstellen, wie unten gezeigt:

`auth/local-auth.guard.ts`

```typescript
import { Injectable } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Injectable()
export class LocalAuthGuard

 extends AuthGuard('local') {}
```

Nun können wir den `/auth/login`-Routenhandler aktualisieren und stattdessen den `LocalAuthGuard` verwenden:

```typescript
@UseGuards(LocalAuthGuard)
@Post('auth/login')
async login(@Request() req) {
  return req.user;
}
```

### JWT-Funktionalität / JWT functionality

Wir sind bereit, zum JWT-Teil unseres Authentifizierungssystems überzugehen. Lass uns unsere Anforderungen überprüfen und verfeinern:

- Ermögliche es Benutzern, sich mit Benutzername/Passwort zu authentifizieren, wobei ein JWT für die Verwendung bei nachfolgenden Aufrufen geschützter API-Endpunkte zurückgegeben wird. Wir sind auf dem besten Weg, diese Anforderung zu erfüllen. Um dies abzuschließen, müssen wir den Code schreiben, der ein JWT ausstellt.
- Erstelle API-Routen, die basierend auf dem Vorhandensein eines gültigen JWT als Trägertoken geschützt sind.

Wir müssen noch ein paar Pakete installieren, um unsere JWT-Anforderungen zu unterstützen:

```bash
$ npm install --save @nestjs/jwt passport-jwt
$ npm install --save-dev @types/passport-jwt
```

Das Paket `@nestjs/jwt` ist ein Dienstprogramm-Paket, das bei der Manipulation von JWTs hilft. Das Paket `passport-jwt` ist das Passport-Paket, das die JWT-Strategie implementiert, und `@types/passport-jwt` stellt die TypeScript-Typdefinitionen bereit.

Werfen wir einen genaueren Blick darauf, wie eine `POST /auth/login`-Anfrage bearbeitet wird. Wir haben die Route mithilfe des eingebauten AuthGuards dekoriert, der von der `passport-local`-Strategie bereitgestellt wird. Das bedeutet:

- Der Routenhandler wird nur aufgerufen, wenn der Benutzer validiert wurde.
- Der Parameter `req` enthält eine Benutzer-Eigenschaft (`user`), die von Passport während des `passport-local`-Authentifizierungsflusses befüllt wird.

Mit diesem Wissen können wir endlich ein echtes JWT generieren und es in dieser Route zurückgeben. Um unsere Dienste sauber zu modularisieren, werden wir das Generieren des JWT im `authService` behandeln. Öffne die Datei `auth.service.ts` im `auth`-Ordner und füge die `login()`-Methode hinzu, und importiere den `JwtService` wie gezeigt:

`auth/auth.service.ts`

```typescript
import { Injectable } from '@nestjs/common';
import { UsersService } from '../users/users.service';
import { JwtService } from '@nestjs/jwt';

@Injectable()
export class AuthService {
  constructor(
    private usersService: UsersService,
    private jwtService: JwtService
  ) {}

  async validateUser(username: string, pass: string): Promise<any> {
    const user = await this.usersService.findOne(username);
    if (user && user.password === pass) {
      const { password, ...result } = user;
      return result;
    }
    return null;
  }

  async login(user: any) {
    const payload = { username: user.username, sub: user.userId };
    return {
      access_token: this.jwtService.sign(payload),
    };
  }
}
```

Wir verwenden die `@nestjs/jwt`-Bibliothek, die eine `sign()`-Funktion bereitstellt, um unser JWT aus einer Teilmenge der Benutzerobjekteigenschaften zu generieren, die wir dann als einfaches Objekt mit einer einzigen `access_token`-Eigenschaft zurückgeben. Hinweis: Wir wählen eine Eigenschaft namens `sub`, um unseren `userId`-Wert zu halten, um den JWT-Standards zu entsprechen. Vergiss nicht, den `JwtService`-Provider in den `AuthService` zu injizieren.

Jetzt müssen wir das `AuthModule` aktualisieren, um die neuen Abhängigkeiten zu importieren und das `JwtModule` zu konfigurieren.

Erstelle zuerst `constants.ts` im `auth`-Ordner und füge den folgenden Code hinzu:

`auth/constants.ts`

```typescript
export const jwtConstants = {
  secret: 'VERWENDE DIESEN WERT NICHT. STATT DESSEN ERSTELLE EIN KOMPLEXES GEHEIMNIS UND HALTE ES SICHER AUSSERHALB DES QUELLCODES.',
};
```

Wir werden dies verwenden, um unseren Schlüssel zwischen den JWT-Signier- und Verifizierungsschritten zu teilen.

**Warnung** / **Warning**  
Gib diesen Schlüssel nicht öffentlich preis. Wir haben dies hier getan, um deutlich zu machen, was der Code tut, aber in einem Produktionssystem musst du diesen Schlüssel durch geeignete Maßnahmen wie einen Geheimnis-Tresor, eine Umgebungsvariable oder einen Konfigurationsdienst schützen.

Öffne nun `auth.module.ts` im `auth`-Ordner und aktualisiere es wie folgt:

`auth/auth.module.ts`

```typescript
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { LocalStrategy } from './local.strategy';
import { UsersModule } from '../users/users.module';
import { PassportModule } from '@nestjs/passport';
import { JwtModule } from '@nestjs/jwt';
import { jwtConstants } from './constants';

@Module({
  imports: [
    UsersModule,
    PassportModule,
    JwtModule.register({
      secret: jwtConstants.secret,
      signOptions: { expiresIn: '60s' },
    }),
  ],
  providers: [AuthService, LocalStrategy],
  exports: [AuthService],
})
export class AuthModule {}
```

Wir konfigurieren das `JwtModule` mit `register()`, indem wir ein Konfigurationsobjekt übergeben. Siehe [hier](https://docs.nestjs.com/) für mehr über das `JwtModule` von Nest und [hier](https://docs.nestjs.com/security/authentication#jwt-token) für weitere Details zu den verfügbaren Konfigurationsoptionen.

Nun können wir die `/auth/login`-Route aktualisieren, um ein JWT zurückzugeben.

`app.controller.ts`

```typescript
import { Controller, Request, Post, UseGuards } from '@nestjs/common';
import { LocalAuthGuard } from './auth/local-auth.guard';
import { AuthService } from './auth/auth.service';

@Controller()
export class AppController {
  constructor(private authService: AuthService) {}

  @UseGuards(LocalAuthGuard)
  @Post('auth/login')
  async login(@Request() req) {
    return this.authService.login(req.user);
  }
}
```

Lass uns die Routen erneut mit cURL testen. Du kannst mit einem der Benutzerobjekte testen, die im `UsersService` fest codiert sind.

```bash
$ # POST to /auth/login
$ curl -X POST http://localhost:3000/auth/login -d '{"username": "john", "password": "changeme"}' -H "Content-Type: application/json"
$ # result -> {"access_token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."}
$ # Hinweis: Obiges JWT gekürzt
```

### Implementierung von Passport JWT / Implementing Passport JWT

Jetzt können wir unsere letzte Anforderung angehen: Endpunkte zu schützen, indem ein gültiges JWT in der Anfrage erforderlich ist. Passport kann uns auch hier helfen. Es bietet die `passport-jwt`-Strategie zum Sichern von RESTful-Endpunkten mit JSON Web Tokens. Erstelle zunächst eine Datei namens `jwt.strategy.ts` im `auth`-Ordner und füge den folgenden Code hinzu:

`auth/jwt.strategy.ts`

```typescript
import { ExtractJwt, Strategy } from 'passport-jwt';
import { PassportStrategy } from '@nestjs/passport';
import { Injectable } from '@nestjs/common';
import { jwtConstants } from './constants';

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor() {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      ignoreExpiration: false,
      secretOrKey: jwtConstants.secret,
    });
  }

  async validate(payload: any) {
    return { userId: payload.sub, username: payload.username };
  }
}
```

Mit unserer `JwtStrategy` haben wir das gleiche Rezept befolgt, das zuvor für alle Passport-Strategien beschrieben wurde. Diese Strategie erfordert einige Initialisierungen, die wir durch die Übergabe eines Optionsobjekts im `super()`-Aufruf durchführen. Weitere Informationen zu den verfügbaren Optionen findest du [hier](https://docs.nestjs.com/security/authentication#jwt-token). In unserem Fall sind diese Optionen:

- `jwtFromRequest`: Gibt die Methode an, mit der das JWT aus der Anfrage extrahiert wird. Wir verwenden die Standardmethode, ein Trägertoken im `Authorization`-Header unserer API-Anfragen bereitzustellen. Weitere Optionen werden [hier](https://docs.nestjs.com/security/authentication#jwt-token) beschrieben.
- `ignoreExpiration`: Um explizit zu sein, wählen wir die Standardeinstellung `false`, die die Verantwortung für die Sicherstellung, dass ein JWT nicht abgelaufen ist, an das Passport-Modul delegiert. Das bedeutet, dass wenn unsere Route ein abgelaufenes JWT erhält, die Anfrage abgelehnt und eine `401 Unauthorized`-Antwort gesendet wird. Passport kümmert sich automatisch um diese Überprüfung.
- `secretOrKey`: Wir verwenden die schnelle Option, einen symmetrischen Schlüssel zur Signierung des Tokens anzugeben. Andere Optionen, wie z. B. ein PEM-kodierter öffentlicher Schlüssel, sind möglicherweise für Produktionsanwendungen besser geeignet (siehe [hier](https://docs.nestjs.com/security/authentication#jwt-token) für mehr Informationen). Wie bereits gewarnt, darf dieses Geheimnis nicht öffentlich gemacht werden.

Die `validate()`-Methode verdient eine Diskussion. Für die `jwt-strategy` überprüft Passport zuerst die Signatur des JWT und dekodiert das JSON. Dann ru

ft es unsere `validate()`-Methode auf und übergibt das dekodierte JSON als einzigen Parameter. Basierend auf der Funktionsweise der JWT-Signierung sind wir garantiert, dass wir ein gültiges Token erhalten, das wir zuvor signiert und an einen gültigen Benutzer ausgegeben haben.

Als Ergebnis all dessen ist unsere Antwort auf den `validate()`-Callback trivial: Wir geben einfach ein Objekt zurück, das die `userId`- und `username`-Eigenschaften enthält. Erinnere dich daran, dass Passport basierend auf dem Rückgabewert unserer `validate()`-Methode ein Benutzerobjekt erstellt und es als Eigenschaft auf dem Anforderungsobjekt anfügt.

Es ist auch erwähnenswert, dass dieser Ansatz uns Raum lässt (sozusagen „Hooks“), um andere Geschäftslogik in den Prozess zu integrieren. Zum Beispiel könnten wir in unserer `validate()`-Methode eine Datenbankabfrage durchführen, um mehr Informationen über den Benutzer zu extrahieren, was zu einem angereicherten Benutzerobjekt im Anforderungspipeline führt. Dies ist auch der Ort, an dem wir weitere Token-Validierungen durchführen können, wie z. B. das Nachschlagen der `userId` in einer Liste von widerrufenen Tokens, wodurch wir die Token-Widerrufung durchführen können. Das hier in unserem Beispielcode implementierte Modell ist ein schnelles, "zustandsloses JWT"-Modell, bei dem jeder API-Aufruf sofort anhand des Vorhandenseins eines gültigen JWT autorisiert wird und ein kleines bisschen Information über den Anfragenden (seine `userId` und `username`) in unserer Anforderungspipeline verfügbar ist.

Füge die neue `JwtStrategy` als Provider im `AuthModule` hinzu:

`auth/auth.module.ts`

```typescript
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { LocalStrategy } from './local.strategy';
import { JwtStrategy } from './jwt.strategy';
import { UsersModule } from '../users/users.module';
import { PassportModule } from '@nestjs/passport';
import { JwtModule } from '@nestjs/jwt';
import { jwtConstants } from './constants';

@Module({
  imports: [
    UsersModule,
    PassportModule,
    JwtModule.register({
      secret: jwtConstants.secret,
      signOptions: { expiresIn: '60s' },
    }),
  ],
  providers: [AuthService, LocalStrategy, JwtStrategy],
  exports: [AuthService],
})
export class AuthModule {}
```

Durch den Import des gleichen Geheimnisses, das wir beim Signieren des JWT verwendet haben, stellen wir sicher, dass die Überprüfungsphase, die von Passport durchgeführt wird, und die Signaturphase, die in unserem `AuthService` durchgeführt wird, ein gemeinsames Geheimnis verwenden.

Schließlich definieren wir die `JwtAuthGuard`-Klasse, die den eingebauten `AuthGuard` erweitert:

`auth/jwt-auth.guard.ts`

```typescript
import { Injectable } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {}
```

### Geschützte Route und JWT-Strategie-Guards implementieren / Implement protected route and JWT strategy guards

Wir können nun unsere geschützte Route und den zugehörigen Guard implementieren.

Öffne die Datei `app.controller.ts` und aktualisiere sie wie unten gezeigt:

`app.controller.ts`

```typescript
import { Controller, Get, Request, Post, UseGuards } from '@nestjs/common';
import { JwtAuthGuard } from './auth/jwt-auth.guard';
import { LocalAuthGuard } from './auth/local-auth.guard';
import { AuthService } from './auth/auth.service';

@Controller()
export class AppController {
  constructor(private authService: AuthService) {}

  @UseGuards(LocalAuthGuard)
  @Post('auth/login')
  async login(@Request() req) {
    return this.authService.login(req.user);
  }

  @UseGuards(JwtAuthGuard)
  @Get('profile')
  getProfile(@Request() req) {
    return req.user;
  }
}
```

Wir wenden erneut den AuthGuard an, den das `@nestjs/passport`-Modul automatisch für uns bereitgestellt hat, als wir das `passport-jwt`-Modul konfiguriert haben. Dieser Guard wird durch seinen Standardnamen `jwt` referenziert. Wenn unsere `GET /profile`-Route aufgerufen wird, wird der Guard automatisch unsere benutzerdefinierte `passport-jwt`-Strategie aufrufen, das JWT validieren und die Benutzer-Eigenschaft auf dem Anforderungsobjekt zuweisen.

Stelle sicher, dass die App läuft, und teste die Routen mit cURL:

```bash
$ # GET /profile
$ curl http://localhost:3000/profile
$ # result -> {"statusCode":401,"message":"Unauthorized"}

$ # POST /auth/login
$ curl -X POST http://localhost:3000/auth/login -d '{"username": "john", "password": "changeme"}' -H "Content-Type: application/json"
$ # result -> {"access_token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...}

$ # GET /profile using access_token returned from previous step as bearer code
$ curl http://localhost:3000/profile -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2Vybm..."
$ # result -> {"userId":1,"username":"john"}
```

Beachte, dass wir im `AuthModule` das JWT so konfiguriert haben, dass es nach 60 Sekunden abläuft. Dies ist wahrscheinlich eine zu kurze Ablaufzeit, und die Details zur Token-Ablaufzeit und -Aktualisierung zu behandeln, würde den Rahmen dieses Artikels sprengen. Wir haben das jedoch gewählt, um eine wichtige Eigenschaft von JWTs und der `passport-jwt`-Strategie zu demonstrieren. Wenn du 60 Sekunden nach der Authentifizierung wartest, bevor du eine `GET /profile`-Anfrage versuchst, erhältst du eine `401 Unauthorized`-Antwort. Das liegt daran, dass Passport das JWT automatisch auf seine Ablaufzeit überprüft und dir diese Arbeit in deiner Anwendung erspart.

Wir haben nun unsere JWT-Authentifizierungsimplementierung abgeschlossen. JavaScript-Clients (wie Angular/React/Vue) und andere JavaScript-Anwendungen können sich nun authentifizieren und sicher mit unserem API-Server kommunizieren.

### Erweiterung von Guards / Extending Guards

In den meisten Fällen reicht es aus, die bereitgestellten `AuthGuard`-Klassen zu verwenden. Es kann jedoch Anwendungsfälle geben, in denen du die standardmäßige Fehlerbehandlung oder Authentifizierungslogik einfach erweitern möchtest. Dazu kannst du die integrierte Klasse erweitern und Methoden in einer Unterklasse überschreiben.

Hier ist ein Beispiel, wie du den `JwtAuthGuard` anpassen kannst:

```typescript
import {
  ExecutionContext,
  Injectable,
  UnauthorizedException,
} from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {
  canActivate(context: ExecutionContext) {
    // Füge hier deine benutzerdefinierte Authentifizierungslogik hinzu
    // Beispielsweise kannst du super.logIn(request) aufrufen, um eine Sitzung zu erstellen.
    return super.canActivate(context);
  }

  handleRequest(err, user, info) {
    // Du kannst eine Ausnahme basierend auf den Argumenten "info" oder "err" werfen
    if (err || !user) {
      throw err || new UnauthorizedException();
    }
    return user;
  }
}
```

Zusätzlich zur Erweiterung der Standardfehlerbehandlung und Authentifizierungslogik können wir die Authentifizierung durch eine Kette von Strategien laufen lassen. Die erste Strategie, die erfolgreich ist, eine Umleitung vornimmt oder einen Fehler verursacht, stoppt die Kette. Authentifizierungsfehler führen dazu, dass jede Strategie in Serie durchlaufen wird, wobei die Authentifizierung letztlich fehlschlägt, wenn alle Strategien fehlschlagen.

```typescript
export class JwtAuthGuard extends AuthGuard(['strategy_jwt_1', 'strategy_jwt_2', '...']) { ... }
```

### Authentifizierung global aktivieren / Enable authentication globally

Wenn die Mehrheit deiner Endpunkte standardmäßig geschützt sein soll, kannst du den Authentifizierungs-Guard als globalen Guard registrieren. Anstatt den `@UseGuards()`-Dekorator für jeden Controller zu verwenden, könntest du einfach festlegen, welche Routen öffentlich sein sollen.

Registriere zuerst den `JwtAuthGuard` als globalen Guard mit der folgenden Konstruktion (in einem beliebigen Modul):

```typescript
providers: [
  {
    provide: APP_GUARD,
    useClass: JwtAuthGuard,
  },
],
```

Mit dieser Konfiguration bindet Nest den `JwtAuthGuard` automatisch an alle Endpunkte.

Nun müssen wir einen Mechanismus bereitstellen, um Routen als öffentlich zu deklarieren. Dazu können wir einen benutzerdefinierten Dekorator mit der `SetMetadata`-Fabrikfunktion erstellen.

```typescript
import { SetMetadata } from '@nestjs/common';

export const IS_PUBLIC_KEY = 'isPublic';
export const Public = () => SetMetadata(IS_PUBLIC_KEY, true);
```

In der obigen Datei haben wir zwei Konstanten exportiert. Eine ist unser Metadaten-Schlüssel namens `IS_PUBLIC_KEY`, und die andere ist unser neuer Dekorator, den wir `Public` nennen (alternativ kannst du ihn `SkipAuth` oder `AllowAnon` nennen, je nachdem, was am besten zu deinem Projekt passt).

Jetzt, da wir einen benutzerdefinierten `@Public()`-Dekorator haben, können wir ihn verwenden, um eine beliebige Methode wie folgt zu dekorieren:

```typescript
@Public()
@Get()
findAll() {
  return [];
}
```

Schließlich muss der `JwtAuthGuard` `true` zurückgeben, wenn die "isPublic"-Metadaten gefunden werden. Dazu verwenden wir die `Reflector`-Klasse (mehr dazu [hier](https://docs.nestjs.com/guards#metadata)).

```typescript
@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {
  constructor(private reflector: Reflector) {
    super();
  }

  canActivate(context: ExecutionContext) {
    const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);
    if (isPublic) {
      return true;
    }
    return super.canActivate(context);
  }
}
```

### Request-spezifische Strategien / Request-scoped strategies

Die Passport-API basiert darauf, Strategien zur globalen Instanz der Bibliothek zu registrieren. Daher sind Strategien nicht dafür ausgelegt, anfrageabhängige Optionen zu haben oder dynamisch pro Anfrage instanziiert zu werden (mehr zu anfrage-spezifischen Providern findest du [hier](https://docs.nestjs.com/fundamentals/injection-scopes#request-scoped-providers)). Wenn du deine Strategie so konfigurierst, dass sie anfrage-spezifisch ist, wird Nest sie nie instanziieren, da sie an keine spezifische Route gebunden ist. Es gibt keinen physischen Weg, um festzustellen, welche "anfrage-spezifischen" Strategien pro Anfrage ausgeführt werden sollen.

Es gibt jedoch Möglichkeiten, anfrage-spezifische Provider innerhalb der Strategie dynamisch zu lösen. Dazu nutzen wir die `ModuleRef`-Funktion.

Öffne zunächst die Datei `local.strategy.ts` und injiziere `ModuleRef` auf die übliche Weise:

```typescript
constructor(private moduleRef: ModuleRef) {
  super({
    passReqToCallback: true,
  });
}
```

**Hinweis** / **Hint**  
Die `ModuleRef`-Klasse wird aus dem `@nestjs/core`-Paket importiert. Stelle sicher, dass du die `passReqToCallback`-Konfigurationseigenschaft auf `true` setzt, wie oben gezeigt.

Im nächsten Schritt wird die Anfrageinstanz verwendet, um die aktuelle Kontextkennung zu erhalten, anstatt eine neue zu generieren (mehr dazu [hier](https://docs.nestjs.com/fundamentals/execution-context#execution-context)).

Jetzt kannst du innerhalb der `validate()`-Methode der `LocalStrategy`-Klasse die `getByRequest()`-Methode der `ContextIdFactory`-Klasse verwenden, um eine Kontextkennung basierend auf dem Anforderungsobjekt zu erstellen und diese an den `resolve()`-Aufruf zu übergeben:

```typescript
async validate(
  request: Request,
  username: string,
  password: string,
) {
  const contextId = ContextIdFactory.getByRequest(request);
  // "AuthService" ist ein anfrage-spezifischer Provider
  const authService = await this.moduleRef.resolve(AuthService, contextId);
  ...
}
```

Im obigen Beispiel gibt die `resolve()`-Methode asynchron die anfrage-spezifische Instanz des `AuthService`-Providers zurück (wir nehmen an, dass `AuthService` als anfrage-spezifischer Provider markiert ist).

### Passport anpassen / Customize Passport

Alle Standardanpassungsoptionen von Passport können auf die gleiche Weise über die `register()`-Methode übergeben werden. Die verfügbaren Optionen hängen von der implementierten Strategie ab. Zum Beispiel:

```typescript
PassportModule.register({ session: true });
```

Du kannst auch Strategien ein Optionsobjekt in ihren Konstruktoren übergeben, um sie zu konfigurieren. Für die lokale Strategie könntest du zum Beispiel Folgendes übergeben:

```typescript
constructor(private authService: AuthService) {
  super({
    usernameField: 'email',
    passwordField: 'password',
  });
}
```

Schau dir die offizielle [Passport-Website](http://www.passportjs.org/) für Eigenschaftsnamen an.

### Benannte Strategien / Named strategies

Beim Implementieren einer Strategie kannst du ihr einen Namen geben, indem du ein zweites Argument an die `PassportStrategy`-Funktion übergibst. Wenn du dies nicht tust, hat jede Strategie einen Standardnamen (z. B. 'jwt' für die `jwt-strategy`):

```typescript
export class JwtStrategy extends PassportStrategy(Strategy, 'myjwt')
```

Dann beziehst du dich darauf über einen Dekorator wie `@UseGuards(AuthGuard('myjwt'))`.

### GraphQL-Unterstützung / GraphQL Support

Um einen `AuthGuard` mit GraphQL zu verwenden, erweitere die eingebaute `AuthGuard`-Klasse und überschreibe die `getRequest()`-Methode.

```typescript
@Injectable()
export class GqlAuthGuard extends AuthGuard('jwt') {
  getRequest(context: ExecutionContext) {
    const ctx = GqlExecutionContext.create(context);
    return ctx.getContext().req;
  }
}
```

Um den aktuell authentifizierten Benutzer in deinem GraphQL-Resolver zu erhalten, kannst du einen `@CurrentUser()`-Dekorator definieren:

```typescript
import { createParamDecorator, ExecutionContext } from '@nestjs/common';
import { GqlExecutionContext } from '@nestjs/graphql';

export const CurrentUser = createParamDecorator(
  (data: unknown, context: ExecutionContext) => {
    const ctx = GqlExecutionContext.create(context);
    return ctx.getContext().req.user;
  },
);
```

Um den obigen Dekorator in deinem Resolver zu verwenden, stelle sicher, dass du ihn als Parameter deiner Abfrage oder Mutation einfügst:

```typescript
@Query(returns => User)
@UseGuards(GqlAuthGuard)
whoAmI(@CurrentUser() user: User) {
  return this.usersService.findById(user.id);
}
```
