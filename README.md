# serviciobiblioteca - Guía 8: Spring Data JPA (con las 4 actividades)

API REST de la biblioteca con **Spring Boot 4 + Spring Data JPA**, resuelta sobre
el proyecto base de la guía (`pe.edu.unsaac.biblioteca`). Incluye las **4 actividades
evaluadas** del apartado *1.1. Aplicación de la guía*.

## Requisitos

- JDK 17 o superior (probado con JDK 21)
- MySQL Server 8 y MySQL Workbench
- Maven (o usar el wrapper incluido: `mvnw` / `mvnw.cmd`)
- Postman para las pruebas

## 1. Base de datos

Opción A (base existente del estudiante): ejecute su script
`bdbiblioteca20261119.sql` y luego la migración de la Actividad 4:

- `sql/actividad4_lector_migracion.sql` → crea la tabla `lector`, migra los
  lectores que ya estaban en `prestamo.lector` a la nueva FK `lector_id` y
  elimina la columna de texto.

Opción B (instalación completa desde cero):

- `sql/bdbiblioteca_completa_con_lector.sql` → crea `bdbiblioteca` con las
  **5 tablas** (autor, categoria, lector, libro, prestamo) y los datos de
  siempre: 9 autores, 7 categorías, 6 lectores, 20 libros y 10 préstamos
  (4 activos y 6 devueltos), más el procedimiento `sp_mantenimiento_autores`.

Revise `src/main/resources/application.properties` y ajuste usuario y clave de
MySQL (por defecto `root/root`, puerto 3307, base `bdbiblioteca`).

## 2. Ejecución

```bash
mvnw spring-boot:run
```

La API queda en `http://localhost:8080/api/...`. Importe en Postman la
colección `Postman_Guia8_Actividades.postman_collection.json` para probar
todos los endpoints.

## 3. Lo que se implementó

| Actividad | Endpoints | Código nuevo/modificado |
|---|---|---|
| 1. CRUD de autores | `POST /api/autores` (201), `PUT /api/autores/{id}` (200/404), `DELETE /api/autores/{id}` (204/409) | `AutorRequest`, `AutorServicio.crear/actualizar/eliminar`, `AutorController` |
| 2. CRUD de categorías | `GET/POST/PUT/DELETE /api/categorias` (201/200/204, 400/404/409) | `CategoriaRequest`, `CategoriaServicio`, `CategoriaController`, `CategoriaRepositorio.findByNombreIgnoreCase`, `LibroRepositorio.existsByCategoriaId` |
| 3. Consultas nuevas | `GET /api/libros/rango?desde=2010&hasta=2020`, `GET /api/libros/mas-prestados` | `LibroRepositorio.findByAnioBetweenOrderByAnio` y JPQL `masPrestados()`, `LibroMasPrestadoDTO`, `LibroServicio`, `LibroController` |
| 4. Préstamos normalizados | `POST /api/prestamos` con `lectorId`, `GET /api/lectores/{id}/prestamos` | entidad `Lector`, `LectorRepositorio`, `LectorController`, `Prestamo` con `@ManyToOne lector_id`, `PrestamoRequest(lectorId)`, `PrestamoDTO`, `PrestamoServicio`, `PrestamoRepositorio.findByLectorIdOrderByFechaPrestamoDesc` |

Reglas de negocio implementadas:

- Autor: nombre de al menos 3 caracteres (400); no se elimina un autor con
  libros registrados (409).
- Categoría: nombre de al menos 3 caracteres (400); no se repite el nombre,
  ignorando mayúsculas/minúsculas, con `findByNombreIgnoreCase` (409); no se
  elimina una categoría con libros (409).
- Préstamo: el libro debe existir (404) y estar disponible (409); el lector
  debe existir (404); registrar préstamo y marcar el libro ocurre en una sola
  transacción (`@Transactional`); la devolución vuelve a dejarlo disponible.
- Los errores salen en JSON gracias a `ManejadorErrores` (`@RestControllerAdvice`).

## 4. Estructura

```
src/main/java/pe/edu/unsaac/biblioteca/
├── controladores/   AutorController, CategoriaController, LibroController,
│                    LectorController, PrestamoController, MantenimientosController
├── dto/             AutorDTO, AutorRequest, CategoriaRequest, CategoriaResumenDTO,
│                    LibroDTO, LibroMasPrestadoDTO, LibroRequest, PrestamoDTO,
│                    PrestamoRequest, AutorMantenimientoDTO
├── entidades/       Autor, Categoria, Lector, Libro, Prestamo
├── excepciones/     NoEncontradoException, ConflictoException,
│                    ValidacionException, ManejadorErrores
├── repositorios/    AutorRepositorio, CategoriaRepositorio, LibroRepositorio,
│                    LectorRepositorio, PrestamoRepositorio
├── servicios/       AutorServicio, CategoriaServicio, LibroServicio,
│                    PrestamoServicio, MantenimientoServicio
├── ServiciobibliotecaApplication.java
└── ServletInitializer.java
```

El módulo de mantenimientos de la Guía 7 (`MantenimientosController`, SP
`sp_mantenimiento_autores`) se mantiene igual, porque no usa la columna
`prestamo.lector`.
