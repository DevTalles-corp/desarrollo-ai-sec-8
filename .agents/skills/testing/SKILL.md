---
name: testing
description: "Usala cuando una tarea crea o cambia un test de escenario, o suma una tabla a la base."
---

# Testing

Es una skill del proyecto. Explica cómo se escriben los tests de escenario y desarrolla la sección Tests de `docs/constitution.md`.

## Cuándo aplica

- Crear el test de un escenario, como el E1 en `RentalScenarioTests`.
- Sumar un escenario a una clase de tests que ya existe.
- Crear una tabla nueva, que también se suma a `clean.sql`.

## Reglas

### Una clase de tests de escenario por feature, en el paquete de la feature

Una clase de tests de escenario por feature, en su paquete. El test recorre toda la feature, así que no va en un subpaquete como `rental/controller`.

```java
package com.devtalles.rental.rental;

@Import(TestcontainersConfiguration.class)
@SpringBootTest
@AutoConfigureMockMvc
@Sql("/db/clean.sql")
class RentalScenarioTests {
    // ...
}
```

**Cómo se verifica:** ninguno todavía.

### La clase se llama `<Recurso>ScenarioTests` y el método dice la situación y el resultado

La clase se llama `<Recurso>ScenarioTests`. El método, en inglés, dice la situación y el resultado. El id del escenario va solo en `@DisplayName`.

```java
@Test
@DisplayName("E1: Alquilar un equipo ya alquilado responde 409")
void rentingAlreadyRentedEquipmentReturns409() throws Exception {
    // ...
}
```

**Cómo se verifica:** ninguno todavía.

### La base se vacía antes de cada test con `@Sql("/db/clean.sql")`

`@Sql("/db/clean.sql")` sobre la clase vacía la base antes de cada test. El script, en `src/test/resources/db/clean.sql`, trunca las tablas de las features.

```java
@Sql("/db/clean.sql")
class RentalScenarioTests {
    // ...
}
```

```sql
TRUNCATE TABLE rental, equipment RESTART IDENTITY CASCADE;
```

**Cómo se verifica:** ninguno todavía.

### Las dependencias llegan con `@Autowired` sobre el campo

`@Autowired` sobre el campo, en lugar del constructor: a la clase de test la crea JUnit, así que las ventajas del constructor no aplican.

```java
@Autowired
private MockMvc mockMvc;

@Autowired
private ObjectMapper objectMapper;
```

**Cómo se verifica:** ninguno todavía.

### El estado inicial se arma por los endpoints

Por los endpoints, con métodos privados del test que hacen el POST con MockMvc. El repository solo cuando la API no permite crear el dato, como un alquiler con fechas pasadas.

```java
private Long createEquipment(EquipmentRequest request) throws Exception {
    String response = mockMvc.perform(post("/equipment")
            .contentType(MediaType.APPLICATION_JSON)
            .content(objectMapper.writeValueAsString(request)))
        .andExpect(status().isCreated())
        .andReturn()
        .getResponse()
        .getContentAsString();

    return objectMapper.readValue(response, EquipmentResponse.class).id();
}
```

**Cómo se verifica:** ninguno todavía.

## Qué evitar

- No crear una clase de tests por cada tarea del SDD ni ubicarla en un subpaquete de capa, como `rental/controller`. Los tests de escenario se agrupan por feature, que es la unidad que permanece cuando la spec se archiva.
- No repetir el id del escenario en el nombre del método. El id va únicamente en `@DisplayName`; el nombre del método describe la situación y el resultado esperado.
- No limpiar la base con `@DirtiesContext`, porque levanta de nuevo la aplicación y es lento, ni anotar la clase de test con `@Transactional`, porque hace que la transacción cubra el controller, el service y el repository y el test deja de correr con las reglas de producción.
- No escribir un constructor en la clase de test para recibir sus dependencias.
- No guardar con el repository un dato que la API permite crear, ni cargar los datos iniciales con un script SQL. El estado inicial se arma por los endpoints, salvo casos que la API no permita.
- No simular nuestras propias capas, como `RentalService` con `@MockitoBean`. Los tests de escenario usan las clases reales del proyecto.
- No usar `@WebMvcTest` ni `@DataJpaTest`, que levantan una sola capa. El test de escenario levanta la aplicación completa con `@SpringBootTest`.