# Server-Sent Events / Server-Sent Events

Server-Sent Events (SSE) ist eine Server-Push-Technologie, die es einem Client ermöglicht, automatische Updates von einem Server über eine HTTP-Verbindung zu empfangen. Jede Benachrichtigung wird als ein Textblock gesendet, der durch ein Paar neuer Zeilen beendet wird (mehr erfahren Sie [hier](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events)).

## Verwendung / Usage#

Um Server-Sent Events auf einer Route zu aktivieren (Route, die innerhalb einer Controller-Klasse registriert ist), annotieren Sie den Method-Handler mit dem @Sse() Dekorator.

```typescript
@Sse('sse')
sse(): Observable<MessageEvent> {
  return interval(1000).pipe(map((_) => ({ data: { hello: 'world' } })));
}
```

**HINWEIS**  
Der @Sse() Dekorator und die MessageEvent-Schnittstelle werden aus @nestjs/common importiert, während Observable, interval und map aus dem rxjs-Paket importiert werden.

**WARNUNG**  
Server-Sent Events-Routen müssen einen Observable-Stream zurückgeben.

Im obigen Beispiel haben wir eine Route namens sse definiert, die es uns ermöglicht, Echtzeit-Updates zu verbreiten. Diese Ereignisse können mit der EventSource API abgehört werden.

Die sse-Methode gibt ein Observable zurück, das mehrere MessageEvent auslöst (in diesem Beispiel wird jede Sekunde ein neues MessageEvent ausgelöst). Das MessageEvent-Objekt sollte die folgende Schnittstelle respektieren, um der Spezifikation zu entsprechen:

```typescript
export interface MessageEvent {
  data: string | object;
  id?: string;
  type?: string;
  retry?: number;
}
```

Mit dieser Einrichtung können wir nun eine Instanz der EventSource-Klasse in unserer Client-seitigen Anwendung erstellen und die /sse Route (die dem Endpunkt entspricht, den wir oben in den @Sse() Dekorator übergeben haben) als Konstruktorargument übergeben.

Die EventSource-Instanz öffnet eine dauerhafte Verbindung zu einem HTTP-Server, der Ereignisse im text/event-stream Format sendet. Die Verbindung bleibt geöffnet, bis sie durch Aufrufen von EventSource.close() geschlossen wird.

Sobald die Verbindung geöffnet ist, werden eingehende Nachrichten vom Server in Form von Ereignissen an Ihren Code geliefert. Wenn es ein event Feld in der eingehenden Nachricht gibt, ist das ausgelöste Ereignis dasselbe wie der Wert des event Feldes. Wenn kein event Feld vorhanden ist, wird ein generisches message Ereignis ausgelöst (Quelle).

```javascript
const eventSource = new EventSource('/sse');
eventSource.onmessage = ({ data }) => {
  console.log('New message', JSON.parse(data));
};
```

## Beispiel / Example#

Ein funktionierendes Beispiel finden Sie [hier](https://github.com/nestjs/nest/tree/master/sample/28-sse).
