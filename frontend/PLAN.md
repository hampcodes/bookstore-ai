# Plan: Clean Code + Seguridad + PayPal + Email — Backend & Frontend

## Context
Proyecto SaaS de librería con Spring Boot + Angular. Tiene ~50 endpoints con código redundante (14 de catálogo haciendo lo mismo con JPQL/Native/SP). Se necesita: limpiar código, agregar funcionalidades (perfil, detalle libro, imagen, slug), mejorar seguridad (OWASP), integrar pagos (PayPal) y notificaciones (Gmail), dockerizar y usar variables de entorno.

---

## Fase 1: Slug en Book

**Backend:**
- Crear `util/SlugUtil.java` — Normalizer NFD, strip acentos, lowercase, hyphens
- `Book.java` — agregar `slug` (String, unique, not null)
- `BookRepository` — `findBySlugWithAuthorAndStore()`, `existsBySlug()`
- `BookResponse` — agregar `slug`
- `BookMapper` — `@Mapping(target = "slug", ignore = true)` en toEntity
- `BookServiceImpl.create()` — auto-generar slug con `SlugUtil.toSlug(title)`, unicidad con sufijo `-2`, `-3`
- `BookServiceImpl.update()` — regenerar slug si título cambió
- `BookController` — `@GetMapping("/slug/{slug}")` público
- `IBookService` — agregar `findBySlug(String slug)`

**Frontend:**
- `book.model.ts` — agregar `slug: string` a BookResponse
- `book.service.ts` — agregar `findBySlug(slug): Observable<BookResponse>`

---

## Fase 2: Catálogo con JPA Specification (14 endpoints → 2)

**Backend — archivos nuevos:**
- `specification/BookSpecification.java` — filtros: title, genre, author, minPrice, maxPrice, storeId, inStock
- `specification/AuthorSpecification.java` — filtros: name, nationality, minBooks
- `service/ICatalogService.java` + `service/impl/CatalogServiceImpl.java`

**Backend — modificar:**
- `BookRepository` — agregar `extends JpaSpecificationExecutor<Book>`, ELIMINAR: searchByTitle, findByGenre, searchByAuthorName, findByPriceRange, findByGenrePaged, findBooksInStockNative, findBooksByAuthorNationalityNative, getBookStatsByGenreNative, getBooksByGenreSP, getBooksByPriceRangeSP
- `AuthorRepository` — agregar `extends JpaSpecificationExecutor<Author>`, ELIMINAR: searchByLastName, findByNationality, searchByName, findByNationalityNative, findAuthorsWithMinBooks, getAuthorsByNationalitySP
- `CatalogController` — reescribir con solo 2 endpoints:
  - `GET /api/catalog/books?title=&genre=&author=&minPrice=&maxPrice=&storeId=&inStock=&page=&size=`
  - `GET /api/catalog/authors?name=&nationality=&minBooks=&page=&size=`
- `IBookService` — eliminar métodos de búsqueda
- `BookServiceImpl` — eliminar implementaciones de búsqueda
- `IAuthorService` — eliminar métodos de búsqueda
- `AuthorServiceImpl` — eliminar implementaciones de búsqueda

**N+1 en Specification:** Patrón two-query (IDs con Specification → batch fetch con JOIN FETCH)

**Frontend:**
- `catalog.service.ts` — un solo método `searchBooks(filters)` con HttpParams dinámicos
- `catalog-page.component.ts` — usar `catalogService.searchBooks()` para browse + search

---

## Fase 3: Imagen en Book

**Backend — archivos nuevos:**
- `service/FileStorageService.java` — saveBookImage(), deleteBookImage(), @PostConstruct init()
- `config/WebConfig.java` — resource handler `/uploads/**`
- Dependencia `thumbnailator:0.4.20` en pom.xml

**Backend — modificar:**
- `Book.java` — agregar `imageUrl` (String)
- `BookResponse` — agregar `imageUrl`
- `BookMapper` — `@Mapping(target = "imageUrl", ignore = true)` en toEntity
- `BookController` — POST/PUT cambian a `@RequestPart("book")` + `@RequestPart("image")`
- `IBookService` — firmas: `create(BookRequest, MultipartFile)`, `update(Long, BookRequest, MultipartFile)`
- `BookServiceImpl` — guardar imagen con FileStorageService usando slug como nombre
- `SecurityConfig` — permitir `GET /uploads/**`
- `application.yaml` — `spring.servlet.multipart.max-file-size: 5MB`

**Frontend:**
- `book.model.ts` — agregar `imageUrl: string | null` a BookResponse
- `book.service.ts` — create/update envían FormData (Blob JSON + File)
- `book-form.component.ts|html` — input file con preview de imagen
- `catalog-page.component.html` — mostrar imagen en cards

---

## Fase 4: Página detalle libro

