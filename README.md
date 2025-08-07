# 🤖 Flujos en n8n para automatizar pruebas y procesos como QA

Este repositorio centraliza los flujos de n8n para la automatización de procesos de **Control de Calidad (QA)**.

El objetivo principal es crear un sistema que pueda:
1.  Generar casos de prueba automáticamente utilizando IA.
2.  Ejecutar esas pruebas en una API utilizando Newman.
3.  Registrar los errores o incidencias en una herramienta de gestión como Jira.

---

### **🚀 Estado de los Flujos**

Actualmente, solo el flujo de **generación de casos de prueba** está completo(adaptable al tipo de prueba necesitada). Los demás se irán añadiendo a medida que se desarrollen.

* **`flujo-generacion-casos-prueba.json`**: **(Listo)** Este flujo usa IA para generar casos de prueba y los formatea en JSON.
* `flujo-ejecucion-pruebas.json`: (En desarrollo)
* `flujo-gestion-jira.json`: (En desarrollo)

---

### **🔑 Configuración de Variables**

El flujo contiene marcadores de posición (`{{ ... }}`) en lugar de datos sensibles. Tenés que reemplazar estos valores para que el flujo funcione.

* **Clave de API de IA**: `{{ TU_CLAVE }}`
* **ID de la credencial de Google Sheets**: `{{ MI_CREDENCIAL_ID }}`

---

### **📝 Instrucciones para el Reemplazo**

1.  Abrí el archivo `flujo-generacion-casos-prueba.json`.
2.  Buscá las variables con el formato `{{ ... }}`.
3.  Reemplazá cada una de ellas con tu clave de API, ID de credencial O ID de hoja de cálculo.
