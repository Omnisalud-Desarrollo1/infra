# MANUAL DE USUARIO — Plataforma Omnisalud Digital

## Tabla de Contenido

1. [Acceso al Sistema](#1-acceso-al-sistema)
2. [Portal Principal](#2-portal-principal)
3. [Módulo Asistencial (app1)](#3-módulo-asistencial-app1)
4. [Módulo Comercial (app2)](#4-módulo-comercial-app2)
5. [Gestión de Usuarios (admin)](#5-gestión-de-usuarios-admin)
6. [Preguntas Frecuentes](#6-preguntas-frecuentes)

---

## 1. Acceso al Sistema

### URL de Acceso

| Entorno | URL |
|---------|-----|
| **Producción** | `https://portalcorporativo.omnisalud.co` |
| **Desarrollo (ngrok)** | URL proporcionada por el equipo técnico |

### Inicio de Sesión

1. Abra la URL del portal en su navegador (Chrome, Firefox o Edge recomendados)
2. Ingrese su **usuario** y **contraseña** corporativa
3. Haga clic en **Iniciar Sesión**

![Pantalla de login con formulario de usuario y contraseña]

> **Nota:** Si olvidó su contraseña, contacte al administrador del sistema.

### Cierre de Sesión

Haga clic en **Cerrar Sesión** en la barra de navegación superior. Esto cerrará su sesión en todas las aplicaciones del ecosistema.

---

## 2. Portal Principal

Después de iniciar sesión, verá el **Dashboard** con tarjetas de acceso a los diferentes módulos:

### Módulos Disponibles

| Tarjeta | Descripción | Acceso |
|---------|-------------|--------|
| **Cotizador** | Genera cotizaciones para clientes | Perfiles: COMERCIAL |
| **Registro de Asistencia** | Control de asistencia del personal | Perfiles: COMERCIAL, Rol: COORDINADOR |
| **Cuadro de Turnos** | Gestión de turnos del personal | Perfiles: COMERCIAL |
| **Módulo Asistencial SUS** | Horarios, agenda y profesionales asistenciales | Todos los perfiles |
| **Módulo Comercial** | Cotizaciones, turnos y asistencia comercial | Perfiles: COMERCIAL |

Las tarjetas se muestran según su **perfil y rol**. Si no ve una tarjeta que esperaba, contacte al administrador.

### Navegación

- Haga clic en cualquier tarjeta para acceder al módulo
- Use la barra superior para ver su nombre de usuario y cerrar sesión
- El logo Omnisalud en la esquina izquierda lo lleva de vuelta al Dashboard

---

## 3. Módulo Asistencial (app1)

### Menú Principal

Al ingresar al Módulo Asistencial, verá las siguientes opciones:

#### Gestión de Horarios _(Perfiles: Asistencial, Administrativo2, Superadmin)_

**Vista Individual** (Perfil Asistencial):
- Visualiza sus horarios semanales asignados
- Muestra tipo de evento (consulta, brigada, telemedicina, etc.)
- Información de sede, hora de inicio/fin, almuerzo

**Vista de Equipo** (Perfiles: Administrativo2, Superadmin):
- Seleccione un profesional y una fecha para ver sus horarios
- Puede editar o eliminar horarios existentes
- Bloqueo de edición para fechas con más de 8 días de antigüedad

### Consulta de Horarios

1. En el menú principal, seleccione **Consulta de Horarios**
2. Filtre por regional y/o fecha
3. El sistema muestra una tabla con los horarios de todos los profesionales del filtro

---

## 4. Módulo Comercial (app2)

### Menú Principal

Al ingresar al Módulo Comercial, verá las siguientes opciones:

#### Cotizador

El Cotizador es una aplicación integrada para generar cotizaciones de servicios médicos:

1. **Seleccione una empresa** — Busque por nombre en el campo de autocompletado
2. **Agregue servicios** — Seleccione de las categorías disponibles (servicios sedes propias, red nacional, etc.)
3. **Configure tarifas** — El sistema calcula automáticamente según tarifas configuradas
4. **Genere PDF** — Exporte la cotización en formato PDF para el cliente
5. **Historial** — Consulte cotizaciones anteriores

> **Acceso rápido:** Use el botón **Cotizador** en el menú principal para abrir la aplicación.

#### Cuadro de Turnos

Gestione los turnos del personal comercial:

- **Vista Individual**: Consulte sus turnos asignados (día, sede, horario)
- **Vista de Equipo**: Administre turnos de todo el equipo (solo supervisores)

#### Registro de Asistencia

Registre y consulte la asistencia del personal:

- **Registrar entrada/salida**: Botones para marcar asistencia
- **Exportar a Excel**: Genere reportes de asistencia por período
- **Filtros**: Por fecha, usuario o sede

---

## 5. Gestión de Usuarios (admin)

> **Acceso restringido:** Solo disponible para usuarios con perfil **ADMIN**.

### Registrar Usuario

1. En el Dashboard del Portal, seleccione **Registrar Usuario**
2. Complete el formulario:
   - Usuario (nombre de acceso)
   - Contraseña
   - Nombre completo
   - Email
   - Documento
3. Asigne **Perfiles** y **Roles** al usuario
4. Haga clic en **Registrar**

### Lista de Usuarios

1. En el Dashboard, seleccione **Usuarios**
2. Verá una tabla con todos los usuarios registrados
3. Puede **buscar** por nombre, **editar** o **ver detalle** de cada usuario

### Editar Usuario

1. En la lista de usuarios, haga clic en **Editar**
2. Modifique los campos necesarios (nombre, email, perfiles, roles, estado)
3. Haga clic en **Guardar Cambios**

---

## 6. Preguntas Frecuentes

### ¿Olvidé mi contraseña?

Contacte al administrador del sistema. El portal usa autenticación centralizada y el restablecimiento de contraseña debe hacerse a través de un administrador.

### ¿Por qué no veo todas las tarjetas en el Dashboard?

Las tarjetas del Dashboard se filtran según su perfil y rol. Si cree que debería tener acceso a un módulo que no aparece, solicite al administrador que revise sus perfiles asignados.

### ¿Cambié de módulo y me pidió iniciar sesión de nuevo?

Su sesión puede haber expirado. La sesión tiene una duración limitada por seguridad. Simplemente inicie sesión nuevamente — será redirigido al módulo donde estaba trabajando.

### ¿Puedo usar el sistema desde mi celular?

Sí, la plataforma es responsiva y funciona en navegadores móviles. Sin embargo, algunos módulos (como el Cotizador) están optimizados para pantallas de escritorio.

### ¿A quién reporto un error?

Contacte al equipo de soporte técnico a través de los canales establecidos por su organización, proporcionando:
- El módulo donde ocurrió el error
- La acción que estaba realizando
- Un mensaje de error (si aparece en pantalla)
- La URL de la página donde ocurrió

### ¿El sistema guarda mi trabajo automáticamente?

Depende del módulo:
- **Cotizador**: Debe guardar explícitamente cada cotización
- **Horarios**: Los cambios se guardan al hacer clic en "Guardar"
- **Asistencia**: El registro de entrada/salida es inmediato

Siempre verifique que sus cambios se hayan guardado antes de cerrar la página.

---

## Atajos de Navegación

| Acción | Ruta |
|--------|------|
| Dashboard Portal | `/app` |
| Módulo Asistencial | `/app1/menu` |
| Módulo Comercial | `/app2/` |
| Cotizador | `/app2/cotizador` |
| Cerrar Sesión | Barra superior → Cerrar Sesión |

---

## Roles y Permisos

| Rol | Permisos principales |
|-----|---------------------|
| **ADMIN** | Acceso total a todos los módulos y gestión de usuarios |
| **Superadmin** | Administración de usuarios, horarios y turnos |
| **Administrativo2** | Gestión de horarios, vista de equipo asistencial |
| **Asistencial** | Vista de horarios individuales |
| **Comercial** | Cotizador, registro de asistencia |
| **Líder / Director** | Cotizador (configuración avanzada) |
| **Coordinador** | Registro de asistencia del equipo |

> Para solicitar un cambio de rol, contacte al administrador del sistema.
