# six flags

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.5</version>
        <relativePath/>
    </parent>

    <groupId>com.sixflags</groupId>
    <artifactId>park-manager</artifactId>
    <version>1.0.0</version>
    <name>ParkManager</name>
    <description>Sistema de venta de boletos y administracion de descuentos para Six Flags</description>

    <properties>
        <java.version>17</java.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
target/
!.mvn/wrapper/maven-wrapper.jar
!**/src/main/**/target/
!**/src/test/**/target/
.idea/
*.iws
*.iml
*.ipr
out/
.apt_generated
.classpath
.factorypath
.project
.settings
.springBeans
.sts4-cache
bin/
.vscode/
.DS_Store
Thumbs.db
*.log
spring.application.name=park-manager
server.port=8080
package com.sixflags.parkmanager;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class ParkManagerApplication {

    public static void main(String[] args) {
        SpringApplication.run(ParkManagerApplication.class, args);
    }
}
package com.sixflags.parkmanager.exception;

public abstract class NegocioException extends RuntimeException {

    private final String codigo;

    protected NegocioException(String codigo, String mensaje) {
        super(mensaje);
        this.codigo = codigo;
    }

    public String getCodigo() {
        return codigo;
    }
}
package com.sixflags.parkmanager.exception;

public class DescuentoInvalidoException extends NegocioException {

    public static final String CODIGO = "DESCUENTO_INVALIDO";

    public DescuentoInvalidoException(String mensaje) {
        super(CODIGO, mensaje);
    }
}
package com.sixflags.parkmanager.exception;

public class TipoBoletoInvalidoException extends NegocioException {

    public static final String CODIGO = "TIPO_BOLETO_INVALIDO";

    public TipoBoletoInvalidoException(String mensaje) {
        super(CODIGO, mensaje);
    }
}
package com.sixflags.parkmanager.exception;

public class SolicitudInvalidaException extends NegocioException {

    public static final String CODIGO = "SOLICITUD_INVALIDA";

    public SolicitudInvalidaException(String mensaje) {
        super(CODIGO, mensaje);
    }
}
package com.sixflags.parkmanager.model;

import com.sixflags.parkmanager.exception.TipoBoletoInvalidoException;

public enum TipoBoleto {
    GENERAL,
    PASE_ANUAL,
    ALUMNO,
    PROFESOR;

    public static TipoBoleto desdeTexto(String valor) {
        if (valor == null || valor.isBlank()) {
            throw new TipoBoletoInvalidoException(
                    "El tipo de boleto debe ser GENERAL, PASE_ANUAL, ALUMNO o PROFESOR");
        }
        try {
            return TipoBoleto.valueOf(valor.trim());
        } catch (IllegalArgumentException ex) {
            throw new TipoBoletoInvalidoException(
                    "Tipo de boleto inválido: '" + valor + "'. Valores permitidos: GENERAL, PASE_ANUAL, ALUMNO, PROFESOR");
        }
    }
}
package com.sixflags.parkmanager.model;

import com.sixflags.parkmanager.exception.CantidadInvalidaException;
import com.sixflags.parkmanager.exception.DescuentoInvalidoException;

import java.math.BigDecimal;
import java.math.RoundingMode;
import java.util.Objects;

public abstract class Boleto {

    public static final int DESCUENTO_MINIMO = 0;
    public static final int DESCUENTO_MAXIMO = 100;
    public static final int CANTIDAD_MINIMA = 1;
    public static final int CANTIDAD_MAXIMA = 10;

    private final TipoBoleto tipo;
    private int porcentajeDescuento;

    protected Boleto(TipoBoleto tipo) {
        this.tipo = Objects.requireNonNull(tipo, "El tipo de boleto no puede ser nulo");
        this.porcentajeDescuento = DESCUENTO_MINIMO;
    }

    public abstract BigDecimal obtenerPrecioBase();

    public TipoBoleto getTipo() {
        return tipo;
    }

    public int getPorcentajeDescuento() {
        return porcentajeDescuento;
    }

    public void configurarDescuento(int porcentaje) {
        if (porcentaje < DESCUENTO_MINIMO || porcentaje > DESCUENTO_MAXIMO) {
            throw new DescuentoInvalidoException("El porcentaje debe estar entre 0 y 100");
        }
        this.porcentajeDescuento = porcentaje;
    }

    public void removerDescuento() {
        this.porcentajeDescuento = DESCUENTO_MINIMO;
    }

