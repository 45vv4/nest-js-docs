## Dynamische Module / Dynamic modules

Das Kapitel über Module behandelt die Grundlagen von Nest-Modulen und enthält eine kurze Einführung in dynamische Module. Dieses Kapitel erweitert das Thema dynamische Module. Nach Abschluss sollten Sie ein gutes Verständnis dafür haben, was sie sind und wie und wann sie verwendet werden.

#### Einführung / Introduction

Die meisten Anwendungsbeispiele im Abschnitt "Übersicht" der Dokumentation verwenden reguläre oder statische Module. Module definieren Gruppen von Komponenten wie Anbieter und Controller, die zusammen als modularer Teil einer gesamten Anwendung funktionieren. Sie bieten einen Ausführungskontext oder Geltungsbereich für diese Komponenten. Beispielsweise sind Anbieter, die in einem Modul definiert sind, für andere Mitglieder des Moduls sichtbar, ohne dass sie exportiert werden müssen. Wenn ein Anbieter außerhalb eines Moduls sichtbar sein muss, wird er zuerst aus seinem Hostmodul exportiert und dann in sein konsumierendes Modul importiert.

Lassen Sie uns ein bekanntes Beispiel durchgehen.

Zuerst definieren wir ein UsersModule, um einen UsersService bereitzustellen und zu exportieren. UsersModule ist das Hostmodul für UsersService.

```typescript
import { Module } from '@nestjs/common';
import { UsersService } from './users.service';

@Module({
  providers: [UsersService],
  exports: [UsersService],
})
export class UsersModule {}
```

Als nächstes definieren wir ein AuthModule, das UsersModule importiert und die exportierten Anbieter von UsersModule im AuthModule verfügbar macht:

```typescript
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { UsersModule } from '../users/users.module';

@Module({
  imports: [UsersModule],
  providers: [AuthService],
  exports: [AuthService],
})
export class AuthModule {}
```

Diese Konstrukte ermöglichen es uns, UsersService beispielsweise in AuthService zu injizieren, das im AuthModule gehostet wird:

```typescript
import { Injectable } from '@nestjs/common';
import { UsersService } from '../users/users.service';

@Injectable()
export class AuthService {
  constructor(private usersService: UsersService) {}
  /*
    Implementierung, die this.usersService verwendet
  */
}
```

Wir bezeichnen dies als statische Modulbindung. Alle Informationen, die Nest benötigt, um die Module miteinander zu verknüpfen, wurden bereits in den Host- und konsumierenden Modulen deklariert. Lassen Sie uns aufschlüsseln, was während dieses Prozesses passiert. Nest macht UsersService im AuthModule verfügbar, indem:

1. UsersModule instanziiert wird, einschließlich der transitiven Importierung anderer Module, die UsersModule selbst konsumiert, und der transitiven Auflösung von Abhängigkeiten (siehe Benutzerdefinierte Anbieter).
2. AuthModule instanziiert wird und die exportierten Anbieter von UsersModule den Komponenten im AuthModule zur Verfügung gestellt werden (als wären sie im AuthModule deklariert).
3. Eine Instanz von UsersService in AuthService injiziert wird.

### Anwendungsfall für dynamische Module / Dynamic module use case

Bei der statischen Modulbindung gibt es keine Möglichkeit für das konsumierende Modul, zu beeinflussen, wie Anbieter aus dem Hostmodul konfiguriert werden. Warum ist das wichtig? Betrachten Sie den Fall, in dem wir ein allgemeines Modul haben, das sich in verschiedenen Anwendungsfällen unterschiedlich verhalten muss. Dies ist analog zum Konzept eines "Plugins" in vielen Systemen, bei denen eine generische Funktionalität eine Konfiguration benötigt, bevor sie von einem Verbraucher verwendet werden kann.

Ein gutes Beispiel bei Nest ist ein Konfigurationsmodul. Viele Anwendungen finden es nützlich, Konfigurationsdetails mithilfe eines Konfigurationsmoduls zu externalisieren. Dies erleichtert es, die Anwendungseinstellungen in verschiedenen Bereitstellungen dynamisch zu ändern: z. B. eine Entwicklungsdatenbank für Entwickler, eine Staging-Datenbank für die Staging-/Testumgebung usw. Indem die Verwaltung der Konfigurationsparameter einem Konfigurationsmodul überlassen wird, bleibt der Quellcode der Anwendung unabhängig von den Konfigurationsparametern.

