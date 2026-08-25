**Birds of Colombia**

Colombia is world-renowned for its biodiversity, particularly for its birds that inhabit multiple thermal floors and regions.

Currently, information about our birds is scattered across multiple sources, making it difficult to access consistent and structured data on their taxonomic classification and ecosystems.

For enthusiasts, academics, and environmental conservation professionals, having a centralized and standardized source of bird information represents a practical need.

For this reason, the development of a REST API is proposed to efficiently manage basic bird information. This API will provide programmatic access to structured data about birds, their taxonomic classification, and ecosystem, facilitating both individual queries and comparative analysis of these animals in the biological and environmental fields.

The data model has three different tables with the following information:

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

Data example:

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

**ID fields must be generated as UUID data type**. This data type must be consistent across primary and foreign keys. **APIs with integer-type ID fields will not be accepted.**

With these specifications, a REST API is to be built implementing the repository pattern, with a layered distribution as follows:

| Controller | Handling of requests and responses associated with HTTP verbs   |
| ----------- | --------------------------------------------------------------- |
| Service    | Implementation of business rule validations                      |
| Repository | Execution of CRUD actions associated with each operation         |
| Model      | Classes that define the state and behavior of entities           |
| Context    | Database connection.                                              |

**Inventory of requests the API is expected to implement**

**Aves**

- List all birds  
   ( GET /api/aves)

- List a bird by Id  
   ( GET /api/ave/{ave_id} )

- Add a bird  
   ( POST /api/aves)

- Update a bird  
   ( PUT /api/ave)

- Delete a bird  
   ( DEL /api/aves/{ ave \_id} )

**Familias**

- List all taxonomic families  
   ( GET /api/familias)

- List a taxonomic family by Id.  
   ( GET /api/familias/{familia_id} )

- List birds by taxonomic family Id  
   ( GET /api/familias/{familia_id} /aves)

- Add a taxonomic family  
   ( POST /api/ familias)

- Update a taxonomic family  
   ( PUT /api/ familias)

- Delete a taxonomic family  
   ( DEL /api/ familias/{familia_id})

**Ecosistemas:**

- List all ecosystems  
   ( GET /api/ecosistemas)

- List an ecosystem by Id.  
   ( GET /api/ ecosistemas /{ecosistema_id} )
- List birds by ecosystem Id.  
   ( GET /api/ ecosistemas /{ecosistema_id}/aves)
- Add an ecosystem  
   ( POST /api/ ecosistemas)
- Update an ecosystem  
   ( PUT /api/ ecosistemas)
- Delete an ecosystem  
   ( DEL /api/ ecosistemas /{ ecosistema_id} )

To validate the API's functionality, it must be used to register 5 birds with their respective family and ecosystem data. Where possible, select birds that share common families and ecosystems so that cross-entity queries can be demonstrated.