    public BigDecimal calcularPrecioUnitarioFinal() {
        BigDecimal precioBase = obtenerPrecioBase().setScale(2, RoundingMode.HALF_UP);
        BigDecimal factorRestante = BigDecimal.valueOf(DESCUENTO_MAXIMO - porcentajeDescuento)
                .divide(BigDecimal.valueOf(DESCUENTO_MAXIMO), 6, RoundingMode.HALF_UP);
        return precioBase.multiply(factorRestante).setScale(2, RoundingMode.HALF_UP);
    }

    public BigDecimal calcularTotal(int cantidad) {
        validarCantidad(cantidad);
        return calcularPrecioUnitarioFinal()
                .multiply(BigDecimal.valueOf(cantidad))
                .setScale(2, RoundingMode.HALF_UP);
    }

    public static void validarCantidad(int cantidad) {
        if (cantidad < CANTIDAD_MINIMA || cantidad > CANTIDAD_MAXIMA) {
            throw new CantidadInvalidaException("La cantidad de boletos debe estar entre 1 y 10");
        }
    }
}
package com.sixflags.parkmanager.model;

import java.math.BigDecimal;

public class BoletoGeneral extends Boleto {

    private static final BigDecimal TARIFA_BASE = new BigDecimal("900.00");

    public BoletoGeneral() {
        super(TipoBoleto.GENERAL);
    }

    @Override
    public BigDecimal obtenerPrecioBase() {
        return TARIFA_BASE;
    }
}
package com.sixflags.parkmanager.model;

import java.math.BigDecimal;

public class BoletoPaseAnual extends Boleto {

    private static final BigDecimal TARIFA_BASE = new BigDecimal("2400.00");

    public BoletoPaseAnual() {
        super(TipoBoleto.PASE_ANUAL);
    }

    @Override
    public BigDecimal obtenerPrecioBase() {
        return TARIFA_BASE;
    }
}
package com.sixflags.parkmanager.model;

import java.math.BigDecimal;

public class BoletoAlumno extends Boleto {

    private static final BigDecimal TARIFA_BASE = new BigDecimal("650.00");

    public BoletoAlumno() {
        super(TipoBoleto.ALUMNO);
    }

    @Override
    public BigDecimal obtenerPrecioBase() {
        return TARIFA_BASE;
    }
}
package com.sixflags.parkmanager.model;

import java.math.BigDecimal;

public class BoletoProfesor extends Boleto {

    private static final BigDecimal TARIFA_BASE = new BigDecimal("700.00");

    public BoletoProfesor() {
        super(TipoBoleto.PROFESOR);
    }

    @Override
    public BigDecimal obtenerPrecioBase() {
        return TARIFA_BASE;
    }
}
package com.sixflags.parkmanager.model;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.Objects;

public final class Compra {

    private final TipoBoleto tipo;
    private final int cantidad;
    private final BigDecimal precioBase;
    private final int descuentoAplicado;
    private final BigDecimal precioUnitarioFinal;
    private final BigDecimal total;
    private final LocalDateTime fechaRegistro;

    public Compra(
            TipoBoleto tipo,
            int cantidad,
            BigDecimal precioBase,
            int descuentoAplicado,
            BigDecimal precioUnitarioFinal,
            BigDecimal total,
            LocalDateTime fechaRegistro) {
        this.tipo = Objects.requireNonNull(tipo, "El tipo no puede ser nulo");
        this.cantidad = cantidad;
        this.precioBase = Objects.requireNonNull(precioBase, "El precio base no puede ser nulo");
        this.descuentoAplicado = descuentoAplicado;
        this.precioUnitarioFinal = Objects.requireNonNull(precioUnitarioFinal, "El precio unitario final no puede ser nulo");
        this.total = Objects.requireNonNull(total, "El total no puede ser nulo");
        this.fechaRegistro = Objects.requireNonNull(fechaRegistro, "La fecha de registro no puede ser nula");
    }

    public TipoBoleto getTipo() {
        return tipo;
    }

    public int getCantidad() {
        return cantidad;
    }

    public BigDecimal getPrecioBase() {
        return precioBase;
    }

    public int getDescuentoAplicado() {
        return descuentoAplicado;
    }

    public BigDecimal getPrecioUnitarioFinal() {
        return precioUnitarioFinal;
    }

    public BigDecimal getTotal() {
        return total;
    }

