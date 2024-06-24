# CLI-Befehlsreferenz / CLI command reference

## nest new

Erstellt ein neues (Standardmodus) Nest-Projekt.

```bash
$ nest new <name> [options]
$ nest n <name> [options]
```

**Beschreibung**  
Erstellt und initialisiert ein neues Nest-Projekt. Fragt nach dem Paketmanager.

- Erstellt einen Ordner mit dem angegebenen <name>
- Füllt den Ordner mit Konfigurationsdateien
- Erstellt Unterordner für den Quellcode (/src) und End-to-End-Tests (/test)
- Füllt die Unterordner mit Standarddateien für Anwendungsbestandteile und Tests

**Argumente**

| Argument | Beschreibung                  |
|----------|-------------------------------|
| <name>   | Der Name des neuen Projekts   |

**Optionen**

| Option                         | Beschreibung                                                             |
|--------------------------------|---------------------------------------------------------------------------|
| --dry-run                      | Meldet Änderungen, die vorgenommen würden, aber verändert das Dateisystem nicht. Alias: -d |
| --skip-git                     | Überspringt die Initialisierung des Git-Repositories. Alias: -g           |
| --skip-install                 | Überspringt die Paketinstallation. Alias: -s                              |
| --package-manager [package-manager] | Paketmanager angeben. Verwenden Sie npm, yarn oder pnpm. Paketmanager muss global installiert sein. Alias: -p |
| --language [language]          | Programmiersprache angeben (TS oder JS). Alias: -l                        |
| --collection [collectionName]  | Schematiksammlung angeben. Verwenden Sie den Paketnamen des installierten npm-Pakets, das das Schematic enthält. Alias: -c |
| --strict                       | Starten Sie das Projekt mit den folgenden aktivierten TypeScript-Compiler-Flags: strictNullChecks, noImplicitAny, strictBindCallApply, forceConsistentCasingInFileNames, noFallthroughCasesInSwitch |

## nest generate

Generiert und/oder ändert Dateien basierend auf einem Schematic.

```bash
$ nest generate <schematic> <name> [options]
$ nest g <schematic> <name> [options]
```

**Argumente**

| Argument    | Beschreibung                                          |
|-------------|-------------------------------------------------------|
| <schematic> | Das Schematic oder die collection:schematic zu generieren. Siehe die Tabelle unten für die verfügbaren Schematics. |
| <name>      | Der Name der generierten Komponente.                 |

**Schematics**

| Name       | Alias | Beschreibung                                                                |
|------------|-------|-----------------------------------------------------------------------------|
| app        |       | Erzeugt eine neue Anwendung innerhalb eines Monorepos (konvertiert zu Monorepo, falls es sich um eine Standardstruktur handelt). |
| library    | lib   | Erzeugt eine neue Bibliothek innerhalb eines Monorepos (konvertiert zu Monorepo, falls es sich um eine Standardstruktur handelt). |
| class      | cl    | Erzeugt eine neue Klasse.                                                   |
| controller | co    | Erzeugt eine Controller-Deklaration.                                        |
| decorator  | d     | Erzeugt einen benutzerdefinierten Decorator.                                |
| filter     | f     | Erzeugt eine Filter-Deklaration.                                            |
| gateway    | ga    | Erzeugt eine Gateway-Deklaration.                                           |
| guard      | gu    | Erzeugt eine Guard-Deklaration.                                             |
| interface  | itf   | Erzeugt eine Schnittstelle.                                                 |
| interceptor| itc   | Erzeugt eine Interceptor-Deklaration.                                       |
| middleware | mi    | Erzeugt eine Middleware-Deklaration.                                        |
| module     | mo    | Erzeugt eine Modul-Deklaration.                                             |
| pipe       | pi    | Erzeugt eine Pipe-Deklaration.                                              |
| provider   | pr    | Erzeugt eine Provider-Deklaration.                                          |
| resolver   | r     | Erzeugt eine Resolver-Deklaration.                                          |
| resource   | res   | Erzeugt eine neue CRUD-Ressource. Siehe den CRUD (resource) Generator für weitere Details. (nur TS) |
| service    | s     | Erzeugt eine Service-Deklaration.                                           |

**Optionen**

| Option                         | Beschreibung                                                             |
|--------------------------------|---------------------------------------------------------------------------|
| --dry-run                      | Meldet Änderungen, die vorgenommen würden, aber verändert das Dateisystem nicht. Alias: -d |
| --project [project]            | Projekt, dem das Element hinzugefügt werden soll. Alias: -p               |
| --flat                         | Erzeugt keinen Ordner für das Element.                                    |
| --collection [collectionName]  | Schematiksammlung angeben. Verwenden Sie den Paketnamen des installierten npm-Pakets, das das Schematic enthält. Alias: -c |
| --spec                         | Erzwingen der Spezifikationsdateigenerierung (Standard)                   |
| --no-spec                      | Deaktivieren der Spezifikationsdateigenerierung                           |

## nest build

Kompiliert eine Anwendung oder einen Arbeitsbereich in einen Ausgabefolder.