**Frontend — archivos nuevos:**
- `features/catalog/book-detail/book-detail.component.ts|html|css`
  - Recibe `:slug` de la ruta
  - Carga libro con `bookService.findBySlug(slug)`
  - Carga reviews con `reviewService.findByBookId(bookId)`
  - Muestra: imagen, título, autor, género, precio, stock, descripción, store
  - Botón "Agregar al carrito" (requiere login)
  - Lista de reviews con estrellas
  - Formulario de review inline (si logueado como CUSTOMER)

**Frontend — modificar:**
- `catalog.routes.ts` — agregar `{ path: ':slug', component: BookDetailComponent }`
- `catalog-page.component.html` — cards enlazan a `/catalog/{{book.slug}}` en vez de agregar directo al carrito

---

## Fase 5: Perfil de usuario

**Backend — archivos nuevos:**
- `dto/response/UserProfileResponse.java` — email, firstName, lastName, dni, phone, roleName
- `dto/request/UpdateProfileRequest.java` — firstName, lastName, phone
- `dto/request/ChangePasswordRequest.java` — currentPassword, newPassword
- `service/IUserService.java` + `service/impl/UserServiceImpl.java`
- `controller/UserController.java` — `/api/users`
  - `GET /profile` — obtener perfil
  - `PUT /profile` — actualizar datos
  - `PUT /profile/password` — cambiar contraseña

**Frontend — archivos nuevos:**
- `models/user.model.ts` — UserProfileResponse, UpdateProfileRequest, ChangePasswordRequest
- `services/user.service.ts` — getProfile(), updateProfile(), changePassword()
- `features/profile/profile.component.ts|html|css` — formulario editar perfil + cambiar contraseña

**Frontend — modificar:**
- `app.routes.ts` — agregar `/profile` con authGuard
- `navbar.component.html` — agregar "Mi Perfil" en menú usuario

---

## Fase 6: Eliminar auto-registro de store owner

**Backend — eliminar:**
- `AuthController.registerStoreOwner()` endpoint
- `IAuthService.registerStoreOwner()` método
- `AuthServiceImpl.registerStoreOwner()` implementación
- `RegisterStoreOwnerRequest.java` archivo

**Backend — crear:**
- `controller/AdminController.java` — `@PreAuthorize("hasRole('ADMIN')")`
  - `POST /api/admin/users` — crear usuario con rol específico (BOOKSTORE_OWNER, CUSTOMER)
- `dto/request/CreateUserRequest.java` — email, password, roleName

**Backend — modificar:**
- `StoreRequest.java` — agregar `ownerId` (opcional)
- `StoreServiceImpl.create()` — si admin y ownerId presente, usar ese owner

**Frontend — modificar:**
- `auth.service.ts` — eliminar `registerStoreOwner()`
- `auth.model.ts` — eliminar `RegisterStoreOwnerRequest`

---

## Fase 7: Eliminar endpoints redundantes

**Backend — eliminar:**
- `GET /api/sales/my-sales` de SaleController (GET /api/sales ya filtra por rol)
- `DELETE /api/stores/{id}` de StoreController
- `GET /api/reviews` findAll de ReviewController

**Backend — agregar:**
- `PUT /api/stores/{id}/deactivate` en StoreController (soft delete)

**Frontend — modificar:**
- Actualizar llamadas que usaban endpoints eliminados

---

## Fase 8: Actualizar data.sql y Postman

- Agregar columnas `slug`, `image_url`, `status`, `paypal_order_id`, `created_at`, `updated_at` a tablas correspondientes
- Generar slugs para los 15 libros existentes
- Eliminar stored procedures del data.sql
- Postman: agregar nuevos endpoints, eliminar los removidos, actualizar POST/PUT books a multipart

---

## Fase 9: Seguridad y buenas prácticas

### 9.1 Validación de password
- Regex: `^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[@#$%^&+=!]).{8,}$`
- `@Pattern` en: `RegisterRequest.password`, `ChangePasswordRequest.newPassword`, `CreateUserRequest.password`
- Mensaje: "Mínimo 8 caracteres, una mayúscula, una minúscula, un número y un carácter especial"
- Frontend: validación reactiva con el mismo patrón + mensaje visual en register y profile

### 9.2 Mensajes genéricos (no exponer datos sensibles)
- Login fallido: solo "Credenciales inválidas"
- Errores: "Recurso no encontrado" (sin IDs ni emails)
- Revisar `GlobalExceptionHandler` y todos los `throw new`

### 9.3 CORS restrictivo
- `SecurityConfig` → origins desde `${CORS_ALLOWED_ORIGINS:http://localhost:4200}`

### 9.4 Validación @Size en todos los DTOs
- Texto largo (description, comment): `@Size(max = 2000)`
- Campos cortos (nombre, email): `@Size(max = 150)`

### 9.5 Auditoría createdAt/updatedAt
- Crear `model/BaseEntity.java` — `@MappedSuperclass`, `@CreatedDate`, `@LastModifiedDate`
- `@EnableJpaAuditing` en BookstoreAiApplication
- Entidades que heredan: User, Sale, Review, Store