    public LocalDateTime getFechaRegistro() {
        return fechaRegistro;
    }
}
package com.sixflags.parkmanager.dto;

public class ConfigurarDescuentoRequest {

    private Integer porcentaje;

    public ConfigurarDescuentoRequest() {
    }

    public ConfigurarDescuentoRequest(Integer porcentaje) {
        this.porcentaje = porcentaje;
    }

    public Integer getPorcentaje() {
        return porcentaje;
    }

    public void setPorcentaje(Integer porcentaje) {
        this.porcentaje = porcentaje;
    }
}
package com.sixflags.parkmanager.dto;

public class DescuentoResponse {

    private String tipo;
    private int porcentaje;
    private String mensaje;

    public DescuentoResponse() {
    }

    public DescuentoResponse(String tipo, int porcentaje, String mensaje) {
        this.tipo = tipo;
        this.porcentaje = porcentaje;
        this.mensaje = mensaje;
    }

    public String getTipo() {
        return tipo;
    }

    public void setTipo(String tipo) {
        this.tipo = tipo;
    }

    public int getPorcentaje() {
        return porcentaje;
    }

    public void setPorcentaje(int porcentaje) {
        this.porcentaje = porcentaje;
    }

    public String getMensaje() {
        return mensaje;
    }

    public void setMensaje(String mensaje) {
        this.mensaje = mensaje;
    }
}
package com.sixflags.parkmanager.dto;

public class RegistrarCompraRequest {

    private String tipo;
    private Integer cantidad;

    public RegistrarCompraRequest() {
    }

    public RegistrarCompraRequest(String tipo, Integer cantidad) {
        this.tipo = tipo;
        this.cantidad = cantidad;
    }

    public String getTipo() {
        return tipo;
    }

    public void setTipo(String tipo) {
        this.tipo = tipo;
    }

    public Integer getCantidad() {
        return cantidad;
    }

    public void setCantidad(Integer cantidad) {
        this.cantidad = cantidad;
    }
}
package com.sixflags.parkmanager.dto;

import java.math.BigDecimal;

public class CompraResponse {

    private String tipo;
    private int cantidad;
    private BigDecimal precioBase;
    private int descuentoAplicado;
    private BigDecimal precioUnitarioFinal;
    private BigDecimal total;
    private BigDecimal ingresosAcumulados;

    public CompraResponse() {
    }

    public CompraResponse(
            String tipo,
            int cantidad,
            BigDecimal precioBase,
            int descuentoAplicado,
            BigDecimal precioUnitarioFinal,
            BigDecimal total,
            BigDecimal ingresosAcumulados) {
        this.tipo = tipo;
        this.cantidad = cantidad;
        this.precioBase = precioBase;
        this.descuentoAplicado = descuentoAplicado;
        this.precioUnitarioFinal = precioUnitarioFinal;
        this.total = total;
        this.ingresosAcumulados = ingresosAcumulados;
    }

    public String getTipo() {
        return tipo;
    }

    public void setTipo(String tipo) {
        this.tipo = tipo;
    }

    public int getCantidad() {
        return cantidad;
    }

    public void setCantidad(int cantidad) {
        this.cantidad = cantidad;
    }

    public BigDecimal getPrecioBase() {
        return precioBase;
    }

    public void setPrecioBase(BigDecimal precioBase) {
        this.precioBase = precioBase;
    }

    public int getDescuentoAplicado() {
        return descuentoAplicado;
    }

    public void setDescuentoAplicado(int descuentoAplicado) {
        this.descuentoAplicado = descuentoAplicado;
    }

    public BigDecimal getPrecioUnitarioFinal() {
        return precioUnitarioFinal;
    }

    public void setPrecioUnitarioFinal(BigDecimal precioUnitarioFinal) {
        this.precioUnitarioFinal = precioUnitarioFinal;
    }

    public BigDecimal getTotal() {
        return total;
    }

    public void setTotal(BigDecimal total) {
        this.total = total;
    }

    public BigDecimal getIngresosAcumulados() {
        return ingresosAcumulados;
    }

    public void setIngresosAcumulados(BigDecimal ingresosAcumulados) {
        this.ingresosAcumulados = ingresosAcumulados;
    }
}
package com.sixflags.parkmanager.dto;

public class ErrorResponse {

    private String codigo;
    private String mensaje;

    public ErrorResponse() {
    }