Der Build-Befehl ist auch verantwortlich für:

- Pfad-Mapping (bei Verwendung von Pfad-Aliasen) über tsconfig-paths
- Anmerkung von DTOs mit OpenAPI-Decorators (wenn @nestjs/swagger CLI-Plugin aktiviert ist)
- Anmerkung von DTOs mit GraphQL-Decorators (wenn @nestjs/graphql CLI-Plugin aktiviert ist)

```bash
$ nest build <name> [options]
```

**Argumente**

| Argument | Beschreibung                  |
|----------|-------------------------------|
| <name>   | Der Name des Projekts, das gebaut werden soll. |

**Optionen**

| Option                    | Beschreibung                                                                                       |
|---------------------------|---------------------------------------------------------------------------------------------------|
| --path [path]             | Pfad zur tsconfig-Datei. Alias -p                                                                  |
| --config [path]           | Pfad zur nest-cli-Konfigurationsdatei. Alias -c                                                    |
| --watch                   | Im Watch-Modus ausführen (Live-Reload). Wenn Sie tsc für die Kompilierung verwenden, können Sie rs eingeben, um die Anwendung neu zu starten (wenn die Option manualRestart auf true gesetzt ist). Alias -w |
| --builder [name]          | Den zu verwendenden Builder für die Kompilierung angeben (tsc, swc oder webpack). Alias -b          |
| --webpack                 | Verwenden Sie webpack für die Kompilierung (veraltet: verwenden Sie stattdessen --builder webpack).|
| --webpackPath             | Pfad zur webpack-Konfiguration.                                                                     |
| --tsc                     | Erzwingen der Verwendung von tsc für die Kompilierung.                                              |

## nest start

Kompiliert und führt eine Anwendung (oder das Standardprojekt in einem Arbeitsbereich) aus.

```bash
$ nest start <name> [options]
```

**Argumente**

| Argument | Beschreibung                  |
|----------|-------------------------------|
| <name>   | Der Name des Projekts, das ausgeführt werden soll. |

**Optionen**

| Option                    | Beschreibung                                                                                       |
|---------------------------|---------------------------------------------------------------------------------------------------|
| --path [path]             | Pfad zur tsconfig-Datei. Alias -p                                                                  |
| --config [path]           | Pfad zur nest-cli-Konfigurationsdatei. Alias -c                                                    |
| --watch                   | Im Watch-Modus ausführen (Live-Reload). Alias -w                                                   |
| --builder [name]          | Den zu verwendenden Builder für die Kompilierung angeben (tsc, swc oder webpack). Alias -b          |
| --preserveWatchOutput     | Behalten Sie veraltete Konsolenausgaben im Watch-Modus anstelle des Löschens des Bildschirms (nur tsc-Watch-Modus) |
| --watchAssets             | Im Watch-Modus ausführen (Live-Reload), beobachtet Nicht-TS-Dateien (Assets). Siehe Assets für weitere Details. |
| --debug [hostport]        | Im Debug-Modus ausführen (mit --inspect-Flag). Alias -d                                            |
| --webpack                 | Verwenden Sie webpack für die Kompilierung (veraltet: verwenden Sie stattdessen --builder webpack).|
| --webpackPath             | Pfad zur webpack-Konfiguration.                                                                     |
| --tsc                     | Erzwingen der Verwendung von tsc für die Kompilierung.                                              |
| --exec [binary]           | Zu ausführendes Binary (Standard: node). Alias -e                                                   |
| -- [key=value]            | Befehlszeilenargumente, die mit process.argv referenziert werden können.                            |

## nest add

Importiert eine Bibliothek, die als Nest-Bibliothek verpackt ist, und führt ihr Installations-Schematic aus.

```bash
$ nest add <name> [options]
```

**Argumente**

| Argument | Beschreibung                  |
|----------|-------------------------------|
| <name>   | Der Name der zu importierenden Bibliothek. |

## nest info

Zeigt Informationen über installierte Nest-Pakete und andere hilfreiche Systeminformationen an. Zum Beispiel:

```bash
$ nest info

 _   _             _      ___  _____  _____  _     _____
| \ | |           | |    |_  |/  ___|/  __ \| |   |_   _|
|  \| |  ___  ___ | |_     | |\ `--. | /  \/| |     | |
| . ` | / _ \/ __|| __|    | | `

--. \| |    | |     | |
| |\  ||  __/\__ \| |_ /\__/ //\__/ /| \__/\| |_____| |_
\_| \_/ \___||___/ \__|\____/ \____/  \____/\_____/\___/

[System Information]
OS Version : macOS High Sierra
NodeJS Version : v16.18.0
[Nest Information]
microservices version : 10.0.0
websockets version : 10.0.0
testing version : 10.0.0
common version : 10.0.0
core version : 10.0.0

## Unterstützen Sie uns / Support us

Nest ist ein Open-Source-Projekt unter MIT-Lizenz. Es kann dank der Unterstützung dieser großartigen Menschen wachsen. Wenn Sie sich ihnen anschließen möchten, lesen Sie hier mehr.
```