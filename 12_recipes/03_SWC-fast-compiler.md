# SWC / SWC

SWC (Speedy Web Compiler) ist eine erweiterbare Plattform auf Basis von Rust, die sowohl zur Kompilierung als auch zum Bündeln verwendet werden kann. Die Verwendung von SWC mit der Nest-CLI ist eine großartige und einfache Möglichkeit, Ihren Entwicklungsprozess erheblich zu beschleunigen.

**HINWEIS / HINT**
SWC ist etwa 20-mal schneller als der Standard-TypeScript-Compiler.

## Installation / Installation

Um loszulegen, installieren Sie zuerst ein paar Pakete:

```shell
$ npm i --save-dev @swc/cli @swc/core
```

## Erste Schritte / Getting started

Sobald der Installationsprozess abgeschlossen ist, können Sie den SWC-Builder mit der Nest-CLI wie folgt verwenden:

```shell
$ nest start -b swc
# ODER / OR nest start --builder swc
```

**HINWEIS / HINT**
Wenn Ihr Repository ein Monorepo ist, sehen Sie sich diesen Abschnitt an.

Anstelle des -b-Flags können Sie auch einfach die Eigenschaft `compilerOptions.builder` in Ihrer `nest-cli.json`-Datei auf "swc" setzen, wie folgt:

```json
{
  "compilerOptions": {
    "builder": "swc"
  }
}
```

Um das Verhalten des Builders anzupassen, können Sie ein Objekt mit zwei Attributen, `type` ("swc") und `options`, wie folgt übergeben:

```json
"compilerOptions": {
  "builder": {
    "type": "swc",
    "options": {
      "swcrcPath": "infrastructure/.swcrc"
    }
  }
}
```

Um die Anwendung im Watch-Modus auszuführen, verwenden Sie den folgenden Befehl:

```shell
$ nest start -b swc -w
# ODER / OR nest start --builder swc --watch
```

## Typprüfung / Type checking

SWC führt keine Typprüfung selbst durch (im Gegensatz zum Standard-TypeScript-Compiler). Um dies zu aktivieren, müssen Sie das `--type-check`-Flag verwenden:

```shell
$ nest start -b swc --type-check
```

Dieser Befehl weist die Nest-CLI an, `tsc` im `noEmit`-Modus zusammen mit SWC auszuführen, was die Typprüfung asynchron durchführt. Auch hier können Sie anstelle des `--type-check`-Flags einfach die Eigenschaft `compilerOptions.typeCheck` in Ihrer `nest-cli.json`-Datei auf `true` setzen, wie folgt:

```json
{
  "compilerOptions": {
    "builder": "swc",
    "typeCheck": true
  }
}
```

## CLI-Plugins (SWC) / CLI Plugins (SWC)

Das `--type-check`-Flag führt automatisch NestJS CLI-Plugins aus und erstellt eine serialisierte Metadatendatei, die dann zur Laufzeit von der Anwendung geladen werden kann.

## SWC-Konfiguration / SWC configuration

Der SWC-Builder ist vorkonfiguriert, um die Anforderungen von NestJS-Anwendungen zu erfüllen. Sie können die Konfiguration jedoch anpassen, indem Sie eine `.swcrc`-Datei im Stammverzeichnis erstellen und die Optionen nach Ihren Wünschen anpassen:

```json
{
  "$schema": "https://json.schemastore.org/swcrc",
  "sourceMaps": true,
  "jsc": {
    "parser": {
      "syntax": "typescript",
      "decorators": true,
      "dynamicImport": true
    },
    "baseUrl": "./"
  },
  "minify": false
}
```

## Monorepo / Monorepo

Wenn Ihr Repository ein Monorepo ist, müssen Sie anstelle des SWC-Builders Webpack konfigurieren, um den `swc-loader` zu verwenden.

Zuerst installieren wir das erforderliche Paket:

```shell
$ npm i --save-dev swc-loader
```

Sobald die Installation abgeschlossen ist, erstellen Sie eine `webpack.config.js`-Datei im Stammverzeichnis Ihrer Anwendung mit folgendem Inhalt:

```javascript
const swcDefaultConfig = require('@nestjs/cli/lib/compiler/defaults/swc-defaults').swcDefaultsFactory().swcOptions;

module.exports = {
  module: {
    rules: [
      {
        test: /\.ts$/,
        exclude: /node_modules/,
        use: {
          loader: 'swc-loader',
          options: swcDefaultConfig,
        },
      },
    ],
  },
};
```

