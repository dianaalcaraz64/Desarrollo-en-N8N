## **Generación de Casos de Prueba con IA en n8n**

Este informe detalla el proceso de desarrollo, los desafíos encontrados y las soluciones aplicadas para establecer un flujo automatizado en n8n que genera casos de prueba utilizando una Inteligencia Artificial (IA) y los guarda directamente en una hoja de cálculo de Google Sheets.

---

### **Objetivo del Proyecto**

El objetivo principal es automatizar la creación de casos de prueba a partir de una historia de usuario, utilizando Gemini 1.5 Flash para generar el contenido y n8n como orquestador para procesar esa información y almacenarla de forma estructurada en Google Sheets.

---

### **Componentes del Flujo en n8n**

El flujo en n8n se compuso de los siguientes nodos clave:

1) **Nodo Manual Trigger**  
   *Configuración esencia*l: Sin parámetros

            *Qué hace paso a paso:*   
1\. Actúa como disparador interno: cuando pulsas **Execute Workflow**, emite un ítem vacío y pone en marcha el flujo.  
2\. No espera peticiones externas.

2) **Nodo Edit Fields**  
   *Configuración esencial:* 2 campos:  
* id de historia a trabajar (string)  
* nombre de hoja de destino (string)  
  *Qué hace paso a paso*:   
* Encapsula los valores que escribes, se convierten en $json.  
* Sirven para filtrar la HU y para indicar la hoja destino.


      **3\)**  **Nodo Google Sheets – Get Rows**  
*Configuración esencial:* 

* Generar credencial y conectarla a la cuenta de google a utilizar.  
* *Sheet Within Document* → **Get Rows**  
* Documento: *By ID (*ID de  documento a leer)  
* Sheet name: hoja con las HU (Nombre de la hoja a leer)  
  *Qué hace paso a paso:*   
* Llama a la API de Sheets.  
* Devuelve una fila (la HU) con todas sus columnas: ID, Funcionalidad, Historia, Criterios, Validaciones, Reglas y **Contexto para razonamiento**.  
* Esa fila pasa a ser $input.first().json para el siguiente nodo


**4\)  Nodo Code (A) – Construir prompt**  
     *Configuración esencial:* JavaScript personalizado.  
     *Qué hace paso a paso*:

* Lee la fila (itemData).  
* Toma el texto de **Contexto para razonamiento**(fila existente en la hoja de HU la cual contiene el contexto para armar los casos); si falta, inserta un contexto por defecto.  
* Arma promptContent con:  
- Instrucciones QA.  
- Lista de claves obligatorias.  
- **Línea antidoble-comillas**: (Importante: …comillas simples o \\").  
* Escapa "→\\" y saltos de línea (\\n→\\\\n).  
* Construye bodyForAI con:  
- temperature: 0.0 (estabilidad).  
- maxOutputTokens: 8192\.  
* Devuelve un único ítem cuyo $json es exactamente bodyForAI


**5\)** **Nodo HTTP Request \- Enviar prompt**  
   *Configuración esencial*:

*  **POST** a https://generativelanguage.googleapis.com/v1beta/models/gemini-1.5-flash:generateContent?Api-Key con clave(la clave que obtenida al conectar con   
  la api de gemini).  
* Authentication: None.  
* SpecifyBody: Using JSON.  
* Body type: **JSON** → {{ $json }}

   *Qué hace paso a paso*:

* Envía el prompt a Gemini.  
* Devuelve en $json.candidates\[0\].content.parts\[0\].text un bloque json \[…\] .


**6\)** **Nodo Code (B) – Parsear respuesta**  
    Configuración esencial*:JavaScript de limpieza y parseo.*  
    *Qué hace paso a paso*: 

* Extrae el texto de la ruta anterior.  
* Elimina json … y \\n escapados.  
* Quita caracteres de control.  
* Recorta hasta \[ o {.  
* Auto-cierra \] o } si falta.  
* JSON.parse → array casos.  
* Emite un ítem por caso (return casos.map(...)).  
* Si falla el parseo, emite un ítem con error\_parsing.

**7\) Nodo Google Sheets – Append Row \- Insertar valores en hoja de destino**  
    *Configuración esencial*: 

- Hoja destino \= {{$node\["Edit Fields"\].json\["nombre de hoja"\]}}.  
- Columna ↔ Campo JSON (mapeo manual).

    *Qué hace paso a paso:* 

* Recibe los ítems del paso anterior.  
* Inserta cada objeto como una fila nueva.  
* Si el parser envía 0 items, las celdas mapeadas aparecen en rojo, señal de que no llegó data.

---

### **Secuencia de ejecución**

1. Pulsar **Execute Workflow** → Manual Trigger inicia.  
2. Edit Fields recibe el ID y nombre de hoja destino.  
3. Get Rows recupera la fila de la HU (incluye “Contexto para razonamiento”).  
4. Code (A) construye el prompt, agrega la instrucción de escape y genera bodyForAI.  
5. HTTP envía a Gemini.  
6. Code (B) limpia y parsea el array JSON de casos.  
7. Opcional Code (C) valida claves mínimas.  
8. Append Row escribe cada caso como nueva fila en la hoja destino.

