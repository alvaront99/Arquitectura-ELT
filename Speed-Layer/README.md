# Microservicio de Evaluación en Tiempo Real y CRUD de Alertas

Este módulo comprende la implementación de la **Ruta Caliente** (Capa *Speed*) de la arquitectura. Se trata de un microservicio independiente que recibe métricas clave desde el emulador, las evalúa bajo reglas dinámicas y gestiona el histórico de incidencias.

## Enfoque Técnico

El sistema separa la **gestión de reglas (CRUD)** de la **evaluación en tiempo real**, garantizando que el monitoreo sea independiente del resto de la arquitectura.

### 1. Motor de Evaluación Dinámico
El corazón del servicio es su capacidad de evaluar flujos de datos contra reglas configurables en caliente. El método de evaluación utiliza lógica matemática de comparación:

```java
// Lógica de evaluación contra reglas activas
public void analizarDatoEnTiempoReal(double valor, String metricaDetectada) {
    List<Alerta> reglasActivas = alertaRepo.findByActivaTrue();

    for (Alerta regla : reglasActivas) {
        if (regla.getMetrica().equalsIgnoreCase(metricaDetectada)) {
            if (evaluarCondicion(valor, regla.getOperador(), regla.getUmbral())) {
                logRepo.save(new LogAlerta(String.format("Alerta! %s detectada", regla.getNombre()), valor));
            }
        }
    }
}

// Evaluación matemática con Switch Expressions (Java 14+)
private boolean evaluarCondicion(double valor, String operador, double umbral) {
    return switch (operador) {
        case ">"  -> valor > umbral;
        case "<"  -> valor < umbral;
        case ">=" -> valor >= umbral;
        case "<=" -> valor <= umbral;
        default   -> false;
    };
}
```

### 2. Gestión de Reglas y Persistencia (CRUD)
El servicio expone una API REST completa para la administración del sistema:
* **Gestión Dinámica (`/config`)**: Permite operaciones CRUD completas para definir umbrales de seguridad. Esto garantiza que las reglas del motor no estén "quemadas" (*hardcoded*) en el código, sino almacenadas en PostgreSQL para su modificación en tiempo real.
* **Historial (`/historial`)**: Consulta de los logs de anomalías detectadas, persistidos en la base de datos para auditoría y visualización.

## Flujo de Trabajo 

1. **Configuración**: El administrador define los umbrales de seguridad mediante la API REST.
2. **Recepción**: El microservicio recibe un `HTTP POST` constante con los datos del emulador en el puerto `8081`.
3. **Validación**: El servicio cruza el dato entrante contra la lista de reglas activas.
4. **Almacenamiento**: Si el dato infringe una regla, se registra inmediatamente un `LogAlerta` en la base de datos.

## Mejoras Futuras
* **Modelos Predictivos:** Integrar un modelo de ML elemental que aprenda de los patrones históricos para predecir alertas antes de alcanzar umbrales críticos.
* **Mensajería Asíncrona:** Migrar la ingesta del endpoint `/analizar` hacia un broker (Kafka/RabbitMQ) para desacoplar totalmente la recepción del dato de su procesamiento, eliminando posibles cuellos de botella en la red.