### 9.6 Variables de entorno + Docker
- `application.yaml` → todo parametrizado con `${VAR:default}`:
  - `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`, `DB_PASSWORD`
  - `JWT_SECRET`, `JWT_EXPIRATION`
  - `OPENAI_API_KEY`, `GOOGLE_CLIENT_ID`
  - `CORS_ALLOWED_ORIGINS`, `SERVER_PORT`
  - `PAYPAL_CLIENT_ID`, `PAYPAL_CLIENT_SECRET`, `PAYPAL_BASE_URL`
  - `MAIL_HOST`, `MAIL_PORT`, `MAIL_USERNAME`, `MAIL_PASSWORD`
- `Dockerfile` — multi-stage (Maven build + JRE slim run)
- `docker-compose.yml` — servicios: api, postgres (pgvector), frontend (nginx)
- `.env.example` — todas las variables sin valores reales

---

## Fase 10: Pago con PayPal (API REST directa)

**Backend — archivos nuevos:**
- `config/PayPalConfig.java` — RestTemplate bean con base URL PayPal
- `service/PayPalService.java`:
  - `getAccessToken()` — OAuth2 client_credentials
  - `createOrder(amount, currency, returnUrl, cancelUrl)` → orderId + approvalUrl
  - `captureOrder(orderId)` → confirma pago
- `dto/response/PayPalOrderResponse.java` — orderId, approvalUrl, status

**Backend — modificar:**
- `Sale.java` — agregar `status` (enum: PENDING, COMPLETED, CANCELLED) y `paypalOrderId` (String)
- `SaleController`:
  - `POST /api/sales` → crea Sale PENDING + createOrder PayPal → retorna approvalUrl
  - `POST /api/sales/confirm?token=X` → captureOrder → Sale COMPLETED
  - `POST /api/sales/cancel?token=X` → Sale CANCELLED
- `SaleResponse` — agregar `status`

**Variables de entorno:** `PAYPAL_CLIENT_ID`, `PAYPAL_CLIENT_SECRET`, `PAYPAL_BASE_URL` (default: sandbox)

**Frontend — archivos nuevos:**
- `features/payment/payment-confirm.component.ts|html` — ruta `/payment/confirm`, confirma pago
- `features/payment/payment-cancel.component.ts|html` — ruta `/payment/cancel`, mensaje + enlace carrito

**Frontend — modificar:**
- `cart.component.ts` — checkout llama POST /api/sales, recibe approvalUrl, `window.location.href = approvalUrl`
- `sale.model.ts` — agregar `status: string` a SaleResponse
- `app.routes.ts` — agregar rutas `/payment/confirm` y `/payment/cancel`

---

## Fase 11: Email al cambiar contraseña (Gmail)

**Backend — agregar dependencia:**
- `spring-boot-starter-mail` en pom.xml

**Backend — archivos nuevos:**
- `service/EmailService.java`:
  - `sendPasswordChangedNotification(toEmail, userName)`
  - Usa `JavaMailSender` + `SimpleMailMessage`
  - Asunto: "Tu contraseña ha sido actualizada"
  - Cuerpo: "Hola {nombre}, tu contraseña fue actualizada. Si no fuiste tú, contacta soporte."

**Backend — modificar:**
- `UserServiceImpl.changePassword()` — después de guardar, llamar `emailService.sendPasswordChangedNotification()`
- `application.yaml` — config mail:
  - `MAIL_HOST` (default: `smtp.gmail.com`)
  - `MAIL_PORT` (default: `587`)
  - `MAIL_USERNAME`, `MAIL_PASSWORD` (Gmail App Password)

**Frontend:**
- `profile.component.ts` — al cambiar password exitosamente, mostrar snackbar "Contraseña actualizada. Revisa tu email."

---

## Verificación
1. `mvn compile` exitoso
2. `ng build` exitoso
3. Catálogo: `GET /api/catalog/books?title=casa&genre=Novela` retorna filtrado combinado
4. Slug: `GET /api/books/slug/cien-anos-de-soledad` retorna el libro
5. Imagen: POST multipart crea libro con imagen comprimida en `/uploads/books/`
6. Detalle: click en card → `/catalog/cien-anos-de-soledad` muestra detalle + reviews
7. Perfil: GET/PUT `/api/users/profile` funciona con JWT
8. Admin: `POST /api/admin/users` crea store owner
9. Postman collection actualizado con ~30 endpoints (antes ~50)
10. Password débil rechazado con mensaje claro
11. Login fallido: solo "Credenciales inválidas"
12. `docker-compose up` levanta API + PostgreSQL + Frontend
13. `.env.example` documenta todas las variables
14. PayPal sandbox: checkout redirige a PayPal → confirma → Sale pasa a COMPLETED
15. Cambio de password envía email real vía Gmail
