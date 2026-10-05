# Sistema de Votaciones — Definición previa al desarrollo del Prototype 1

## 1. Objetivo

El equipo desarrollará un **sistema de votaciones distribuido** cuyo primer prototipo implementará el *happy path* de una votación electrónica, desde la autenticación del votante hasta el registro de una papeleta cifrada y la entrega de un recibo.

El diseño debe cumplir con los requisitos del Prototype 1 de Software Architecture:

- Definir el dominio y el enfoque funcional del sistema.
- Definir el alcance funcional del prototipo.
- Utilizar una arquitectura distribuida.
- Incluir un componente de presentación web.
- Incluir al menos dos componentes de lógica.
- Incluir una base de datos relacional y una NoSQL.
- Utilizar al menos dos tipos diferentes de conectores basados en HTTP.
- Utilizar al menos tres lenguajes de propósito general.
- Utilizar un despliegue orientado a contenedores.

---

# 2. Alcance del Prototype 1

## 2.1. Happy path

El flujo principal que se pretende demostrar es:

```text
Login
  ↓
MFA
  ↓
Validación en padrón
  ↓
Emisión de token anónimo
  ↓
Firma ciega
  ↓
Descarga de parámetros de elección
  ↓
Cifrado local del voto
  ↓
Generación de prueba de validez
  ↓
Envío de papeleta
  ↓
Validación del token y prueba
  ↓
Registro en MongoDB
  ↓
Recibo firmado
  ↓
Confirmación del voto
```

El prototipo debe concentrarse inicialmente en este camino exitoso. Funcionalidades como recuperación ante errores, escenarios excepcionales, auditorías completas, cierre de urna y escrutinio pueden implementarse posteriormente o quedar documentadas como trabajo futuro, dependiendo del alcance acordado por el equipo.

---

# 3. Arquitectura propuesta

La arquitectura conceptual propuesta es:

```text
                    ┌─────────────────────────┐
                    │   Navegador (Next.js)   │
                    │   Cifra el voto         │
                    └────────────┬────────────┘
                                 │
                                 ▼
                    ┌─────────────────────────┐
                    │ Cloud Armor +            │
                    │ Load Balancer            │
                    └────────────┬────────────┘
                                 │
              ┌──────────────────┴──────────────────┐
              │                                     │
              ▼                                     ▼
    ┌──────────────────────┐             ┌──────────────────────┐
    │    Zona identidad    │             │     Zona votación    │
    │                      │             │                      │
    │ Python: identidad    │             │ Go: recepción        │
    │ PostgreSQL           │             │ Redis                │
    │ Pub/Sub: eventos     │             │ Pub/Sub: votos       │
    │ TypeScript: avisos   │             │ Go: escritor         │
    │                      │             │ MongoDB               │
    │                      │             │ Rust: boletín         │
    │                      │             │ Escrutinio            │
    └──────────────────────┘             └──────────────────────┘
```

La arquitectura se divide conceptualmente en dos zonas:

- **Zona de identidad:** maneja la autenticación y la relación entre una persona y su derecho a votar.
- **Zona de votación:** recibe y procesa papeletas sin necesitar conocer la identidad del votante.

La separación entre ambas zonas es fundamental para evitar que la información utilizada durante el login pueda vincularse posteriormente con la papeleta.

---

# 4. Componentes

| Componente | Responsabilidad | Tecnología |
|---|---|---|
| Navegador | Interfaz, MFA, generación/cegado del token, cifrado del voto y pruebas | Next.js / TypeScript |
| Identidad | Usuarios, roles, padrón, elecciones y emisión de tokens | Python |
| Base de identidad | Padrón, usuarios, roles y estado de votación | PostgreSQL |
| Eventos de identidad | Avisos y auditoría | Pub/Sub |
| Avisos | Notificaciones por correo/SMS | TypeScript |
| Recepción | Validación del token y prueba; entrada de papeletas | Go |
| Tokens usados | Control de tokens consumidos | Redis |
| Eventos de votación | Transporte de eventos/papeletas | Pub/Sub |
| Escritor | Persistencia de papeletas y encadenamiento por hash | Go |
| Base de papeletas | Almacenamiento de votos cifrados | MongoDB |
| Boletín | Publicación de información de solo lectura | Rust |
| Escrutinio | Procesamiento de resultados al cierre | Por definir |

---

# 5. Happy path detallado

## 5.1. Login — Python

El usuario inicia sesión mediante el navegador.

1. El navegador solicita autenticación.
2. El usuario se autentica mediante MFA.
3. Python valida las credenciales y el segundo factor.
4. Python consulta PostgreSQL.
5. Se verifica que:
   - el usuario pertenece al padrón de la elección/urna;
   - la urna está abierta;
   - el usuario está habilitado para votar.

