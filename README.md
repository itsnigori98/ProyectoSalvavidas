# Taller 3 – Módulo de usuarios, roles y permisos

**Computación en Internet II – Universidad Icesi**

Aplicación web con el patrón **MVC** (Spring Boot, Thymeleaf y Spring Security) que implementa un módulo de gestión de usuarios, roles y permisos. El módulo está protegido: solo se entra después de iniciar sesión (autenticación) y lo que cada usuario puede ver y hacer depende de los roles y permisos que tenga asignados (autorización).

El código del proyecto está en la carpeta `ProyectoSalvavidas/`.

## Tecnologías

- Java 17 y Spring Boot 3.5
- Spring MVC + Thymeleaf (vistas del lado del servidor)
- Spring Security (formulario de inicio de sesión, CSRF, `@PreAuthorize`, BCrypt)
- Spring Data JPA con base de datos H2 en memoria
- JUnit 5 y Mockito (pruebas)
- Maven (se usa el *wrapper* incluido, no hace falta instalarlo)

## Cómo ejecutar la aplicación

**Requisitos:** JDK 17 o superior (`java -version` debe mostrar 17+) y conexión a internet la primera vez, para que Maven descargue las dependencias.

Desde la raíz del repositorio:

```bash
cd ProyectoSalvavidas

# Linux / macOS
./mvnw spring-boot:run

# Windows
mvnw.cmd spring-boot:run
```

También se puede ejecutar desde IntelliJ: abrir la carpeta `ProyectoSalvavidas` como proyecto Maven y correr la clase `IntroSpringBootApplication`.

La aplicación queda disponible en:

| Qué | URL |
|---|---|
| Inicio de sesión | http://localhost:8080/WebApp6/usuario/login |
| Registro | http://localhost:8080/WebApp6/usuario/registro |
| Consola de la base H2 | http://localhost:8080/WebApp6/h2 |

Para la consola H2 se usa: JDBC URL `jdbc:h2:mem:testdb`, usuario `sa`, contraseña vacía.

> La base de datos es **en memoria**: se crea y se llena con `src/main/resources/data.sql` cada vez que arranca la aplicación. Los datos creados durante una ejecución se pierden al detenerla.

### Usuarios de prueba

| Correo | Contraseña | Roles |
|---|---|---|
| `carlos.ramirez@correo.com` | `admin123` | `ROLE_ADMIN` |
| `andres.gomez@correo.com` | `andres123` | `ROLE_USUARIO` + `ROLE_MODERADOR` |
| `maria.lopez@correo.com` | `maria123` | `ROLE_USUARIO` |
| `sofia.torres@correo.com` | `sofia123` | `ROLE_USUARIO` |
| `julian.vargas@correo.com` | `julian123` | `ROLE_USUARIO` |
| `diana.ospina@correo.com` | `diana123` | `ROLE_USUARIO` (estado INACTIVO: no puede iniciar sesión) |

Las contraseñas se guardan hasheadas con BCrypt; nunca en texto plano.

## Cómo probar la aplicación

Cada punto del taller se prueba así. Los pasos 4 a 9 se hacen como administrador (`carlos.ramirez@correo.com` / `admin123`).

1. **Registrar usuario:** en `/usuario/registro` llenar el formulario. El usuario queda con el rol `ROLE_USUARIO`. Si el correo ya existe se muestra un mensaje de error.
2. **Iniciar sesión:** en `/usuario/login` entrar con un usuario de prueba. Si las credenciales son incorrectas, o el usuario está INACTIVO, se muestra un error. Al entrar se llega a `/profile/me`, que muestra los roles y permisos del usuario.
3. **Control de acceso:** iniciar sesión con `maria.lopez@correo.com` e intentar abrir `/usuario`, `/rol` o `/permiso`. El servidor responde **403 (acceso denegado)**. Sin sesión, cualquier página redirige al login.
4. **Listar usuarios:** desde el perfil del administrador, botón **Usuarios** (`/usuario`).
5. **Crear roles:** `/rol` → **Crear rol**. El nombre se guarda en mayúsculas y con el prefijo `ROLE_` (por ejemplo, `vendedor` queda `ROLE_VENDEDOR`). Un nombre repetido o vacío muestra un error.
6. **Asociar permisos a un rol:** `/rol` → clic en el nombre del rol → marcar los permisos → **Guardar permisos**. Desmarcar un permiso y guardar lo quita.
7. **Asociar roles a un usuario:** `/usuario` → **Gestionar roles** → marcar los roles en «Asociar roles» → **Guardar roles**.
8. **Quitar roles a un usuario:** en esa misma pantalla, en «Roles actuales», botón **Quitar** junto al rol.
9. **Eliminar roles:** `/rol` → **Eliminar** (también se borran sus asociaciones con usuarios y permisos).
10. **Permisos:** `/permiso` lista los permisos y **Crear permiso** agrega uno nuevo.
11. **Cerrar sesión:** botón **Cerrar sesión** en el perfil.

Para comprobar que los permisos cambian lo que ve cada usuario, asociar un rol nuevo (o quitarle uno) a `maria.lopez@correo.com`, cerrar sesión, volver a entrar con ella y revisar la lista de roles y permisos en su perfil.

## Reglas de seguridad

Configuradas en `security/WebSecurityConfig.java` (`SecurityFilterChain`):

| Ruta | Acceso |
|---|---|
| `/usuario/login`, `/usuario/registro`, `/css/**` | Público (`permitAll`) |
| `/usuario/**`, `/rol/**`, `/permiso/**` | Solo `ROLE_ADMIN` (`hasRole`) |
| `/categoria/**` | Permiso `GESTIONAR_CATEGORIAS` (`hasAuthority`) |
| Cualquier otra ruta | Usuario autenticado |
| `/api/**` | API REST con JWT (cadena de seguridad aparte) |

