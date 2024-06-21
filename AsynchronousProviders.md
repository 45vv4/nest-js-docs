## Asynchrone Provider / Asynchronous providers

Manchmal sollte der Start der Anwendung verzögert werden, bis eine oder mehrere asynchrone Aufgaben abgeschlossen sind. Zum Beispiel möchten Sie möglicherweise keine Anfragen akzeptieren, bevor die Verbindung zur Datenbank hergestellt wurde. Dies können Sie mit asynchronen Providern erreichen.

Die Syntax hierfür besteht darin, async/await mit der `useFactory`-Syntax zu verwenden. Die Factory-Funktion gibt ein Promise zurück und kann asynchrone Aufgaben erwarten. Nest wartet auf die Auflösung des Promises, bevor es eine Klasse instanziiert, die von einem solchen Provider abhängt (injektziert).

```typescript
{
  provide: 'ASYNC_CONNECTION',
  useFactory: async () => {
    const connection = await createConnection(options);
    return connection;
  },
}
```

### Hinweis
Erfahren Sie mehr über die Syntax benutzerdefinierter Provider [hier](https://docs.nestjs.com/fundamentals/custom-providers).

### Injektion

Asynchrone Provider werden wie jeder andere Provider durch ihre Tokens in andere Komponenten injiziert. Im obigen Beispiel würden Sie die Konstruktion `@Inject('ASYNC_CONNECTION')` verwenden.

### Beispiel

Das TypeORM-Rezept enthält ein umfangreicheres Beispiel für einen asynchronen Provider.
