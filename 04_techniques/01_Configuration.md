## Konfiguration

Anwendungen laufen oft in unterschiedlichen Umgebungen. Abhängig von der Umgebung sollten unterschiedliche Konfigurationseinstellungen verwendet werden. Zum Beispiel verwendet die lokale Umgebung in der Regel spezifische Datenbank-Zugangsdaten, die nur für die lokale DB-Instanz gültig sind. Die Produktionsumgebung würde ein separates Set von DB-Zugangsdaten verwenden. Da sich Konfigurationsvariablen ändern, ist es am besten, sie in der Umgebung zu speichern.

Extern definierte Umgebungsvariablen sind innerhalb von Node.js durch das globale `process.env` sichtbar. Wir könnten versuchen, das Problem mehrerer Umgebungen zu lösen, indem wir die Umgebungsvariablen separat in jeder Umgebung setzen. Dies kann schnell unübersichtlich werden, besonders in der Entwicklungs- und Testumgebung, wo diese Werte leicht nachgebildet und/oder geändert werden müssen.

In Node.js-Anwendungen ist es üblich, `.env`-Dateien zu verwenden, die Schlüssel-Wert-Paare enthalten, wobei jeder Schlüssel einen bestimmten Wert darstellt, um jede Umgebung darzustellen. Das Ausführen einer App in verschiedenen Umgebungen ist dann einfach eine Frage des Austauschs der richtigen `.env`-Datei.

Ein guter Ansatz für die Verwendung dieser Technik in Nest besteht darin, ein `ConfigModule` zu erstellen, das einen `ConfigService` bereitstellt, der die entsprechende `.env`-Datei lädt. Während Sie ein solches Modul selbst schreiben können, bietet Nest zur Bequemlichkeit das `@nestjs/config`-Paket direkt an. Wir behandeln dieses Paket im aktuellen Kapitel.

### Installation

Um es zu verwenden, installieren wir zuerst die erforderliche Abhängigkeit:

```bash
$ npm i --save @nestjs/config
```

### HINWEIS
Das `@nestjs/config`-Paket verwendet intern dotenv.

### Getting started

Sobald die Installation abgeschlossen ist, können wir das `ConfigModule` importieren. Typischerweise importieren wir es in das Hauptmodul `AppModule` und steuern sein Verhalten mit der `.forRoot()`-Methode. Während dieses Schrittes werden Schlüssel/Wert-Paare der Umgebungsvariablen analysiert und aufgelöst. Später werden wir mehrere Möglichkeiten sehen, auf die `ConfigService`-Klasse des `ConfigModule` in unseren anderen Feature-Modulen zuzugreifen.

```typescript
// app.module.ts

import { Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';

@Module({
  imports: [ConfigModule.forRoot()],
})
export class AppModule {}
```

Der obige Code lädt und analysiert eine `.env`-Datei aus dem Standardverzeichnis (dem Projektstammverzeichnis), kombiniert Schlüssel/Wert-Paare aus der `.env`-Datei mit den Umgebungsvariablen, die `process.env` zugewiesen sind, und speichert das Ergebnis in einer privaten Struktur, auf die Sie über den `ConfigService` zugreifen können. Die `forRoot()`-Methode registriert den `ConfigService`-Provider, der eine `get()`-Methode zum Lesen dieser analysierten/kombinierten Konfigurationsvariablen bereitstellt.

Ein Beispiel für eine `.env`-Datei sieht folgendermaßen aus:

```
DATABASE_USER=test
DATABASE_PASSWORD=test
```

### Benutzerdefinierter Pfad für die .env-Datei

Standardmäßig sucht das Paket nach einer `.env`-Datei im Stammverzeichnis der Anwendung. Um einen anderen Pfad für die `.env`-Datei anzugeben, setzen Sie die `envFilePath`-Eigenschaft eines (optional) Optionsobjekts, das Sie an `forRoot()` übergeben:

```typescript
ConfigModule.forRoot({
  envFilePath: '.development.env',
});
```

Sie können auch mehrere Pfade für `.env`-Dateien angeben:

```typescript
ConfigModule.forRoot({
  envFilePath: ['.env.development.local', '.env.development'],
});
```

Wenn eine Variable in mehreren Dateien gefunden wird, hat die erste Vorrang.

