# Áreas Asignadas

**Versión:** 2.5.0 · **Requiere:** WordPress 5.8+ · PHP 7.4+ · **Text domain:** `areas-asignadas`

> Panel para admins de empresa: gestiona áreas y usuarios, sigue el avance en tests y cursos, envía recordatorios y genera informes PDF y Excel.

## ¿Qué es?

**Áreas Asignadas** es un plugin de WordPress que añade un panel de autogestión para las empresas cliente. Se inserta con el shortcode **`[areas_asignadas]`** y solo lo ven los usuarios con el rol **`empresa_admin`**. A cualquier otro usuario se le muestra una pantalla de acceso denegado.

Desde ese panel, el administrador de una empresa ve las áreas contratadas, consulta los usuarios de cada una y su avance (test y cursos de LearnDash), da de alta usuarios nuevos, les envía recordatorios y descarga informes en PDF y Excel. Todo queda limitado a los datos de su propia empresa.

---

## Funcionalidades

### 1. Listado de áreas de la empresa
- Tabla con todas las áreas cuyo `grupo_id` coincide con el ID del `empresa_admin` conectado.
- Por cada área muestra:
  - **Nombre** y **fecha de expiración** (o «Sin fecha de expiración»).
  - **Cupos**: usados / totales, con una barra de ocupación que cambia de color según el porcentaje.
  - **Estado** (badge, por ejemplo ACTIVO).
  - Botón **«Ver usuarios»**.
  - Botón **«Informe general»** del área. Se desactiva si el área no tiene usuarios o si ninguno ha completado el test.
- Buscador y paginación del listado.
- Menú **«Acciones»** con:
  - Informe general (de toda la empresa)
  - Descargar estado de usuario (Excel)
  - Recordar registro

### 2. Modal «Ver usuarios» de un área
- La búsqueda, el filtro por estado del test y la paginación se hacen **en el servidor**, no en el navegador.
- Por cada usuario muestra: estado del test (Completado / Incompleto), estado del perfil (Completo / Incompleto) y **avance real en LearnDash**. El avance cuenta los cursos con inscripción directa y también los asignados por **grupos de LearnDash**.
- Botón **«Ver resultados»** que genera el **PDF individual** del usuario.
- Botón de **campana** para enviar un recordatorio de test pendiente a un usuario con el test INCOMPLETO.
- Diseño adaptable: cabecera fija, paginación anclada abajo y, en pantallas de 640 px o menos, la tabla se convierte en tarjetas.

### 3. Alta de usuarios desde un área («Agregar usuario»)
1. Solo pide el **correo electrónico**. Si ya existe un usuario con ese correo, se detiene sin crear nada.
2. Crea el usuario en WordPress con rol `subscriber`, contraseña aleatoria y `first_name`/`nickname` tomados de la parte del correo antes de la @. Además genera un `user_login` único y un `user_nicename` con un sufijo aleatorio.
3. Lo inscribe en `{prefix}user_members` en esa área, con `status = ACTIVO`, `days = 0` y como fecha final de membresía la fecha de expiración del área.
4. **Replica el alta** en `wmslatam.com/wp-json/wms/v1/create-user` (con `tipo_programa = inspira`).
5. Si WMS confirma el alta, dispara el webhook `creacion-inspira-wms`.
6. Si la respuesta de WMS trae un `area_octagono_id`, también da de alta al usuario en **Octágono** (`octagono.wmslatam.com/.../create-user-area`) y después avisa al webhook `creacion-inspira-octagono`.
7. Si falla cualquier paso de la cadena, se detienen los siguientes, pero el usuario creado en el sitio local se conserva. En ese caso se muestra el aviso de «sincronización pendiente».
- Los cupos se actualizan en pantalla sin recargar la página.
- Cuando el área está llena, el botón aparece deshabilitado. El servidor también comprueba el cupo, así que no basta con saltarse la interfaz.

### 4. Informes en PDF
Los resultados del test se obtienen en el servidor y se envían a un webhook de n8n (`webhook/pdf_aprobandome`, o el que devuelva `ipw_get_webhook('area')` si está activo el plugin «Webhooks de generación de PDF (Inspira)»). El webhook responde con `{ pdfUrl }` y el navegador descarga el archivo.

| Informe | Endpoint AJAX | Fuente de datos |
|---|---|---|
| Por usuario | `ap_get_resultados_pdf` | `get_results_user()` |
| Por área | `ap_get_resultados_area_pdf` | `get_results_area()` |
| General de empresa | `ap_get_resultados_grupo_pdf` | `get_results_area()` de cada área |