    public ErrorResponse(String codigo, String mensaje) {
        this.codigo = codigo;
        this.mensaje = mensaje;
    }

    public String getCodigo() {
        return codigo;
    }

    public void setCodigo(String codigo) {
        this.codigo = codigo;
    }

    public String getMensaje() {
        return mensaje;
    }

    public void setMensaje(String mensaje) {
        this.mensaje = mensaje;
    }
}
package com.sixflags.parkmanager.service;

import com.sixflags.parkmanager.model.Boleto;
import com.sixflags.parkmanager.model.BoletoAlumno;
import com.sixflags.parkmanager.model.BoletoGeneral;
import com.sixflags.parkmanager.model.BoletoPaseAnual;
import com.sixflags.parkmanager.model.BoletoProfesor;
import com.sixflags.parkmanager.model.TipoBoleto;
import org.springframework.stereotype.Service;

import java.util.Collections;
import java.util.EnumMap;
import java.util.Map;

@Service
public class CatalogoBoletosService {

    private final Map<TipoBoleto, Boleto> boletosPorModalidad;

    public CatalogoBoletosService() {
        EnumMap<TipoBoleto, Boleto> mapa = new EnumMap<>(TipoBoleto.class);
        mapa.put(TipoBoleto.GENERAL, new BoletoGeneral());
        mapa.put(TipoBoleto.PASE_ANUAL, new BoletoPaseAnual());
        mapa.put(TipoBoleto.ALUMNO, new BoletoAlumno());
        mapa.put(TipoBoleto.PROFESOR, new BoletoProfesor());
        this.boletosPorModalidad = Collections.unmodifiableMap(mapa);
    }

    public Boleto obtenerBoleto(TipoBoleto tipo) {
        return boletosPorModalidad.get(tipo);
    }

    public Boleto obtenerBoleto(String tipoTexto) {
        TipoBoleto tipo = TipoBoleto.desdeTexto(tipoTexto);
        return obtenerBoleto(tipo);
    }
}
package com.sixflags.parkmanager.service;

import com.sixflags.parkmanager.dto.ConfigurarDescuentoRequest;
import com.sixflags.parkmanager.dto.DescuentoResponse;
import com.sixflags.parkmanager.exception.SolicitudInvalidaException;
import com.sixflags.parkmanager.model.Boleto;
import org.springframework.stereotype.Service;

@Service
public class DescuentoService {

    private final CatalogoBoletosService catalogoBoletosService;

    public DescuentoService(CatalogoBoletosService catalogoBoletosService) {
        this.catalogoBoletosService = catalogoBoletosService;
    }

    public synchronized DescuentoResponse configurarDescuento(String tipoTexto, ConfigurarDescuentoRequest request) {
        Boleto boleto = catalogoBoletosService.obtenerBoleto(tipoTexto);
        if (request == null || request.getPorcentaje() == null) {
            throw new SolicitudInvalidaException("El campo 'porcentaje' es obligatorio");
        }
        boleto.configurarDescuento(request.getPorcentaje());
        return new DescuentoResponse(
                boleto.getTipo().name(),
                boleto.getPorcentajeDescuento(),
                "Descuento configurado"
        );
    }

    public synchronized DescuentoResponse removerDescuento(String tipoTexto) {
        Boleto boleto = catalogoBoletosService.obtenerBoleto(tipoTexto);
        boleto.removerDescuento();
        return new DescuentoResponse(
                boleto.getTipo().name(),
                boleto.getPorcentajeDescuento(),
                "Descuento removido"
        );
    }
}
package com.sixflags.parkmanager.service;

import com.sixflags.parkmanager.dto.CompraResponse;
import com.sixflags.parkmanager.dto.RegistrarCompraRequest;
import com.sixflags.parkmanager.exception.SolicitudInvalidaException;
import com.sixflags.parkmanager.model.Boleto;
import com.sixflags.parkmanager.model.Compra;
import org.springframework.stereotype.Service;

import java.math.BigDecimal;
import java.math.RoundingMode;
import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

@Service
public class CompraService {

    private final CatalogoBoletosService catalogoBoletosService;
    private final List<Compra> historialCompras;
    private int boletosAcumulados;
    private BigDecimal ingresosAcumulados;

