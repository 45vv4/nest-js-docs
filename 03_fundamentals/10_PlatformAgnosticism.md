# Plattformunabhängigkeit / Platform agnosticism

Nest ist ein plattformunabhängiges Framework. Das bedeutet, dass Sie wiederverwendbare logische Teile entwickeln können, die in verschiedenen Arten von Anwendungen verwendet werden können. Zum Beispiel können die meisten Komponenten ohne Änderungen in verschiedenen zugrunde liegenden HTTP-Server-Frameworks (z.B. Express und Fastify) und sogar in verschiedenen Arten von Anwendungen (z.B. HTTP-Server-Frameworks, Microservices mit verschiedenen Transportschichten und Websockets) wiederverwendet werden.

## Einmal bauen, überall verwenden / Build once, use everywhere

Der Abschnitt "Überblick" der Dokumentation zeigt hauptsächlich Codierungstechniken unter Verwendung von HTTP-Server-Frameworks (z.B. Apps, die eine REST-API bereitstellen oder eine serverseitig gerenderte MVC-ähnliche App bereitstellen). Alle diese Bausteine können jedoch auch auf verschiedenen Transportschichten (Microservices oder Websockets) verwendet werden.

Darüber hinaus kommt Nest mit einem dedizierten GraphQL-Modul. Sie können GraphQL als Ihre API-Schicht abwechselnd mit der Bereitstellung einer REST-API verwenden.

Zusätzlich hilft die Anwendungskontext-Funktion, jede Art von Node.js-Anwendung zu erstellen - einschließlich Dingen wie CRON-Jobs und CLI-Apps - auf Basis von Nest.

Nest strebt danach, eine vollwertige Plattform für Node.js-Apps zu sein, die ein höheres Maß an Modularität und Wiederverwendbarkeit in Ihre Anwendungen bringt. Einmal bauen, überall verwenden!
