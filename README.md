# Visor de Aplicaciones Web desde Google Drive

Este proyecto te permite ejecutar aplicaciones web sencillas (compuestas por HTML, CSS y JavaScript) directamente desde una carpeta compartida en Google Drive.

El objetivo es crear un sistema donde puedas simplemente subir tus archivos a una carpeta, compartirla y acceder a tu aplicación funcional a través de una URL especial, sin necesidad de un hosting web tradicional.

---

## Cómo Funciona (El Concepto)

La arquitectura se basa en tres componentes que trabajan en conjunto:

1.  **Google Drive (El Almacén):** Simplemente actúa como el lugar donde guardas los archivos de código fuente (`index.html`, `styles.css`, `app.js`).
2.  **API de Google Apps Script (El Intermediario Seguro):** Un pequeño script alojado en tu propia cuenta de Google que actúa como una API personal. Su única función es leer los archivos de una carpeta de Drive específica (cuando se le pide) y devolver su contenido. Es el puente seguro entre el mundo exterior y tus archivos.
3.  **Visor Universal (El Intérprete):** Una única página web estática, alojada en un servicio gratuito como GitHub Pages. Esta página es la "aplicación principal". Cuando la abres con un ID de carpeta en la URL, llama a tu API, recibe el código y lo "renderiza" para que puedas verlo y usarlo.

---

## Paso 1: Crear tu API en Google Apps Script

Esta API es tu "servidor" personal y seguro. Necesitas crear tu propia copia.