El resultado de esta etapa permite al usuario continuar con la obtención de un token anónimo.

---

## 5.2. Emisión del token anónimo — Python

El objetivo es obtener un token firmado que permita votar sin revelar posteriormente la identidad del votante.

1. El navegador genera un token aleatorio.
2. El navegador ciega el token mediante un esquema de firma ciega.
3. El navegador envía el token cegado a Python.
4. Python verifica en PostgreSQL que el usuario todavía no haya recibido un token para esa elección.
5. Debe existir una restricción equivalente a:

```text
UNIQUE(user_id, election_id)
```

6. Python firma el token cegado.
7. Python registra el evento de emisión del token.
8. El navegador recibe la firma.
9. El navegador desciega el token.
10. El resultado es un token firmado que Python no puede vincular posteriormente con la papeleta.

### Propiedad buscada

Python conoce:

```text
usuario → recibió token
```

pero no debe poder conocer:

```text
usuario → papeleta X
```

---

# 6. Parámetros de la elección

El navegador descarga los parámetros necesarios para construir una papeleta válida:

- Identificador de la elección.
- Clave pública de la elección.
- Lista de opciones.
- Firma de los parámetros.

El navegador debe poder verificar que los parámetros utilizados corresponden a una elección válida antes de cifrar el voto.

---

# 7. Cifrado local

El usuario selecciona una opción en el navegador.

El navegador:

1. Obtiene la opción seleccionada.
2. Cifra el voto utilizando la clave pública de la elección.
3. Genera la prueba de validez correspondiente.
4. Calcula un identificador de papeleta.

El voto en claro no debe enviarse al servidor.

El servidor de votación recibe únicamente la información necesaria para validar y registrar la papeleta cifrada.

---

# 8. Auditoría opcional — Benaloh

El usuario puede disponer de una opción de auditoría.

Conceptualmente:

```text
Usuario selecciona opción
        ↓
Se genera voto cifrado
        ↓
¿Auditar?
   ┌────┴────┐
   │         │
  Sí         No
   │         │
Auditar    Emitir
   │
Repetir proceso
```

El alcance exacto de esta funcionalidad deberá definirse antes de implementarla. El equipo debe decidir qué parte de la auditoría Benaloh será realmente demostrada en el Prototype 1.

---

# 9. Envío de la papeleta — Go

El navegador envía la papeleta al componente de recepción.

La petición debe contener conceptualmente:

```json
{
  "election_id": "...",
  "encrypted_vote": "...",
  "proof": "...",
  "signed_token": "..."
}
```

### Restricción importante

Esta petición debe realizarse:

- sin cookies;
- sin sesión de usuario;
- sin identificadores de identidad;
- sin información que permita vincular directamente la papeleta con el login.

El objetivo es mantener la separación entre la identidad y la papeleta.

---

# 10. Validación y registro — Go

El componente Go realiza la validación de la papeleta.

### 10.1. Token

Go verifica la firma del token utilizando la clave pública correspondiente de Python.

Go **no debe necesitar llamar a Python** para realizar esta verificación.

### 10.2. Prueba

Go valida la prueba de validez asociada con el voto cifrado.

### 10.3. Token gastado y papeleta

Si las validaciones son exitosas:

1. Se verifica que el token no haya sido utilizado anteriormente.
2. Se registra el token como gastado.
3. Se registra la papeleta.
4. La papeleta se encadena mediante hash con la papeleta anterior.

La operación de consumo del token y registro de la papeleta debe diseñarse como una operación atómica para evitar que un token pueda utilizarse dos veces.

---

# 11. MongoDB

MongoDB almacenará las papeletas de la elección.

Un esquema conceptual inicial podría ser:

```text
ballot
├── ballot_id
├── election_id
├── encrypted_vote
├── proof
├── token_hash
├── previous_hash
├── ballot_hash
└── timestamp
```

El esquema definitivo debe ser acordado por el equipo antes de implementar el componente.

La papeleta debe poder formar parte de una cadena de hashes:

```text
Ballot 1
   │
   ▼
Ballot 2
   │
   ▼
Ballot 3
   │
   ▼
...
```

De esta forma se puede detectar una modificación de la secuencia publicada.

---

# 12. Recibo

Después de registrar correctamente la papeleta:

1. Go genera un recibo.
2. El recibo está asociado al hash de la papeleta.
3. Go firma el recibo.
4. El navegador recibe el recibo.
5. El navegador lo muestra al usuario.

Conceptualmente:

```text
Papeleta
   ↓
Hash
   ↓
Recibo firmado
   ↓
Usuario
```