Die Herausforderung besteht darin, dass das Konfigurationsmodul selbst, da es generisch ist (ähnlich wie ein "Plugin"), von seinem konsumierenden Modul angepasst werden muss. Hier kommen dynamische Module ins Spiel. Mithilfe dynamischer Modulfunktionen können wir unser Konfigurationsmodul dynamisch gestalten, sodass das konsumierende Modul eine API verwenden kann, um zu steuern, wie das Konfigurationsmodul beim Import angepasst wird.

Mit anderen Worten, dynamische Module bieten eine API zum Importieren eines Moduls in ein anderes und zur Anpassung der Eigenschaften und des Verhaltens dieses Moduls beim Import, im Gegensatz zu den statischen Bindungen, die wir bisher gesehen haben.

### Beispiel für ein Konfigurationsmodul / Config module example

Wir verwenden die grundlegende Version des Beispielcodes aus dem Kapitel zur Konfiguration für diesen Abschnitt. Die vollständige Version am Ende dieses Kapitels ist hier als Arbeitsbeispiel verfügbar.

Unsere Anforderung ist es, dass ConfigModule ein Optionsobjekt akzeptiert, um es anzupassen. Hier ist die Funktion, die wir unterstützen möchten. Das einfache Beispiel legt den Speicherort der .env-Datei im Projektstammverzeichnis fest. Angenommen, wir möchten dies konfigurierbar machen, sodass Sie Ihre .env-Dateien in jedem gewünschten Ordner verwalten können. Zum Beispiel, stellen Sie sich vor, Sie möchten Ihre verschiedenen .env-Dateien in einem Ordner unter dem Projektstamm namens config (d. h. ein Unterordner zu src) speichern. Sie möchten in der Lage sein, verschiedene Ordner zu wählen, wenn Sie das ConfigModule in verschiedenen Projekten verwenden.

Dynamische Module geben uns die Möglichkeit, Parameter an das importierte Modul zu übergeben, sodass wir sein Verhalten ändern können. Schauen wir uns an, wie das funktioniert. Es ist hilfreich, wenn wir vom Endziel ausgehend betrachten, wie dies aus der Perspektive des konsumierenden Moduls aussehen könnte, und dann rückwärts arbeiten. Zuerst überprüfen wir schnell das Beispiel für das statische Importieren des ConfigModule (d. h. ein Ansatz, der keine Möglichkeit bietet, das Verhalten des importierten Moduls zu beeinflussen). Achten Sie genau auf das Imports-Array im @Module()-Dekorator:

```typescript
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { ConfigModule } from './config/config.module';

@Module({
  imports: [ConfigModule],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

Betrachten wir, wie ein dynamischer Modulimport, bei dem wir ein Konfigurationsobjekt übergeben, aussehen könnte. Vergleichen Sie den Unterschied im Imports-Array zwischen diesen beiden Beispielen:

```typescript
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { ConfigModule } from './config/config.module';

