### CORS / CORS

Cross-Origin Resource Sharing (CORS) ist ein Mechanismus, der es ermöglicht, Ressourcen von einer anderen Domäne anzufordern. Im Hintergrund verwendet Nest die Express cors- oder Fastify @fastify/cors-Pakete, je nach zugrunde liegender Plattform. Diese Pakete bieten verschiedene Optionen, die Sie basierend auf Ihren Anforderungen anpassen können.

#### Erste Schritte / Getting started#

Um CORS zu aktivieren, rufen Sie die Methode `enableCors()` auf dem Nest-Anwendungsobjekt auf.

```typescript
const app = await NestFactory.create(AppModule);
app.enableCors();
await app.listen(3000);
```

Die Methode `enableCors()` nimmt ein optionales Konfigurationsobjekt als Argument entgegen. Die verfügbaren Eigenschaften dieses Objekts sind in der offiziellen CORS-Dokumentation beschrieben. Eine andere Möglichkeit besteht darin, eine Callback-Funktion zu übergeben, die es Ihnen ermöglicht, das Konfigurationsobjekt basierend auf der Anfrage asynchron zu definieren (on the fly).

Alternativ können Sie CORS über das Optionsobjekt der Methode `create()` aktivieren. Setzen Sie die Eigenschaft `cors` auf `true`, um CORS mit den Standardeinstellungen zu aktivieren. Oder übergeben Sie ein CORS-Konfigurationsobjekt oder eine Callback-Funktion als Wert der Eigenschaft `cors`, um das Verhalten anzupassen.

```typescript
const app = await NestFactory.create(AppModule, { cors: true });
await app.listen(3000);
```