Además, los servicios usan `@PreAuthorize` (por ejemplo, `hasRole('ADMIN')` en roles y permisos, y `hasAuthority('CREATE_USER')` al crear usuarios).

Todos los formularios usan `th:action`, por lo que Thymeleaf incluye automáticamente el token CSRF.

## Estructura del proyecto

```
ProyectoSalvavidas/
├── pom.xml
└── src/
    ├── main/
    │   ├── java/edu/co/icesi/introspringboot/
    │   │   ├── controller/   AuthController, UsuarioController, RolController,
    │   │   │                 PermisoController, ProfileController, ...
    │   │   ├── service/      interfaces de los servicios
    │   │   │   └── impl/     implementaciones (lógica y reglas)
    │   │   ├── repo/         repositorios Spring Data JPA
    │   │   ├── entity/       entidades JPA
    │   │   ├── security/     WebSecurityConfig, CustomUserDetailService, JWT
    │   │   └── api/          API REST (/api/v1)
    │   └── resources/
    │       ├── templates/    usuario/, rol/, permiso/, ... (Thymeleaf)
    │       ├── static/css/   estilos de las vistas
    │       ├── application.properties
    │       └── data.sql      datos iniciales (roles, permisos y usuarios)
    └── test/                 pruebas unitarias y de integración
```

Cada controlador delega la lógica en los servicios y envía los datos a la vista con el `Model`. Los métodos `GET` devuelven la plantilla y cada `POST` termina con un `redirect:`.

## Pruebas

```bash
cd ProyectoSalvavidas
./mvnw test          # Windows: mvnw.cmd test
```

- `src/test/java/.../unit/services/`: pruebas unitarias (JUnit 5 + Mockito) de los servicios, entre ellos `UsuarioService`, `RolService`, `PermisoService`, `UsuarioRolService` y `CustomUserDetailService` (el servicio de autenticación).
- `src/test/java/.../integration/services/`: prueba de integración del servicio de publicaciones contra H2.

Las pruebas usan su propio `src/test/resources/application.properties` (H2 con `create-drop` y sin `data.sql`).

## Despliegue en Apache Tomcat

El proyecto se empaqueta como WAR con el nombre `WebApp6`, por lo que Tomcat lo publica en el contexto `/WebApp6`.

```bash
cd ProyectoSalvavidas
./mvnw clean package        # genera target/WebApp6.war
```

1. Copiar `target/WebApp6.war` a la carpeta `webapps/` del Tomcat (o subirlo desde el *Manager* de Tomcat).
2. Entrar a `http://<servidor>:<puerto>/WebApp6/usuario/login`.

Requisitos del servidor: Tomcat 10.1 o superior (Jakarta EE) y Java 17 o superior.

> En Tomcat, el contexto lo define el nombre del WAR; la propiedad `server.servlet.context-path` solo aplica al ejecutar con `spring-boot:run`.

### Despliegue en IAsLab

La aplicación está desplegada en el **Apache Tomcat 11** de IAsLab.

| Dato | Valor |
|---|---|
| Servidor | `192.168.131.112` (computador `104m12`) |
| Usuario del sistema | `computacion` |
| Servidor de aplicaciones | Apache Tomcat 11.0.27 en `~/Desktop/apache-tomcat-11.0.27` |
| Java | OpenJDK 17 |
| Puerto | 8080 |
| Archivo desplegado | `WebApp6.war` |
| Contexto | `/WebApp6` |

> Se requiere estar en la red de IAsLab (o conectado por VPN/ZeroTier) y usar `http://`, no `https://`.

| Qué | URL |
|---|---|
| Inicio de sesión | http://192.168.131.112:8080/WebApp6/usuario/login |
| Registro | http://192.168.131.112:8080/WebApp6/usuario/registro |

Los usuarios de prueba son los mismos de la tabla [Usuarios de prueba](#usuarios-de-prueba), y la guía [Cómo probar la aplicación](#cómo-probar-la-aplicación) aplica igual cambiando `localhost` por `192.168.131.112`.

**Pasos que se siguieron para desplegar:**

1. Generar el WAR:

   ```bash
   cd ProyectoSalvavidas
   ./mvnw clean package        # genera target/WebApp6.war
   ```

2. Copiar el WAR al servidor:

   ```bash
   scp target/WebApp6.war computacion@192.168.131.112:Desktop/apache-tomcat-11.0.27/webapps/
   ```

   Con Tomcat encendido, el WAR se despliega solo en unos segundos al aparecer en `webapps/`.

3. Si Tomcat está apagado (por ejemplo, después de reiniciar el computador), entrar al servidor e iniciarlo:

   ```bash
   ssh computacion@192.168.131.112
   cd ~/Desktop/apache-tomcat-11.0.27
   ./bin/startup.sh
   ```

4. Verificar el despliegue en el log de Tomcat:

   ```bash
   tail -f ~/Desktop/apache-tomcat-11.0.27/logs/catalina.out
   ```

   Debe aparecer `Deployment of web application archive [.../webapps/WebApp6.war] has finished`. Para salir del log: `Ctrl + C` (Tomcat sigue corriendo).

Para actualizar la aplicación basta con reemplazar `WebApp6.war` en `webapps/`. Para retirarla, borrar `WebApp6.war` de `webapps/`; Tomcat la desmonta sola. En el mismo Tomcat hay otras aplicaciones desplegadas, así que **no** se debe usar `./bin/shutdown.sh` solo para actualizar esta.

> Como la base de datos es H2 en memoria, cada vez que se vuelve a desplegar el WAR o se reinicia Tomcat, los datos vuelven al estado de `data.sql`.
