### Verschlüsselung und Hashing / Encryption and Hashing

Verschlüsselung ist der Prozess der Kodierung von Informationen. Dieser Prozess wandelt die ursprüngliche Darstellung der Informationen, bekannt als Klartext, in eine alternative Form, bekannt als Chiffretext, um. Idealerweise können nur autorisierte Parteien einen Chiffretext wieder in Klartext entschlüsseln und auf die ursprünglichen Informationen zugreifen. Verschlüsselung verhindert selbst keine Eingriffe, sondern verweigert einem potenziellen Abfänger den verständlichen Inhalt. Verschlüsselung ist eine bidirektionale Funktion; was verschlüsselt ist, kann mit dem richtigen Schlüssel wieder entschlüsselt werden.

Hashing ist der Prozess, einen gegebenen Schlüssel in einen anderen Wert umzuwandeln. Eine Hash-Funktion wird verwendet, um den neuen Wert gemäß einem mathematischen Algorithmus zu erzeugen. Einmal gehasht, sollte es unmöglich sein, vom Ausgangswert auf den Eingabewert zurückzuschließen.

### Verschlüsselung / Encryption

Node.js bietet ein integriertes Crypto-Modul, das Sie verwenden können, um Zeichenketten, Zahlen, Buffer, Streams und mehr zu verschlüsseln und zu entschlüsseln. Nest selbst bietet kein zusätzliches Paket zu diesem Modul, um unnötige Abstraktionen zu vermeiden.

Als Beispiel verwenden wir den AES (Advanced Encryption System) 'aes-256-ctr' Algorithmus im CTR-Verschlüsselungsmodus.

```typescript
import { createCipheriv, randomBytes, scrypt } from 'crypto';
import { promisify } from 'util';

const iv = randomBytes(16);
const password = 'Password used to generate key';

// Die Schlüssellänge hängt vom Algorithmus ab.
// In diesem Fall für aes256 sind es 32 Bytes.
const key = (await promisify(scrypt)(password, 'salt', 32)) as Buffer;
const cipher = createCipheriv('aes-256-ctr', key, iv);

const textToEncrypt = 'Nest';
const encryptedText = Buffer.concat([
  cipher.update(textToEncrypt),
  cipher.final(),
]);
```

Nun, um den Wert von `encryptedText` zu entschlüsseln:

```typescript
import { createDecipheriv } from 'crypto';

const decipher = createDecipheriv('aes-256-ctr', key, iv);
const decryptedText = Buffer.concat([
  decipher.update(encryptedText),
  decipher.final(),
]);
```

### Hashing / Hashing

Zum Hashing empfehlen wir die Verwendung der Pakete `bcrypt` oder `argon2`. Nest selbst bietet keine zusätzlichen Wrapper für diese Module, um unnötige Abstraktionen zu vermeiden (und die Lernkurve kurz zu halten).

Als Beispiel verwenden wir bcrypt, um ein zufälliges Passwort zu hashen.

Zuerst die erforderlichen Pakete installieren:

```bash
$ npm i bcrypt
$ npm i -D @types/bcrypt
```

Nach Abschluss der Installation können Sie die Hash-Funktion wie folgt verwenden:

```typescript
import * as bcrypt from 'bcrypt';

const saltOrRounds = 10;
const password = 'random_password';
const hash = await bcrypt.hash(password, saltOrRounds);
```

Um ein Salt zu generieren, verwenden Sie die Funktion `genSalt`:

```typescript
const salt = await bcrypt.genSalt();
```

Um ein Passwort zu vergleichen/prüfen, verwenden Sie die Funktion `compare`:

```typescript
const isMatch = await bcrypt.compare(password, hash);
```

Weitere Informationen zu den verfügbaren Funktionen finden Sie [hier](https://www.npmjs.com/package/bcrypt).
