---
name: solid
description: "Usala cuando una tarea crea o hace crecer un service, agrega una interfaz o una implementación, suma una variante de un comportamiento o hace que una feature use otra."
---

# Principios SOLID

Explica cómo se aplican los principios SOLID en una aplicación Spring Boot organizada por features. Las reglas valen para cualquier proyecto, y los ejemplos son de este. Si una regla choca con `docs/constitution.md`, gana la constitución.

## Cuándo aplica

- Crear un service o sumarle un método. Ejemplo: sumarle a `RentalService` el registro de un alquiler.
- Sumar una variante de un comportamiento. Ejemplo: la tarifa por mes.
- Crear una interfaz o una implementación. Ejemplo: `PricingStrategy` y `WeeklyPricing`.
- Hacer que una feature use otra. Ejemplo: `rental` necesita consultar un equipo de `catalog`.

## Reglas

### Responsabilidad única: cada clase tiene un solo motivo para cambiar

Lo que cambia por el mismo motivo va junto, y lo que cambia por motivos distintos va en clases distintas. Antes de sumar un método a una clase, el agente se pregunta quién pediría ese cambio. Si es otro motivo, como las tarifas, una notificación o una integración externa, va en su propia clase. Una clase puede tener varios métodos si todos cambian por el mismo motivo.

Al registrar un alquiler, `RentalService` valida que las fechas no se superpongan con otro alquiler del mismo equipo y lo guarda con su precio. Todo eso cambia cuando cambia la forma de alquilar, así que va junto en `RentalService`. El cálculo del precio cambia cuando cambian las tarifas, así que va en su propia clase, y `RentalService` le pide el precio.

**Cómo se verifica:** ninguno todavía.

### Abierto/cerrado: una clase está abierta a extensión y cerrada a modificación

Una variante nueva de un comportamiento se suma con una clase nueva. La variante implementa la interfaz del comportamiento, sobrescribe su método y lleva `@Component` con nombre. El service recibe por constructor un `Map<String, Interfaz>` y busca la variante por el nombre que trae el pedido; si el nombre no está en el Map, lanza una excepción de la feature, creada según la skill `error-handling`. Las clases que ya existen quedan como están. Se aplica cuando las variantes crecen; con dos variantes que no cambian, alcanza una clase con dos métodos. El código completo está en `examples/strategy.md`: leelo antes de sumar una variante.

Para agregar la tarifa por mes se crea `MonthlyPricing` con `@Component("monthly")`, y `RentalService` queda como está.

**Cómo se verifica:** el `git diff` de la tarea muestra la clase nueva y ningún cambio en el código de producción del service; los tests pueden cambiar.

### Sustitución de Liskov: una implementación devuelve siempre lo que la interfaz promete

El compilador controla la firma, pero no que el resultado llegue siempre. Para cualquier entrada que la interfaz acepta, la implementación devuelve lo que promete, nunca una excepción ni un null que la interfaz no prometa. Vale igual para una subclase, que devuelve siempre lo que promete la clase que extiende. Si el negocio pide una condición, como un mínimo de días, la valida el service antes de llamar a la implementación.

`PricingStrategy` promete un `BigDecimal` para cualquier cantidad de días, y `WeeklyPricing` lo devuelve también con 3 días. Si el negocio exige 7 días como mínimo, lo valida `RentalService`.

**Cómo se verifica:** el mismo test, con los mismos datos, pasa con cada implementación.

### Segregación de interfaces: nadie depende de métodos que no usa

Una interfaz tiene solo los métodos que necesita quien la usa. Cuando una feature usa otra, lo hace con una interfaz por rol en el subpaquete `api`, y si necesita algo más, se suma ese método. Si una implementación queda obligada a tener métodos que no le corresponden, la interfaz se divide en interfaces más chicas. Vale igual para una clase base: si una subclase queda obligada a heredar un método que no puede cumplir, ese método sale de la clase base y pasa a una interfaz, que implementan solo las subclases que lo necesitan.

`RentalService` usa `CatalogApi`, que tiene solo `findEquipment`, y no `EquipmentService` con sus cinco métodos.

**Cómo se verifica:** ninguno todavía.

### Inversión de dependencias: entre features, se depende de una abstracción

Cuando una feature usa otra feature, tiene que depender solo de una interfaz del subpaquete `api` de esa feature, nunca de sus otros subpaquetes, como `service` o `repository`. Dentro de una feature, las clases dependen de clases concretas, sin una interfaz en el medio, salvo que la pida un patrón, como Strategy, o el framework, como el repository de Spring Data.

`RentalService` depende de `CatalogApi`, y no de `EquipmentService`. `EquipmentController` depende de `EquipmentService` directamente.

**Cómo se verifica:** ninguno todavía.

## Qué evitar

- No sumarle a un service un método que cambia por otro motivo, como el cálculo de las tarifas o el envío de una notificación, porque el service termina con varios motivos para cambiar y cada cambio toca código que no tiene que ver.
- No elegir la variante de un comportamiento con un `switch` o una cadena de `if` en el service, porque cada variante nueva obliga a modificar el service en lugar de agregar una clase nueva.
- No rechazar un caso dentro de una implementación con una excepción o un `null` que la interfaz no promete, como una tarifa que exige un mínimo de días, porque la condición del negocio la debe validar el service antes de llamarla.
- No darle a otra feature una interfaz espejo que copie todos los métodos de una clase, porque quien la usa termina dependiendo de métodos que no llama.
- No crear una interfaz por cada service por las dudas, como `EquipmentService` con su `EquipmentServiceImpl`, porque las clases de una feature cambian juntas y esa interfaz no protege nada.
- No usar desde una feature el service, el repository ni la entidad de otra, porque se acoplan sus implementaciones; se usa solo su `api`, que devuelve un record con los datos que necesita quien la usa, como `EquipmentSummary`.
