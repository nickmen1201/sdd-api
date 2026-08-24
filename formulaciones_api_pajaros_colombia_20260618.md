**Aves de Colombia**

Colombia es reconocida mundialmente por su riqueza en biodiversidad, particularmente por sus aves que habitan en múltiples pisos térmicos y regiones.

Actualmente, la información sobre nuestras aves se encuentra dispersa en múltiples fuentes que dificultan el acceso a datos consistentes y estructurados sobre su clasificación taxonómica y sus ecosistemas.

Para aficionados, académicos y profesionales de la conservación ambiental, contar con una fuente centralizada y estandarizada de información sobre las aves representa una necesidad práctica.

Por esta razón, se propone el desarrollo de una API REST que permita gestionar de manera eficiente la información básica de las aves. Esta API proporcionará acceso programático a datos estructurados sobre las aves, sus clasificación taxonómica y ecosistema, facilitando tanto la consulta individual como el análisis comparativo de estos animales en las áreas biológicas y la ambientales.

El modelo de datos tiene tres tablas con la siguiente información:

**Aves:**

- id (PK)
- nombre_cientifico
- nombre_comun
- nombre_indigena
- lengua_indigena
- familia (FK a tabla familias)
- ecosistema (FK a tabla ecosistemas)

**Familias Taxonómica:**

- id (PK)
- nombre
- orden

**Ecosistema:**

- id (PK)
- nombre
- Zona Geográfica.

Datos Ejemplo:

**Ave:**

| **Nombre Científico** | Vultur gryphus      |
| --------------------- | ------------------- |
| **Nombre Común**      | Cóndor de los Andes |
| **Nombre Indígena**   | Kuntur              |
| **Lengua Indígena**   | Quechua             |
| **Familia**           | Cathartidae         |

**Familias Taxonómica:**

| **Nombre** | Cathartidae    |
| ---------- | -------------- |
| **Orden**  | Cathartiformes |

**Ecosistema:**

| **Nombre**          | Páramo |
| ------------------- | ------ |
| **Zona Geográfica** | Andina |

**Los campos ID son generados como tipo de dato UUID**. Este tipo de datos debe ser consistente en claves primarias y foráneas. **No se acepta API con campos ID tipo entero.**

Con estas especificaciones, se desea construir una API REST que implemente el patrón repositorio, con una distribución por capas de la siguiente manera:

| Controlador | Manejo de peticiones y respuestas asociadas a verbos HTTP      |
| ----------- | -------------------------------------------------------------- |
| Servicio    | Implementación de las validaciones de las reglas del negocio   |
| Repositorio | Ejecución de acciones CRUD asociadas a cada operación          |
| Modelo      | Clases que definen el estado y comportamiento de las entidades |
| Contexto    | Conexión a la base de datos.                                   |

**Inventario de peticiones que se espera implemente la API**

**Aves**

- Listar todas las aves  
   ( GET /api/aves)

- Listar un ave por Id  
   ( GET /api/ave/{ave_id} )

- Agregar un ave  
   ( POST /api/aves)

- Actualizar un ave  
   ( PUT /api/ave)

- Eliminar un ave  
   ( DEL /api/aves/{ ave \_id} )

**Familias**

- Listar todas las familias taxonómicas  
   ( GET /api/familias)

- Listar una familia taxonómica por Id.  
   ( GET /api/familias/{familia_id} )

- Listar aves por Id de familia taxonómica  
   ( GET /api/familias/{familia_id} /aves)

- Agregar una familia taxonómica  
   ( POST /api/ familias)

- Actualizar una familia taxonómica  
   ( PUT /api/ familias)

- Eliminar una familia taxonómica  
   ( DEL /api/ familias/{familia_id})

**Ecosistemas:**

- Listar todos los ecosistemas  
   ( GET /api/ecosistemas)

- Listar un ecosistema por Id.  
   ( GET /api/ ecosistemas /{ecosistema_id} )
- Listar aves por Id de ecosistema.  
   ( GET /api/ ecosistemas /{ecosistema_id}/aves)
- Agregar un ecosistema  
   ( POST /api/ ecosistemas)

- Actualizar un ecosistema  
   ( PUT /api/ ecosistemas)

- Eliminar un ecosistema  
   ( DEL /api/ ecosistemas /{ ecosistema_id} )

Para validar el funcionamiento del API, debe utilizarse para registrar 5 aves con sus respectivos datos de familias y ecosistemas. En lo posible, seleccione aves que tengan familias y ecosistemas comunes para que pueda demostrar las consultas cruzadas entre entidades.