### Laden von Umgebungsvariablen deaktivieren

Wenn Sie die `.env`-Datei nicht laden möchten, sondern stattdessen nur auf Umgebungsvariablen aus der Laufzeitumgebung (wie OS-Shell-Exporte) zugreifen möchten, setzen Sie die `ignoreEnvFile`-Eigenschaft des Optionsobjekts auf `true`:

```typescript
ConfigModule.forRoot({
  ignoreEnvFile: true,
});
```

### Modul global verwenden

Wenn Sie `ConfigModule` in anderen Modulen verwenden möchten, müssen Sie es importieren (wie bei jedem Nest-Modul üblich). Alternativ deklarieren Sie es als globales Modul, indem Sie die `isGlobal`-Eigenschaft des Optionsobjekts auf `true` setzen. In diesem Fall müssen Sie `ConfigModule` nicht in anderen Modulen importieren, sobald es im Hauptmodul (z.B. `AppModule`) geladen wurde:

```typescript
ConfigModule.forRoot({
  isGlobal: true,
});
```

### Benutzerdefinierte Konfigurationsdateien

Für komplexere Projekte können Sie benutzerdefinierte Konfigurationsdateien verwenden, um verschachtelte Konfigurationsobjekte zurückzugeben. Dies ermöglicht es Ihnen, verwandte Konfigurationseinstellungen nach Funktion zu gruppieren (z.B. datenbankbezogene Einstellungen) und verwandte Einstellungen in einzelnen Dateien zu speichern, um sie unabhängig zu verwalten.

Eine benutzerdefinierte Konfigurationsdatei exportiert eine Factory-Funktion, die ein Konfigurationsobjekt zurückgibt. Das Konfigurationsobjekt kann ein beliebiges verschachteltes einfaches JavaScript-Objekt sein. Das `process.env`-Objekt enthält die vollständig aufgelösten Schlüssel/Wert-Paare der Umgebungsvariablen (mit `.env`-Datei und extern definierten Variablen aufgelöst und kombiniert wie oben beschrieben). Da Sie das zurückgegebene Konfigurationsobjekt steuern, können Sie jede erforderliche Logik hinzufügen, um Werte in einen geeigneten Typ zu konvertieren, Standardwerte festzulegen usw. Zum Beispiel:

```typescript
// config/configuration.ts

export default () => ({
  port: parseInt(process.env.PORT, 10) || 3000,
  database: {
    host: process.env.DATABASE_HOST,
    port: parseInt(process.env.DATABASE_PORT, 10) || 5432
  }
});
```

Wir laden diese Datei mit der `load`-Eigenschaft des Optionsobjekts, das wir an die `ConfigModule.forRoot()`-Methode übergeben:

```typescript
import configuration from './config/configuration';

@Module({
  imports: [
    ConfigModule.forRoot({
      load: [configuration],
    }),
  ],
})
export class AppModule {}
```

### Hinweis
Der Wert, der der `load`-Eigenschaft zugewiesen wird, ist ein Array, sodass Sie mehrere Konfigurationsdateien laden können (z.B. `load: [databaseConfig, authConfig]`).

### YAML-Konfigurationsdateien

Mit benutzerdefinierten Konfigurationsdateien können wir auch benutzerdefinierte Dateien wie YAML-Dateien verwalten. Hier ist ein Beispiel für eine Konfiguration im YAML-Format:

```yaml
http:
  host: 'localhost'
  port: 8080

db:
  postgres:
    url: 'localhost'
    port: 5432
    database: 'yaml-db'

  sqlite:
    database: 'sqlite.db'
```

Um YAML-Dateien zu lesen und zu analysieren, können wir das `js-yaml`-Paket verwenden.

```bash
$ npm i js-yaml
$ npm i -D @types/js-yaml
```

Sobald das Paket installiert ist, verwenden wir die `yaml#load`-Funktion, um die oben erstellte YAML-Datei zu laden.

```typescript
// config/configuration.ts

import { readFileSync } from 'fs';
import * as yaml from 'js-yaml';
import { join } from 'path';

const YAML_CONFIG_FILENAME = 'config.yaml';

export default () => {
  return yaml.load(
    readFileSync(join(__dirname, YAML_CONFIG_FILENAME), 'utf8'),
  ) as Record<string, any>;
};
```

