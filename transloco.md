
## **Schritt 1: Transloco in das Projekt installieren**
Öffne ein Terminal im Stammverzeichnis deines Angular-Projekts und führe folgenden Befehl aus:

```sh
ng add @jsverse/transloco
```

Während der Installation wirst du einige Fragen beantworten müssen, z. B.:
- Welche Sprachen du unterstützen möchtest (z. B. `en`, `es`).
- Ob du serverseitiges Rendering (SSR) verwendest.
- Ob du die Standalone- oder NgModule-Version nutzen willst.

Wähle **Standalone**, wenn deine Anwendung keine NgModule verwendet.

Nach Abschluss der Installation wurden einige Dateien und Konfigurationen automatisch erstellt.

---

## **Schritt 2: Konfiguration von Transloco**
Die `app.config.ts`-Datei wurde angepasst, um Transloco bereitzustellen. Falls nicht, erstelle oder bearbeite die Datei:

### **`src/app/app.config.ts`**
```typescript
import { ApplicationConfig, isDevMode } from '@angular/core';
import { provideHttpClient } from '@angular/common/http';
import { provideTransloco } from '@jsverse/transloco';

import { TranslocoHttpLoader } from './transloco-loader';

export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(),
    provideTransloco({
      config: {
        availableLangs: ['en', 'es'], // Unterstützte Sprachen
        defaultLang: 'en', // Standardsprache
        reRenderOnLangChange: true, // Entferne dies, falls Sprachwechsel zur Laufzeit nicht benötigt wird
        prodMode: !isDevMode(),
      },
      loader: TranslocoHttpLoader, // Verwende den HTTP-Loader für Übersetzungsdateien
    }),
  ],
};
```

---

## **Schritt 3: Transloco Loader erstellen**
Erstelle eine Datei `transloco-loader.ts` im `src/app/`-Verzeichnis und implementiere den HTTP-Loader, um die Übersetzungsdateien zu laden:

### **`src/app/transloco-loader.ts`**
```typescript
import { inject, Injectable } from '@angular/core';
import { Translation, TranslocoLoader } from '@jsverse/transloco';
import { HttpClient } from '@angular/common/http';

@Injectable({ providedIn: 'root' })
export class TranslocoHttpLoader implements TranslocoLoader {
  private http = inject(HttpClient);

  getTranslation(lang: string) {
    return this.http.get<Translation>(`/assets/i18n/${lang}.json`);
  }
}
```

Falls du Probleme mit der Ladepfad-Struktur hast (z. B. wenn dein Build-Prozess die Dateien verschiebt), kannst du eine relative URL verwenden:

```typescript
getTranslation(langPath: string) {
  return this.http.get(`./assets/i18n/${langPath}.json`);
}
```

---

## **Schritt 4: Übersetzungsdateien erstellen**
Die Übersetzungen werden in JSON-Dateien im `assets/i18n/`-Verzeichnis gespeichert.

Erstelle das Verzeichnis `assets/i18n/` und füge folgende JSON-Dateien hinzu:

### **`src/assets/i18n/en.json`**
```json
{
  "title": "Welcome",
  "description": "This is an Angular app with Transloco"
}
```

### **`src/assets/i18n/es.json`**
```json
{
  "title": "Bienvenido",
  "description": "Esta es una aplicación Angular con Transloco"
}
```

---

## **Schritt 5: Transloco in einer Standalone-Komponente nutzen**
Jetzt kannst du Transloco in deinen Komponenten verwenden.

### **`src/app/app.component.ts`**
```typescript
import { Component } from '@angular/core';
import { TranslocoModule, TranslocoService } from '@jsverse/transloco';

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [TranslocoModule],
  template: `
    <ng-container *transloco="let t">
      <h1>{{ t('title') }}</h1>
      <p>{{ t('description') }}</p>
      <button (click)="changeLanguage('en')">English</button>
      <button (click)="changeLanguage('es')">Español</button>
    </ng-container>
  `,
})
export class AppComponent {
  constructor(private translocoService: TranslocoService) {}

  changeLanguage(lang: string) {
    this.translocoService.setActiveLang(lang);
  }
}
```

---

## **Schritt 6: Globale Konfiguration für Tools & Plugins**
Falls du Plugins wie den `keysManager` verwenden möchtest, erstelle die Datei `transloco.config.ts`:

### **`src/transloco.config.ts`**
```typescript
import { TranslocoGlobalConfig } from '@jsverse/transloco-utils';

const config: TranslocoGlobalConfig = {
  rootTranslationsPath: 'src/assets/i18n/',
  langs: ['en', 'es'],
  keysManager: {},
};

export default config;
```

---

## **Schritt 7: Anwendung starten**
Starte deine Anwendung mit:

```sh
ng serve
```

Öffne sie im Browser (`http://localhost:4200`), und du solltest nun die Übersetzungen sehen können. Klicke auf die Buttons, um die Sprache zu wechseln.

---

## **Fazit**
Jetzt hast du `@jsverse/transloco` erfolgreich in eine Angular-Standalone-Anwendung integriert! Die wichtigsten Schritte waren:
1. Transloco mit `ng add @jsverse/transloco` installieren.
2. `app.config.ts` für Transloco konfigurieren.
3. Einen `TranslocoHttpLoader` erstellen, um Übersetzungen über HTTP zu laden.
4. JSON-Dateien mit Übersetzungen in `assets/i18n/` ablegen.
5. Transloco in einer Standalone-Komponente nutzen.
6. Optional eine globale Konfigurationsdatei für Plugins erstellen.

Damit ist dein Angular-Projekt bereit für mehrsprachige Inhalte!