    public CompraService(CatalogoBoletosService catalogoBoletosService) {
        this.catalogoBoletosService = catalogoBoletosService;
        this.historialCompras = new ArrayList<>();
        this.boletosAcumulados = 0;
        this.ingresosAcumulados = BigDecimal.ZERO.setScale(2, RoundingMode.HALF_UP);
    }

    public synchronized CompraResponse registrarCompra(RegistrarCompraRequest request) {
        if (request == null
                || request.getTipo() == null
                || request.getTipo().isBlank()
                || request.getCantidad() == null) {
            throw new SolicitudInvalidaException("Los campos 'tipo' y 'cantidad' son obligatorios");
        }

        Boleto boleto = catalogoBoletosService.obtenerBoleto(request.getTipo());

        int cantidad = request.getCantidad();
        BigDecimal precioBase = boleto.obtenerPrecioBase().setScale(2, RoundingMode.HALF_UP);
        int descuentoAplicado = boleto.getPorcentajeDescuento();
        BigDecimal precioUnitarioFinal = boleto.calcularPrecioUnitarioFinal();
        BigDecimal total = boleto.calcularTotal(cantidad);

        this.boletosAcumulados += cantidad;
        this.ingresosAcumulados = this.ingresosAcumulados.add(total).setScale(2, RoundingMode.HALF_UP);

        Compra compra = new Compra(
                boleto.getTipo(),
                cantidad,
                precioBase,
                descuentoAplicado,
                precioUnitarioFinal,
                total,
                LocalDateTime.now()
        );
        this.historialCompras.add(compra);

        return new CompraResponse(
                boleto.getTipo().name(),
                cantidad,
                precioBase,
                descuentoAplicado,
                precioUnitarioFinal,
                total,
                this.ingresosAcumulados
        );
    }

    public synchronized int getBoletosAcumulados() {
        return boletosAcumulados;
    }

    public synchronized BigDecimal getIngresosAcumulados() {
        return ingresosAcumulados;
    }

    public synchronized List<Compra> getHistorialCompras() {
        return Collections.unmodifiableList(new ArrayList<>(historialCompras));
    }
}
package com.sixflags.parkmanager.controller;

import com.sixflags.parkmanager.dto.ConfigurarDescuentoRequest;
import com.sixflags.parkmanager.dto.DescuentoResponse;
import com.sixflags.parkmanager.service.DescuentoService;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.DeleteMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PutMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/api/descuentos")
public class DescuentoController {

    private final DescuentoService descuentoService;

    public DescuentoController(DescuentoService descuentoService) {
        this.descuentoService = descuentoService;
    }

    @PutMapping("/{tipo}")
    public ResponseEntity<DescuentoResponse> configurarDescuento(
            @PathVariable String tipo,
            @RequestBody(required = false) ConfigurarDescuentoRequest request) {
        return ResponseEntity.ok(descuentoService.configurarDescuento(tipo, request));
    }

    @DeleteMapping("/{tipo}")
    public ResponseEntity<DescuentoResponse> removerDescuento(@PathVariable String tipo) {
        return ResponseEntity.ok(descuentoService.removerDescuento(tipo));
    }
}
package com.sixflags.parkmanager.controller;

import com.sixflags.parkmanager.dto.CompraResponse;
import com.sixflags.parkmanager.dto.RegistrarCompraRequest;
import com.sixflags.parkmanager.service.CompraService;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/api/compras")
public class CompraController {

    private final CompraService compraService;

    public CompraController(CompraService compraService) {
        this.compraService = compraService;
    }

    @PostMapping
    public ResponseEntity<CompraResponse> registrarCompra(
            @RequestBody(required = false) RegistrarCompraRequest request) {
        return ResponseEntity.ok(compraService.registrarCompra(request));
    }
}
package com.sixflags.parkmanager.controller;

import com.sixflags.parkmanager.dto.ErrorResponse;
import com.sixflags.parkmanager.exception.NegocioException;
import com.sixflags.parkmanager.exception.SolicitudInvalidaException;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.http.converter.HttpMessageNotReadableException;
import org.springframework.web.HttpMediaTypeNotSupportedException;
import org.springframework.web.bind.MethodArgumentTypeMismatchException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(NegocioException.class)
    public ResponseEntity<ErrorResponse> manejarExcepcionNegocio(NegocioException ex) {
        ErrorResponse error = new ErrorResponse(ex.getCodigo(), ex.getMessage());
        return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(error);
    }

    @ExceptionHandler({
            HttpMessageNotReadableException.class,
            HttpMediaTypeNotSupportedException.class,
            MethodArgumentTypeMismatchException.class
    })
    public ResponseEntity<ErrorResponse> manejarSolicitudMalFormada(Exception ex) {
        ErrorResponse error = new ErrorResponse(
                SolicitudInvalidaException.CODIGO,
                "La solicitud está mal formada o contiene tipos de datos inválidos"
        );
        return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(error);
    }
}
package com.sixflags.parkmanager.controller;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.http.MediaType;
import org.springframework.test.annotation.DirtiesContext;
import org.springframework.test.web.servlet.MockMvc;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.delete;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.put;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

