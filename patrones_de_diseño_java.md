# Guía de Patrones de Diseño en Java / Spring Boot

> Guía práctica con explicación, código listo para copiar (Java 17 + Spring Boot 3), casos de uso y beneficios de cada patrón.

---

## Índice

1. [¿Qué son los patrones de diseño?](#1-qué-son-los-patrones-de-diseño)
2. [Tabla resumen](#2-tabla-resumen)
3. **Patrones creacionales**
   - [Singleton](#31-singleton)
   - [Factory Method](#32-factory-method)
   - [Builder](#33-builder)
4. **Patrones estructurales**
   - [Adapter](#41-adapter)
   - [Decorator](#42-decorator)
   - [Facade](#43-facade)
   - [Proxy](#44-proxy)
5. **Patrones de comportamiento**
   - [Strategy](#51-strategy)
   - [Observer](#52-observer)
   - [Template Method](#53-template-method)
   - [Chain of Responsibility](#54-chain-of-responsibility)
   - [Command](#55-command)
6. **Patrones de arquitectura / empresariales**
   - [Dependency Injection](#61-dependency-injection)
   - [Repository](#62-repository)
   - [DTO (Data Transfer Object)](#63-dto-data-transfer-object)
   - [Service Layer](#64-service-layer)
7. [Cómo elegir el patrón correcto](#7-cómo-elegir-el-patrón-correcto)
8. [Antipatrones comunes](#8-antipatrones-comunes)

---

## 1. ¿Qué son los patrones de diseño?

Un **patrón de diseño** es una solución probada y reutilizable a un problema recurrente en el diseño de software. No es código terminado, sino una **plantilla** de cómo organizar clases y objetos para resolver un problema concreto.

Se popularizaron con el libro *Design Patterns* (1994) del "Gang of Four" (GoF), que los clasifica en tres grupos:

| Categoría | Propósito | Pregunta que responde |
|---|---|---|
| **Creacionales** | Cómo se crean los objetos | ¿Quién crea el objeto y cómo? |
| **Estructurales** | Cómo se componen clases y objetos | ¿Cómo encajan las piezas? |
| **Comportamiento** | Cómo se comunican los objetos | ¿Quién hace qué y cuándo? |

Spring Boot **usa internamente muchos de estos patrones**, así que entenderlos te ayuda a entender el framework y a escribir código que encaje naturalmente con él.

---

## 2. Tabla resumen

| Patrón | Categoría | Problema que resuelve | Dónde lo usa Spring |
|---|---|---|---|
| Singleton | Creacional | Una sola instancia compartida | Scope por defecto de los beans |
| Factory Method | Creacional | Crear objetos sin acoplarse a la clase concreta | `BeanFactory`, `@Bean` |
| Builder | Creacional | Construir objetos complejos paso a paso | `ResponseEntity`, `WebClient.builder()` |
| Adapter | Estructural | Hacer compatibles interfaces distintas | `HandlerAdapter` en Spring MVC |
| Decorator | Estructural | Añadir comportamiento sin modificar la clase | `BeanPostProcessor`, wrappers de `HttpServletRequest` |
| Facade | Estructural | Simplificar un subsistema complejo | `JdbcTemplate`, `RestTemplate` |
| Proxy | Estructural | Interceptar llamadas a un objeto | AOP, `@Transactional`, `@Cacheable` |
| Strategy | Comportamiento | Intercambiar algoritmos en tiempo de ejecución | `PasswordEncoder`, `AuthenticationProvider` |
| Observer | Comportamiento | Notificar cambios a múltiples interesados | `ApplicationEvent`, `@EventListener` |
| Template Method | Comportamiento | Esqueleto de algoritmo con pasos variables | `JdbcTemplate`, `AbstractController` |
| Chain of Responsibility | Comportamiento | Procesar una petición en una cadena de manejadores | `Filter` de Spring Security |
| Command | Comportamiento | Encapsular una acción como objeto | `Runnable` en `TaskExecutor` |
| Dependency Injection | Arquitectura | Desacoplar dependencias | Núcleo de Spring (IoC container) |
| Repository | Arquitectura | Abstraer el acceso a datos | Spring Data JPA |
| DTO | Arquitectura | Transferir datos entre capas | Requests/Responses REST |
| Service Layer | Arquitectura | Centralizar la lógica de negocio | `@Service` |

---

# 3. Patrones creacionales

## 3.1 Singleton

### ¿A qué corresponde?
Garantiza que una clase tenga **una única instancia** en toda la aplicación y proporciona un punto de acceso global a ella.

En Spring **no necesitas implementarlo manualmente**: todos los beans (`@Component`, `@Service`, `@Repository`, `@Configuration`) son singleton por defecto dentro del contenedor.

### Ejemplo de código

**Singleton clásico en Java puro (para entender el concepto):**

```java
public final class ConfiguracionGlobal {

    // 1. La instancia se crea una sola vez (thread-safe gracias al class loader)
    private static final ConfiguracionGlobal INSTANCIA = new ConfiguracionGlobal();

    private final String entorno;

    // 2. Constructor privado: nadie más puede hacer "new"
    private ConfiguracionGlobal() {
        this.entorno = System.getenv().getOrDefault("APP_ENV", "dev");
    }

    // 3. Punto de acceso global
    public static ConfiguracionGlobal getInstancia() {
        return INSTANCIA;
    }

    public String getEntorno() {
        return entorno;
    }
}
```

**Singleton al estilo Spring (la forma recomendada):**

```java
import org.springframework.stereotype.Component;
import java.util.concurrent.ConcurrentHashMap;
import java.util.Map;

@Component // Singleton por defecto: Spring crea UNA instancia y la inyecta donde se necesite
public class CacheEnMemoria {

    // Debe ser thread-safe porque la instancia es compartida entre todas las peticiones
    private final Map<String, Object> cache = new ConcurrentHashMap<>();

    public void guardar(String clave, Object valor) {
        cache.put(clave, valor);
    }

    public Object obtener(String clave) {
        return cache.get(clave);
    }
}
```

**Explicación:**
- En la versión clásica, el constructor privado y el campo `static final` garantizan una sola instancia.
- En Spring, el contenedor IoC gestiona el ciclo de vida. Si necesitas otra cosa, cambias el scope: `@Scope("prototype")`, `@RequestScope`, `@SessionScope`.
- ⚠️ Como la instancia es compartida entre hilos, **no guardes estado mutable no thread-safe** en un bean singleton.

### Casos de uso
- Servicios sin estado (`@Service`).
- Cachés en memoria, pools de conexiones, clientes HTTP (`WebClient`, `RestClient`).
- Configuración global de la aplicación.

### Beneficios
| Beneficio | Descripción |
|---|---|
| Ahorro de memoria | Una sola instancia en lugar de miles |
| Estado compartido controlado | Un único punto de verdad |
| Rendimiento | Evita el costo de crear objetos pesados repetidamente |

---

## 3.2 Factory Method

### ¿A qué corresponde?
Define una forma de **crear objetos sin especificar la clase concreta** que se instanciará. El cliente pide "un objeto que haga X" y la fábrica decide qué implementación entregar.

### Ejemplo de código

```java
// 1. Contrato común
public interface Notificador {
    TipoNotificacion getTipo();
    void enviar(String destinatario, String mensaje);
}

public enum TipoNotificacion { EMAIL, SMS, PUSH }
```

```java
import org.springframework.stereotype.Component;

// 2. Implementaciones concretas (cada una es un bean)
@Component
public class EmailNotificador implements Notificador {
    @Override public TipoNotificacion getTipo() { return TipoNotificacion.EMAIL; }
    @Override public void enviar(String destinatario, String mensaje) {
        System.out.println("📧 Email a " + destinatario + ": " + mensaje);
    }
}

@Component
public class SmsNotificador implements Notificador {
    @Override public TipoNotificacion getTipo() { return TipoNotificacion.SMS; }
    @Override public void enviar(String destinatario, String mensaje) {
        System.out.println("📱 SMS a " + destinatario + ": " + mensaje);
    }
}

@Component
public class PushNotificador implements Notificador {
    @Override public TipoNotificacion getTipo() { return TipoNotificacion.PUSH; }
    @Override public void enviar(String destinatario, String mensaje) {
        System.out.println("🔔 Push a " + destinatario + ": " + mensaje);
    }
}
```

```java
import org.springframework.stereotype.Component;
import java.util.EnumMap;
import java.util.List;
import java.util.Map;

// 3. La fábrica
@Component
public class NotificadorFactory {

    private final Map<TipoNotificacion, Notificador> notificadores = new EnumMap<>(TipoNotificacion.class);

    // Spring inyecta TODAS las implementaciones de Notificador automáticamente
    public NotificadorFactory(List<Notificador> implementaciones) {
        implementaciones.forEach(n -> notificadores.put(n.getTipo(), n));
    }

    public Notificador crear(TipoNotificacion tipo) {
        Notificador notificador = notificadores.get(tipo);
        if (notificador == null) {
            throw new IllegalArgumentException("Tipo de notificación no soportado: " + tipo);
        }
        return notificador;
    }
}
```

```java
// 4. Uso desde un servicio
@Service
public class PedidoService {

    private final NotificadorFactory factory;

    public PedidoService(NotificadorFactory factory) {
        this.factory = factory;
    }

    public void confirmarPedido(String cliente, TipoNotificacion preferencia) {
        factory.crear(preferencia).enviar(cliente, "Tu pedido fue confirmado ✅");
    }
}
```

**Explicación:**
- `PedidoService` no conoce `EmailNotificador` ni `SmsNotificador`; solo conoce la interfaz.
- El truco de inyectar `List<Notificador>` hace que **agregar un nuevo canal** (ej. WhatsApp) sea tan simple como crear una nueva clase `@Component`. La fábrica no se modifica.

### Casos de uso
- Elegir proveedor de pagos (Stripe, PayPal, MercadoPago) según el país.
- Generar reportes en distintos formatos (PDF, Excel, CSV).
- Seleccionar parsers según el tipo de archivo.

### Beneficios
| Beneficio | Descripción |
|---|---|
| Bajo acoplamiento | El cliente depende de la interfaz, no de la clase concreta |
| Principio Open/Closed | Nuevas implementaciones sin tocar código existente |
| Centralización | La lógica de creación vive en un solo lugar |

---

## 3.3 Builder

### ¿A qué corresponde?
Separa la **construcción de un objeto complejo** de su representación, permitiendo crearlo paso a paso con una API fluida. Ideal cuando un objeto tiene muchos parámetros, varios de ellos opcionales.

### Ejemplo de código

**Builder manual:**

```java
import java.math.BigDecimal;
import java.util.ArrayList;
import java.util.List;

public final class Factura {

    private final String cliente;
    private final String rfc;
    private final List<String> conceptos;
    private final BigDecimal subtotal;
    private final BigDecimal iva;
    private final String notas; // opcional

    // Constructor privado: solo el Builder puede crear instancias
    private Factura(Builder b) {
        this.cliente = b.cliente;
        this.rfc = b.rfc;
        this.conceptos = List.copyOf(b.conceptos); // inmutable
        this.subtotal = b.subtotal;
        this.iva = b.subtotal.multiply(new BigDecimal("0.16"));
        this.notas = b.notas;
    }

    public static Builder builder() { return new Builder(); }

    public BigDecimal getTotal() { return subtotal.add(iva); }

    public static class Builder {
        private String cliente;
        private String rfc;
        private final List<String> conceptos = new ArrayList<>();
        private BigDecimal subtotal = BigDecimal.ZERO;
        private String notas;

        public Builder cliente(String cliente) { this.cliente = cliente; return this; }
        public Builder rfc(String rfc) { this.rfc = rfc; return this; }
        public Builder concepto(String c, BigDecimal monto) {
            this.conceptos.add(c);
            this.subtotal = this.subtotal.add(monto);
            return this;
        }
        public Builder notas(String notas) { this.notas = notas; return this; }

        public Factura build() {
            // Validaciones centralizadas antes de construir
            if (cliente == null || rfc == null) {
                throw new IllegalStateException("Cliente y RFC son obligatorios");
            }
            if (conceptos.isEmpty()) {
                throw new IllegalStateException("La factura debe tener al menos un concepto");
            }
            return new Factura(this);
        }
    }
}
```

**Uso:**

```java
Factura factura = Factura.builder()
        .cliente("Juan Pérez")
        .rfc("PEJJ800101ABC")
        .concepto("Desarrollo web", new BigDecimal("15000"))
        .concepto("Hosting anual", new BigDecimal("2400"))
        .notas("Pago a 30 días")
        .build();
```

**Con Lombok (lo más común en Spring Boot):**

```java
import lombok.Builder;
import lombok.Singular;
import lombok.Value;
import java.util.List;

@Value      // Clase inmutable con getters
@Builder    // Genera el builder automáticamente
public class Usuario {
    String nombre;
    String email;
    @Builder.Default boolean activo = true;
    @Singular List<String> roles; // permite .role("ADMIN").role("USER")
}

// Uso:
Usuario u = Usuario.builder()
        .nombre("Ana")
        .email("ana@mail.com")
        .role("ADMIN")
        .build();
```

**Explicación:**
- Evita el "constructor telescópico" (`new Factura(a, b, null, null, c, null...)`).
- La validación ocurre en `build()`, garantizando que nunca exista un objeto en estado inválido.
- Spring lo usa en: `ResponseEntity.ok().header(...).body(...)`, `WebClient.builder()`, `UriComponentsBuilder`.

### Casos de uso
- Entidades o DTOs con muchos campos opcionales.
- Construcción de queries dinámicas o criterios de búsqueda.
- Configuración de clientes HTTP.
- Datos de prueba en tests (`UsuarioTestBuilder`).

### Beneficios
| Beneficio | Descripción |
|---|---|
| Legibilidad | Cada parámetro se nombra explícitamente |
| Inmutabilidad | Objetos finales y thread-safe |
| Validación centralizada | Nunca se crea un objeto inválido |
| Flexibilidad | Parámetros opcionales sin múltiples constructores |

---

# 4. Patrones estructurales

## 4.1 Adapter

### ¿A qué corresponde?
Convierte la interfaz de una clase en **otra interfaz que el cliente espera**. Es como un adaptador de enchufe: permite que dos componentes incompatibles trabajen juntos.

### Ejemplo de código

```java
// Interfaz que usa NUESTRA aplicación
public interface ProcesadorPago {
    ResultadoPago cobrar(String clienteId, BigDecimal monto);
}

public record ResultadoPago(boolean exitoso, String transaccionId, String mensaje) {}
```

```java
// SDK de terceros con una API distinta (no podemos modificarla)
public class StripeClient {
    public StripeCharge createCharge(long amountInCents, String currency, String customer) {
        // ... llamada real a Stripe
        return new StripeCharge("ch_123", "succeeded");
    }
}

public record StripeCharge(String id, String status) {}
```

```java
import org.springframework.stereotype.Component;
import java.math.BigDecimal;

// El ADAPTER: implementa nuestra interfaz y traduce hacia Stripe
@Component
public class StripeAdapter implements ProcesadorPago {

    private final StripeClient stripe = new StripeClient();

    @Override
    public ResultadoPago cobrar(String clienteId, BigDecimal monto) {
        // Traducción: pesos -> centavos, y nuestra firma -> la de Stripe
        long centavos = monto.multiply(BigDecimal.valueOf(100)).longValueExact();
        StripeCharge charge = stripe.createCharge(centavos, "mxn", clienteId);

        // Traducción de la respuesta de Stripe a nuestro modelo
        boolean ok = "succeeded".equals(charge.status());
        return new ResultadoPago(ok, charge.id(), ok ? "Pago aprobado" : "Pago rechazado");
    }
}
```

**Explicación:**
- Tu lógica de negocio solo conoce `ProcesadorPago`. Si mañana cambias a PayPal, creas `PaypalAdapter` y nada más cambia.
- El SDK externo queda **aislado** en una sola clase.

### Casos de uso
- Integrar APIs o SDKs de terceros.
- Migrar de un sistema legacy a uno nuevo de forma gradual.
- Unificar múltiples proveedores bajo una misma interfaz (envíos, SMS, almacenamiento).

### Beneficios
| Beneficio | Descripción |
|---|---|
| Aislamiento | Cambios en el proveedor externo afectan solo al adapter |
| Reutilización | Usas código existente sin modificarlo |
| Testabilidad | Fácil de mockear la interfaz propia |

---

## 4.2 Decorator

### ¿A qué corresponde?
Añade **responsabilidades adicionales a un objeto dinámicamente**, envolviéndolo en otro objeto que implementa la misma interfaz. Es una alternativa flexible a la herencia.

### Ejemplo de código

```java
public interface ServicioClima {
    String obtenerClima(String ciudad);
}
```

```java
// Implementación base
public class ServicioClimaApi implements ServicioClima {
    @Override
    public String obtenerClima(String ciudad) {
        // Simula llamada lenta a una API externa
        try { Thread.sleep(1000); } catch (InterruptedException ignored) {}
        return "Soleado, 25°C en " + ciudad;
    }
}
```

```java
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

// Decorador 1: agrega caché
public class ClimaConCache implements ServicioClima {
    private final ServicioClima envuelto;
    private final Map<String, String> cache = new ConcurrentHashMap<>();

    public ClimaConCache(ServicioClima envuelto) { this.envuelto = envuelto; }

    @Override
    public String obtenerClima(String ciudad) {
        return cache.computeIfAbsent(ciudad, envuelto::obtenerClima);
    }
}

// Decorador 2: agrega logging y medición de tiempo
public class ClimaConLog implements ServicioClima {
    private final ServicioClima envuelto;

    public ClimaConLog(ServicioClima envuelto) { this.envuelto = envuelto; }

    @Override
    public String obtenerClima(String ciudad) {
        long inicio = System.currentTimeMillis();
        String resultado = envuelto.obtenerClima(ciudad);
        System.out.printf("[LOG] clima(%s) tardó %d ms%n", ciudad, System.currentTimeMillis() - inicio);
        return resultado;
    }
}
```

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

// Composición de decoradores en Spring
@Configuration
public class ClimaConfig {

    @Bean
    public ServicioClima servicioClima() {
        // Log -> Cache -> API real
        return new ClimaConLog(new ClimaConCache(new ServicioClimaApi()));
    }
}
```

**Explicación:**
- Cada decorador hace **una sola cosa** y delega el resto.
- Puedes combinarlos en el orden que quieras sin crear subclases como `ClimaConCacheYLog`.
- Diferencia con Proxy: el Decorator **agrega funcionalidad**; el Proxy **controla el acceso**. En la práctica se parecen mucho.

### Casos de uso
- Añadir caché, logging, métricas, reintentos o compresión.
- Envolver `HttpServletRequest` para leer el body varias veces (`ContentCachingRequestWrapper`).
- Streams de Java (`BufferedInputStream(new FileInputStream(...))`).

### Beneficios
| Beneficio | Descripción |
|---|---|
| Composición sobre herencia | Evita explosión de subclases |
| Responsabilidad única | Cada decorador tiene un propósito |
| Flexibilidad | Se combinan y reordenan en tiempo de ejecución |

---

## 4.3 Facade

### ¿A qué corresponde?
Proporciona una **interfaz simplificada** a un subsistema complejo compuesto por muchas clases. El cliente llama a un solo método y la fachada orquesta todo por dentro.

### Ejemplo de código

```java
// Subsistemas complejos (cada uno un @Service independiente)
@Service class InventarioService {
    public void reservarStock(Long productoId, int cantidad) { /* ... */ }
}
@Service class PagoService {
    public String cobrar(Long clienteId, BigDecimal monto) { return "TX-001"; }
}
@Service class EnvioService {
    public String programarEnvio(Long pedidoId, String direccion) { return "GUIA-777"; }
}
@Service class EmailService {
    public void enviarConfirmacion(String email, String guia) { /* ... */ }
}
```

```java
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

public record CompraRequest(Long clienteId, String email, Long productoId,
                            int cantidad, BigDecimal monto, String direccion) {}

public record CompraResponse(String transaccionId, String guiaEnvio) {}

// LA FACHADA
@Service
public class CheckoutFacade {

    private final InventarioService inventario;
    private final PagoService pagos;
    private final EnvioService envios;
    private final EmailService emails;

    public CheckoutFacade(InventarioService inventario, PagoService pagos,
                          EnvioService envios, EmailService emails) {
        this.inventario = inventario;
        this.pagos = pagos;
        this.envios = envios;
        this.emails = emails;
    }

    // Un solo método para una operación que involucra 4 subsistemas
    @Transactional
    public CompraResponse realizarCompra(CompraRequest req) {
        inventario.reservarStock(req.productoId(), req.cantidad());
        String tx = pagos.cobrar(req.clienteId(), req.monto());
        String guia = envios.programarEnvio(req.productoId(), req.direccion());
        emails.enviarConfirmacion(req.email(), guia);
        return new CompraResponse(tx, guia);
    }
}
```

```java
// El controller queda limpio
@RestController
@RequestMapping("/api/checkout")
public class CheckoutController {

    private final CheckoutFacade checkout;

    public CheckoutController(CheckoutFacade checkout) { this.checkout = checkout; }

    @PostMapping
    public ResponseEntity<CompraResponse> comprar(@RequestBody CompraRequest req) {
        return ResponseEntity.ok(checkout.realizarCompra(req));
    }
}
```

**Explicación:**
- El controller no necesita conocer el orden ni los detalles de los 4 servicios.
- Spring usa este patrón en `JdbcTemplate` (oculta conexiones, statements, result sets y manejo de excepciones).

### Casos de uso
- Procesos de negocio que orquestan varios servicios (checkout, registro de usuario, onboarding).
- Simplificar librerías complejas (JDBC, APIs de AWS).
- Exponer una API limpia de un módulo hacia otros módulos.

### Beneficios
| Beneficio | Descripción |
|---|---|
| Simplicidad | El cliente usa una API pequeña y clara |
| Desacoplamiento | Los clientes no dependen de los subsistemas internos |
| Mantenibilidad | Cambios internos no afectan a quien usa la fachada |

---

## 4.4 Proxy

### ¿A qué corresponde?
Proporciona un **sustituto o intermediario** de otro objeto para controlar el acceso a él. El proxy intercepta las llamadas y puede ejecutar lógica antes y/o después.

Es **el patrón más importante de Spring**: `@Transactional`, `@Cacheable`, `@Async`, `@PreAuthorize` y toda la AOP funcionan con proxies.

### Ejemplo de código

**Proxy implícito de Spring:**

```java
@Service
public class CuentaService {

    // Spring NO te inyecta CuentaService directamente,
    // sino un PROXY que abre/cierra la transacción alrededor de tu método
    @Transactional
    public void transferir(Long origen, Long destino, BigDecimal monto) {
        // debitar(origen, monto);
        // acreditar(destino, monto);
        // Si ocurre una RuntimeException, el proxy hace rollback
    }

    @Cacheable("saldos") // El proxy revisa la caché antes de ejecutar el método
    public BigDecimal consultarSaldo(Long cuentaId) {
        return BigDecimal.TEN;
    }
}
```

**Proxy propio con AOP (anotación personalizada):**

```java
import java.lang.annotation.*;

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface MedirTiempo {}
```

```java
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;

@Aspect
@Component
public class MedicionAspect {

    private static final Logger log = LoggerFactory.getLogger(MedicionAspect.class);

    @Around("@annotation(MedirTiempo)")
    public Object medir(ProceedingJoinPoint pjp) throws Throwable {
        long inicio = System.nanoTime();
        try {
            return pjp.proceed(); // ejecuta el método real
        } finally {
            long ms = (System.nanoTime() - inicio) / 1_000_000;
            log.info("⏱ {} tardó {} ms", pjp.getSignature().toShortString(), ms);
        }
    }
}
```

```java
@Service
public class ReporteService {

    @MedirTiempo
    public byte[] generarReporteMensual() {
        // lógica pesada...
        return new byte[0];
    }
}
```

> Dependencia necesaria: `spring-boot-starter-aop`

**Explicación:**
- ⚠️ **Trampa clásica (self-invocation):** si un método de la misma clase llama a otro método anotado con `@Transactional` usando `this.metodo()`, **el proxy no intercepta** la llamada y la anotación no tiene efecto. Solución: mover el método a otro bean.
- Los proxies solo interceptan métodos **públicos** llamados desde fuera del bean.

### Casos de uso
- Transacciones, caché, seguridad, logging, métricas, reintentos.
- Lazy loading (Hibernate usa proxies para relaciones `LAZY`).
- Clientes remotos (`@FeignClient`, `@HttpExchange`).

### Beneficios
| Beneficio | Descripción |
|---|---|
| Separación de responsabilidades | Lógica transversal fuera del código de negocio |
| Código limpio | Una anotación reemplaza decenas de líneas repetidas |
| Control de acceso | Seguridad, validación o rate limiting centralizados |

---

# 5. Patrones de comportamiento

## 5.1 Strategy

### ¿A qué corresponde?
Define una **familia de algoritmos**, encapsula cada uno y los hace **intercambiables**. El cliente elige qué estrategia usar en tiempo de ejecución, eliminando largas cadenas de `if/else` o `switch`.

### Ejemplo de código

**❌ Sin patrón:**

```java
public BigDecimal calcularDescuento(String tipoCliente, BigDecimal total) {
    if (tipoCliente.equals("REGULAR")) {
        return BigDecimal.ZERO;
    } else if (tipoCliente.equals("VIP")) {
        return total.multiply(new BigDecimal("0.15"));
    } else if (tipoCliente.equals("EMPLEADO")) {
        return total.multiply(new BigDecimal("0.30"));
    }
    // Cada nuevo tipo = modificar este método 😩
    return BigDecimal.ZERO;
}
```

**✅ Con Strategy:**

```java
import java.math.BigDecimal;

public interface EstrategiaDescuento {
    String tipo();
    BigDecimal calcular(BigDecimal total);
}
```

```java
import org.springframework.stereotype.Component;
import java.math.BigDecimal;

@Component
public class DescuentoRegular implements EstrategiaDescuento {
    public String tipo() { return "REGULAR"; }
    public BigDecimal calcular(BigDecimal total) { return BigDecimal.ZERO; }
}

@Component
public class DescuentoVip implements EstrategiaDescuento {
    public String tipo() { return "VIP"; }
    public BigDecimal calcular(BigDecimal total) { return total.multiply(new BigDecimal("0.15")); }
}

@Component
public class DescuentoEmpleado implements EstrategiaDescuento {
    public String tipo() { return "EMPLEADO"; }
    public BigDecimal calcular(BigDecimal total) { return total.multiply(new BigDecimal("0.30")); }
}
```

```java
import org.springframework.stereotype.Service;
import java.math.BigDecimal;
import java.util.List;
import java.util.Map;
import java.util.function.Function;
import java.util.stream.Collectors;

@Service
public class DescuentoService {

    private final Map<String, EstrategiaDescuento> estrategias;

    public DescuentoService(List<EstrategiaDescuento> lista) {
        this.estrategias = lista.stream()
                .collect(Collectors.toMap(EstrategiaDescuento::tipo, Function.identity()));
    }

    public BigDecimal aplicar(String tipoCliente, BigDecimal total) {
        EstrategiaDescuento estrategia = estrategias.getOrDefault(tipoCliente, estrategias.get("REGULAR"));
        return total.subtract(estrategia.calcular(total));
    }
}
```

**Explicación:**
- Cada regla de negocio vive en su propia clase, fácil de testear de forma aislada.
- Para agregar `ESTUDIANTE` solo creas `DescuentoEstudiante` con `@Component`.
- Spring lo usa en `PasswordEncoder` (BCrypt, Argon2, etc.) y en `AuthenticationProvider`.

> 💡 **Strategy vs Factory:** la Factory se enfoca en **crear/entregar** el objeto; la Strategy se enfoca en **intercambiar el comportamiento**. Es muy común combinarlos, como en este ejemplo.

### Casos de uso
- Cálculo de descuentos, impuestos, comisiones o tarifas de envío.
- Algoritmos de ordenamiento o búsqueda configurables.
- Exportación a distintos formatos.
- Validaciones que cambian según el contexto.

### Beneficios
| Beneficio | Descripción |
|---|---|
| Elimina condicionales | Adiós a `if/else` gigantes |
| Open/Closed | Nuevos algoritmos sin modificar los existentes |
| Testabilidad | Cada estrategia se prueba por separado |

---

## 5.2 Observer

### ¿A qué corresponde?
Define una relación **uno-a-muchos**: cuando un objeto (el sujeto) cambia de estado, todos sus **observadores** son notificados automáticamente. En Spring se implementa con el sistema de **eventos**.

### Ejemplo de código

```java
// 1. El evento (un record inmutable es perfecto)
public record UsuarioRegistradoEvent(Long usuarioId, String email, String nombre) {}
```

```java
import org.springframework.context.ApplicationEventPublisher;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

// 2. El publicador (sujeto)
@Service
public class RegistroService {

    private final ApplicationEventPublisher publisher;

    public RegistroService(ApplicationEventPublisher publisher) {
        this.publisher = publisher;
    }

    @Transactional
    public void registrar(String nombre, String email) {
        Long nuevoId = 1L; // usuarioRepository.save(...).getId();

        // El servicio NO sabe quién escucha: solo anuncia lo que pasó
        publisher.publishEvent(new UsuarioRegistradoEvent(nuevoId, email, nombre));
    }
}
```

```java
import org.springframework.context.event.EventListener;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Component;
import org.springframework.transaction.event.TransactionPhase;
import org.springframework.transaction.event.TransactionalEventListener;

// 3. Los observadores (listeners): independientes entre sí
@Component
public class BienvenidaEmailListener {

    // Se ejecuta SOLO si la transacción hizo commit (evita enviar email si hubo rollback)
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    @Async // no bloquea al usuario mientras se envía el correo
    public void enviarBienvenida(UsuarioRegistradoEvent e) {
        System.out.println("📧 Bienvenido " + e.nombre() + " -> " + e.email());
    }
}

@Component
public class AuditoriaListener {

    @EventListener
    public void registrarAuditoria(UsuarioRegistradoEvent e) {
        System.out.println("📝 Auditoría: nuevo usuario " + e.usuarioId());
    }
}

@Component
public class MetricasListener {

    @EventListener
    public void incrementarContador(UsuarioRegistradoEvent e) {
        System.out.println("📊 +1 registro");
    }
}
```

```java
// Habilitar @Async en la aplicación
@SpringBootApplication
@EnableAsync
public class MiApplication {
    public static void main(String[] args) {
        SpringApplication.run(MiApplication.class, args);
    }
}
```

**Explicación:**
- `RegistroService` no depende de email, auditoría ni métricas. Puedes agregar o quitar listeners sin tocarlo.
- `@TransactionalEventListener` es clave para no ejecutar efectos secundarios si la transacción falla.
- `@Async` ejecuta el listener en otro hilo.

### Casos de uso
- Enviar correos o notificaciones tras una acción.
- Auditoría, métricas e invalidación de cachés.
- Comunicación entre módulos de un monolito modular (Spring Modulith se basa en esto).
- Paso previo a migrar hacia mensajería (Kafka, RabbitMQ).

### Beneficios
| Beneficio | Descripción |
|---|---|
| Desacoplamiento total | Publicador y suscriptores no se conocen |
| Extensibilidad | Nuevas reacciones sin modificar el origen |
| Asincronía | Tareas lentas fuera del flujo principal |

---

## 5.3 Template Method

### ¿A qué corresponde?
Define el **esqueleto de un algoritmo** en una clase base y deja que las subclases implementen ciertos **pasos específicos** sin cambiar la estructura general.

### Ejemplo de código

```java
import java.util.List;

// Clase base con el algoritmo fijo
public abstract class ImportadorArchivo<T> {

    // Método plantilla: "final" para que nadie altere el orden de los pasos
    public final ResultadoImportacion importar(byte[] contenido) {
        List<String> lineas = leer(contenido);                 // paso común
        List<T> registros = lineas.stream()
                .map(this::parsear)                            // paso variable
                .filter(this::esValido)                        // paso variable
                .toList();
        guardar(registros);                                    // paso variable
        despuesDeImportar(registros.size());                   // hook opcional
        return new ResultadoImportacion(lineas.size(), registros.size());
    }

    // Paso común implementado en la base
    private List<String> leer(byte[] contenido) {
        return new String(contenido).lines().skip(1).toList(); // omite encabezado
    }

    // Pasos que cada subclase DEBE implementar
    protected abstract T parsear(String linea);
    protected abstract boolean esValido(T registro);
    protected abstract void guardar(List<T> registros);

    // Hook: implementación vacía que las subclases PUEDEN sobrescribir
    protected void despuesDeImportar(int total) {}
}

public record ResultadoImportacion(int leidos, int importados) {}
```

```java
import org.springframework.stereotype.Component;
import java.math.BigDecimal;
import java.util.List;

public record Producto(String sku, String nombre, BigDecimal precio) {}

@Component
public class ImportadorProductosCsv extends ImportadorArchivo<Producto> {

    @Override
    protected Producto parsear(String linea) {
        String[] c = linea.split(",");
        return new Producto(c[0].trim(), c[1].trim(), new BigDecimal(c[2].trim()));
    }

    @Override
    protected boolean esValido(Producto p) {
        return p.precio().compareTo(BigDecimal.ZERO) > 0;
    }

    @Override
    protected void guardar(List<Producto> productos) {
        // productoRepository.saveAll(...)
        System.out.println("Guardando " + productos.size() + " productos");
    }

    @Override
    protected void despuesDeImportar(int total) {
        System.out.println("✅ Importación de productos terminada: " + total);
    }
}
```

**Explicación:**
- El flujo (leer → parsear → validar → guardar) está garantizado en todas las implementaciones.
- Spring lo usa en `JdbcTemplate`, `RestTemplate`, `TransactionTemplate` y `OncePerRequestFilter` (tú implementas `doFilterInternal`).

### Casos de uso
- Importadores/exportadores de archivos.
- Procesos batch con pasos fijos.
- Filtros HTTP (`OncePerRequestFilter`).
- Generación de documentos con estructura común (encabezado, cuerpo, pie).

### Beneficios
| Beneficio | Descripción |
|---|---|
| Reutilización | La lógica común vive una sola vez |
| Consistencia | Todas las variantes siguen el mismo flujo |
| Puntos de extensión claros | Solo se sobrescribe lo necesario |

---

## 5.4 Chain of Responsibility

### ¿A qué corresponde?
Pasa una petición a lo largo de una **cadena de manejadores**. Cada manejador decide si la procesa, la rechaza o la pasa al siguiente. Spring Security es literalmente una cadena de filtros (`SecurityFilterChain`).

### Ejemplo de código

```java
import java.math.BigDecimal;

public record SolicitudCredito(String clienteId, BigDecimal monto, int scoreBuro, BigDecimal ingresoMensual) {}

public record Resultado(boolean aprobado, String motivo) {
    public static Resultado continuar() { return new Resultado(true, "OK"); }
    public static Resultado rechazar(String motivo) { return new Resultado(false, motivo); }
}

// Cada validación de la cadena
public interface ValidadorCredito {
    Resultado validar(SolicitudCredito s);
}
```

```java
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;
import java.math.BigDecimal;

@Component @Order(1)
public class ValidadorMontoMinimo implements ValidadorCredito {
    public Resultado validar(SolicitudCredito s) {
        return s.monto().compareTo(new BigDecimal("1000")) < 0
                ? Resultado.rechazar("Monto mínimo: $1,000")
                : Resultado.continuar();
    }
}

@Component @Order(2)
public class ValidadorScore implements ValidadorCredito {
    public Resultado validar(SolicitudCredito s) {
        return s.scoreBuro() < 600
                ? Resultado.rechazar("Score de buró insuficiente")
                : Resultado.continuar();
    }
}

@Component @Order(3)
public class ValidadorCapacidadPago implements ValidadorCredito {
    public Resultado validar(SolicitudCredito s) {
        BigDecimal limite = s.ingresoMensual().multiply(BigDecimal.valueOf(10));
        return s.monto().compareTo(limite) > 0
                ? Resultado.rechazar("El monto excede 10 veces el ingreso mensual")
                : Resultado.continuar();
    }
}
```

```java
import org.springframework.stereotype.Service;
import java.util.List;

@Service
public class EvaluacionCreditoService {

    private final List<ValidadorCredito> cadena; // Spring respeta el @Order

    public EvaluacionCreditoService(List<ValidadorCredito> cadena) {
        this.cadena = cadena;
    }

    public Resultado evaluar(SolicitudCredito solicitud) {
        for (ValidadorCredito validador : cadena) {
            Resultado r = validador.validar(solicitud);
            if (!r.aprobado()) {
                return r; // corta la cadena en el primer rechazo
            }
        }
        return new Resultado(true, "Crédito aprobado 🎉");
    }
}
```

**Explicación:**
- `@Order` define la posición de cada eslabón. Las validaciones más baratas deberían ir primero.
- Agregar una validación nueva = una clase nueva con `@Component` y `@Order`.

### Casos de uso
- Validaciones de negocio encadenadas.
- Filtros HTTP y de seguridad (JWT, CORS, rate limiting).
- Pipelines de procesamiento (sanitizar → enriquecer → persistir).
- Flujos de aprobación (supervisor → gerente → director).

### Beneficios
| Beneficio | Descripción |
|---|---|
| Desacoplamiento | El emisor no sabe quién procesa la petición |
| Configurabilidad | Se reordenan o agregan eslabones fácilmente |
| Responsabilidad única | Cada eslabón valida una sola regla |

---

## 5.5 Command

### ¿A qué corresponde?
Encapsula una **solicitud o acción como un objeto**, permitiendo parametrizarla, encolarla, registrarla, reintentarla o deshacerla.

### Ejemplo de código

```java
// El comando
public interface Comando {
    void ejecutar();
    default void deshacer() { throw new UnsupportedOperationException("No se puede deshacer"); }
    String descripcion();
}
```

```java
import java.math.BigDecimal;

// Comandos concretos
public class DepositarComando implements Comando {
    private final Cuenta cuenta;
    private final BigDecimal monto;

    public DepositarComando(Cuenta cuenta, BigDecimal monto) {
        this.cuenta = cuenta;
        this.monto = monto;
    }

    public void ejecutar() { cuenta.depositar(monto); }
    public void deshacer() { cuenta.retirar(monto); }
    public String descripcion() { return "Depositar " + monto; }
}

public class Cuenta {
    private BigDecimal saldo = BigDecimal.ZERO;
    public void depositar(BigDecimal m) { saldo = saldo.add(m); }
    public void retirar(BigDecimal m) { saldo = saldo.subtract(m); }
    public BigDecimal getSaldo() { return saldo; }
}
```

```java
import org.springframework.stereotype.Component;
import java.util.ArrayDeque;
import java.util.Deque;

// El invocador: ejecuta, registra historial y permite deshacer
@Component
public class InvocadorComandos {

    private final Deque<Comando> historial = new ArrayDeque<>();

    public void ejecutar(Comando comando) {
        comando.ejecutar();
        historial.push(comando);
        System.out.println("▶ " + comando.descripcion());
    }

    public void deshacerUltimo() {
        if (!historial.isEmpty()) {
            Comando c = historial.pop();
            c.deshacer();
            System.out.println("↩ Deshecho: " + c.descripcion());
        }
    }
}
```

**Explicación:**
- La acción se convierte en un objeto que puede **guardarse, reintentarse o revertirse**.
- En Spring aparece con `Runnable`/`Callable` enviados a un `TaskExecutor`, y en arquitecturas CQRS (Command Query Responsibility Segregation).

### Casos de uso
- Funcionalidad de deshacer/rehacer.
- Colas de tareas y jobs programados.
- CQRS: separar comandos (escritura) de consultas (lectura).
- Registro de operaciones para auditoría o reintentos.

### Beneficios
| Beneficio | Descripción |
|---|---|
| Deshacer/rehacer | Cada acción sabe cómo revertirse |
| Encolamiento | Acciones diferidas o en segundo plano |
| Trazabilidad | Historial completo de operaciones |

---

# 6. Patrones de arquitectura / empresariales

## 6.1 Dependency Injection

### ¿A qué corresponde?
En lugar de que una clase **cree sus dependencias** con `new`, estas le son **proporcionadas desde fuera** (inyectadas). Es la base de la **Inversión de Control (IoC)** y el corazón de Spring.

### Ejemplo de código

**❌ Sin DI (acoplamiento fuerte):**

```java
public class OrdenService {
    private final OrdenRepository repo = new OrdenRepositoryMySql(); // atado a MySQL
    private final EmailService email = new EmailServiceSmtp();       // imposible de mockear
}
```

**✅ Con DI por constructor (forma recomendada):**

```java
import org.springframework.stereotype.Service;

@Service
public class OrdenService {

    private final OrdenRepository repo;
    private final EmailService email;

    // Con un único constructor, @Autowired es opcional
    public OrdenService(OrdenRepository repo, EmailService email) {
        this.repo = repo;
        this.email = email;
    }
}
```

**Con Lombok:**

```java
import lombok.RequiredArgsConstructor;

@Service
@RequiredArgsConstructor // genera el constructor con todos los campos final
public class OrdenService {
    private final OrdenRepository repo;
    private final EmailService email;
}
```

**Elegir entre varias implementaciones:**

```java
@Service
public class ReporteService {

    private final Exportador exportador;

    // @Qualifier indica qué bean usar cuando hay varios del mismo tipo
    public ReporteService(@Qualifier("exportadorPdf") Exportador exportador) {
        this.exportador = exportador;
    }
}
```

### Comparativa de tipos de inyección

| Tipo | Ejemplo | Recomendado | Motivo |
|---|---|---|---|
| Constructor | `public X(Dep d)` | ✅ Sí | Campos `final`, inmutables, fácil de testear |
| Setter | `@Autowired setDep(Dep d)` | ⚠️ Solo opcionales | Permite dependencias opcionales |
| Campo | `@Autowired private Dep d;` | ❌ No | Oculta dependencias, difícil de testear sin Spring |

**Test sin levantar Spring (gracias a DI por constructor):**

```java
import static org.mockito.Mockito.*;

class OrdenServiceTest {

    @Test
    void creaOrden() {
        OrdenRepository repoMock = mock(OrdenRepository.class);
        EmailService emailMock = mock(EmailService.class);

        OrdenService service = new OrdenService(repoMock, emailMock); // inyección manual

        // ... asserts y verify
    }
}
```

### Casos de uso
- Prácticamente en todas las clases de una aplicación Spring.
- Cambiar implementaciones por perfil (`@Profile("dev")` vs `@Profile("prod")`).
- Tests unitarios con mocks.

### Beneficios
| Beneficio | Descripción |
|---|---|
| Bajo acoplamiento | Dependes de interfaces, no de implementaciones |
| Testabilidad | Dependencias fácilmente reemplazables por mocks |
| Configuración centralizada | El contenedor gestiona el ciclo de vida |

---

## 6.2 Repository

### ¿A qué corresponde?
Media entre la lógica de negocio y la capa de datos, ofreciendo una **interfaz tipo colección** para acceder a las entidades. La lógica de negocio no sabe si los datos vienen de MySQL, PostgreSQL, MongoDB o una API.

### Ejemplo de código

```java
import jakarta.persistence.*;
import java.math.BigDecimal;
import java.time.LocalDateTime;

@Entity
@Table(name = "productos")
public class Producto {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String nombre;
    private String categoria;
    private BigDecimal precio;
    private boolean activo;
    private LocalDateTime creadoEn;
    // getters y setters
}
```

```java
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Modifying;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import java.math.BigDecimal;
import java.util.List;
import java.util.Optional;

public interface ProductoRepository extends JpaRepository<Producto, Long> {

    // 1. Query derivada del nombre del método (Spring genera el SQL)
    List<Producto> findByCategoriaAndActivoTrue(String categoria);

    Optional<Producto> findByNombreIgnoreCase(String nombre);

    boolean existsByNombre(String nombre);

    // 2. Con paginación
    Page<Producto> findByPrecioBetween(BigDecimal min, BigDecimal max, Pageable pageable);

    // 3. JPQL personalizada
    @Query("SELECT p FROM Producto p WHERE p.precio > :precio ORDER BY p.precio DESC")
    List<Producto> buscarCaros(@Param("precio") BigDecimal precio);

    // 4. Operación de modificación
    @Modifying
    @Query("UPDATE Producto p SET p.activo = false WHERE p.categoria = :categoria")
    int desactivarPorCategoria(@Param("categoria") String categoria);
}
```

**Uso:**

```java
@Service
@RequiredArgsConstructor
public class CatalogoService {

    private final ProductoRepository productos;

    public Page<Producto> buscarPorRango(BigDecimal min, BigDecimal max, int pagina) {
        return productos.findByPrecioBetween(min, max, PageRequest.of(pagina, 20, Sort.by("precio")));
    }
}
```

**Explicación:**
- Solo declaras la **interfaz**; Spring Data genera la implementación en tiempo de ejecución (¡usando un Proxy!).
- `JpaRepository` ya incluye `save`, `findById`, `findAll`, `deleteById`, `count`, etc.

### Casos de uso
- Todo acceso a base de datos en una app Spring.
- Aislar la persistencia para poder cambiar de motor de BD.
- Consultas paginadas y ordenadas.

### Beneficios
| Beneficio | Descripción |
|---|---|
| Menos código | CRUD completo sin implementar nada |
| Abstracción | El negocio no depende del motor de datos |
| Testabilidad | Fácil de mockear o probar con `@DataJpaTest` |

---

## 6.3 DTO (Data Transfer Object)

### ¿A qué corresponde?
Objeto simple que **transporta datos entre capas o sistemas** (ej. entre el backend y el frontend). Evita exponer directamente las entidades JPA.

### Ejemplo de código

```java
import jakarta.validation.constraints.*;

// DTO de entrada con validaciones (records de Java 17 son ideales)
public record CrearUsuarioRequest(
        @NotBlank(message = "El nombre es obligatorio")
        String nombre,

        @Email(message = "Email inválido")
        @NotBlank
        String email,

        @Size(min = 8, message = "La contraseña debe tener al menos 8 caracteres")
        String password
) {}

// DTO de salida: NO incluye la contraseña ni campos internos
public record UsuarioResponse(Long id, String nombre, String email) {}
```

```java
// Mapper manual (alternativa: MapStruct)
@Component
public class UsuarioMapper {

    public Usuario toEntity(CrearUsuarioRequest req, String passwordHash) {
        Usuario u = new Usuario();
        u.setNombre(req.nombre());
        u.setEmail(req.email());
        u.setPassword(passwordHash);
        return u;
    }

    public UsuarioResponse toResponse(Usuario u) {
        return new UsuarioResponse(u.getId(), u.getNombre(), u.getEmail());
    }
}
```

**Con MapStruct (genera el mapper en compilación):**

```java
import org.mapstruct.Mapper;

@Mapper(componentModel = "spring")
public interface UsuarioMapStruct {
    UsuarioResponse toResponse(Usuario usuario);
    List<UsuarioResponse> toResponseList(List<Usuario> usuarios);
}
```

```java
@RestController
@RequestMapping("/api/usuarios")
@RequiredArgsConstructor
public class UsuarioController {

    private final UsuarioService service;

    @PostMapping
    public ResponseEntity<UsuarioResponse> crear(@Valid @RequestBody CrearUsuarioRequest req) {
        UsuarioResponse creado = service.crear(req);
        return ResponseEntity.status(HttpStatus.CREATED).body(creado);
    }
}
```

**Explicación:**
- `@Valid` activa las validaciones del DTO antes de llegar al servicio.
- Nunca se expone el `password` ni relaciones lazy que causarían `LazyInitializationException` o recursión infinita en JSON.

### Casos de uso
- Requests y responses de APIs REST.
- Proyecciones de consultas (solo los campos necesarios).
- Comunicación entre microservicios.

### Beneficios
| Beneficio | Descripción |
|---|---|
| Seguridad | No se exponen campos sensibles |
| Contrato estable | La API no cambia si cambia la entidad |
| Validación | Reglas de entrada declarativas |
| Rendimiento | Solo se envían los datos necesarios |

---

## 6.4 Service Layer

### ¿A qué corresponde?
Capa que **concentra la lógica de negocio** y coordina repositorios y otros servicios. Forma parte de la arquitectura en capas típica de Spring Boot:

```
Controller  →  Service  →  Repository  →  Base de datos
 (HTTP)       (negocio)     (datos)
```

### Ejemplo de código

```java
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.security.crypto.password.PasswordEncoder;

@Service
@RequiredArgsConstructor
@Transactional(readOnly = true) // por defecto, solo lectura (optimiza consultas)
public class UsuarioService {

    private final UsuarioRepository repo;
    private final UsuarioMapper mapper;
    private final PasswordEncoder passwordEncoder;

    public UsuarioResponse obtener(Long id) {
        return repo.findById(id)
                .map(mapper::toResponse)
                .orElseThrow(() -> new RecursoNoEncontradoException("Usuario " + id + " no existe"));
    }

    @Transactional // sobrescribe: este método sí escribe
    public UsuarioResponse crear(CrearUsuarioRequest req) {
        if (repo.existsByEmail(req.email())) {
            throw new ReglaNegocioException("El email ya está registrado");
        }
        Usuario guardado = repo.save(mapper.toEntity(req, passwordEncoder.encode(req.password())));
        return mapper.toResponse(guardado);
    }
}
```

```java
// Manejo centralizado de excepciones de negocio
@RestControllerAdvice
public class ManejadorGlobalExcepciones {

    @ExceptionHandler(RecursoNoEncontradoException.class)
    public ProblemDetail noEncontrado(RecursoNoEncontradoException ex) {
        return ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, ex.getMessage());
    }

    @ExceptionHandler(ReglaNegocioException.class)
    public ProblemDetail reglaNegocio(ReglaNegocioException ex) {
        return ProblemDetail.forStatusAndDetail(HttpStatus.CONFLICT, ex.getMessage());
    }
}
```

**Explicación:**
- El controller solo traduce HTTP ↔ Java; el servicio decide **qué** se permite hacer.
- `@Transactional(readOnly = true)` a nivel de clase y `@Transactional` en métodos de escritura es una práctica muy común.

### Casos de uso
- Toda regla de negocio: validaciones, cálculos, orquestación.
- Punto de definición de límites transaccionales.

### Beneficios
| Beneficio | Descripción |
|---|---|
| Separación de capas | Controllers delgados, negocio reutilizable |
| Reutilización | La misma lógica sirve para REST, jobs o mensajería |
| Transacciones claras | Límites transaccionales bien definidos |

---

## 7. Cómo elegir el patrón correcto

| Si tienes este problema… | Usa… |
|---|---|
| Un `if/else` o `switch` que crece con cada nueva regla | **Strategy** (+ Factory) |
| Un objeto con muchos parámetros opcionales | **Builder** |
| Necesitas integrar una API externa con otra firma | **Adapter** |
| Lógica repetida en muchos métodos (logs, métricas, seguridad) | **Proxy / AOP** |
| Una acción debe disparar varias reacciones independientes | **Observer** (eventos) |
| Varios procesos con los mismos pasos pero detalles distintos | **Template Method** |
| Validaciones secuenciales que pueden cortar el flujo | **Chain of Responsibility** |
| Un controller que llama a 5 servicios en orden | **Facade** |
| Agregar caché o reintentos sin tocar la clase original | **Decorator** |
| Necesitas deshacer, encolar o auditar acciones | **Command** |
| Elegir implementación según un parámetro en tiempo de ejecución | **Factory** |
| Exponer datos por API sin revelar la entidad | **DTO** |

---

## 8. Antipatrones comunes

| Antipatrón | Síntoma | Solución |
|---|---|---|
| **God Service** | Un `@Service` de 2,000 líneas que hace de todo | Dividir por responsabilidad, usar Facade + servicios pequeños |
| **Exponer entidades JPA** | `@RestController` devuelve `@Entity` directamente | Usar DTOs |
| **Inyección por campo** | `@Autowired private X x;` en todas partes | Inyección por constructor |
| **Self-invocation con proxies** | `@Transactional` o `@Cacheable` "no funciona" | Mover el método a otro bean |
| **Estado mutable en singletons** | Variables de instancia modificadas por cada request | Beans sin estado o estructuras thread-safe |
| **Sobreingeniería** | Aplicar 5 patrones a un CRUD simple | Usar patrones solo cuando resuelven un problema real |
| **Lógica en el controller** | Validaciones y cálculos dentro del `@RestController` | Mover a la capa de servicio |

---

> **Regla de oro:** los patrones son herramientas, no objetivos. Aplícalos cuando el código **empiece a doler** (duplicación, condicionales gigantes, acoplamiento), no por anticipado.
