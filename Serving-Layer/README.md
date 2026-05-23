# Serving Layer: Visualización y Explotación del Dato

Esta capa constituye el punto final del ciclo de vida del dato en la arquitectura actual. Su propósito es presentar la información procesada tanto por la **Ruta Caliente** (alertas en tiempo real) como por la **Ruta Fría** (análisis batch), permitiendo la supervisión del sistema y la validación de los datos.

## Enfoque: De la Interfaz Custom a la Inteligencia de Negocio

Actualmente, el sistema cuenta con una interfaz básica orientada a la validación técnica durante la fase de desarrollo. Sin embargo, la estrategia definida para esta capa es clara: **la migración hacia Power BI.**

* **Estado Actual:** En esta fase de prototipado, el sistema se centra exclusivamente en la explotación de los datos de la **Ruta Fría (Batch)**. Esto permite validar que los cálculos, la ingeniería de características y las agregaciones de la capa Gold son correctos antes de escalar el volumen de datos.
* **Visión a Futuro (Power BI):** El objetivo es eliminar la capa de visualización desarrollada a medida. Power BI no solo ofrece una capacidad técnica superior para la creación de gráficos, sino que permitirá a los usuarios finales:
    * **Exploración profunda:** Navegar entre diferentes dimensiones temporales y métricas climáticas con mucha mayor agilidad.
    * **Escalabilidad:** Manejar grandes volúmenes de datos históricos (Capa Gold) sin penalizar el rendimiento del sistema.
    * **IA de Usuario:** Aprovechar las capacidades integradas de *AutoML* y detección de patrones de Power BI sobre los datos ya estructurados en PostgreSQL.

## Alcance de la Explotación Actual

Debido a que el objetivo principal es la transición hacia una herramienta de BI dedicada y potente, la visualización actual tiene un alcance limitado:

* **Cumplimiento de requisitos (TFG):** La interfaz web desarrollada cumple con los requisitos funcionales mínimos exigidos en la documentación académica del TFG para la visualización y validación del flujo de datos.
* **Monitorización de la Ruta Fría**: Visualización de los indicadores clave generados en las tablas `gold_ciclos`, `gold_tendencias` y `gold_bioclima`.
* **Validación de Integridad**: Verificación técnica de que el proceso Batch (orquestado por el Vigía) ha persistido los resultados correctamente sin duplicidades.
* **Gestión de Alertas**: Consulta del histórico de logs generados por la evaluación en tiempo real.

> **Nota de Diseño:** Al no ser un rol orientado al desarrollo frontend, la interfaz actual se ha limitado estrictamente a los requerimientos técnicos del proyecto. Mi prioridad ha sido dedicar el esfuerzo a la robustez del backend y a la calidad del dato en origen. La migración estratégica a Power BI nos permitirá escalar esta visualización hacia un entorno de explotación real mucho más eficiente.

## Hoja de Ruta de Migración

* **Fase 1 (Actual):** Explotación vía herramientas de consulta y visualización ligera sobre las tablas Gold en PostgreSQL.
* **Fase 2 (Próxima):** Conexión directa del modelo de datos de PostgreSQL a Power BI mediante el conector nativo.
* **Fase 3 (Final):** Desmantelamiento de los componentes de visualización *custom* y publicación de los cuadros de mando (Dashboards) sobre el servicio de Power BI, permitiendo la actualización automática programada de los datos.