## Monorepo und CLI-Plugins / Monorepo and CLI plugins

Wenn Sie CLI-Plugins verwenden, wird `swc-loader` diese nicht automatisch laden. Stattdessen müssen Sie eine separate Datei erstellen, die sie manuell lädt. Dazu deklarieren Sie eine `generate-metadata.ts`-Datei in der Nähe der `main.ts`-Datei mit folgendem Inhalt:

```typescript
import { PluginMetadataGenerator } from '@nestjs/cli/lib/compiler/plugins';
import { ReadonlyVisitor } from '@nestjs/swagger/dist/plugin';

const generator = new PluginMetadataGenerator();
generator.generate({
  visitors: [new ReadonlyVisitor({ introspectComments: true, pathToSource: __dirname })],
  outputDir: __dirname,
  watch: true,
  tsconfigPath: 'apps/<name>/tsconfig.app.json',
});
```

**HINWEIS / HINT**
In diesem Beispiel haben wir das `@nestjs/swagger`-Plugin verwendet, aber Sie können jedes Plugin Ihrer Wahl verwenden.

Die `generate()`-Methode akzeptiert folgende Optionen:

- `watch`: Ob das Projekt auf Änderungen überwacht werden soll.
- `tsconfigPath`: Pfad zur `tsconfig.json`-Datei. Relativ zum aktuellen Arbeitsverzeichnis (`process.cwd()`).
- `outputDir`: Pfad zum Verzeichnis, in dem die Metadatendatei gespeichert wird.
- `visitors`: Ein Array von Besuchern, die zur Generierung der Metadaten verwendet werden.
- `filename`: Der Name der Metadatendatei. Standardmäßig `metadata.ts`.
- `printDiagnostics`: Ob Diagnosen in der Konsole ausgegeben werden sollen. Standardmäßig `true`.

Abschließend können Sie das `generate-metadata`-Skript in einem separaten Terminalfenster mit folgendem Befehl ausführen:

```shell
$ npx ts-node src/generate-metadata.ts
# ODER / OR npx ts-node apps/{YOUR_APP}/src/generate-metadata.ts
```

## Häufige Fallstricke / Common pitfalls

Wenn Sie TypeORM/MikroORM oder ein anderes ORM in Ihrer Anwendung verwenden, können Sie auf Probleme mit zirkulären Importen stoßen. SWC kann zirkuläre Importe nicht gut handhaben, daher sollten Sie den folgenden Workaround verwenden:

```typescript
@Entity()
export class User {
  @OneToOne(() => Profile, (profile) => profile.user)
  profile: Relation<Profile>; // <--- siehe "Relation<>"-Typ hier anstelle von nur "Profile"
}
```

**HINWEIS / HINT**
Der `Relation`-Typ wird aus dem `typeorm`-Paket exportiert.

Dies verhindert, dass der Typ der Eigenschaft im transpilierten Code in den Eigenschaftsmetadaten gespeichert wird, was Probleme mit zirkulären Abhängigkeiten verhindert.

Wenn Ihr ORM keinen ähnlichen Workaround bietet, können Sie den Wrapper-Typ selbst definieren:

```typescript
/**
 * Wrapper-Typ zur Umgehung des ESM-Modul-Problems bei zirkulären Abhängigkeiten,
 * verursacht durch das Speichern des Typs der Eigenschaft in den Reflektionsmetadaten.
 */
export type WrapperType<T> = T; // WrapperType === Relation
```

Für alle zirkulären Abhängigkeitsinjektionen in Ihrem Projekt müssen Sie auch den benutzerdefinierten Wrapper-Typ wie oben beschrieben verwenden:

```typescript
@Injectable()
export class UserService {
  constructor(
    @Inject(forwardRef(() => ProfileService))
    private readonly profileService: WrapperType<ProfileService>,
  ) {}
}
```

## Jest + SWC / Jest + SWC

Um SWC mit Jest zu verwenden, müssen Sie die folgenden Pakete installieren:

```shell
$ npm i --save-dev jest @swc/core @swc/jest
```

Sobald die Installation abgeschlossen ist, aktualisieren Sie die Datei `package.json/jest.config.js` (abhängig von Ihrer Konfiguration) mit folgendem Inhalt:

```json
{
  "jest": {
    "transform": {
      "^.+\\.(t|j)s?$": ["@swc/jest"]
    }
  }
}
```

