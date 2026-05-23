# Capa Batch (Ruta Fría): Orquestación y Procesamiento ELT

Este módulo contiene la implementación de la **Ruta Fría** de la arquitectura meteorológica. Su responsabilidad es extraer los datos en crudo (RAW) almacenados previamente por el emulador, transformarlos para generar nuevas características de negocio (Feature Engineering) y cargarlos en tablas analíticas estructuradas.

> **💡 Nota sobre la elección tecnológica (Spark vs. Pandas):** > El uso de Apache Spark en esta capa tiene un fin puramente formativo. Soy consciente de que el volumen de datos generado por el emulador es mínimo y podría haberse procesado de forma mucho más sencilla utilizando librerías en memoria como Pandas. Sin embargo, el objetivo del proyecto era enfrentarme a la lógica de un motor de procesamiento distribuido (configuración de sesión, gestión de memoria del *driver*, operaciones sobre *DataFrames* distribuidos) para interiorizar los estándares utilizados en entornos reales de Big Data.

Además, para la orquestación, en lugar de desplegar herramientas empresariales complejas desde el primer día, he optado por desarrollar un *script* a medida en Python. El objetivo era comprender a bajo nivel cómo se gestionan los ciclos de ejecución, la asignación de recursos y la tolerancia a fallos en trabajos por lotes (*batch*).

## Tecnologías Utilizadas

* **Apache Spark (PySpark)**: Motor de procesamiento de datos distribuidos.
* **Python 3**: Lenguaje base para el orquestador y los scripts de Spark.
* **PostgreSQL (Driver JDBC)**: Base de datos relacional para lectura (Capa Bronce) y escritura (Capa Gold).
* **Linux (WSL2)**: Entorno de ejecución para la gestión nativa de procesos (`subprocess`).

## Enfoque Técnico y Componentes Clave

El módulo se divide en dos piezas fundamentales que operan de forma desacoplada: el orquestador (Vigía) y el motor de procesamiento (Spark).

### 1. Orquestación a Bajo Nivel 

El orquestador es un script en Python diseñado para simular ventanas de tiempo (micro-ciclos diarios). Se encarga de levantar la máquina virtual de Java y el entorno de Spark de forma controlada, asignar la memoria necesaria (`--driver-memory 2g`) y monitorizar el código de salida. 

Si el trabajo de Spark falla por cualquier motivo, el Vigía atrapa el error y permite que el sistema espere al siguiente ciclo sin que el proceso orquestador principal se caiga (tolerancia a fallos básica).

```python
def ejecutar_spark(dia):
    """Lanza el proceso Spark para procesar el día indicado asignando recursos."""
    print(f"\n>>> [{time.strftime('%H:%M:%S')}] Despertando a Spark para el Día {dia}...")
    
    comando = [
        "spark-submit",
        "--master", "local[2]", # Limitado a 2 núcleos para pruebas locales
        "--driver-memory", "2g", 
        "--jars", "postgresql-42.7.10.jar",
        "procesamiento.py", 
        str(dia)
    ]
    
    # El proceso principal espera la finalización del trabajo Spark
    resultado = subprocess.run(comando)
    return resultado.returncode == 0

def orquestar():
    dia_actual = 1
    while True:
        time.sleep(TIEMPO_DIA_MINUTOS * 60) # Espera del ciclo batch
        exito = ejecutar_spark(dia_actual)
        
        if exito:
            dia_actual += 1
        else:
            # Tolerancia a fallos: no detenemos el bucle infinito
            print(f"[FALLO] Spark tuvo un error en el Día {dia_actual}. Se reintentará en el próximo ciclo.")
```

### 2. Procesamiento ELT e Ingeniería de Características (PySpark)

El script de Spark se encarga de la carga pesada. Lee los datos base (Capa Bronce) y realiza cálculos físicos (como el *Wind Chill* o la energía del viento) mediante transformaciones puras (`withColumn`). Posteriormente, realiza agregaciones (`groupBy`) para separar los datos en tres vistas analíticas (Capa Gold): Resúmenes de ciclo, Tendencias (anomalías) y Bioclima.

```python
# 1. Ingeniería de Características Base (Ej. Sensación Térmica / Wind Chill)
df_base = df_bronce.withColumn("wind_chill", 
    F.when((F.col("temperatura_c") <= 10) & (F.col("velocidad_viento_kmh") > 4.8),
        13.12 + 0.6215 * F.col("temperatura_c") - 11.37 * F.pow(F.col("velocidad_viento_kmh"), 0.16) + 
        0.3965 * F.col("temperatura_c") * F.pow(F.col("velocidad_viento_kmh"), 0.16)
    ).otherwise(F.col("temperatura_c"))
)

# 2. Agregación Analítica (Capa Gold: Ciclos)
df_ciclos = df_base.groupBy().agg(
    F.lit(num_dia).alias("num_ciclo"),
    F.max("temperatura_c").alias("t_max"),
    F.min("temperatura_c").alias("t_min"),
    F.avg("precipitacion_mm").alias("precip_total")
).withColumn("confort_predom", 
    F.when(F.col("t_max") > 30, "Calor Extremo")
     .when(F.col("t_min") < 5, "Frio Intenso")
     .otherwise("Optimo")
)

# 3. Escritura en PostgreSQL (Modelo Multitabla Analítico)
df_ciclos.write.jdbc(url=url_jdbc, table="gold_ciclos", mode="append", properties=properties)
```

## Flujo de Trabajo

1. **Espera Activa**: El *Vigía* espera el tiempo configurado para el cierre del ciclo (simulación diaria).
2. **Invocación (Subprocess)**: Se dispara `spark-submit`, inyectando el número de ciclo como parámetro al script.
3. **Lectura JDBC (Extract)**: Spark lee la tabla RAW de PostgreSQL y la carga en memoria (`df.cache()`).
4. **Transformación (Transform)**: Se aplican fórmulas estadísticas y clasificación de anomalías.
5. **Escritura (Load)**: Spark persiste el resultado estructurado en tres tablas `gold_` diferentes empleando el modo `append`.
6. **Cierre de Ciclo**: El orquestador registra el éxito/fracaso y pasa al siguiente lote si corresponde.

## Mejoras Futuras Identificadas

Al tratarse de una primera versión del flujo de datos, se ha detectado un punto de mejora técnica prioritario de cara a entornos de producción reales:

* **Idempotencia en la Escritura:** Actualmente, el volcado en las tablas analíticas utiliza el modo `append`. Si el script de PySpark se llegara a ejecutar dos veces para el mismo ciclo (por ejemplo, debido a un reinicio forzado del orquestador o un fallo en la red), los registros de ese día se duplicarían en la base de datos. Una mejora clave sería implementar lógica de control para limpiar o sobrescribir de manera segura únicamente los datos pertenecientes al ciclo activo (`num_dia`) antes de realizar la inserción, asegurando que el resultado final sea siempre el mismo independientemente de cuántas veces se lance el proceso.