@SpringBootTest
@AutoConfigureMockMvc
@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_EACH_TEST_METHOD)
class ParkManagerIntegrationTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    void flujoCompletoDescuentoCompraYRemocionIdempotente() throws Exception {
        mockMvc.perform(put("/api/descuentos/ALUMNO")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("{\"porcentaje\": 20}"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.tipo").value("ALUMNO"))
                .andExpect(jsonPath("$.porcentaje").value(20))
                .andExpect(jsonPath("$.mensaje").value("Descuento configurado"));

        mockMvc.perform(post("/api/compras")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("{\"tipo\": \"ALUMNO\", \"cantidad\": 3}"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.tipo").value("ALUMNO"))
                .andExpect(jsonPath("$.cantidad").value(3))
                .andExpect(jsonPath("$.precioBase").value(650.00))
                .andExpect(jsonPath("$.descuentoAplicado").value(20))
                .andExpect(jsonPath("$.precioUnitarioFinal").value(520.00))
                .andExpect(jsonPath("$.total").value(1560.00))
                .andExpect(jsonPath("$.ingresosAcumulados").value(1560.00));

        mockMvc.perform(delete("/api/descuentos/ALUMNO"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.tipo").value("ALUMNO"))
                .andExpect(jsonPath("$.porcentaje").value(0));

        mockMvc.perform(delete("/api/descuentos/ALUMNO"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.tipo").value("ALUMNO"))
                .andExpect(jsonPath("$.porcentaje").value(0));

        mockMvc.perform(post("/api/compras")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("{\"tipo\": \"ALUMNO\", \"cantidad\": 1}"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.precioBase").value(650.00))
                .andExpect(jsonPath("$.descuentoAplicado").value(0))
                .andExpect(jsonPath("$.precioUnitarioFinal").value(650.00))
                .andExpect(jsonPath("$.total").value(650.00))
                .andExpect(jsonPath("$.ingresosAcumulados").value(2210.00));
    }

    @Test
    void compraInvalidaNoDebeAlterarIngresosAcumulados() throws Exception {
        mockMvc.perform(post("/api/compras")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("{\"tipo\": \"GENERAL\", \"cantidad\": 1}"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.ingresosAcumulados").value(900.00));

        mockMvc.perform(post("/api/compras")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("{\"tipo\": \"GENERAL\", \"cantidad\": 11}"))
                .andExpect(status().isBadRequest())
                .andExpect(jsonPath("$.codigo").value("CANTIDAD_INVALIDA"));

        mockMvc.perform(post("/api/compras")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("{\"tipo\": \"PROFESOR\", \"cantidad\": 1}"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.ingresosAcumulados").value(1600.00));
    }

    @Test
    void debeResponderCodigosDeErrorEspecificadosParaSolicitudesInvalidas() throws Exception {
        mockMvc.perform(put("/api/descuentos/PASE_ANUAL")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("{\"porcentaje\": 120}"))
                .andExpect(status().isBadRequest())
                .andExpect(jsonPath("$.codigo").value("DESCUENTO_INVALIDO"));

        mockMvc.perform(put("/api/descuentos/VIP")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("{\"porcentaje\": 10}"))
                .andExpect(status().isBadRequest())
                .andExpect(jsonPath("$.codigo").value("TIPO_BOLETO_INVALIDO"));

        mockMvc.perform(post("/api/compras")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("{\"tipo\": \"GENERAL\"}"))
                .andExpect(status().isBadRequest())
                .andExpect(jsonPath("$.codigo").value("SOLICITUD_INVALIDA"));

        mockMvc.perform(post("/api/compras")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("{mal_formado"))
                .andExpect(status().isBadRequest())
                .andExpect(jsonPath("$.codigo").value("SOLICITUD_INVALIDA"));
    }
}