@Module({
  imports: [ConfigModule.register({ folder: './config' })],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

Schauen wir uns an, was im dynamischen Beispiel oben passiert. Was sind die beweglichen Teile?

- ConfigModule ist eine normale Klasse, sodass wir daraus schließen können, dass es eine statische Methode namens register() haben muss. Wir wissen, dass sie statisch ist, weil wir sie auf der ConfigModule-Klasse aufrufen, nicht auf einer Instanz der Klasse. Hinweis: Diese Methode, die wir bald erstellen werden, kann einen beliebigen Namen haben, aber konventionell sollten wir sie entweder forRoot() oder register() nennen.
- Die register()-Methode wird von uns definiert, sodass wir beliebige Eingabeargumente akzeptieren können. In diesem Fall akzeptieren wir ein einfaches Optionsobjekt mit geeigneten Eigenschaften, was der typische Fall ist.
- Wir können davon ausgehen, dass die register()-Methode etwas wie ein Modul zurückgeben muss, da ihr Rückgabewert in der vertrauten Imports-Liste erscheint, die wir bisher gesehen haben und die eine Liste von Modulen enthält.

Tatsächlich wird unsere register()-Methode ein DynamicModule zurückgeben. Ein dynamisches Modul ist nichts anderes als ein zur Laufzeit erstelltes Modul mit genau denselben Eigenschaften wie ein statisches Modul plus einer zusätzlichen Eigenschaft namens module. Lassen Sie uns schnell eine Beispieldeklaration für ein statisches Modul überprüfen, wobei wir besonders auf die Moduloptionen achten, die an den Dekorator übergeben werden:

```typescript
@Module({
  imports: [DogsModule],
  controllers: [CatsController],
  providers: [CatsService],
  exports: [CatsService]
})
```

Dynamische Module müssen ein Objekt mit genau demselben Interface zurückgeben, plus einer zusätzlichen Eigenschaft namens module. Die module-Eigenschaft dient als Name des Moduls und sollte mit dem Klassennamen des Moduls übereinstimmen, wie im folgenden Beispiel gezeigt.

HINWEIS
Für ein dynamisches Modul sind alle Eigenschaften des Moduloptionsobjekts optional, außer module.

Wie sieht es mit der statischen register()-Methode aus? Wir können nun sehen, dass ihre Aufgabe darin besteht, ein Objekt zurückzugeben, das das DynamicModule-Interface hat. Wenn

 wir es aufrufen, stellen wir effektiv ein Modul für die Imports-Liste zur Verfügung, ähnlich wie wir es im statischen Fall durch Auflisten eines Modulklassennamens tun würden. Mit anderen Worten, die API für dynamische Module gibt einfach ein Modul zurück, aber anstatt die Eigenschaften im @Module-Dekorator festzulegen, spezifizieren wir sie programmatisch.

Es gibt noch ein paar Details zu klären, um das Bild vollständig zu machen:

- Wir können nun feststellen, dass die Imports-Eigenschaft des @Module()-Dekorators nicht nur einen Modulklassennamen (z. B. imports: [UsersModule]) annehmen kann, sondern auch eine Funktion, die ein dynamisches Modul zurückgibt (z. B. imports: [ConfigModule.register(...)]).
- Ein dynamisches Modul kann selbst andere Module importieren. Wir werden dies in diesem Beispiel nicht tun, aber wenn das dynamische Modul von Anbietern anderer Module abhängt, würden Sie diese mit der optionalen Imports-Eigenschaft importieren. Auch dies ist genau analog zur Deklaration von Metadaten für ein statisches Modul mithilfe des @Module()-Dekorators.

Mit diesem Verständnis können wir uns nun ansehen, wie unsere dynamische ConfigModule-Deklaration aussehen muss. Lassen Sie uns einen Versuch wagen.

```typescript
import { DynamicModule, Module } from '@nestjs/common';
import { ConfigService } from './config.service';

@Module({})
export class ConfigModule {
  static register(): DynamicModule {
    return {
      module: ConfigModule,
      providers: [ConfigService],
      exports: [ConfigService],
    };
  }
}
```

Es sollte nun klar sein, wie die Teile zusammenpassen. Das Aufrufen von ConfigModule.register(...) gibt ein DynamicModule-Objekt mit Eigenschaften zurück, die im Wesentlichen dieselben sind wie die, die wir bisher als Metadaten über den @Module-Dekorator bereitgestellt haben.

HINWEIS
Importieren Sie DynamicModule aus @nestjs/common.

Unser dynamisches Modul ist jedoch noch nicht sehr interessant, da wir noch keine Fähigkeit eingeführt haben, es zu konfigurieren, wie wir es gerne möchten. Lassen Sie uns das als Nächstes angehen.

### Modulanpassung / Module configuration

Die offensichtliche Lösung zur Anpassung des Verhaltens des ConfigModule besteht darin, ihm ein Optionsobjekt in der statischen register()-Methode zu übergeben, wie wir oben vermutet haben. Schauen wir uns noch einmal die Imports-Eigenschaft unseres konsumierenden Moduls an:

```typescript
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { ConfigModule } from './config/config.module';

@Module({
  imports: [ConfigModule.register({ folder: './config' })],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

Das regelt das Übergeben eines Optionsobjekts an unser dynamisches Modul. Wie verwenden wir dann dieses Optionsobjekt im ConfigModule? Lassen Sie uns das kurz überlegen. Wir wissen, dass unser ConfigModule im Wesentlichen ein Host für die Bereitstellung und den Export eines injizierbaren Dienstes - des ConfigService - zur Verwendung durch andere Anbieter ist. Tatsächlich muss unser ConfigService das Optionsobjekt lesen, um sein Verhalten anzupassen. Angenommen, wir wissen irgendwie, wie wir die Optionen aus der register()-Methode in den ConfigService übergeben. Mit dieser Annahme können wir einige Änderungen am Dienst vornehmen, um sein Verhalten basierend auf den Eigenschaften des Optionsobjekts anzupassen. (Hinweis: Vorläufig, da wir noch nicht bestimmt haben, wie wir es übergeben, werden wir die Optionen einfach hartkodieren. Wir werden dies in einer Minute beheben).

```typescript
import { Injectable } from '@nestjs/common';
import * as dotenv from 'dotenv';
import * as fs from 'fs';
import * as path from 'path';
import { EnvConfig } from './interfaces';

@Injectable()
export class ConfigService {
  private readonly envConfig: EnvConfig;

  constructor() {
    const options = { folder: './config' };

    const filePath = `${process.env.NODE_ENV || 'development'}.env`;
    const envFile = path.resolve(__dirname, '../../', options.folder, filePath);
    this.envConfig = dotenv.parse(fs.readFileSync(envFile));
  }

  get(key: string): string {
    return this.envConfig[key];
  }
}
```

Jetzt weiß unser ConfigService, wie er die .env-Datei im angegebenen Ordner der Optionen findet.

Unsere verbleibende Aufgabe besteht darin, das Optionsobjekt aus dem Register-Schritt in unseren ConfigService zu injizieren. Und natürlich werden wir dafür die Abhängigkeitsinjektion verwenden. Dies ist ein wichtiger Punkt, also stellen Sie sicher, dass Sie ihn verstehen. Unser ConfigModule stellt ConfigService bereit. ConfigService wiederum hängt von dem Optionsobjekt ab, das nur zur Laufzeit bereitgestellt wird. Also müssen wir zur Laufzeit das Optionsobjekt zuerst an den Nest-IoC-Container binden und dann von Nest in unseren ConfigService injizieren lassen. Denken Sie daran, dass Anbieter nicht nur Dienste umfassen können, sondern auch beliebige Werte, sodass wir die Abhängigkeitsinjektion verwenden können, um ein einfaches Optionsobjekt zu handhaben.

Lassen Sie uns zuerst das Binden des Optionsobjekts an den IoC-Container angehen. Wir tun dies in unserer statischen register()-Methode. Denken Sie daran, dass wir ein Modul dynamisch konstruieren und eine der Eigenschaften eines Moduls seine Anbieterliste ist. Was wir also tun müssen, ist, unser Optionsobjekt als Anbieter zu definieren. Dies macht es injizierbar in den ConfigService, was wir im nächsten Schritt nutzen werden. Im folgenden Code achten Sie auf das Providers-Array:

```typescript
import { DynamicModule, Module } from '@nestjs/common';
import { ConfigService } from './config.service';

@Module({})
export class ConfigModule {
  static register(options: Record<string, any>): DynamicModule {
    return {
      module: ConfigModule,
      providers: [
        {
          provide: 'CONFIG_OPTIONS',
          useValue: options,
        },
        ConfigService,
      ],
      exports: [ConfigService],
    };
  }
}
```

Nun können wir den Prozess abschließen, indem wir den 'CONFIG_OPTIONS'-Anbieter in den ConfigService injizieren. Denken Sie daran, dass wir bei der Definition eines Anbieters mithilfe eines nicht-klassenbasierten Tokens den @Inject()-Dekorator verwenden müssen, wie hier beschrieben:

```typescript
import * as dotenv from 'dotenv';
import * as fs from 'fs';
import * as path from 'path';
import { Injectable, Inject } from '@nestjs/common';
import { EnvConfig } from './interfaces';

@Injectable()
export class ConfigService {
  private readonly envConfig: EnvConfig;

  constructor(@Inject('CONFIG_OPTIONS') private options: Record<string, any>) {
    const filePath = `${process.env.NODE_ENV || 'development'}.env`;
    const envFile = path.resolve(__dirname, '../../', options.folder, filePath);
    this.envConfig = dotenv.parse(fs.readFileSync(envFile));
  }

  get(key: string): string {
    return this.envConfig[key];
  }
}
```

Ein abschließender Hinweis: Zur Vereinfachung haben wir oben einen string-basierten Injektionstoken ('CONFIG_OPTIONS') verwendet, aber bewährte Praxis ist es, ihn als Konstante (oder Symbol) in einer separaten Datei zu definieren und diese Datei zu importieren. Beispielsweise:

```typescript
export const CONFIG_OPTIONS = 'CONFIG_OPTIONS';
```

#### Beispiel / Example

Ein vollständiges Beispiel des Codes in diesem Kapitel finden Sie hier.

### Community-Richtlinien / Community guidelines

Sie haben vielleicht die Verwendung von Methoden wie forRoot, register und forFeature bei einigen der @nestjs/-Pakete gesehen und fragen sich, was der Unterschied zwischen all diesen Methoden ist. Es gibt keine feste Regel dazu, aber die @nestjs/-Pakete versuchen, diesen Richtlinien zu folgen:

- Wenn Sie ein Modul mit register erstellen, erwarten Sie, dass Sie ein dynamisches Modul mit einer spezifischen Konfiguration nur für das aufrufende Modul konfigurieren. Beispielsweise mit Nest's @nestjs/axios: HttpModule.register({ baseUrl: 'someUrl' }). Wenn Sie in einem anderen Modul HttpModule.register({ baseUrl: 'somewhere else' }) verwenden, wird es eine andere Konfiguration haben. Sie können dies für so viele Module tun, wie Sie möchten.
- Wenn Sie ein Modul mit forRoot erstellen, erwarten Sie, dass Sie ein dynamisches Modul einmal konfigurieren und diese Konfiguration an mehreren Stellen wiederverwenden (obwohl möglicherweise unbewusst, da es abstrahiert ist). Deshalb haben Sie eine GraphQLModule.forRoot(), eine TypeOrmModule.forRoot() usw.
- Wenn Sie ein Modul mit forFeature erstellen, erwarten Sie, dass Sie die Konfiguration eines dynamischen Moduls forRoot verwenden, aber einige spezifische Konfigurationen für die Bedürfnisse des aufrufenden Moduls anpassen müssen (z. B. welches Repository dieses Modul verwenden soll oder den Kontext, den ein Logger verwenden soll).

All diese Methoden haben in der Regel auch ihre asynchronen Gegenstücke, registerAsync, forRootAsync und forFeatureAsync, die dasselbe bedeuten, aber die Abhängigkeitsinjektion von Nest für die Konfiguration verwenden.

### Baukasten für konfigurierbare Module / Configurable module builder

Da das manuelle Erstellen hochkonfigurierbarer, dynamischer Module, die asynchrone Methoden (registerAsync, forRootAsync usw.) bereitstellen, ziemlich kompliziert ist, insbesondere für Neulinge, stellt Nest die Klasse Configurable

ModuleBuilder zur Verfügung, die diesen Prozess erleichtert und es Ihnen ermöglicht, eine Modul-"Blaupause" in nur wenigen Zeilen Code zu erstellen.

Beispielsweise nehmen wir das oben verwendete Beispiel (ConfigModule) und konvertieren es, um den ConfigurableModuleBuilder zu verwenden. Bevor wir beginnen, stellen wir sicher, dass wir eine dedizierte Schnittstelle erstellen, die darstellt, welche Optionen unser ConfigModule akzeptiert:

```typescript
export interface ConfigModuleOptions {
  folder: string;
}
```

Mit dieser Grundlage erstellen wir eine neue dedizierte Datei (neben der bestehenden config.module.ts-Datei) und benennen sie config.module-definition.ts. In dieser Datei nutzen wir den ConfigurableModuleBuilder, um die ConfigModule-Definition zu erstellen.

```typescript
import { ConfigurableModuleBuilder } from '@nestjs/common';
import { ConfigModuleOptions } from './interfaces/config-module-options.interface';

export const { ConfigurableModuleClass, MODULE_OPTIONS_TOKEN } =
  new ConfigurableModuleBuilder<ConfigModuleOptions>().build();
```

Öffnen wir nun die config.module.ts-Datei und ändern ihre Implementierung, um die automatisch generierte ConfigurableModuleClass zu nutzen:

```typescript
import { Module } from '@nestjs/common';
import { ConfigService } from './config.service';
import { ConfigurableModuleClass } from './config.module-definition';

@Module({
  providers: [ConfigService],
  exports: [ConfigService],
})
export class ConfigModule extends ConfigurableModuleClass {}
```

Durch das Erweitern der ConfigurableModuleClass bedeutet dies, dass das ConfigModule nun nicht nur die register-Methode bereitstellt (wie zuvor mit der benutzerdefinierten Implementierung), sondern auch die registerAsync-Methode, die es Verbrauchern ermöglicht, dieses Modul asynchron zu konfigurieren, z. B. durch Bereitstellung asynchroner Fabriken:

```typescript
@Module({
  imports: [
    ConfigModule.register({ folder: './config' }),
    // oder alternativ:
    // ConfigModule.registerAsync({
    //   useFactory: () => {
    //     return {
    //       folder: './config',
    //     }
    //   },
    //   inject: [...any extra dependencies...]
    // }),
  ],
})
export class AppModule {}
```

Zuletzt aktualisieren wir die ConfigService-Klasse, um den generierten Modulanbieter anstelle des bisher verwendeten 'CONFIG_OPTIONS' zu injizieren:

```typescript
@Injectable()
export class ConfigService {
  constructor(@Inject(MODULE_OPTIONS_TOKEN) private options: ConfigModuleOptions) { ... }
}
```

### Benutzerdefinierter Methodenname / Custom method key

ConfigurableModuleClass bietet standardmäßig die Methoden register und ihr Gegenstück registerAsync. Um einen anderen Methodennamen zu verwenden, verwenden Sie die Methode ConfigurableModuleBuilder#setClassMethodName, wie folgt:

```typescript
export const { ConfigurableModuleClass, MODULE_OPTIONS_TOKEN } =
  new ConfigurableModuleBuilder<ConfigModuleOptions>().setClassMethodName('forRoot').build();
```

Diese Konstruktion weist den ConfigurableModuleBuilder an, eine Klasse zu generieren, die forRoot und forRootAsync anstelle von register und registerAsync bereitstellt. Beispiel:

```typescript
@Module({
  imports: [
    ConfigModule.forRoot({ folder: './config' }), // <-- Beachten Sie die Verwendung von "forRoot" anstelle von "register"
    // oder alternativ:
    // ConfigModule.forRootAsync({
    //   useFactory: () => {
    //     return {
    //       folder: './config',
    //     }
    //   },
    //   inject: [...any extra dependencies...]
    // }),
  ],
})
export class AppModule {}
```

### Benutzerdefinierte Optionsfabrikklasse / Custom options factory class

Da die registerAsync-Methode (oder forRootAsync oder ein anderer Name, je nach Konfiguration) dem Verbraucher ermöglicht, eine Anbieterdefinition zu übergeben, die zur Modulkonfiguration führt, könnte ein Bibliotheksverbraucher potenziell eine Klasse bereitstellen, die zur Konstruktion des Konfigurationsobjekts verwendet wird:

```typescript
@Module({
  imports: [
    ConfigModule.registerAsync({
      useClass: ConfigModuleOptionsFactory,
    }),
  ],
})
export class AppModule {}
```

Diese Klasse muss standardmäßig die create()-Methode bereitstellen, die ein Modulkonfigurationsobjekt zurückgibt. Wenn Ihre Bibliothek jedoch einer anderen Namenskonvention folgt, können Sie dieses Verhalten ändern und den ConfigurableModuleBuilder anweisen, eine andere Methode zu erwarten, z. B. createConfigOptions, mithilfe der Methode ConfigurableModuleBuilder#setFactoryMethodName:

```typescript
export const { ConfigurableModuleClass, MODULE_OPTIONS_TOKEN } =
  new ConfigurableModuleBuilder<ConfigModuleOptions>().setFactoryMethodName('createConfigOptions').build();
```

Nun muss die ConfigModuleOptionsFactory-Klasse die Methode createConfigOptions (anstelle von create) bereitstellen:

```typescript
@Module({
  imports: [
    ConfigModule.registerAsync({
      useClass: ConfigModuleOptionsFactory, // <-- Diese Klasse muss die Methode "createConfigOptions" bereitstellen
    }),
  ],
})
export class AppModule {}
```

### Zusätzliche Optionen / Extra options

Es gibt Grenzfälle, in denen Ihr Modul zusätzliche Optionen benötigt, die bestimmen, wie es sich verhalten soll (ein schönes Beispiel für eine solche Option ist das isGlobal-Flag oder einfach global), die gleichzeitig nicht im MODULE_OPTIONS_TOKEN-Anbieter enthalten sein sollten (da sie für die in diesem Modul registrierten Dienste/Anbieter irrelevant sind, z. B. muss ConfigService nicht wissen, ob sein Hostmodul als globales Modul registriert ist).

In solchen Fällen kann die Methode ConfigurableModuleBuilder#setExtras verwendet werden. Sehen Sie sich das folgende Beispiel an:

```typescript
export const { ConfigurableModuleClass, MODULE_OPTIONS_TOKEN } = new ConfigurableModuleBuilder<ConfigModuleOptions>()
  .setExtras(
    {
      isGlobal: true,
    },
    (definition, extras) => ({
      ...definition,
      global: extras.isGlobal,
    }),
  )
  .build();
```

Im obigen Beispiel ist das erste Argument, das an die Methode setExtras übergeben wird, ein Objekt, das Standardwerte für die "zusätzlichen" Eigenschaften enthält. Das zweite Argument ist eine Funktion, die eine automatisch generierte Moduldeklination (mit Anbieter, Exporte usw.) und ein Extras-Objekt, das zusätzliche Eigenschaften darstellt (entweder vom Verbraucher angegeben oder Standardwerte), übernimmt. Der Rückgabewert dieser Funktion ist eine modifizierte Moduldeklination. In diesem speziellen Beispiel nehmen wir die Eigenschaft extras.isGlobal und weisen sie der globalen Eigenschaft der Moduldeklination zu (was wiederum bestimmt, ob ein Modul global ist oder nicht, lesen Sie hier mehr).

Wenn Sie dieses Modul jetzt konsumieren, kann das zusätzliche isGlobal-Flag übergeben werden, wie folgt:

```typescript
@Module({
  imports: [
    ConfigModule.register({
      isGlobal: true,
      folder: './config',
    }),
  ],
})
export class AppModule {}
```

Da isGlobal jedoch als "zusätzliche" Eigenschaft deklariert ist, wird es im MODULE_OPTIONS_TOKEN-Anbieter nicht verfügbar sein:

```typescript
@Injectable()
export class ConfigService {
  constructor(@Inject(MODULE_OPTIONS_TOKEN) private options: ConfigModuleOptions) {
    // Das "options"-Objekt wird die "isGlobal"-Eigenschaft nicht haben
    // ...
  }
}
```

### Erweitern der automatisch generierten Methoden / Extending auto-generated methods

Die automatisch generierten statischen Methoden (register, registerAsync usw.) können bei Bedarf erweitert werden, wie folgt:

```typescript
import { Module } from '@nestjs/common';
import { ConfigService } from './config.service';
import { ConfigurableModuleClass, ASYNC_OPTIONS_TYPE, OPTIONS_TYPE } from './config.module-definition';

@Module({
  providers: [ConfigService],
  exports: [ConfigService],
})
export class ConfigModule extends ConfigurableModuleClass {
  static register(options: typeof OPTIONS_TYPE): DynamicModule {
    return {
      // Ihre benutzerdefinierte Logik hier
      ...super.register(options),
    };
  }

  static registerAsync(options: typeof ASYNC_OPTIONS_TYPE): DynamicModule {
    return {
      // Ihre benutzerdefinierte Logik hier
      ...super.registerAsync(options),
    };
  }
}
```

Beachten Sie die Verwendung der Typen OPTIONS_TYPE und ASYNC_OPTIONS_TYPE, die aus der Moduldeklinationsdatei exportiert werden müssen:

```typescript
export const { ConfigurableModuleClass, MODULE_OPTIONS_TOKEN, OPTIONS_TYPE, ASYNC_OPTIONS_TYPE } = new ConfigurableModuleBuilder<ConfigModuleOptions>().build();
```