---

### **Desafíos Encontrados y Soluciones Aplicadas**

El proceso no fue lineal y se encontró un desafío principal que requería una depuración y ajustes:

1. **Problema Inicial: Falla en el Parseo del JSON (Unexpected token ''\` )**  
* **Síntoma***:* La primera vez que se intentó procesar la respuesta de la IA con un nodo Code para extraer el JSON, n8n arrojaba un error "error\_details": "Unexpected token '', "json\\n\[\\n"... is not valid JSON"\`. Esto indicaba claramente que la función \`JSON.parse()\` estaba recibiendo una cadena que comenzaba con los caracteres de Markdown (\` json ), en lugar de un \[o{\` que es lo que se espera al inicio de un JSON.  
* **Causa Raíz:** Aunque la IA generaba el JSON correctamente, lo envolvía dentro de un bloque de código Markdown (\`\`\`json\\n...json\_data...\\n\`\`\`). El script inicial en el nodo Code no estaba eliminando estos delimitadores de forma efectiva, o la forma en que los extraía no siempre era perfecta.  
* **Primera Solución Intentada*:*** *Se propuso una expresión regular (aiResponseText.match(/\`\`\`json\\s*(\[\\s\\S\]\*?)\\s\*\`\`\`/)) para extraer el contenido entre los delimitadores de Markdown. Aunque esto funcionó en algunos casos, la variabilidad en las respuestas de la IA (por ejemplo, espacios extra, saltos de línea, o incluso la ausencia de un delimitador de cierre) hacía que el parseo siguiera fallando intermitentemente.  
* **Solución Definitiva (Iteración en el nodo Code):** Reconociendo que el problema persistía en la limpieza, se implementó una estrategia más robusta y "agresiva" en el nodo Code. Esta estrategia incluyó:  
- **Búsqueda Explícita de Delimitadores JSON:** En lugar de depender únicamente de los delimitadores Markdown, el código ahora busca el **primer corchete de apertura (\[) o llave de apertura ({)** y el **último corchete de cierre (\]) o llave de cierre (})** en la cadena de texto. Se extrae solo la porción de la cadena entre estos caracteres, asumiendo que esa es la estructura JSON deseada.  
- **Limpieza Agresiva de Markdown:** Se añadió un paso explícito para eliminar cualquier \`\`\`json o \`\`\` restante dentro de la cadena extraída, garantizando que solo el JSON puro se pasará a JSON.parse().  
- **Manejo de Caracteres de Control:** Se incluyó un filtro para eliminar caracteres de control Unicode (\[\\uFEFF\\u0000-\\u001F\\u007F-\\u009F\]) que a veces pueden colarse en las respuestas y romper el parseo JSON.  
- **Depuración Detallada (debugInfo):** Se introdujeron múltiples campos de depuración (last50CharsOriginal, jsonStringForParse\_first50, jsonStringLength, etc.) en el objeto de salida del nodo Code. Esto permitió inspeccionar exactamente qué cadena se estaba intentando parsear en cada etapa, lo cual fue fundamental para diagnosticar los problemas persistentes y validar la efectividad de las limpiezas.  
2. **Problema Adicional: Columnas de Depuración en Google Sheets**  
* **Síntoma:** Una vez que el parseo del JSON funcionó, se observó que la hoja de Google Sheets mostraba columnas adicionales con los nombres de las propiedades de debugInfo (ej. last50CharsOriginal, jsonStringLength).  
* **Causa Raíz:** Estas columnas fueron el resultado directo de nuestra solución de depuración. Al adjuntar ...debugInfo al objeto JSON de los casos de prueba (items.push({ json: { ...data, ...debugInfo } })), estábamos incluyendo intencionalmente esa información en la salida del nodo Code para depuración. Cuando los datos se enviaban a Google Sheets, el nodo de Google Sheets los interpretaba como nuevas columnas.  
* **Solución:** Se modificaron las líneas donde se construyen los ítems de salida en el nodo Code para **excluir explícitamente ...debugInfo**. Esto aseguró que solo las propiedades intrínsecas de los casos de prueba (ID, descripción, etc.) fueran enviadas a Google Sheets. Se mantuvo la generación de debugInfo internamente en el nodo Code para futuras depuraciones si fuera necesario, pero sin que afecte la salida final.

---

### 

### **Resultado Final**

Gracias a la depuración sistemática y las iteraciones en el script del nodo Code, el flujo ahora es robusto y funcional. La IA genera los casos de prueba en formato JSON, el nodo Code los procesa y limpia de manera confiable, y el nodo Google Sheets los guarda de forma estructurada en la hoja de cálculo, sin columnas adicionales indeseadas.

Este proceso destaca la importancia de una buena estrategia de depuración y la flexibilidad de n8n, particularmente del nodo Code, para adaptarse a las particularidades de las respuestas de las APIs y garantizar la integridad de los datos en el flujo.