El usuario debe poder conservar el recibo para posteriormente verificar que su papeleta aparece en la información pública.

---

# 13. Confirmación del voto

Después de recibir el recibo, debe definirse cómo se informa al componente de identidad que el usuario ya votó.

El diseño propuesto contempla:

```text
Go
 ↓
Recibo
 ↓
Navegador
 ↓
Confirmación
 ↓
Python
 ↓
Usuario marcado como "votó"
```

También puede considerarse un mecanismo asíncrono.

### Decisión pendiente

El equipo debe definir cómo realizar esta confirmación **sin introducir una relación entre la identidad del votante y la papeleta registrada**.

Esta es una de las decisiones arquitectónicas que debe resolverse antes de implementar el flujo completo.

---

# 14. Cierre y escrutinio

Al cerrar la urna:

1. Se detiene la recepción de nuevos votos.
2. Los responsables utilizan la clave correspondiente distribuida mediante un esquema de umbral.
3. Se descifran las papeletas.
4. Se calcula el resultado.
5. Se publican los resultados.
6. Se publican las pruebas necesarias para verificar el proceso.
7. Los ciudadanos pueden verificar sus recibos contra la lista pública de papeletas.

El alcance exacto del escrutinio debe definirse según lo que el equipo quiera demostrar en el prototipo.

---

# 15. API: contratos que deben definirse

Antes de comenzar el desarrollo, el equipo debe definir los contratos HTTP de cada componente.

Como mínimo, se deben especificar operaciones equivalentes a:

```text
POST /login
POST /tokens/blind
GET  /elections/{election_id}/parameters
POST /ballots
POST /vote-confirmation
```

Para cada endpoint se debe documentar:

- Método HTTP.
- URL.
- Request.
- Response.
- Headers.
- Autenticación.
- Códigos HTTP.
- Errores posibles.
- Información que puede contener la petición.
- Información que está prohibido enviar.
- Componente responsable.

Los nombres definitivos de los endpoints quedan a decisión del equipo.

---

# 16. Conectores HTTP

El Prototype 1 exige al menos **dos tipos diferentes de conectores basados en HTTP**.

El equipo debe decidir explícitamente cuáles utilizará y en qué relaciones entre componentes.

Esta decisión debe quedar documentada en la vista C&C y en los contratos de las APIs.

---

# 17. Modelo de datos

## 17.1. PostgreSQL

Como punto de partida:

```text
users
roles
elections
voter_roll
issued_tokens
```

Debe definirse:

- Relaciones.
- Claves primarias.
- Claves foráneas.
- Índices.
- Restricciones de unicidad.
- Estado de la elección.
- Estado del votante.
- Registro de tokens emitidos.

Restricción crítica:

```text
UNIQUE(user_id, election_id)
```

## 17.2. MongoDB

Colección principal:

```text
ballots
```

Debe definirse el esquema de:

- Papeleta cifrada.
- Prueba.
- Elección.
- Hash anterior.
- Hash propio.
- Identificador de papeleta.
- Información temporal necesaria.

---

# 18. Seguridad y criptografía

Antes de implementar, el equipo debe acordar explícitamente:

- Algoritmo de firma de Python.
- Algoritmo de firma del recibo.
- Esquema de firma ciega.
- Algoritmo de cifrado del voto.
- Esquema de generación de pruebas de validez.
- Formato de los tokens.
- Generación de aleatoriedad.
- Gestión de claves.
- Almacenamiento de claves privadas.
- Distribución de claves públicas.
- Esquema de clave repartida en umbral.
- Qué información conoce cada componente.
- Qué información nunca debe cruzar entre las zonas.

No se deben comenzar las implementaciones criptográficas sin haber definido primero estas decisiones y sus interfaces.

---

# 19. Despliegue orientado a contenedores

La arquitectura debe poder ejecutarse mediante contenedores.

Una organización inicial podría ser:

```text
docker-compose
│
├── frontend
├── identity-python
├── voting-go
├── ballot-writer
├── bulletin-rust
├── postgres
├── mongodb
├── redis
└── pubsub
```

El despliegue definitivo dependerá de la infraestructura elegida por el equipo.

Se debe definir:

- Imagen de cada componente.
- Variables de entorno.
- Redes.
- Puertos.
- Dependencias.
- Volúmenes.
- Configuración de bases de datos.
- Gestión de secretos.
- Orden/inicialización de servicios.

---

# 20. Lenguajes

La arquitectura propuesta contempla al menos:

- **TypeScript** — navegador/Next.js y avisos.
- **Python** — identidad.
- **Go** — recepción y escritura de papeletas.
- **Rust** — boletín.

Esto satisface el requisito de utilizar al menos tres lenguajes de propósito general.

