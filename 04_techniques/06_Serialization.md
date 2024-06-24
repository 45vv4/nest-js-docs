# Serialisierung / Serialization

Serialisierung ist ein Prozess, der stattfindet, bevor Objekte in einer Netzwerkantwort zurückgegeben werden. Dies ist ein geeigneter Ort, um Regeln für die Transformation und Bereinigung der Daten festzulegen, die dem Client zurückgegeben werden sollen. Zum Beispiel sollten sensible Daten wie Passwörter immer von der Antwort ausgeschlossen werden. Oder bestimmte Eigenschaften benötigen möglicherweise eine zusätzliche Transformation, wie das Senden nur eines Teilsets von Eigenschaften einer Entität. Diese Transformationen manuell durchzuführen, kann mühsam und fehleranfällig sein und kann dazu führen, dass nicht alle Fälle abgedeckt werden.

## Überblick / Overview

Nest bietet eine eingebaute Fähigkeit, um sicherzustellen, dass diese Operationen auf einfache Weise durchgeführt werden können. Der ClassSerializerInterceptor-Interceptor verwendet das leistungsstarke class-transformer-Paket, um eine deklarative und erweiterbare Möglichkeit zur Transformation von Objekten zu bieten. Die grundlegende Operation, die er ausführt, besteht darin, den Wert zu nehmen, der von einem Methoden-Handler zurückgegeben wird, und die instanceToPlain()-Funktion von class-transformer anzuwenden. Dabei können Regeln angewendet werden, die durch class-transformer-Dekoratoren in einer Entitäts-/DTO-Klasse ausgedrückt werden, wie unten beschrieben.

**HINWEIS**

Die Serialisierung gilt nicht für StreamableFile-Antworten.

## Eigenschaften ausschließen / Exclude properties

Angenommen, wir möchten automatisch eine Passwort-Eigenschaft aus einer Benutzerentität ausschließen. Wir annotieren die Entität wie folgt:

```typescript
import { Exclude } from 'class-transformer';

export class UserEntity {
  id: number;
  firstName: string;
  lastName: string;

  @Exclude()
  password: string;

  constructor(partial: Partial<UserEntity>) {
    Object.assign(this, partial);
  }
}
```

Betrachten Sie nun einen Controller mit einem Methoden-Handler, der eine Instanz dieser Klasse zurückgibt.

```typescript
@UseInterceptors(ClassSerializerInterceptor)
@Get()
findOne(): UserEntity {
  return new UserEntity({
    id: 1,
    firstName: 'Kamil',
    lastName: 'Mysliwiec',
    password: 'password',
  });
}
```

**WARNUNG**

Beachten Sie, dass wir eine Instanz der Klasse zurückgeben müssen. Wenn Sie ein einfaches JavaScript-Objekt zurückgeben, zum Beispiel { user: new UserEntity() }, wird das Objekt nicht ordnungsgemäß serialisiert.

**HINWEIS**

Der ClassSerializerInterceptor wird aus @nestjs/common importiert.

Wenn dieser Endpunkt angefordert wird, erhält der Client die folgende Antwort:

```json
{
  "id": 1,
  "firstName": "Kamil",
  "lastName": "Mysliwiec"
}
```

Beachten Sie, dass der Interceptor anwendungsweit angewendet werden kann (wie [hier](https://github.com/nestjs/nest/tree/master/sample/21-serializer) beschrieben). Die Kombination aus dem Interceptor und der Deklaration der Entitätsklasse stellt sicher, dass jede Methode, die eine UserEntity zurückgibt, die Passwort-Eigenschaft entfernt. Dies bietet eine zentrale Durchsetzung dieser Geschäftsregel.

## Eigenschaften exponieren / Expose properties

Sie können den @Expose()-Dekorator verwenden, um Alias-Namen für Eigenschaften bereitzustellen oder eine Funktion auszuführen, um einen Eigenschaftswert zu berechnen (analog zu Getter-Funktionen), wie unten gezeigt.

```typescript
@Expose()
get fullName(): string {
  return `${this.firstName} ${this.lastName}`;
}
```

## Transformieren / Transform

Sie können zusätzliche Datenumwandlungen mit dem @Transform()-Dekorator durchführen. Zum Beispiel gibt die folgende Konstruktion die name-Eigenschaft der RoleEntity zurück, anstatt das gesamte Objekt zurückzugeben.

```typescript
@Transform(({ value }) => value.name)
role: RoleEntity;
```

## Optionen übergeben / Pass options

Möglicherweise möchten Sie das Standardverhalten der Transformationsfunktionen ändern. Um Standardeinstellungen zu überschreiben, übergeben Sie sie in einem options-Objekt mit dem @SerializeOptions()-Dekorator.

```typescript
@SerializeOptions({
  excludePrefixes: ['_'],
})
@Get()
findOne(): UserEntity {
  return new UserEntity();
}
```

**HINWEIS**

Der @SerializeOptions()-Dekorator wird aus @nestjs/common importiert.

Optionen, die über @SerializeOptions() übergeben werden, werden als zweites Argument der zugrunde liegenden instanceToPlain()-Funktion übergeben. In diesem Beispiel schließen wir automatisch alle Eigenschaften aus, die mit dem Präfix _ beginnen.

## Beispiel / Example

Ein funktionierendes Beispiel ist [hier](https://github.com/nestjs/nest/tree/master/sample/21-serializer) verfügbar.

## WebSockets und Microservices / WebSockets and Microservices

Während dieses Kapitel Beispiele mit HTTP-Stil-Anwendungen (z. B. Express oder Fastify) zeigt, funktioniert der ClassSerializerInterceptor genauso für WebSockets und Microservices, unabhängig von der verwendeten Transportmethode.

## Mehr erfahren / Learn more

Lesen Sie hier mehr über verfügbare Dekoratoren und Optionen, die vom class-transformer-Paket bereitgestellt werden.

## Unterstützen Sie uns / Support us

Nest ist ein Open-Source-Projekt, das unter der MIT-Lizenz steht. Es kann dank der Unterstützung dieser großartigen Menschen wachsen. Wenn Sie sich ihnen anschließen möchten, lesen Sie [hier](https://github.com/typestack/class-transformer) mehr.