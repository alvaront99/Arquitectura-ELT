# Emulador IoT Concurrente: Capa de Generación de Datos

Este módulo contiene el código fuente del **Emulador IoT multihilo**, el componente encargado de actuar como fuente de datos (*Data Source*) de la arquitectura meteorológica.

El objetivo de este desarrollo no es simplemente generar números aleatorios, sino resolver un problema clásico de la ingeniería de datos en origen: garantizar la **consistencia temporal y la atomicidad** de las lecturas físicas mediante programación concurrente, asegurando que los registros sean íntegros antes de su distribución.

## Tecnologías Utilizadas

* **Java**: Lenguaje principal (hilos nativos, `ExecutorService`, `CyclicBarrier`).
* **Spring Boot**: Orquestación y despliegue del componente.
* **Spring Data JPA**: Mapeo objeto-relacional (ORM) y persistencia.
* **PostgreSQL**: Base de datos relacional para el almacenamiento en crudo.

## Enfoque Técnico 

El emulador implementa un modelo de concurrencia avanzada para simular distintos sensores (temperatura, humedad, viento, precipitación) operando al mismo tiempo. 

### 1. Concurrencia y Sincronización 

Para evitar condiciones de carrera (por ejemplo, que el sistema registre la temperatura de un segundo y la humedad del segundo siguiente), se utiliza una barrera de sincronización. Un `ExecutorService` lanza los hilos y la `CyclicBarrier` los obliga a esperarse mutuamente. El paquete de datos solo se ensambla cuando las 5 métricas están calculadas.

```java
    private void ejecutarCiclo() {
        SiLlueve(); // Determina el estado climático base para correlacionar variables
        final double tempCalculada = Temperatura();
        
        Map<String, Object> lecturaActual = new ConcurrentHashMap<>();

        // La barrera: el bloque interno SOLO se ejecuta cuando los 5 hilos llegan al .await()
        CyclicBarrier barrera = new CyclicBarrier(5, () -> {
            procesarYEnviar(lecturaActual);
        });

        // Lanzamiento concurrente de los "sensores"
        poolSensores.execute(() -> registrarDato("0_fecha", LocalDateTime.now().format(formatter), lecturaActual, barrera));
        poolSensores.execute(() -> registrarDato("1_temperatura", tempCalculada, lecturaActual, barrera));
        poolSensores.execute(() -> registrarDato("2_humedad", Humedad(tempCalculada), lecturaActual, barrera));
        poolSensores.execute(() -> registrarDato("3_viento", Viento(), lecturaActual, barrera));
        poolSensores.execute(() -> registrarDato("4_precipitacion", Precipitacion(), lecturaActual, barrera));
    }

    private void registrarDato(String clave, Object valor, Map<String, Object> mapa, CyclicBarrier b) {
        try {
            mapa.put(clave, valor);
            b.await(); // El hilo se bloquea aquí esperando al resto
        } catch (InterruptedException | BrokenBarrierException e) {
            Thread.currentThread().interrupt();
        }
    }
```

### 2. Ensamblaje y Persistencia en Origen

Una vez que la barrera de sincronización se libera, el orquestador toma el control. Su responsabilidad en esta fase es extraer las métricas calculadas concurrentemente, mapearlas contra el modelo de datos estricto y persistirlas de manera segura en la capa RAW (PostgreSQL), actuando como el punto de entrada a la arquitectura de datos.

```java
    private void procesarYEnviar(Map<String, Object> datosDesordenados) {
        try {
            // 1. Ensamblaje y mapeo de la entidad
            Estacion entidad = new Estacion();
            entidad.setFechaHora(LocalDateTime.now());
            entidad.setTemperatura((double) datosDesordenados.get("1_temperatura"));
            entidad.setHumedad((double) datosDesordenados.get("2_humedad"));
            entidad.setVelocidadViento((double) datosDesordenados.get("3_viento"));
            entidad.setPrecipitacion((double) datosDesordenados.get("4_precipitacion"));

            // 2. Persistencia en la capa RAW (Punto de entrada de la arquitectura)
            repository.save(entidad);
            
            System.out.println("\n[SISTEMA] >>> ¡Ciclo persistido en la base de datos con éxito! <<<");

        } catch (Exception e) {
            System.err.println("Error crítico al guardar en PostgreSQL: " + e.getMessage());
        }
    }
```

## Flujo de Trabajo 

1. **Micro-batching**: Un `ScheduledExecutorService` dispara el ciclo de lectura cada 5 segundos.
2. **Cálculo de métricas**: Los hilos simulan las lecturas aplicando distribuciones estadísticas correlacionadas (ej. la lluvia afecta a la humedad).
3. **Sincronización Atómica**: La `CyclicBarrier` detiene los procesos hasta tener una foto exacta e instantánea de todos los sensores.
4. **Persistencia**: El dato íntegro se guarda en PostgreSQL, dejando el registro listo para los procesos de ingesta y transformación (ELT) posteriores.


## 🔮 Mejoras Futuras Identificadas

Al tratarse de una primera iteración, existen puntos de mejora técnica identificados para aumentar la resiliencia del sistema en un entorno de producción:

* **Prevención de *Deadlocks* (Timeout en la Barrera):** Actualmente, el método `b.await()` bloquea el hilo indefinidamente. Una mejora crítica sería implementar un tiempo máximo de espera (ej. `b.await(2, TimeUnit.SECONDS)`). Así, si el hilo de un sensor falla o se queda colgado, se evita que todo el ciclo de lectura quede bloqueado y se puede gestionar el descarte del paquete.
* **Gestión de Hilos Huérfanos:** Muy ligado al punto anterior, implementar una estrategia de captura de la excepción `TimeoutException` para limpiar el `ConcurrentHashMap`, resetear la barrera y asegurar que el *pool* de conexiones se recupera limpiamente para el siguiente ciclo de 5 segundos.