El equipo debe decidir cuáles serán realmente incluidos en el Prototype 1 y evitar introducir un lenguaje únicamente para cumplir el requisito si no tiene una responsabilidad arquitectónica clara.

---

# 21. Responsabilidades del equipo

Antes de desarrollar, se debe asignar explícitamente la responsabilidad de cada componente.

Por ejemplo:

| Área | Responsable |
|---|---|
| Next.js / navegador | Por definir |
| Python / identidad | Por definir |
| PostgreSQL | Por definir |
| Go / recepción | Por definir |
| Go / escritor | Por definir |
| MongoDB | Por definir |
| Redis | Por definir |
| Pub/Sub | Por definir |
| Rust / boletín | Por definir |
| Criptografía | Por definir |
| Docker / integración | Por definir |
| Documentación | Por definir |

Una persona puede tener varias responsabilidades, pero debe existir un responsable claro para cada componente.

---

# 22. Decisiones que deben estar cerradas antes de programar

La reunión de arquitectura debería terminar con respuestas concretas a estas preguntas:

### Dominio y funcionalidad

- ¿Qué problema exacto resuelve el sistema?
- ¿Qué actores existen?
- ¿Qué es una elección?
- ¿Qué es una urna?
- ¿Qué es un votante?
- ¿Qué constituye un voto válido?

### Alcance

- ¿Qué partes del happy path estarán implementadas?
- ¿Qué partes quedarán fuera del Prototype 1?
- ¿Qué funcionalidades serán simuladas?

### Arquitectura

- ¿Cuáles son exactamente los componentes?
- ¿Cuál es la responsabilidad de cada uno?
- ¿Qué componente puede comunicarse con cuál?
- ¿Qué información puede cruzar entre la zona de identidad y la zona de votación?

### APIs

- ¿Cuáles son los endpoints?
- ¿Qué requests y responses utilizan?
- ¿Qué autenticación utiliza cada endpoint?
- ¿Cuáles son los dos tipos de conectores HTTP?
- ¿Qué endpoint recibe la papeleta sin sesión?

### Datos

- ¿Cuál es el esquema de PostgreSQL?
- ¿Cuál es el esquema de MongoDB?
- ¿Qué información se almacena en Redis?
- ¿Qué eventos se envían mediante Pub/Sub?

### Seguridad

- ¿Cómo funciona la firma ciega?
- ¿Cómo se genera y verifica el token?
- ¿Cómo se cifra el voto?
- ¿Cómo se genera la prueba?
- ¿Cómo se valida?
- ¿Cómo se firman los recibos?
- ¿Cómo se evita el doble voto?
- ¿Cómo se evita vincular identidad y papeleta?

### Infraestructura

- ¿Cómo se ejecutan todos los componentes?
- ¿Qué contenedores existen?
- ¿Cómo se conectan?
- ¿Cómo se configuran?
- ¿Cómo se inicia el sistema desde cero?

### Equipo

- ¿Quién desarrolla cada componente?
- ¿Quién integra?
- ¿Quién valida la seguridad?
- ¿Quién documenta?
- ¿Cómo se integran las ramas y cambios?

---

# 23. Artefactos que deberíamos producir antes del desarrollo

El equipo debería cerrar estos seis artefactos:

1. **Alcance funcional del Prototype 1**
2. **Diagrama C&C definitivo**
3. **Contratos de las APIs**
4. **Modelo de datos PostgreSQL + MongoDB**
5. **Diseño de seguridad y criptografía**
6. **Diseño de despliegue con contenedores**

Después de esto, el desarrollo puede dividirse por componentes sin que cada integrante tenga que tomar decisiones arquitectónicas contradictorias.

---

# 24. Resultado esperado

Al finalizar esta etapa, cualquier integrante del equipo debería poder responder:

> **¿Qué estamos construyendo?**
>
> Un sistema de votación distribuido.

> **¿Qué vamos a demostrar en el Prototype 1?**
>
> El happy path desde el login hasta el registro de una papeleta cifrada y la generación de un recibo.

> **¿Cómo está dividido?**
>
> En componentes de presentación, identidad, recepción, persistencia, publicación y escrutinio.

> **¿Cómo evitamos vincular identidad y voto?**
>
> Mediante la separación entre la zona de identidad y la zona de votación y el uso de un token obtenido mediante firma ciega.

> **¿Qué falta decidir?**
>
> Los contratos HTTP, los esquemas definitivos de datos, los algoritmos criptográficos, la confirmación del voto, los conectores HTTP concretos, el despliegue y la distribución de responsabilidades.

Solo después de cerrar estas decisiones debería comenzar la implementación coordinada del prototipo.