Zusätzlich müssen Sie die folgenden Transform-Eigenschaften zu Ihrer `.swcrc`-Datei hinzufügen: `legacyDecorator`, `decoratorMetadata`:

```json
{
  "$schema": "https://json.schemastore.org/swcrc",
  "sourceMaps": true,
  "jsc": {
    "parser": {
      "syntax": "typescript",
      "decorators": true,
      "dynamicImport": true
    },
    "transform": {
      "legacyDecorator": true,
      "decoratorMetadata": true
    },
    "baseUrl": "./"
  },
  "minify": false
}
```

Wenn Sie NestJS CLI-Plugins in Ihrem Projekt verwenden, müssen Sie `PluginMetadataGenerator` manuell ausführen. Navigieren Sie zu diesem Abschnitt, um mehr zu erfahren.

## Vitest / Vitest

Vitest ist ein schneller und leichtgewichtiger Testrunner, der für die Arbeit mit Vite entwickelt wurde. Es bietet eine moderne,

 schnelle und einfach zu bedienende Testlösung, die in NestJS-Projekte integriert werden kann.

## Installation / Installation

Um loszulegen, installieren Sie zuerst die erforderlichen Pakete:

```shell
$ npm i --save-dev vitest unplugin-swc @swc/core @vitest/coverage-v8
```

## Konfiguration / Configuration

Erstellen Sie eine `vitest.config.ts`-Datei im Stammverzeichnis Ihrer Anwendung mit folgendem Inhalt:

```typescript
import swc from 'unplugin-swc';
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    globals: true,
    root: './',
  },
  plugins: [
    // Dies ist erforderlich, um die Testdateien mit SWC zu erstellen
    swc.vite({
      // Setzen Sie explizit den Modultyp, um zu vermeiden, dass dieser Wert aus einer `.swcrc`-Konfigurationsdatei übernommen wird
      module: { type: 'es6' },
    }),
  ],
});
```

Diese Konfigurationsdatei richtet die Vitest-Umgebung, das Stammverzeichnis und das SWC-Plugin ein. Sie sollten auch eine separate Konfigurationsdatei für E2E-Tests erstellen, mit einem zusätzlichen `include`-Feld, das den Testpfad-Regex angibt:

```typescript
import swc from 'unplugin-swc';
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    include: ['**/*.e2e-spec.ts'],
    globals: true,
    root: './',
  },
  plugins: [swc.vite()],
});
```

Zusätzlich können Sie die `alias`-Optionen setzen, um TypeScript-Pfade in Ihren Tests zu unterstützen:

```typescript
import swc from 'unplugin-swc';
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    include: ['**/*.e2e-spec.ts'],
    globals: true,
    alias: {
      '@src': './src',
      '@test': './test',
    },
    root: './',
  },
  resolve: {
    alias: {
      '@src': './src',
      '@test': './test',
    },
  },
  plugins: [swc.vite()],
});
```

## Aktualisieren von Importen in E2E-Tests / Update imports in E2E tests

Ändern Sie alle E2E-Testimporte von `import * as request from 'supertest'` zu `import request from 'supertest'`. Dies ist notwendig, da Vitest, wenn es mit Vite gebündelt wird, einen Standardimport für `supertest` erwartet. Die Verwendung eines Namespace-Imports kann in diesem speziellen Setup Probleme verursachen.

Zuletzt aktualisieren Sie die Testskripte in Ihrer `package.json`-Datei wie folgt:

```json
{
  "scripts": {
    "test": "vitest run",
    "test:watch": "vitest",
    "test:cov": "vitest run --coverage",
    "test:debug": "vitest --inspect-brk --inspect --logHeapUsage --threads=false",
    "test:e2e": "vitest run --config ./vitest.config.e2e.ts"
  }
}
```

Diese Skripte konfigurieren Vitest zum Ausführen von Tests, Überwachen von Änderungen, Erzeugen von Codeabdeckungsberichten und Debuggen. Das `test:e2e`-Skript ist speziell zum Ausführen von E2E-Tests mit einer benutzerdefinierten Konfigurationsdatei.

Mit diesem Setup können Sie nun die Vorteile der Verwendung von Vitest in Ihrem NestJS-Projekt genießen, einschließlich schnellerer Testausführung und einer moderneren Testerfahrung.

**HINWEIS / HINT**
Sie können sich ein funktionierendes Beispiel in diesem Repository ansehen.
