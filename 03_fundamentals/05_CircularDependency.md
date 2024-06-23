
# Zyklische Abhängigkeit / Circular dependency

Eine zyklische Abhängigkeit tritt auf, wenn zwei Klassen voneinander abhängen. Zum Beispiel benötigt Klasse A Klasse B, und Klasse B benötigt auch Klasse A. Zyklische Abhängigkeiten können in Nest zwischen Modulen und zwischen Providern auftreten.

Während zyklische Abhängigkeiten nach Möglichkeit vermieden werden sollten, ist dies nicht immer möglich. In solchen Fällen ermöglicht Nest die Auflösung zyklischer Abhängigkeiten zwischen Providern auf zwei Arten. In diesem Kapitel beschreiben wir die Verwendung von Vorwärtsreferenzierung als eine Technik und die Verwendung der ModuleRef-Klasse zum Abrufen einer Provider-Instanz aus dem DI-Container als eine andere.

Wir beschreiben auch die Auflösung zyklischer Abhängigkeiten zwischen Modulen.

**WARNUNG**: Eine zyklische Abhängigkeit kann auch durch die Verwendung von "Barrel-Dateien"/index.ts-Dateien zum Gruppieren von Imports verursacht werden. Barrel-Dateien sollten weggelassen werden, wenn es um Modul-/Provider-Klassen geht. Zum Beispiel sollten Barrel-Dateien nicht verwendet werden, wenn Dateien innerhalb desselben Verzeichnisses wie die Barrel-Datei importiert werden, d.h. cats/cats.controller sollte cats nicht importieren, um die cats/cats.service-Datei zu importieren. Weitere Details finden Sie auch in diesem GitHub-Issue.

## Vorwärtsreferenz / Forward reference

Eine Vorwärtsreferenz ermöglicht es Nest, Klassen zu referenzieren, die noch nicht definiert sind, indem die forwardRef()-Dienstprogrammfunktion verwendet wird. Zum Beispiel, wenn CatsService und CommonService voneinander abhängen, können beide Seiten der Beziehung @Inject() und das forwardRef()-Dienstprogramm verwenden, um die zyklische Abhängigkeit aufzulösen. Andernfalls wird Nest sie nicht instanziieren, weil alle wesentlichen Metadaten nicht verfügbar sind. Hier ist ein Beispiel:

```typescript
// cats.service.ts

@Injectable()
export class CatsService {
  constructor(
    @Inject(forwardRef(() => CommonService))
    private commonService: CommonService,
  ) {}
}
```

**TIPP**: Die forwardRef()-Funktion wird aus dem @nestjs/common-Paket importiert.

Das deckt eine Seite der Beziehung ab. Nun machen wir dasselbe mit CommonService:

```typescript
// common.service.ts

@Injectable()
export class CommonService {
  constructor(
    @Inject(forwardRef(() => CatsService))
    private catsService: CatsService,
  ) {}
}
```

**WARNUNG**: Die Reihenfolge der Instanziierung ist unbestimmt. Stellen Sie sicher, dass Ihr Code nicht davon abhängt, welcher Konstruktor zuerst aufgerufen wird. Das Abhängigkeitsverhältnis von Providern mit Scope.REQUEST kann zu undefinierten Abhängigkeiten führen. Weitere Informationen finden Sie hier.

## Alternative ModuleRef-Klasse / ModuleRef class alternative

Eine Alternative zur Verwendung von forwardRef() besteht darin, Ihren Code zu refaktorisieren und die ModuleRef-Klasse zu verwenden, um einen Provider auf einer Seite der (ansonsten) zyklischen Beziehung abzurufen. Erfahren Sie mehr über die ModuleRef-Dienstprogramklasse hier.

## Modul-Vorwärtsreferenz / Module forward reference

Um zyklische Abhängigkeiten zwischen Modulen aufzulösen, verwenden Sie die gleiche forwardRef()-Dienstprogramfunktion auf beiden Seiten der Modulassoziation. Zum Beispiel:

```typescript
// common.module.ts

@Module({
  imports: [forwardRef(() => CatsModule)],
})
export class CommonModule {}
```

Das deckt eine Seite der Beziehung ab. Nun machen wir dasselbe mit CatsModule:

```typescript
// cats.module.ts

@Module({
  imports: [forwardRef(() => CommonModule)],
})
export class CatsModule {}
```