### 5. Exportación a Excel («Descargar estado de usuario»)
- Un modal permite elegir qué áreas incluir.
- Genera un archivo **.xlsx** con un escritor propio (`AP_Xlsx_Writer`), sin librerías externas. El nombre del archivo es `usuarios-AAAA-MM-DD.xlsx`.
- Columnas: **Nombre, Correo, Área, Estado de perfil, Test, Cursos asignados, Avance general**.
- Los estados se colorean: «Completado» y «Realizado» en verde, el resto en gris.

### 6. «Recordar registro» (usuarios con perfil incompleto)
- Está disponible en el menú «Acciones» del listado de áreas y en el modal de usuarios.
- Funciona en dos pasos, **selección** y **confirmación**:
  - Lista los usuarios que no han completado su perfil.
  - Incluye un buscador por correo y un filtro por área.
  - Permite filtrar por último recordatorio: todos, sin recordatorios o un rango de fechas.
  - La selección se conserva al cambiar de filtro.
- El perfil se considera completo según los campos obligatorios del plugin **Editar Perfil de Suscriptor** (`ESP_User_Fields`). Si ese plugin no está activo, basta con que el usuario tenga nombre y apellidos.
- El envío sale **del servidor**: hace un único POST JSON con el array de correos, sin duplicados.
- Justo antes de enviar, vuelve a comprobar que cada usuario pertenece a la empresa y que su perfil sigue incompleto.
- La fecha del último recordatorio se guarda en el usermeta `ap_ultimo_recordatorio_registro` solo si el webhook acepta la petición. Se muestra en hora de Colombia, con el formato `dd/mm/aaaa hh:mm:ss AM/PM`.

### 7. Pantalla de administración (wp-admin)
- Menú **Áreas asignadas → Recordatorios de registro**, visible solo con `manage_options`.
- Sirve para configurar las URL de dos webhooks:
  - Recordar registro (`ap_webhook_recordar_registro`)
  - Recordatorio de test (`ap_webhook_recordatorio_test`)
- Si no se configura nada, usa las URL por defecto de `api.efrenmartinezortiz-ia.com`.
- Incluye un **probador de webhook**: se envía a un correo concreto y muestra el código HTTP y la respuesta, sin modificar la fecha de ningún usuario.
- Muestra un aviso en el panel si el rol `empresa_admin` no existe en el sitio.

---

## Seguridad
- Todas las peticiones AJAX comprueban un **nonce** (`ap_nonce`) y el rol `empresa_admin`.
- **Protección contra IDOR:** antes de devolver datos o actuar, se comprueba que cada área o usuario pertenece a la empresa del admin conectado.
- Las URL de los webhooks de recordatorio ya no aparecen en el JS público: se resuelven en el servidor.
- El SQL usa siempre `$wpdb->prepare()` y está concentrado en `AP_Database`.

## Arquitectura

```
areas-asignadas/
├── areas-asignadas.php          # Bootstrap (plugins_loaded)
├── includes/
│   ├── class-loader.php         # Conecta los módulos
│   ├── class-security.php       # Rol + nonces
│   ├── class-database.php       # Toda la capa SQL / LearnDash
│   ├── class-frontend.php       # Shortcode, plantillas, assets
│   ├── class-ajax.php           # Endpoints wp_ajax_*
│   ├── class-reminders.php      # Envío y fechas de recordatorios
│   ├── class-reminders-admin.php# Pantalla de ajustes en wp-admin
│   ├── class-xlsx-writer.php    # Generador .xlsx nativo
│   ├── class-admin.php          # Avisos de entorno
│   └── class-helpers.php        # Formateo
├── templates/                   # HTML (listado + modales)
└── assets/                      # CSS y JS del panel
```

- El CSS y el JS se cargan **desde el propio shortcode**, así que funcionan igual si el shortcode está dentro de Elementor, de un widget o de una plantilla. Si el CSS no se imprimió en el head, se imprime en el footer.
- Todos los textos se pueden traducir y se pasan al JS con `wp_localize_script`.

## Dependencias externas
- Tablas `{prefix}areas` y `{prefix}user_members`, que crea y mantiene otro sistema.
- Funciones `get_results_user()` y `get_results_area()`, del sistema de evaluaciones.
- **LearnDash**, para el avance en los cursos.
- **SweetAlert2** para los diálogos. Si no está cargado, se usa `alert()`.
- Opcionales: **Editar Perfil de Suscriptor** (`ESP_User_Fields`) e `ipw_get_webhook()`.
- Servicios remotos: la API de WMS, la API de Octágono y los webhooks de n8n en `api.efrenmartinezortiz-ia.com`.