1.  **Ve a Google Apps Script:** Abre [script.google.com](https://script.google.com) y haz clic en **`Nuevo proyecto`**.
2.  **Pega el Código:** Borra el contenido por defecto y pega el siguiente código:

    ```javascript
    /**
     * Función principal que se ejecuta cuando se llama a la URL de la API (GET).
     * @param {Object} e - Objeto de evento con los parámetros de la URL.
     * @returns {ContentService.TextOutput} - Respuesta en formato JSON.
     */
    function doGet(e) {
      const folderId = e.parameter.folderId;
    
      if (!folderId) {
        return createJsonResponse({ error: "Parámetro 'folderId' no encontrado." });
      }
    
      try {
        const folder = DriveApp.getFolderById(folderId);
        const files = folder.getFiles();
        const fileContents = {};
    
        while (files.hasNext()) {
          const file = files.next();
          const fileName = file.getName().toLowerCase();
          const content = file.getBlob().getDataAsString('UTF-8');
          fileContents[fileName] = content;
        }
    
        return createJsonResponse(fileContents);
    
      } catch (error) {
        return createJsonResponse({ 
          error: "No se pudo acceder a la carpeta.",
          details: error.message
        });
      }
    }
    
    /**
     * Función auxiliar para crear una respuesta JSON estandarizada.
     */
    function createJsonResponse(data) {
      return ContentService
        .createTextOutput(JSON.stringify(data))
        .setMimeType(ContentService.MimeType.JSON);
    }
    ```

3.  **Despliega la API:**
    * Haz clic en el botón azul **`Desplegar`** > **`Nuevo despliegue`**.
    * Haz clic en el ícono del engranaje (⚙️) y selecciona **`Aplicación web`**.
    * En **"Quién tiene acceso"**, selecciona **`Cualquier persona`**.
    * Haz clic en `Desplegar`.
    * **Autoriza los permisos** que te solicite Google. Es seguro, ya que solo le das permiso a tu propio script.
    * Al final, **copia y guarda la `URL de la aplicación web`**. ¡Esta es la URL de tu API!

---

## Paso 2: Crear y Alojar el Visor Universal

Esta es la página pública que interpretará tus aplicaciones. La alojaremos en GitHub Pages por ser gratuito y sencillo.

1.  **Crea un Repositorio en GitHub:** Ve a [GitHub](https://github.com), crea una cuenta si no la tienes y crea un nuevo repositorio público (ej. `visor-drive`).
2.  **Crea el Archivo `index.html`:** Dentro del repositorio, crea un archivo llamado `index.html` y pega el siguiente código.

    ```html
    <!DOCTYPE html>
    <html lang="es">
    <head>
        <meta charset="UTF-8">
        <title>Visor de Aplicaciones de Google Drive</title>
        <style>
            body, html { margin: 0; padding: 0; width: 100%; height: 100%; font-family: sans-serif; }
            .container { display: flex; align-items: center; justify-content: center; width: 100%; height: 100%; background-color: #f0f2f5; }
            .message { text-align: center; padding: 40px; background-color: white; border-radius: 8px; box-shadow: 0 4px 12px rgba(0,0,0,0.1); }
            iframe { width: 100%; height: 100%; border: none; display: none; }
        </style>
    </head>
    <body>
        <iframe id="viewerFrame"></iframe>
        <div id="messageContainer" class="container">
            <div class="message">
                <h1 id="messageTitle">Cargando...</h1>
                <p id="messageText">Por favor, espera.</p>
            </div>
        </div>
        <script>
            // --- CONFIGURACIÓN ---
            const APPS_SCRIPT_URL = 'URL_DE_TU_API_AQUI'; // <-- ¡IMPORTANTE: Pega tu URL aquí!
    
            window.onload = function() {
                const folderId = window.location.hash.substring(1);
                const msgContainer = document.getElementById('messageContainer');
                const msgTitle = document.getElementById('messageTitle');
                const msgText = document.getElementById('messageText');
                const viewerFrame = document.getElementById('viewerFrame');
    
                if (!folderId) {
                    msgTitle.textContent = 'Bienvenido al Visor';
                    msgText.innerHTML = 'Para usar, añade el ID de una carpeta de Drive a la URL después de un #.';
                    return;
                }
    
                fetch(`${APPS_SCRIPT_URL}?folderId=${folderId}`)
                    .then(response => response.json())
                    .then(data => {
                        if (data.error) throw new Error(data.details || data.error);
    
                        const html = data['index.html'] || '';
                        const css = data['styles.css'] || data['style.css'] || '';
                        const js = data['app.js'] || data['script.js'] || '';
    
                        viewerFrame.srcdoc = `<html><head><style>${css}</style></head><body>${html}<script>${js}<\/script></body></html>`;
                        viewerFrame.style.display = 'block';
                        msgContainer.style.display = 'none';
                    })
                    .catch(error => {
                        msgTitle.textContent = 'Error al Cargar';
                        msgText.textContent = `Razón: ${error.message}`;
                    });
            };
        </script>
    </body>
    </html>
    ```

3.  **Configura tu API:** **¡Paso crucial!** En el código anterior, reemplaza el texto `'URL_DE_TU_API_AQUI'` con la URL real de tu API que obtuviste en el Paso 1.
4.  **Habilita GitHub Pages:**
    * En tu repositorio de GitHub, ve a **`Settings`** > **`Pages`**.
    * En la sección `Branch`, selecciona `main` y haz clic en `Save`.
    * Espera unos minutos y GitHub te dará la URL pública de tu visor (ej. `https://tu-usuario.github.io/visor-drive/`).

---

## Paso 3: Usar tu Visor

¡Ya tienes todo listo! Ahora, para ver cualquier aplicación:

1.  **Crea tu App:** Crea una nueva carpeta en Google Drive y sube tus archivos `index.html`, `styles.css` y `app.js`.
2.  **Comparte la Carpeta:** Haz clic derecho en la carpeta > `Compartir` > `Compartir` y asegúrate de que el acceso general sea **`Cualquier persona con el enlace`**.
3.  **Obtén el ID:** Mira la URL de tu carpeta en el navegador. El ID es la larga cadena de caracteres después de `/folders/`.
    `https://drive.google.com/drive/folders/`**`1Abc-lKzYv5X0T8s19GABC-DEF123456`**
4.  **Crea el Enlace Final:** Une la URL de tu visor con el ID de la carpeta usando un `#`.

    **Ejemplo:** `https://tu-usuario.github.io/visor-drive/#1Abc-lKzYv5X0T8s19GABC-DEF123456`

Este es el enlace que puedes usar y compartir. Cualquiera que lo abra verá la aplicación contenida en esa carpeta de Drive.

---

Este proyecti se distribuye bajo la licencia [Creative Commons Atribución-CompartirIgual 4.0 Internacional (CC BY-SA 4.0)](https://creativecommons.org/licenses/by-sa/4.0/).

**Atribución:** Este script fue generado en un entorno colaborativo con la asistencia de Gemini, un LLM de Google.
