# Arquitectura de pruebas unitarias

**Backend TypeScript + NestJS · Arquitectura hexagonal con DDD · Vitest + StrykerJS**

Guía práctica del equipo: qué pruebas escribimos, con qué criterio, con qué herramientas y cómo mantenemos el tiempo de ejecución bajo control.

> Este documento es la guía operativa. El informe de investigación de 64 páginas (`03_Pruebas_Unitarias_DDD_Mutation_Testing.pdf`) contiene las fuentes, las citas y las correcciones a la documentación que circula mal.

---

## Índice

1. [Las decisiones, en una página](#1-las-decisiones-en-una-página)
2. [Vocabulario del dominio](#2-vocabulario-del-dominio)
3. [Las dos escuelas: Detroit y Londres](#3-las-dos-escuelas-detroit-y-londres)
4. [La arquitectura de pruebas: tres capas](#4-la-arquitectura-de-pruebas-tres-capas)
5. [Dónde está la frontera entre unitaria e integración](#5-dónde-está-la-frontera-entre-unitaria-e-integración)
6. [La política de dobles](#6-la-política-de-dobles)
7. [Fakes y suite de contrato](#7-fakes-y-suite-de-contrato)
8. [Cómo se escribe una prueba](#8-cómo-se-escribe-una-prueba)
9. [Qué NO probamos](#9-qué-no-probamos)
10. [Herramientas: Vitest](#10-herramientas-vitest)
11. [Herramientas: StrykerJS](#11-herramientas-strykerjs)
12. [Mutation testing: conceptos](#12-mutation-testing-conceptos)
13. [El tiempo de ejecución](#13-el-tiempo-de-ejecución)
14. [Estrategia de CI y pull requests](#14-estrategia-de-ci-y-pull-requests)
15. [Plan de adopción](#15-plan-de-adopción)
16. [Antipatrones prohibidos](#16-antipatrones-prohibidos)
17. [Definición de «terminado»](#17-definición-de-terminado)
18. [Glosario](#18-glosario)

---

## 1. Las decisiones, en una página

| # | Decisión | Resumen |
|---|---|---|
| 1 | **Escuela por defecto: Detroit** | Objetos de dominio reales, aserciones sobre resultado y estado. Londres queda reducido a una costura estrecha en el borde del hexágono |
| 2 | **La unidad es el comportamiento, no la clase** | Un test unitario invoca el caso de uso a través de su puerto, con value objects, entidades y agregados reales dentro |
| 3 | **Fake para dependencias gestionadas, mock solo para las no gestionadas** | Fake el repositorio (es nuestro), mock el correo y los pagos (alguien de fuera lo nota) |
| 4 | **Suite de contrato del puerto** | Un único cuerpo de aserciones ejecutado contra el fake y contra el adaptador real |
| 5 | **Mutation testing con StrykerJS sobre Vitest** | Acotado a `dominio/` y `aplicacion/` |
| 6 | **Dos ejecuciones de Stryker** | Incremental y acotada en el PR (bloquea), completa de noche (informa) |
| 7 | **Al principio reporta, no bloquea** | `thresholds.break: null` en las primeras semanas |
| 8 | **Un módulo piloto antes de extender** | Se mide el número real de mutantes y el tiempo real antes de diseñar los umbrales |

**La frase que lo resume:** *Londres en la frontera, Detroit en el núcleo.*

---

## 2. Vocabulario del dominio

### Value object

Se define por su **contenido**. Dos instancias con los mismos datos son el mismo valor, así que son intercambiables.

- **La pregunta:** *¿me importa cuál de los dos es?* → **No** → value object.
- Se valida al construirse (si existe, es válido).
- Es inmutable: no se modifica, se crea otro.
- Se compara campo a campo.

Ejemplos: `Email`, `Money`, `Direccion`, `Sku`, `RangoDeFechas`.

### Entidad

Se define por su **identidad**. Tiene un identificador y sigue siendo la misma aunque cambien todos sus datos.

- **La pregunta:** *¿me importa cuál es, concretamente este y no otro?* → **Sí** → entidad.
- Se compara por identificador, nunca por datos.

Ejemplos: `Usuario`, `Pedido`, `LineaDePedido`.

### Agregado

Un grupo de objetos que solo tiene sentido completo y que se guarda, se valida y se recupera **entero**.

Se compone de:

- **Una entidad raíz** — siempre exactamente una, y siempre una entidad.
- **Cero o más entidades hijas.**
- **Cero o más value objects.**

Lo más común es que no haya entidades hijas en absoluto: `Usuario` con sus value objects es un agregado perfectamente normal.

```
  ,--- frontera de consistencia: lo que se guarda y se valida SIEMPRE junto ---.
  |                                                                            |
  |  +--------------------------+        +------------------------------+      |
  |  | Pedido         [RAÍZ]    |        | VALUE OBJECTS                |      |
  |  |                          |------->| Money (total)                |      |
  |  | añadirLinea()            |        | Direccion (envío)            |      |
  |  | enviar()                 |        | Sku                          |      |
  |  +--------------------------+        +------------------------------+      |
  |             | gobierna                                                     |
  |             v                                                              |
  |  +--------------------------+                                              |
  |  | LineaDePedido  [ENTIDAD] |  xN                                          |
  |  +--------------------------+                                              |
  |                                                                            |
  `----------------------------------------------------------------------------'
     ^                                        ^
     | entrar por la raíz: SÍ                 | acceso directo a las hijas: NO
```

**¿Dónde termina el grupo?** Lo que tiene que cuadrar **en el mismo instante** va junto. Lo demás, fuera.

De ahí sale una regla operativa:

> **Entre agregados distintos se referencia por identificador, no por objeto.**

```ts
class Pedido {
  private clienteId: UsuarioId;   // ✅ solo el id
  // private cliente: Usuario;    // ❌ arrastraría otro agregado entero
}
```

Y otra: **un repositorio por raíz de agregado, no uno por tabla.** Si `Pedido` vive en tres tablas, sigue habiendo un solo `RepositorioPedidos`.

### Invariante

Algo que tiene que ser verdad **siempre**, en todo momento en que alguien pueda mirar desde fuera. No «al crear» ni «al guardar»: siempre.

La **validación es el momento**; el **invariante es la promesa**. La validación es cómo proteges el invariante.

> **Cada invariante es, literalmente, una prueba unitaria.** Si listas los invariantes de un agregado, ya tienes escrito su plan de pruebas.

### Puerto

Una promesa, escrita como interfaz. Dice qué se puede pedir, no cómo se hace. No contiene lógica.

### Adaptador

La implementación concreta que cumple esa promesa y habla con el mundo real. Del mismo puerto puede haber varios: el de Postgres (producción) y el fake en memoria (pruebas).

### Caso de uso

Una operación completa del negocio, de principio a fin. Su trabajo es **coordinar**: pedir datos al repositorio, llamar al dominio, guardar el resultado, avisar a quien haga falta. **Las reglas de negocio no viven aquí**, viven en el dominio.

### Repositorio

El almacén de los agregados. Está diseñado para **parecer una colección en memoria**, no una base de datos — por eso el fake con un `Map` es la implementación más fiel al concepto, no un apaño.

No confundir con DAO:

| | DAO | Repositorio |
|---|---|---|
| Se organiza por | Tabla | **Agregado** |
| Devuelve | Filas o DTOs | **Objetos de dominio** |
| Operaciones | CRUD genérico | Las que el dominio necesita |

---

## 3. Las dos escuelas: Detroit y Londres

### Escuela de Detroit

*También llamada: clásica, classicist, Chicago, inside-out.*

Estilo de TDD en el que la unidad bajo prueba es una **unidad de comportamiento**, los colaboradores se ejercitan **reales** siempre que sea viable, y la verificación se hace sobre el **estado final del SUT** o sobre el **valor devuelto**.

| | |
|---|---|
| Qué es una «unidad» | Una porción de comportamiento con sentido de negocio. Puede abarcar varias clases |
| Qué significa «aislamiento» | Aislar **las pruebas entre sí**, para que corran en paralelo y en cualquier orden |
| Uso de dobles | Solo cuando lo real resulta inviable: lento, no determinista, o fuera del proceso |
| Dirección de diseño | *Middle-out*: se parte del dominio hacia fuera |
| Acoplamiento de la prueba | Al **contrato público** del objeto |
| Origen del nombre | La comunidad original de XP en el proyecto C3 de Chrysler, en Detroit |

### Escuela de Londres

*También llamada: mockista, mockist, outside-in.*

Estilo de TDD en el que la unidad es típicamente **una clase**, **todos** sus colaboradores se sustituyen por mocks, y la verificación se hace sobre la **interacción**: qué llamadas emitió el SUT, con qué argumentos y en qué orden.

| | |
|---|---|
| Qué es una «unidad» | Normalmente una clase o un método |
| Qué significa «aislamiento» | Aislar **el SUT de sus dependencias** |
| Uso de dobles | Mock para cualquier colaborador con comportamiento no trivial |
| Dirección de diseño | *Outside-in* / **Need-Driven Development**: las interfaces se descubren a partir de lo que el objeto *necesita* |
| Acoplamiento de la prueba | Al **protocolo de interacción** entre objetos |
| Origen del nombre | La comunidad de XP de Londres. Publicada en XP2000 y en *Extreme Programming Examined*, 2001 |

### La distinción en una línea

> **Detroit asevera el resultado**: el estado en que queda el SUT, o el valor que devuelve.
> **Londres asevera la interacción**: las llamadas que el SUT hizo a sus colaboradores.

Fowler separó explícitamente estos dos ejes en la revisión de 2007 de *Mocks Aren't Stubs*. Los términos *classicist* y *mockist* son suyos; nadie se puso esos nombres a sí mismo. **Fowler no declara ganador**: dice que él personalmente es clasicista y reconoce que su experiencia con el estilo mockista es limitada. Citarlo como si zanjara el debate es mal uso de la fuente.

### El ejemplo: el mismo caso de uso en los dos estilos

```ts
// ---------- ESTILO LONDRES (solitaria): todo doblado, se asevera LA LLAMADA
const repo = { guardar: vi.fn(), buscarPorEmail: vi.fn().mockResolvedValue(null) };
const notificador = { enviarBienvenida: vi.fn() };
const caso = new RegistrarUsuario(repo, notificador, reloj);

await caso.ejecutar({ email: 'ANA@acme.com', nombre: 'Ana' });

expect(repo.buscarPorEmail).toHaveBeenCalledWith('ana@acme.com');
expect(repo.guardar).toHaveBeenCalledWith(expect.objectContaining({ nombre: 'Ana' }));
expect(notificador.enviarBienvenida).toHaveBeenCalledOnce();
```

```ts
// ---------- ESTILO DETROIT (social): dominio real, fake en el repositorio
const repo = new RepositorioUsuariosEnMemoria();
const notificador = { enviarBienvenida: vi.fn() };
const caso = new RegistrarUsuario(repo, notificador, reloj);

await caso.ejecutar({ email: 'ANA@acme.com', nombre: 'Ana' });

const guardado = await repo.buscarPorEmail(Email.crear('ana@acme.com'));
expect(guardado!.nombre).toBe('Ana');
expect(guardado!.registradoEl).toEqual(FECHA);
expect(notificador.enviarBienvenida).toHaveBeenCalledOnce();
```

**Tres cosas que la versión Detroit detecta y la de Londres no:**

1. Que `Email` normaliza a minúsculas. En la de Londres el repositorio es un `vi.fn()`, así que **la prueba pasa aunque `Email` esté roto**.
2. Si mañana fusionamos `buscarPorEmail` y `guardar` en un solo `guardarSiNoExiste`, la prueba de Londres **se pone roja sin que el comportamiento haya cambiado**.
3. **Un mutante que rompa la normalización de `Email` sobrevive** en la versión de Londres, porque esa prueba nunca llegó a ejecutar `Email`.

El tercer punto es el que une las pruebas unitarias con el mutation testing.

### SUT, social y solitaria

**SUT** (*System Under Test*) es la cosa que se está probando en esa prueba concreta. Todo lo demás que participa son sus **colaboradores**. El término es de Meszaros. **Es un rol, no un tipo:** la misma clase es SUT en una prueba y colaborador en otra.

| Archivo | SUT | Colaboradores |
|---|---|---|
| `Money.spec.ts` | `Money` | — |
| `Pedido.spec.ts` | `Pedido` | `Money`, `LineaDePedido`, `Direccion` |
| `RegistrarUsuario.spec.ts` | `RegistrarUsuario` | `Usuario`, `Email`, repositorio fake, notificador |

Con ese término, las definiciones quedan exactas:

- **Solitaria:** los colaboradores del SUT se sustituyen por dobles.
- **Social:** los colaboradores del SUT se ejecutan reales.

**Tres precisiones:**

1. **Describe la topología, no la aserción.** Es un eje sobre qué grafo de objetos instancia la prueba, no sobre qué se comprueba al final. Por eso es **independiente** de Detroit/Londres.
2. **Se predica respecto de un colaborador concreto**, no de una capa entera. Por eso una misma prueba puede ser las dos cosas. Y en una prueba sin colaboradores —un value object puro— **el eje no aplica en absoluto**.
3. **Determina qué código de producción ejecuta la prueba**, y por tanto qué mutantes puede matar.

Los términos los acuñó **Jay Fields**; Fowler los recoge en su bliki *UnitTest*, donde además se niega a fijar una definición de «unidad». Su formulación es asimétrica a propósito: los clasicistas **tienden a** pruebas sociales, los mockistas **insisten en** solitarias.

> **Nuestra posición, en una frase: sociales hacia el dominio, solitarias hacia los puertos.**

**Nota sobre el fake:** un fake **sigue siendo un doble** (está en la taxonomía de Meszaros), así que técnicamente la prueba es solitaria hacia ese puerto. Lo que cambia el fake no es la clasificación sino las consecuencias: permite aseverar *estado* en vez de *llamadas*, con lo que el estilo de verificación se mantiene en Detroit.

### Los tres estilos de Khorikov

El vocabulario más preciso que existe sobre esto distingue **tres** estilos, no dos:

| Estilo | Definición | Escuela | Preferencia |
|---|---|---|---|
| **Basado en salida** (funcional) | Entrada → se asevera el valor devuelto. Solo para código sin efectos secundarios | Detroit | 1.º |
| **Basado en estado** | Se asevera el estado en que queda el SUT tras la operación | Detroit | 2.º |
| **Basado en comunicación** | Se asevera que los colaboradores fueron invocados correctamente | Londres | 3.º |

El basado en salida tiene la mejor protección frente a falsos positivos, porque su entrada y su salida cambian menos durante los refactors. **Una prueba de value object o de invariante de agregado es casi siempre del primer tipo** — y eso no es una elección de estilo, es una propiedad de la arquitectura.

---

## 4. La arquitectura de pruebas: tres capas

> **La discusión de Detroit contra Londres solo existe en una de las tres capas.** En las otras dos no hay nada que decidir.

### Capa 1 — Dominio

| | |
|---|---|
| **Contiene** | `Email`, `Money`, `Pedido`, `LineaDePedido` |
| **Prueba** | Unitaria sencilla: `new`, llamas, compruebas |
| **Dobles** | **Ninguno.** El dominio no depende de nada |
| **Escuela** | **No aplica.** Sin colaboradores externos no hay elección posible |
| **¿Se muta?** | **Sí**, y es donde mejor funciona |
| **Cuántas** | 4–6 por value object · **una por cada invariante** del agregado |

### Capa 2 — Aplicación

| | |
|---|---|
| **Contiene** | `RegistrarUsuario`, `EnviarPedido` (casos de uso) |
| **Prueba** | Unitaria también, pero con dobles en los puertos |
| **Dobles** | Fake la BD · mock el correo y los pagos · stub el reloj |
| **Escuela** | **Aquí, y solo aquí, existe la decisión.** Elegimos Detroit |
| **¿Se muta?** | **Sí** |
| **Cuántas** | Una por escenario de negocio |

### Capa 3 — Infraestructura

| | |
|---|---|
| **Contiene** | `TypeOrmPedidos`, `NotificadorSmtp`, el controlador |
| **Prueba** | **Integración**, no unitaria |
| **Dobles** | **Ninguno** — Postgres de verdad |
| **Escuela** | No aplica |
| **¿Se muta?** | **No** — lo cubren las de integración, que Stryker no ejecuta |
| **Cuántas** | Fidelidad del mapeo, ida y vuelta, restricciones de la BD |

**La simetría que lo hace fácil de recordar:** los dobles solo aparecen en la capa del medio. Arriba no hay nada que doblar; abajo, el adaptador *es* la cosa real que queremos probar.

### Qué aseveramos en cada elemento

| Elemento | Qué comprobamos | Dobles |
|---|---|---|
| **Value object** | Creación válida · un rechazo **en el límite** · normalización · igualdad por valor · inmutabilidad | Ninguno |
| **Entidad** | Igualdad por identificador · cada transición de estado y su guarda | Ninguno |
| **Agregado** | **Un caso por invariante** · que la operación ilegal lance · que un fallo no deje efectos a medias | Ninguno |
| **Caso de uso** | El resultado devuelto · el estado que deja (vía fake) · el comando saliente (vía mock) | Fake, mock, stub |
| **Puerto** | Nada: es una interfaz. Su contrato se prueba con la suite compartida | — |
| **Repositorio real** | Fidelidad del mapeo · ida y vuelta · restricciones de la BD | Ninguno |
| **Módulo de Nest** | Que todos los tokens se resuelven. Una sola prueba por módulo | El contenedor real |

---

## 5. Dónde está la frontera entre unitaria e integración

**El criterio es físico, no de tamaño:** una prueba deja de ser unitaria **en el momento en que sale del proceso**. Mientras todo viva en memoria es unitaria, aunque instancie quince objetos reales. Cuando necesita que exista algo más corriendo —un Postgres, un puerto de red, un fichero en disco, un servidor SMTP— es de integración.

No hay zona gris: **la frontera es la del proceso, y coincide exactamente con la frontera del hexágono.**

```
  +--------------+        +----------------------+        +-----------------+
  |  ADAPTADOR   |        |   APLICACIÓN         |        |   ADAPTADOR     |
  |  DE ENTRADA  |   |    |   + DOMINIO          |   |    |   DE SALIDA     |
  |              |   |    |                      |   |    |                 |
  | Controller   |---|--->| RegistrarUsuario     |---|--->| TypeOrm -> PG   |
  |              |   |    |   +-> Usuario, Email |   |    | Smtp -> correo  |
  +--------------+   |    +----------------------+   |    +-----------------+
                     |                               |
                  PUERTO                          PUERTO
            (frontera del hexágono)        (frontera del hexágono)

      todo en memoria  ------------------------->|  cruza el proceso  ->

  UNITARIA      ..............[=====================].......................
  INTEGRACIÓN   ..[=========]...........................[===============]..
  E2E           [=======================================================]
```

### Los tres niveles

| | **Unitaria** | **Integración** | **E2E** |
|---|---|---|---|
| **Pregunta que responde** | ¿la regla de negocio es correcta? | ¿el adaptador guarda y lee bien de verdad? | ¿está todo cableado? |
| **Qué es real** | Dominio y caso de uso | El adaptador y Postgres | Todo |
| **Qué se sustituye** | Solo los puertos | Nada | Nada |
| **Qué NO asevera** | Nada sobre el mapeo de persistencia | **Nada sobre reglas de negocio** | Detalle fino |
| **Fichero** | `*.spec.ts` | `*.int-spec.ts` | `*.e2e-spec.ts` |
| **Coste** | milisegundos | segundos | decenas de segundos |
| **Cuántas** | cientos | decenas | una o dos por flujo |

```ts
// 1) UNITARIA - RegistrarUsuario.spec.ts   (milisegundos, sin infraestructura)
const repo        = new RepositorioUsuariosEnMemoria();                // fake
const notificador = { enviarBienvenida: vi.fn() };                     // mock
const reloj       = new FixedClock(new Date('2026-10-20T00:00:00Z'));  // stub
const caso        = new RegistrarUsuario(repo, notificador, reloj);

await caso.ejecutar({ email: 'ana@acme.com', nombre: 'Ana' });

expect((await repo.buscarPorEmail(Email.crear('ana@acme.com')))!.nombre).toBe('Ana');
expect(notificador.enviarBienvenida).toHaveBeenCalledTimes(1);

// 2) INTEGRACIÓN - TypeOrmUsuarios.int-spec.ts  (Postgres real, cero reglas de negocio)
const repo = new TypeOrmUsuarios(await dataSourceDePrueba());
await repo.guardar(unUsuario({ email: 'ana@acme.com' }));

const leido = await repo.buscarPorEmail(Email.crear('ANA@acme.com'));
expect(leido?.email.valor).toBe('ana@acme.com');   // fidelidad del mapeo
```

### El reparto de responsabilidades

Las reglas de negocio se aseveran **solo** en el nivel unitario. El nivel de integración asevera algo más humilde: que el mapeo no pierde campos y que el índice único existe de verdad en el esquema.

No es criterio nuestro: la guía prescriptiva de AWS sobre arquitecturas hexagonales es explícita en que **los tests de integración de adaptadores secundarios no deben validar lógica de negocio**.

Si una regla se comprobara en los dos sitios, el día que cambie habría que editar dos capas de pruebas. **Ese es el síntoma de que la frontera está mal puesta.**

### Y el hueco que ninguno de los dos cubre

**El cableado.** El test unitario instancia el caso de uso con `new`, así que no comprueba que el módulo de NestJS inyecte el adaptador correcto; el de integración ejercita el adaptador suelto, no el camino desde HTTP. Para eso existe el e2e, y por eso basta con uno o dos por flujo.

---

## 6. La política de dobles

> **Regla única: no se dobla nada que no cruce la frontera del proceso.**

```
                 ¿la dependencia vive FUERA de mi proceso?
                                   |
            +----------------------+----------------------+
           NO                                            SI
            |                                             |
   OBJETO REAL, siempre            ¿alguien fuera de la aplicación lo notaría?
  Money, Email, LineaDePedido                   |
  (no hay escuela que elegir)      +------------+------------+
                                  NO                        SI
                                   |                         |
                           FAKE en memoria              MOCK / SPY
                      tu BD, tu caché, tu outbox    SMTP, pagos, bus compartido
                      aserción sobre el ESTADO      aserción sobre LA LLAMADA

  Caso aparte: entradas que solo necesitas CONTROLAR  ->  STUB de valor fijo
  Clock, IdGenerator (FixedClock). Nunca se asevera que fueron llamados.
  Y el Logger que el constructor exige pero el test no observa  ->  DUMMY.
```

### La taxonomía de dobles (Meszaros)

| Doble | ¿Tiene lógica? | ¿Recuerda? | Qué aseveras | Coste |
|---|---|---|---|---|
| **Dummy** | No | No | **Nada** | Trivial |
| **Stub** | No | No | El resultado obtenido | 1 línea |
| **Spy** | No | Registra llamadas | Que la llamada ocurrió | 1 línea |
| **Mock** | No | Registra llamadas | Que la llamada ocurrió, con qué argumentos | 1 línea |
| **Fake** | **Sí** | **Sí** | **El estado en que quedó** | ~30 líneas, una vez |

### Casos resueltos

| Lo que usas | ¿Se dobla? | Por qué |
|---|---|---|
| `lodash`, `date-fns`, `zod` | **No** | Vive en tu proceso, es puro y rápido |
| Paquete propio compartido (`@empresa/validaciones`) | **No** | Igual: código en memoria |
| Otro módulo del monolito, llamada directa | **No** | Sigue sin cruzar el proceso |
| Ese mismo módulo, pero por HTTP a otro servicio | **Mock** | Ahora cruza, y otro equipo depende de esa llamada |
| Tu Postgres | **Fake** | Cruza, pero nadie de fuera sabe cómo guardas |
| SMTP, pasarela de pagos | **Mock** | Cruza y **alguien de fuera lo nota** |
| `new Date()`, `crypto.randomUUID()` | **Stub** | Determinismo |
| El logger | **Dummy** | No es comportamiento observable del negocio |

**Ojo con «externo»:** no significa «lo escribió otro», significa «está fuera de mi proceso». `date-fns` dentro de un value object se usa real aunque sea de terceros; Postgres se dobla aunque la base de datos sea tuya.

### Las dos excepciones legítimas

Se permite doblar un colaborador **interno** solo en dos casos, y se documenta el motivo en un comentario:

1. **Es lento aunque esté en memoria.** El caso clásico: el hasheo de contraseñas (`argon2`, `bcrypt`) está diseñado para tardar 100 ms o más. Multiplicado por 200 pruebas son 20 segundos.
2. **Es no determinista.** El reloj y el aleatorio.

### Un detalle que arruina el determinismo

**Un agregado que llama a `new Date()` o a `crypto.randomUUID()` por dentro vuelve sus propios invariantes imposibles de testear deterministamente.** Inyecta un `Clock` y un `IdGenerator` como puertos de pleno derecho; un `FixedClock` es un doble de una línea.

---

## 7. Fakes y suite de contrato

### Qué es un fake, exactamente

Una **implementación funcional con un atajo que la hace inadecuada para producción** (definición de Fowler; su ejemplo canónico es precisamente una base de datos en memoria).

**No se construye con ninguna librería de mocking. Es una clase que escribes a mano**, con la misma interfaz que la real:

```ts
// test/fakes/RepositorioUsuariosEnMemoria.ts
export class RepositorioUsuariosEnMemoria implements RepositorioUsuarios {
  private readonly datos = new Map<string, Usuario>();

  async guardar(usuario: Usuario): Promise<void> {
    this.datos.set(usuario.email.valor, usuario);
  }

  async buscarPorEmail(email: Email): Promise<Usuario | null> {
    return this.datos.get(email.valor) ?? null;
  }

  async contar(): Promise<number> {
    return this.datos.size;
  }
}
```

El «atajo» es que al apagar el proceso se pierde todo, no tiene transacciones y no maneja concurrencia. Para una prueba, nada de eso importa.

### La diferencia clave: el fake recuerda

Para probar «no se admite el mismo email dos veces»:

```ts
// ❌ Con stub: programas a mano la secuencia de respuestas
repo.buscarPorEmail = vi.fn()
  .mockResolvedValueOnce(null)
  .mockResolvedValueOnce(usuarioFalso);
// Estás escribiendo la respuesta que deberías estar comprobando.

// ✅ Con fake: ejecutas el escenario
const repo = new RepositorioUsuariosEnMemoria();
const caso = new RegistrarUsuario(repo, notificador, reloj);

await caso.ejecutar({ email: 'ana@acme.com' });
await expect(caso.ejecutar({ email: 'ANA@acme.com' }))
  .rejects.toThrow(EmailYaRegistrado);
expect(await repo.contar()).toBe(1);
```

El primer registro **realmente se guardó**, así que el segundo **realmente lo encuentra**. Y el `ANA@` en mayúsculas comprueba de paso que `Email` normaliza — algo invisible con un stub.

### Coste real

**Uno por puerto, no uno por prueba.** Unas 20–30 líneas, escritas una sola vez por el dev que crea el puerto, en el mismo PR. Vive en el repositorio como código mantenido.

En la práctica, **uno o dos fakes por módulo**: el repositorio y poco más. El reloj y el generador de IDs son tan triviales que les basta un stub de una línea.

### La suite de contrato

El único remedio contra la deriva del fake: un **único cuerpo de aserciones** que expresa la promesa del puerto, ejecutado dos veces.

```ts
// test/contratos/RepositorioUsuarios.contract.ts
export function contratoRepositorioUsuarios(crear: () => Promise<RepositorioUsuarios>) {
  it('lo que guardas, lo recuperas por su email', async () => {
    const repo = await crear();
    const usuario = unUsuario({ email: 'ana@acme.com' });
    await repo.guardar(usuario);
    expect(await repo.buscarPorEmail(Email.crear('ANA@acme.com'))).toEqual(usuario);
  });

  it('rechaza un email ya existente', async () => { /* ... */ });
}

// RepositorioUsuariosEnMemoria.spec.ts      -> unitaria, milisegundos
contratoRepositorioUsuarios(async () => new RepositorioUsuariosEnMemoria());

// TypeOrmUsuarios.int-spec.ts               -> integración, Postgres real
contratoRepositorioUsuarios(async () => new TypeOrmUsuarios(await dataSourceDePrueba()));
```

### Cuándo el fake NO es la herramienta correcta

Si el puerto es de **consulta con proyecciones complejas** —filtros, ordenaciones, agregaciones SQL— el fake acaba reimplementando semántica de SQL en TypeScript. En ese caso se salta directamente a integración con Postgres real.

---

## 8. Cómo se escribe una prueba

```ts
// RegistrarUsuario.spec.ts
it('rechaza un email ya registrado y no envía bienvenida', async () => {
  const repo        = new RepositorioUsuariosEnMemoria([unUsuario({ email: 'ana@acme.com' })]);
  const notificador = { enviarBienvenida: vi.fn() };
  const caso        = new RegistrarUsuario(repo, notificador, new FixedClock(FECHA));

  await expect(caso.ejecutar({ email: 'ana@acme.com' })).rejects.toThrow(EmailYaRegistrado);

  expect(await repo.contar()).toBe(1);                          // estado
  expect(notificador.enviarBienvenida).not.toHaveBeenCalled();  // efecto externo
});
```

### Convenciones

| Convención | Por qué |
|---|---|
| **Se instancia con `new`, no con el `TestingModule` de NestJS** | Es instantáneo y no mete el framework en el bucle. **Aviso:** la documentación oficial de Nest recomienda lo contrario (auto-mocking), así que alguien lo citará en una revisión de código. Conviene tenerlo decidido de antemano |
| **Se nombra por comportamiento, no por método** | `rechaza un email ya registrado`, no `test de validarEmail` |
| **Constructores de datos** (`unUsuario({ ... })`) | El día que el agregado gane un campo obligatorio se toca un fichero, no doscientas pruebas |
| **Un concepto por prueba** | Si falla, el nombre ya dice qué se rompió |
| **Ficheros de spec pequeños** | Mejora la reutilización del modo incremental de Stryker, que en Vitest rastrea cambios **por fichero** |
| **Separación por sufijo, no por carpeta** | `*.spec.ts` / `*.int-spec.ts` / `*.e2e-spec.ts`, para que el CI pueda correr solo el nivel rápido |
| **Pruebas independientes del orden** | `coverageAnalysis: perTest` está forzado en el runner de Vitest y lo exige |

### Los constructores de datos

```ts
// test/builders/usuario.ts
export const unUsuario = (over: Partial<DatosUsuario> = {}): Usuario =>
  Usuario.registrar(
    Email.crear(over.email ?? 'test@acme.com'),
    over.nombre ?? 'Test',
    over.registradoEl ?? new Date('2026-01-01T00:00:00Z'),
  );
```

---

## 9. Qué NO probamos

Decir esto vale tanto como lo anterior, porque es lo que evita que la suite se llene de ruido:

- Getters y setters sin lógica
- DTOs y mappers triviales
- Adaptadores y repositorios reales → **eso es integración**
- El framework (NestJS)
- **Excepción:** una sola prueba por módulo que compruebe que el contenedor resuelve todos los tokens

### Una defensa estructural que cuesta muy poco

Una aserción estilo ArchUnit que compruebe que `dominio` no importa nada de `infraestructura` ni de `@nestjs/*`. Es en sí mismo un test unitario baratísimo y evita la clase entera de fugas de persistencia en tests de dominio.

---

## 10. Herramientas: Vitest

**Vitest es el ejecutor de pruebas**: encuentra los ficheros, los corre y reporta. Es el sustituto moderno de Jest, construido sobre Vite, mucho más rápido arrancando — lo cual importa el triple aquí, porque Stryker lo va a invocar miles de veces.

```ts
// vitest.config.ts — separación del nivel rápido y el de integración
export default defineConfig({
  test: {
    include: ['src/**/*.spec.ts', 'test/**/*.spec.ts'],
    exclude: ['**/*.int-spec.ts', '**/*.e2e-spec.ts'],
    environment: 'node',   // NUNCA jsdom/happy-dom para dominio: cuesta tiempo por mutante
    globals: false,
  },
});
```

```json
// package.json
{
  "scripts": {
    "test:unit": "vitest run",
    "test:int":  "vitest run --include '**/*.int-spec.ts'",
    "test:e2e":  "vitest run --include '**/*.e2e-spec.ts'",
    "mutation":  "stryker run"
  }
}
```

---

## 11. Herramientas: StrykerJS

**StrykerJS no ejecuta pruebas: orquesta a Vitest.** Genera los mutantes y le pregunta a Vitest si las pruebas los detectan.

### Cómo funciona por dentro: mutant schemata

```
  SIN schemata (la forma naíf)            CON schemata (Stryker >= 4.0)
  ----------------------------            -----------------------------
  por cada uno de los 2.000 mutantes:     compila UNA vez el código instrumentado,
    1. modificar el fichero                con TODOS los mutantes dentro, detrás de
    2. COMPILAR TypeScript  <-- caro       interruptores:
    3. correr las pruebas
    4. revertir                              activo === 7 ? precio / cantidad
                                           : activo === 8 ? precio + cantidad
  = 2.000 compilaciones = inviable        : /* original */ precio * cantidad

                                          luego, por mutante: activar un interruptor
                                          con una variable de entorno y correr SOLO
                                          las pruebas que cubren esa línea (perTest)
```

Esto llegó en **Stryker 4.0**. En TypeScript con Vitest **no hay recompilación por mutante**, lo cual es la diferencia entre horas y minutos.

### Requisitos y rupturas de versión

| Versión | Cambio que puede morderte |
|---|---|
| **v10** | **Node ≥ 22.** Si estás en Node 20, o actualizas o fijas StrykerJS en la línea 9.x. **Es el bloqueador más común** |
| v9 | Vitest ≥ 2 |
| v7 | `disableTypeChecks: true` por defecto |
| v6 | ESM puro: la config es `stryker.config.mjs`, y un fichero que *exporte una función* es un error |
| v5 | `coverageAnalysis: perTest` por defecto (exige tests independientes del orden) |

Versión actual verificada: **`@stryker-mutator/core@10.0.0`**.

```bash
npm i -D @stryker-mutator/core \
         @stryker-mutator/vitest-runner \
         @stryker-mutator/typescript-checker
```

### Configuración

```js
// stryker.config.mjs
export default {
  packageManager: 'npm',
  testRunner: 'vitest',
  reporters: ['html', 'clear-text', 'progress'],

  // coverageAnalysis se OMITE deliberadamente: el runner de Vitest lo ignora
  // y siempre usa perTest. La mayor palanca de tiempo está siempre activa.

  mutate: [
    'src/**/dominio/**/*.ts',
    'src/**/aplicacion/**/*.ts',
    '!src/**/*.spec.ts',
    '!src/**/*.int-spec.ts',
    '!src/**/*.e2e-spec.ts',
    '!src/**/*.module.ts',
    '!src/**/*.dto.ts',
  ],

  ignoreStatic: true,            // legal aquí porque perTest está forzado
  checkers: ['typescript'],
  tsconfigFile: 'tsconfig.json',

  incremental: true,
  incrementalFile: 'reports/stryker-incremental.json',

  concurrency: 4,                // ajustar a los núcleos reales del runner de CI
  thresholds: { high: 90, low: 80, break: null },  // break: null al principio
};
```

**Advertencia sobre el glob:** `*.entity.ts` solo debe excluirse si en tu proyecto significa «entidad de ORM». Si significa «entidad de dominio», es justo lo contrario y hay que mutarla.

---

## 12. Mutation testing: conceptos

**La cobertura mide qué líneas se ejecutaron. No mide si alguna aserción se habría enterado del error.** Se puede llegar al 100 % de cobertura sin una sola aserción útil.

```
tu código  ->  StrykerJS genera los mutantes  ->  Vitest corre las pruebas
                                                   que cubren esa línea
                                                          |
                                           ¿alguna se puso roja?
                                                          |
                          +-------------------------------+---------------+
                         SÍ                                              NO
                          |                                               |
                     MUERTO                                        SOBREVIVE
          esa línea está vigilada                 te falta una comprobación ahí
```

### Cómo se hacen los mutantes

**No los escribimos nosotros.** Stryker lee el código y aplica un catálogo de cambios pequeños:

| Lo que había | Lo que pone el mutante |
|---|---|
| `a + b` | `a - b` |
| `x > 0` | `x >= 0` |
| `a && b` | `a \|\| b` |
| `if (algo)` | `if (true)` |
| `return true` | `return false` |
| `'texto'` | `''` |
| una línea | la borra entera |

```ts
// Nuestra regla
enviar(): void {
  if (this.lineas.length === 0) throw new PedidoSinLineas();
  this.estado = EstadoPedido.ENVIADO;
}

// Lo que Stryker probará, él solo:
//   if (this.lineas.length === 1)
//   if (this.lineas.length !== 0)
//   borrar el throw entero
```

Si borramos el `throw` y todo sigue en verde, **no tenemos ninguna prueba que compruebe «un pedido sin líneas no se puede enviar»**.

### Cubre las dos clases de prueba unitaria

Stryker muta el **código de producción**, no las pruebas. Así que la pregunta no es «¿a qué pruebas se aplica?» sino «¿qué código mutamos?»:

| Código que se muta | Quién mata esos mutantes | ¿Esas pruebas usan dobles? |
|---|---|---|
| `Email`, `Money`, `Pedido` | Las unitarias **sencillas** | **No, ninguno** |
| `RegistrarUsuario` | Las unitarias **con fake y mock** | Sí |
| `TypeOrmPedidos` | *No se muta* | — |

**Los value objects son el mejor terreno de todos.** El catálogo de mutantes (`>` → `>=`, `===` → `!==`) son exactamente los **errores de límite**, que es justo lo que un value object protege. Mutation testing sobre value objects es, en la práctica, una máquina que encuentra los casos límite olvidados — y son funciones puras que corren en microsegundos.

### Estados del mutante

| Estado | Significa | ¿Cuenta en el score? |
|---|---|---|
| **Killed** | Alguna prueba falló | Sí |
| **Survived** | Ninguna prueba falló | Sí |
| **No coverage** | Ninguna prueba toca esa línea | Sí (en el denominador total) |
| **Timeout** | La prueba se colgó — cuenta como muerto | Sí |
| **Runtime error** | El mutante revienta por otro motivo | **No** |
| **Compile error** | El mutante no compila | **No** |
| **Ignored** | Suprimido con una razón | **No** |

**Dos consecuencias que hay que interiorizar:**

1. Los mutantes `Ignored`, `Runtime error` y `Compile error` **no están en el denominador**, así que suprimir un mutante con una razón es *neutral* para el score, no inflacionario. La supresión dirigida es higiene legítima, no trampa.
2. **Lee siempre las dos cifras juntas** (score global y score sobre código cubierto). Una brecha grande significa que tu problema es de cobertura, no de fuerza de las aserciones.

### El techo matemático

Entre el **5,7 % y el 7,4 %** de los mutantes son **equivalentes**: cambios que no alteran el comportamiento observable y que por tanto es **imposible** matar. Cualquier umbral por encima de **~93 %** es matemáticamente irracional.

**No existe ninguna cifra publicada de «buen mutation score».**

### Por qué merece la pena

- Entre el **70 % y el 73 %** de los fallos reales están acoplados a al menos un mutante ordinario (Just et al., FSE 2014; replicación industrial de Google, ICSE 2021).
- El mutation score correlaciona con la detección de fallos reales **significativamente mejor que la cobertura de sentencias**.
- **87,5 %** de los mutantes mueren con los tests existentes (dato de Google): la mayor parte del trabajo ya está hecha.

### El síntoma que destapa

**Cobertura cercana al 100 % con mutation score cercano a cero** es la firma del test tautológico (*testing the mock*, Pereira 2010): la expectativa del mock y la implementación son la misma frase escrita dos veces, así que el test no puede fallar cuando el comportamiento es incorrecto, solo cuando la implementación cambia.

---

## 13. El tiempo de ejecución

### El modelo de coste

```
  tiempo total  ~=  Nº de mutantes  x  tiempo de las pruebas que cubren cada mutante
                    |                  |
                    |                  +- palanca B: pruebas rápidas (fakes, no BD)
                    |                                perTest (forzado en Vitest)
                    |
                    +- palanca A: acotar `mutate` a dominio + aplicación,
                                  ignoreStatic, typescript-checker, incremental

  Es un PRODUCTO, no una suma: partir un factor por la mitad parte el total.
```

Fórmula más precisa:

```
T ≈ T_dry_run + Σ_mutantes T_tests_que_lo_cubren   (+ T_typecheck si hay checker)
```

**La `dry run` inicial es obligatoria y no se puede eliminar**: descubre los tests, calcula la cobertura per-test y comprueba que Stryker arranca. Se ejecuta incluso en modo incremental.

> **La decisión arquitectónica de la sección 6 es, de hecho, la mayor optimización de mutation testing del proyecto.** Si las unitarias usaran una base de datos real, cada una tardaría cientos de milisegundos; multiplicado por 2.000 mutantes el pipeline sería inviable. Con fakes en memoria tardan milisegundos. Esa batalla ya está ganada.

### Las palancas, ordenadas por retorno sobre esfuerzo

| # | Palanca | Qué hace | Esfuerzo | Contrapartida |
|---|---|---|---|---|
| 1 | **`coverageAnalysis: perTest`** | Cada mutante solo dispara las pruebas que cubren esa línea, no la suite entera | **Ninguno**: el runner de Vitest lo fuerza | No hay escotilla de escape: si tu suite depende del orden, hay que arreglar los tests |
| 2 | **Acotar `mutate` a dominio + aplicación** | Elimina de raíz los mutantes de adaptadores, DTOs, módulos y config | 5 minutos de glob | Dejas de medir esas capas (que ya cubre integración) |
| 3 | **Pruebas rápidas** (fakes, `environment: 'node'`) | Baja el segundo factor del producto | Ya hecho por arquitectura | Ninguna |
| 4 | **`ignoreStatic: true`** | Los mutantes en inicializadores estáticos no se re-evalúan por prueba y son desproporcionadamente caros | Una línea | Los saca del denominador. Mide el coste en tu repo antes de fijarlo |
| 5 | **`checkers: ['typescript']`** | Descarta mutantes que no compilan **sin ejecutar pruebas** | Una línea | Añade `T_typecheck` |
| 6 | **Modo incremental** | Reutiliza resultados de código no cambiado | Persistir un artefacto en CI | Caché potencialmente obsoleta: riesgo de *veracidad de la métrica* |
| 7 | **Ficheros de spec pequeños** | Mejora la reutilización incremental, que en Vitest es **por fichero** | Reorganizar specs | Ninguna |
| 8 | **`concurrency`** | Paralelismo | Una línea | Ajustar a los núcleos reales del runner, no a los del portátil |

### Una ventaja estructural de TypeScript

En TS con Vitest **no hay recompilación por mutante** (mutant schemata: un solo bundle con interruptores). Compáralo con C++, donde un caso documentado tardó **47.009 s** frente a suites de 0,1–0,8 s. La arquitectura de la herramienta domina sobre la configuración, y aquí está en el lado bueno.

**La consecuencia incómoda pero clarificadora:** en nuestro stack, **el tiempo de la suite unitaria es el multiplicador dominante**, mucho más que en Java o C++. Toda inversión en que las unitarias sean rápidas se multiplica por miles.

---

## 14. Estrategia de CI y pull requests

### ¿Podemos correr solo lo que modificamos?

**Sí**, y es exactamente lo que hace Google en producción: mutan únicamente **líneas cambiadas y ya cubiertas por tests**, con un máximo de **un mutante por línea**. Su mediana es de **820 mutantes por changelist** sobre un monorepo de 5.000 changelists muestreados.

Hay **dos formas** de hacerlo, y la diferencia importa:

#### Opción A — Acotar con `--mutate` desde el diff

```bash
CHANGED=$(git diff --name-only origin/main...HEAD -- 'src/**/*.ts' \
  | grep -v -e '\.spec\.ts$' -e '\.int-spec\.ts$' | paste -sd, -)
npx stryker run --mutate "$CHANGED"
```

StrykerJS admite además **rangos de líneas**: `--mutate "fichero:inicio-fin"`, lo que convierte «mutar solo lo que toqué» en un script sobre `git diff`.

- **Requiere `fetch-depth: 0`** en CI, porque sin historial no existe `origin/main` para comparar.
- **Limitación:** solo te informa de los ficheros cambiados. Pierdes la foto del proyecto.

#### Opción B — Modo incremental (recomendada)

Stryker guarda los resultados de cada mutante y, en la siguiente pasada, **reutiliza** los del código que no cambió.

- **Ventaja decisiva:** sigue reportando el mutation score del **proyecto completo** pagando solo el coste de lo que cambió.
- Ejemplo oficial: **3.731 de 3.965 resultados reutilizados (~94 % de reducción)** tras cambiar 1 fichero fuente y 2 de test.
- Requiere Stryker ≥ 6.2. Se activa con `--incremental`.

### Dos correcciones a documentación que circula mal

> **1. StrykerJS NO tiene un flag `--since` ni `--in-diff`.**
> Eso es **Stryker.NET**. Infection (PHP) tiene `--git-diff-lines`. Vas a encontrar artículos citándolos para JavaScript y están equivocados.

> **2. El modo incremental compara el CONTENIDO de los ficheros** (con `diff-match-patch` de Google), **no usa git.**
> Por tanto `--incremental` **no** necesita `fetch-depth: 0`. Eso solo hace falta para el acotado manual de la opción A. Lo que sí necesita es que el fichero incremental **sobreviva entre ejecuciones de CI**.

### El detalle que todo el mundo olvida: cachear el fichero incremental

```yaml
- uses: actions/cache@v4
  with:
    path: reports/stryker-incremental.json
    key: stryker-incremental-${{ github.sha }}
    restore-keys: stryker-incremental-
```

Si no sobrevive, cada pasada incremental es en realidad una pasada completa.

### Limitación del incremental con Vitest

**El runner de Vitest rastrea cambios de test «por fichero»** (Jest y CucumberJS dan detalle completo). Consecuencia muy real: **tocar un test en un fichero con 40 tests invalida los resultados de todos los mutantes cubiertos por ese fichero.**

De ahí la convención de **ficheros de spec pequeños**.

### Por qué el diff solo no basta

Si únicamente mutas lo que cambió, **nunca detectas**:

- Que una prueba **ya existente** se debilitó con el tiempo (alguien le quitó una aserción en otro PR).
- Mutantes en código **cuyo llamante** cambió: modificas el caso de uso y eso altera qué ramas del agregado se ejercitan, pero el agregado no está en tu diff.

**Por eso van dos ejecuciones con propósitos distintos.**

### Los niveles de CI

| | **Nivel 1 — Cada PR** | **Nivel 2 — Nocturno en `main`** | **Nivel 3 — Semanal / pre-release** |
|---|---|---|---|
| **Qué corre** | Incremental, acotado a lo afectado | Pasada completa de dominio + aplicación | Pasada completa + informes históricos |
| **Duración objetivo** | **< 5 min** | Lo que haga falta | Sin límite |
| **¿Bloquea?** | **Sí** — es la única puerta | **No** (`continue-on-error: true`) | No |
| **Para qué** | Que no entre código nuevo sin comprobar | Detectar decaimiento de la suite | Tendencia y dashboard |
| **Reporte** | Comentario en el PR | Artefacto + dashboard | Histórico |

### Sobre los umbrales

En `thresholds` **solo `break` bloquea la build**; `high` y `low` son colores del reporte, no puertas.

```js
thresholds: { high: 90, low: 80, break: null }   // fase 1: reporta, no bloquea
```

**Recomendación de arranque:**

1. **Fase 1:** `break: null`. La herramienta reporta. El equipo se acostumbra a leer el informe.
2. **Fase 2:** `break: 90` sobre el diff del PR.
3. **Fase 3:** subir progresivamente, **nunca por encima de ~93 %**.

> Un `break: 100` el primer día va a chocar con mutantes equivalentes en el diff, y la primera experiencia del equipo será «esta herramienta me bloquea por algo imposible» — exactamente la impresión que no quieres crear.

**Y una advertencia sobre porcentajes en pasadas acotadas:** si un PR toca tres líneas y genera 4 mutantes, matar 3 da 75 % y falla un umbral del 80 % sin que eso signifique nada. Los denominadores pequeños hacen el porcentaje ruidoso. Para el PR es más sano **revisar la lista de mutantes supervivientes en el código nuevo** y reservar el umbral porcentual para la pasada nocturna.

### Esqueleto del workflow

```yaml
name: calidad
on: [pull_request]

jobs:
  unitarias:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: npm }
      - run: npm ci
      - run: npm run test:unit

  mutacion:
    needs: unitarias
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }      # solo si usas acotado manual por diff
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: npm }
      - run: npm ci
      - uses: actions/cache@v4
        with:
          path: reports/stryker-incremental.json
          key: stryker-incremental-${{ github.sha }}
          restore-keys: stryker-incremental-
      - run: npx stryker run --incremental
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: informe-mutacion
          path: reports/mutation/
```

---

## 15. Plan de adopción

### Fase 0 — Verificación previa

- [ ] Confirmar **Node ≥ 22** (bloqueador más común de StrykerJS 10)
- [ ] Confirmar Vitest ≥ 2
- [ ] Decidir y documentar la convención de sufijos (`*.spec.ts` / `*.int-spec.ts` / `*.e2e-spec.ts`)

### Fase 1 — Un módulo piloto

1. Elegir **un** módulo con lógica de negocio real.
2. Clasificar sus dependencias con el árbol de decisión de la sección 6.
3. **Enumerar los invariantes** de sus agregados. Esa lista es el plan de pruebas.
4. Escribir las pruebas de value objects, agregado y caso de uso.
5. Escribir el fake del repositorio y la suite de contrato.
6. Correr Stryker acotado a ese módulo, con `break: null`.
7. **Medir dos números reales:** cuántos mutantes genera y cuánto tarda la pasada completa.

### Fase 2 — Diseñar el pipeline con datos

Con esos dos números, y no antes, se fija la duración objetivo del PR, el `concurrency` y el primer umbral. **No se estima: se mide.**

### Fase 3 — Extender

- Un módulo por sprint.
- Revisar los supervivientes en cada PR, no el porcentaje.
- Subir el umbral progresivamente.

### Fase 4 — Régimen

- Nocturna completa con histórico.
- El umbral se congela por debajo de ~93 %.

---

## 16. Antipatrones prohibidos

| # | Antipatrón | Por qué |
|---|---|---|
| 1 | **Mockear el dominio para probar un caso de uso** | El más frecuente y el más caro. Si mockeas el agregado, la prueba no comprueba ninguna regla |
| 2 | `expect(repositorio.guardar).toHaveBeenCalledWith(...)` | Asevera implementación. Si mañana agrupas dos guardados en uno, la prueba se rompe aunque el comportamiento sea idéntico |
| 3 | **Aserciones sobre el logger** | No es comportamiento observable del negocio |
| 4 | **Repetir reglas de negocio en el nivel de integración** | Cada regla se asevera en un solo sitio |
| 5 | **Un fichero de test espejo uno a uno de cada fichero de producción** | Es la definición londinense de «unidad» adoptada sin decidirlo |
| 6 | **Usar la entidad del ORM como entidad de dominio** | El ORM necesita constructores vacíos y propiedades públicas; un agregado necesita lo contrario. Pierdes la capacidad de proteger invariantes |
| 7 | **`new Date()` dentro de un agregado** | Vuelve sus propios invariantes imposibles de testear deterministamente |
| 8 | **Perseguir el 100 % de mutation score** | Matemáticamente imposible: hay mutantes equivalentes |
| 9 | **Perseguir el 100 % de cobertura como objetivo** | Se alcanza sin una sola aserción útil |

---

## 17. Definición de «terminado»

Un PR que añade o modifica lógica de dominio está terminado cuando:

- [ ] Cada **invariante** nuevo o modificado tiene su caso de prueba
- [ ] Los **value objects** nuevos tienen: creación válida, un rechazo **en el límite**, normalización, igualdad
- [ ] Si se creó un **puerto**, se entregaron en el mismo PR: la interfaz, el adaptador real, el **fake** y la **suite de contrato**
- [ ] Las pruebas de caso de uso **no mockean el dominio**
- [ ] No hay aserciones sobre llamadas a dependencias **gestionadas**
- [ ] El módulo tiene su prueba de resolución de tokens
- [ ] La pasada incremental de Stryker **no deja mutantes supervivientes nuevos sin justificar**
- [ ] Los ficheros de spec son pequeños (favorece el incremental)

---

## 18. Glosario

| Término | Significado |
|---|---|
| **Agregado** | Grupo de objetos que se guarda, valida y recupera entero |
| **Adaptador** | Implementación concreta de un puerto |
| **Arid node** | Nodo de código sin lógica de negocio donde mutar no aporta señal |
| **Coupling Effect** | Hipótesis de que los tests que detectan fallos simples detectan también los complejos |
| **Dependencia gestionada** | Fuera de proceso, bajo tu control total (tu BD) → **fake** |
| **Dependencia no gestionada** | Fuera de proceso, sin control total (SMTP, pagos) → **mock** |
| **Detroit / clásica** | Escuela que asevera estado y salida |
| **Dry run** | Pasada inicial obligatoria de Stryker: descubre tests y calcula cobertura |
| **Entidad** | Objeto definido por su identidad |
| **Frontera de consistencia** | El borde del agregado |
| **Invariante** | Afirmación que debe ser cierta siempre |
| **Londres / mockista** | Escuela que asevera interacción |
| **Mutant schemata** | Instrumentar todos los mutantes a la vez detrás de interruptores |
| **Mutante** | Versión del código con un cambio pequeño introducido a propósito |
| **Mutante equivalente** | Mutante que no altera el comportamiento observable: imposible de matar |
| **Mutation score** | `killed / (mutantes válidos)` |
| **Puerto** | Interfaz que expresa una promesa, sin lógica |
| **Raíz del agregado** | Única entidad a la que se habla desde fuera |
| **Social (sociable)** | Los colaboradores del SUT corren reales |
| **Solitaria** | Los colaboradores del SUT están doblados |
| **SUT** | *System Under Test*: lo que se está probando en esa prueba |
| **TDD tautológica** | Test cuya expectativa y cuya implementación son la misma frase |
| **Value object** | Objeto definido por su contenido, inmutable y autovalidado |

---

*Documento de arquitectura de pruebas · 21 de octubre de 2026*