### Hinweis
Die Nest CLI verschiebt Ihre "Assets" (nicht-TS-Dateien) während des Build-Prozesses nicht automatisch in den `dist`-Ordner. Um sicherzustellen, dass Ihre YAML-Dateien kopiert werden, müssen Sie dies im `compilerOptions#assets`-Objekt in der `nest-cli.json`-Datei angeben. Wenn sich der `config`-Ordner auf derselben Ebene wie der `src`-Ordner befindet, fügen Sie `compilerOptions#assets` mit dem Wert `"assets": [{"include": "../config/*.yaml", "outDir": "./dist/config"}]` hinzu. Lesen Sie mehr [hier](https://docs.nestjs.com).

### Verwendung des ConfigService

Um Konfigurationswerte aus unserem `ConfigService` zu erhalten, müssen wir zuerst `ConfigService` injizieren. Wie bei jedem Provider müssen wir das enthaltene Modul - das `ConfigModule` - in das Modul importieren, das es verwenden wird (es sei denn, Sie setzen die `isGlobal`-Eigenschaft im Optionsobjekt, das an die `ConfigModule.forRoot()`-Methode übergeben wird, auf `true`). Importieren Sie es in ein Feature-Modul wie unten gezeigt:

```typescript
// feature.module.ts

@Module({
  imports: [ConfigModule],
  // ...
})
export class FeatureModule {}
```

Dann können wir es mit der Standard-Konstruk

torinjektion injizieren:

```typescript
constructor(private configService: ConfigService) {}
```

### HINWEIS
Der `ConfigService` wird aus dem `@nestjs/config`-Paket importiert.

Und verwenden Sie ihn in unserer Klasse:

```typescript
// einfache Umgebungsvariable abrufen
const dbUser = this.configService.get<string>('DATABASE_USER');

// benutzerdefinierten Konfigurationswert abrufen
const dbHost = this.configService.get<string>('database.host');
```

Wie oben gezeigt, verwenden Sie die `configService.get()`-Methode, um eine einfache Umgebungsvariable abzurufen, indem Sie den Variablennamen übergeben. Sie können TypeScript-Typ-Hinweise geben, indem Sie den Typ übergeben, wie oben gezeigt (z.B. `get<string>(...)`). Die `get()`-Methode kann auch ein verschachteltes benutzerdefiniertes Konfigurationsobjekt durchlaufen (erstellt über eine benutzerdefinierte Konfigurationsdatei), wie im zweiten Beispiel oben gezeigt.

Sie können auch das gesamte verschachtelte benutzerdefinierte Konfigurationsobjekt mit einer Schnittstelle als Typ-Hinweis abrufen:

```typescript
interface DatabaseConfig {
  host: string;
  port: number;
}

const dbConfig = this.configService.get<DatabaseConfig>('database');

// Sie können jetzt `dbConfig.port` und `dbConfig.host` verwenden
const port = dbConfig.port;
```

Die `get()`-Methode nimmt auch ein optionales zweites Argument, das einen Standardwert definiert, der zurückgegeben wird, wenn der Schlüssel nicht existiert:

```typescript
// "localhost" verwenden, wenn "database.host" nicht definiert ist
const dbHost = this.configService.get<string>('database.host', 'localhost');
```

Der `ConfigService` hat zwei optionale Generika (Typ-Argumente). Das erste soll verhindern, dass auf eine Konfigurationseigenschaft zugegriffen wird, die nicht existiert. Verwenden Sie es wie unten gezeigt:

```typescript
interface EnvironmentVariables {
  PORT: number;
  TIMEOUT: string;
}

// irgendwo im Code
constructor(private configService: ConfigService<EnvironmentVariables>) {
  const port = this.configService.get('PORT', { infer: true });

  // TypeScript-Fehler: Dies ist ungültig, da die URL-Eigenschaft in EnvironmentVariables nicht definiert ist
  const url = this.configService.get('URL', { infer: true });
}
```

Mit der `infer`-Eigenschaft auf `true` gesetzt, wird die `ConfigService#get`-Methode den Eigenschaftstyp basierend auf der Schnittstelle automatisch ableiten, sodass beispielsweise `typeof port === "number"` (wenn Sie nicht das `strictNullChecks`-Flag von TypeScript verwenden), da `PORT` in der `EnvironmentVariables`-Schnittstelle einen Zahlentyp hat.

Auch mit der `infer`-Funktion können Sie den Typ einer Eigenschaft eines verschachtelten benutzerdefinierten Konfigurationsobjekts ableiten, selbst wenn Sie Punktnotation verwenden:

```typescript
constructor(private configService: ConfigService<{ database: { host: string } }>) {
  const dbHost = this.configService.get('database.host', { infer: true })!;
  // typeof dbHost === "string"                                          |
  //                                                                     +--> Non-Null Assertion Operator
}
```

Das zweite Generikum baut auf dem ersten auf und dient als Typassertion, um alle undefined-Typen zu entfernen, die Methoden von `ConfigService` zurückgeben können, wenn `strictNullChecks` aktiviert ist. Zum Beispiel:

```typescript
// ...
constructor(private configService: ConfigService<{ PORT: number }, true>) {
  //                                                               ^^^^
  const port = this.configService.get('PORT', { infer: true });
  //    ^^^ Der Typ von port wird 'number' sein, sodass Sie keine TypeScript-Typ-Assertions mehr benötigen
}
```

### Konfigurations-Namensräume

Das `ConfigModule` ermöglicht es Ihnen, mehrere benutzerdefinierte Konfigurationsdateien zu definieren und zu laden, wie im Abschnitt `Benutzerdefinierte Konfigurationsdateien` oben gezeigt. Sie können komplexe Konfigurationsobjekthierarchien mit verschachtelten Konfigurationsobjekten verwalten, wie in diesem Abschnitt gezeigt. Alternativ können Sie ein "namensbasiertes" Konfigurationsobjekt mit der `registerAs()`-Funktion zurückgeben:

```typescript
// config/database.config.ts

export default registerAs('database', () => ({
  host: process.env.DATABASE_HOST,
  port: process.env.DATABASE_PORT || 5432
}));
```

### HINWEIS
Die `registerAs`-Funktion wird aus dem `@nestjs/config`-Paket exportiert.

Laden Sie eine namensbasierte Konfiguration mit der `load`-Eigenschaft der `forRoot()`-Methode des Optionsobjekts, genauso wie Sie eine benutzerdefinierte Konfigurationsdatei laden:

```typescript
import databaseConfig from './config/database.config';

@Module({
  imports: [
    ConfigModule.forRoot({
      load: [databaseConfig],
    }),
  ],
})
export class AppModule {}
```

Um den `host`-Wert aus dem `database`-Namensraum abzurufen, verwenden Sie die Punktnotation. Verwenden Sie `database` als Präfix für den Eigenschaftsnamen, entsprechend dem Namen des Namensraums (der als erstes Argument an die `registerAs()`-Funktion übergeben wird):

```typescript
const dbHost = this.configService.get<string>('database.host');
```

Eine sinnvolle Alternative ist es, den `database`-Namensraum direkt zu injizieren. Dies ermöglicht es uns, von der starken Typisierung zu profitieren:

```typescript
constructor(
  @Inject(databaseConfig.KEY)
  private dbConfig: ConfigType<typeof databaseConfig>,
) {}
```

### HINWEIS
Der `ConfigType` wird aus dem `@nestjs/config`-Paket exportiert.

### Umgebungsvariablen zwischenspeichern

Da der Zugriff auf `process.env` langsam sein kann, können Sie die `cache`-Eigenschaft des Optionsobjekts, das an `ConfigModule.forRoot()` übergeben wird, setzen, um die Leistung der `ConfigService#get`-Methode zu erhöhen, wenn es um Variablen geht, die in `process.env` gespeichert sind:

```typescript
ConfigModule.forRoot({
  cache: true,
});
```

### Partielle Registrierung

Bisher haben wir Konfigurationsdateien in unserem Hauptmodul (z.B. `AppModule`) mit der `forRoot()`-Methode verarbeitet. Vielleicht haben Sie eine komplexere Projektstruktur mit feature-spezifischen Konfigurationsdateien, die in mehreren verschiedenen Verzeichnissen gespeichert sind. Anstatt all diese Dateien im Hauptmodul zu laden, bietet das `@nestjs/config`-Paket eine Funktion namens partielle Registrierung, die nur die Konfigurationsdateien referenziert, die mit jedem Feature-Modul verbunden sind. Verwenden Sie die `forFeature()`-Methode innerhalb eines Feature-Moduls, um diese partielle Registrierung durchzuführen:

```typescript
import databaseConfig from './config/database.config';

@Module({
  imports: [ConfigModule.forFeature(databaseConfig)],
})
export class DatabaseModule {}
```

### WARNUNG
Unter bestimmten Umständen müssen Sie möglicherweise auf Eigenschaften zugreifen, die über die partielle Registrierung geladen wurden, indem Sie den `onModuleInit()`-Hook verwenden, anstatt in einem Konstruktor. Dies liegt daran, dass die `forFeature()`-Methode während der Modulinitialisierung ausgeführt wird und die Reihenfolge der Modulinitialisierung unbestimmt ist. Wenn Sie auf Werte zugreifen, die auf diese Weise von einem anderen Modul geladen wurden, in einem Konstruktor, ist das Modul, von dem die Konfiguration abhängt, möglicherweise noch nicht initialisiert. Die `onModuleInit()`-Methode wird erst ausgeführt, nachdem alle Module, von denen sie abhängt, initialisiert wurden, sodass diese Technik sicher ist.

### Schema-Validierung

Es ist übliche Praxis, während des Anwendungsstarts eine Ausnahme zu werfen, wenn erforderliche Umgebungsvariablen nicht bereitgestellt wurden oder bestimmte Validierungsregeln nicht erfüllen. Das `@nestjs/config`-Paket ermöglicht zwei verschiedene Möglichkeiten, dies zu tun:

- Joi-Built-in-Validator. Mit Joi definieren Sie ein Objektschema und validieren JavaScript-Objekte dagegen.
- Eine benutzerdefinierte `validate()`-Funktion, die Umgebungsvariablen als Eingabe übernimmt.

Um Joi zu verwenden, müssen wir das Joi-Paket installieren:

```bash
$ npm install --save joi
```

Jetzt können wir ein Joi-Validierungsschema definieren und es über die `validationSchema`-Eigenschaft des Optionsobjekts der `forRoot()`-Methode übergeben:

```typescript
// app.module.ts

import * as Joi from 'joi';

@Module({
  imports: [
    ConfigModule.forRoot({
      validationSchema: Joi.object({
        NODE_ENV: Joi.string()
          .valid('development', 'production', 'test', 'provision')
          .default('development'),
        PORT: Joi.number().port().default(3000),
      }),
    }),
  ],
})
export class AppModule {}
```

Standardmäßig sind alle Schema-Schlüssel optional. Hier setzen wir Standardwerte für `NODE_ENV` und `PORT`, die verwendet werden, wenn wir diese Variablen nicht in der Umgebung bereitstellen (.env-Datei oder Laufzeitumgebung). Alternativ können wir die `required()`-

Validierungsmethode verwenden, um zu verlangen, dass ein Wert in der Umgebung (.env-Datei oder Laufzeitumgebung) definiert sein muss. In diesem Fall wird der Validierungsschritt eine Ausnahme werfen, wenn wir die Variable nicht in der Umgebung bereitstellen. Weitere Informationen darüber, wie Validierungsschemas erstellt werden, finden Sie in den Joi-Validierungsmethoden.

Standardmäßig sind unbekannte Umgebungsvariablen (Umgebungsvariablen, deren Schlüssel nicht im Schema vorhanden sind) erlaubt und lösen keine Validierungsausnahme aus. Standardmäßig werden alle Validierungsfehler gemeldet. Sie können dieses Verhalten ändern, indem Sie ein Optionsobjekt über den Schlüssel `validationOptions` des `forRoot()`-Optionsobjekts übergeben. Dieses Optionsobjekt kann beliebige Standard-Validierungsoptions-Eigenschaften enthalten, die von Joi-Validierungsoptionen bereitgestellt werden. Zum Beispiel, um die beiden oben genannten Einstellungen umzukehren, übergeben Sie die Optionen wie folgt:

```typescript
// app.module.ts

import * as Joi from 'joi';

@Module({
  imports: [
    ConfigModule.forRoot({
      validationSchema: Joi.object({
        NODE_ENV: Joi.string()
          .valid('development', 'production', 'test', 'provision')
          .default('development'),
        PORT: Joi.number().port().default(3000),
      }),
      validationOptions: {
        allowUnknown: false,
        abortEarly: true,
      },
    }),
  ],
})
export class AppModule {}
```

Das `@nestjs/config`-Paket verwendet standardmäßig die folgenden Einstellungen:

- `allowUnknown`: Steuert, ob unbekannte Schlüssel in den Umgebungsvariablen erlaubt sind oder nicht. Standardwert ist `true`.
- `abortEarly`: Wenn `true`, wird die Validierung beim ersten Fehler abgebrochen; wenn `false`, werden alle Fehler zurückgegeben. Standardwert ist `false`.

Beachten Sie, dass, wenn Sie sich entscheiden, ein `validationOptions`-Objekt zu übergeben, alle Einstellungen, die Sie nicht explizit übergeben, auf die Joi-Standardwerte (nicht die `@nestjs/config`-Standardwerte) zurückgesetzt werden. Zum Beispiel, wenn Sie `allowUnknown` in Ihrem benutzerdefinierten `validationOptions`-Objekt nicht angeben, wird es den Joi-Standardwert von `false` haben. Daher ist es wahrscheinlich am sichersten, beide dieser Einstellungen in Ihrem benutzerdefinierten Objekt anzugeben.

### Benutzerdefinierte validate-Funktion

Alternativ können Sie eine synchronisierende `validate`-Funktion angeben, die ein Objekt enthält, das die Umgebungsvariablen (aus der .env-Datei und dem Prozess) enthält, und ein Objekt zurückgibt, das validierte Umgebungsvariablen enthält, sodass Sie diese bei Bedarf konvertieren/mutieren können. Wenn die Funktion einen Fehler wirft, wird das Bootstrapping der Anwendung verhindert.

In diesem Beispiel verwenden wir die `class-transformer` und `class-validator` Pakete. Zuerst müssen wir:

- eine Klasse mit Validierungseinschränkungen definieren,
- eine `validate`-Funktion definieren, die die `plainToInstance`- und `validateSync`-Funktionen verwendet.

```typescript
// env.validation.ts

import { plainToInstance } from 'class-transformer';
import { IsEnum, IsNumber, Max, Min, validateSync } from 'class-validator';

enum Environment {
  Development = "development",
  Production = "production",
  Test = "test",
  Provision = "provision",
}

class EnvironmentVariables {
  @IsEnum(Environment)
  NODE_ENV: Environment;

  @IsNumber()
  @Min(0)
  @Max(65535)
  PORT: number;
}

export function validate(config: Record<string, unknown>) {
  const validatedConfig = plainToInstance(
    EnvironmentVariables,
    config,
    { enableImplicitConversion: true },
  );
  const errors = validateSync(validatedConfig, { skipMissingProperties: false });

  if (errors.length > 0) {
    throw new Error(errors.toString());
  }
  return validatedConfig;
}
```

Mit dieser Einrichtung verwenden wir die `validate`-Funktion als Konfigurationsoption des `ConfigModule`, wie folgt:

```typescript
// app.module.ts

import { validate } from './env.validation';

@Module({
  imports: [
    ConfigModule.forRoot({
      validate,
    }),
  ],
})
export class AppModule {}
```

### Benutzerdefinierte Getter-Funktionen

`ConfigService` definiert eine generische `get()`-Methode, um einen Konfigurationswert nach Schlüssel abzurufen. Wir können auch Getter-Funktionen hinzufügen, um einen etwas natürlicheren Programmierstil zu ermöglichen:

```typescript
@Injectable()
export class ApiConfigService {
  constructor(private configService: ConfigService) {}

  get isAuthEnabled(): boolean {
    return this.configService.get('AUTH_ENABLED') === 'true';
  }
}
```

Nun können wir die Getter-Funktion wie folgt verwenden:

```typescript
// app.service.ts

@Injectable()
export class AppService {
  constructor(apiConfigService: ApiConfigService) {
    if (apiConfigService.isAuthEnabled) {
      // Authentifizierung ist aktiviert
    }
  }
}
```

### Hook für geladene Umgebungsvariablen

Wenn eine Modulkonfiguration von den Umgebungsvariablen abhängt und diese Variablen aus der `.env`-Datei geladen werden, können Sie den `ConfigModule.envVariablesLoaded`-Hook verwenden, um sicherzustellen, dass die Datei geladen wurde, bevor Sie mit dem `process.env`-Objekt interagieren:

```typescript
export async function getStorageModule() {
  await ConfigModule.envVariablesLoaded;
  return process.env.STORAGE === 'S3' ? S3StorageModule : DefaultStorageModule;
}
```

Diese Konstruktion garantiert, dass, nachdem die `ConfigModule.envVariablesLoaded`-Promise aufgelöst wurde, alle Konfigurationsvariablen geladen sind.

### Bedingte Modulk

onfiguration

Manchmal möchten Sie möglicherweise ein Modul bedingt laden und die Bedingung in einer Umgebungsvariablen angeben. Glücklicherweise bietet `@nestjs/config` ein `ConditionalModule`, das genau das ermöglicht:

```typescript
@Module({
  imports: [ConfigModule.forRoot(), ConditionalModule.registerWhen(FooModule, 'USE_FOO')],
})
export class AppModule {}
```

Das obige Modul würde das `FooModule` nur laden, wenn in der `.env`-Datei kein falscher Wert für die Umgebungsvariable `USE_FOO` angegeben ist. Sie können auch eine benutzerdefinierte Bedingung selbst übergeben, eine Funktion, die die `process.env`-Referenz erhält und einen booleschen Wert zurückgeben sollte:

```typescript
@Module({
  imports: [ConfigModule.forRoot(), ConditionalModule.registerWhen(FooBarModule, (env: NodeJS.ProcessEnv) => !!env['foo'] && !!env['bar'])],
})
export class AppModule {}
```

Es ist wichtig sicherzustellen, dass beim Verwenden des `ConditionalModule` auch das `ConfigModule` in der Anwendung geladen ist, sodass der `ConfigModule.envVariablesLoaded`-Hook ordnungsgemäß referenziert und verwendet werden kann. Wenn der Hook nicht innerhalb von 5 Sekunden oder einer vom Benutzer festgelegten Zeitüberschreitung in Millisekunden im dritten Optionsparameter der `registerWhen`-Methode aktiviert wird, wirft das `ConditionalModule` einen Fehler und Nest wird das Starten der Anwendung abbrechen.

### Erweiterbare Variablen

Das `@nestjs/config`-Paket unterstützt die Erweiterung von Umgebungsvariablen. Mit dieser Technik können Sie verschachtelte Umgebungsvariablen erstellen, bei denen eine Variable innerhalb der Definition einer anderen Variable referenziert wird. Zum Beispiel:

```
APP_URL=mywebsite.com
SUPPORT_EMAIL=support@${APP_URL}
```

Mit dieser Konstruktion wird die Variable `SUPPORT_EMAIL` auf 'support@mywebsite.com' aufgelöst. Beachten Sie die Verwendung der `${...}`-Syntax, um den Wert der Variablen `APP_URL` innerhalb der Definition von `SUPPORT_EMAIL` aufzulösen.

### HINWEIS
Für diese Funktion verwendet das `@nestjs/config`-Paket intern `dotenv-expand`.

Aktivieren Sie die Erweiterung von Umgebungsvariablen, indem Sie die `expandVariables`-Eigenschaft im Optionsobjekt, das an die `forRoot()`-Methode des `ConfigModule` übergeben wird, wie unten gezeigt, setzen:

```typescript
// app.module.ts

@Module({
  imports: [
    ConfigModule.forRoot({
      // ...
      expandVariables: true,
    }),
  ],
})
export class AppModule {}
```

### Verwendung in der `main.ts`

Obwohl unsere Konfiguration in einem Dienst gespeichert ist, kann sie dennoch in der `main.ts`-Datei verwendet werden. Auf diese Weise können Sie sie verwenden, um Variablen wie den Anwendungsport oder den CORS-Host zu speichern.

Um darauf zuzugreifen, müssen Sie die `app.get()`-Methode verwenden, gefolgt von der Dienstreferenz:

```typescript
const configService = app.get(ConfigService);
```

Sie können sie dann wie gewohnt verwenden, indem Sie die `get`-Methode mit dem Konfigurationsschlüssel aufrufen:

```typescript
const port = configService.get('PORT');
```

