# Análisis Forense, Tráfico Local e Ingeniería Inversa (CyberOps)

## 1. Información General

- **Institución:** Instituto Tecnológico de las Américas (ITLA), República Dominicana
- **Materia:** Seguridad de Redes
- **Práctica:** Reto CyberOps - Ingeniería Inversa y Análisis de Sockets
- **Estudiante:** Edwin De Paula (Vervin Daniel Figuereo Mariñez)
- **Matrícula:** 2024-2415
- **Enlace del Video:** [Insertar enlace de YouTube aquí]

---

## 2. Objetivos del Laboratorio

### Objetivo General
Analizar un binario de software comprometido u ofuscado (`WinFormsApp1.exe`) aplicando técnicas de ingeniería inversa estática/dinámica y auditoría de redes para comprender su flujo lógico, evadir sus contramedidas defensivas y extraer credenciales de acceso dinámicas validadas remotamente.

### Objetivos Específicos
- **Enumeración de Sockets:** Identificar servicios locales e interactuar con puertos dinámicos creados en tiempo de ejecución.
- **Evadir Contramedidas (Anti-Debugging):** Desarmar el encapsulamiento de hilos asíncronos (`async/await`) generado por el compilador .NET.
- **Análisis Forense de Memoria:** Volcar variables críticas directamente desde la memoria RAM para anular los señuelos criptográficos del binario.

---

## 3. Requisitos y Parámetros del Entorno

### Requisitos del Sistema
- **Sistema Operativo:** Windows 10/11 (Entorno de Análisis Controlado / Sandbox)
- **Herramientas de Reversing:** dnSpy (v6.1.8 o superior) con privilegios de depuración
- **Herramientas de Red:** Wireshark con soporte para capturas en interfaz Loopback

### Comandos de Enumeración e Interacción
```bash
# Paso 1: Localizar el PID y el puerto TCP dinámico en escucha
netstat -ano | findstr WinFormsApp1

# Paso 2: Conexión interactiva al servidor local de pistas (Decoy Server)
telnet 127.0.0.1 <PUERTO_DESCUBIERTO>

```

---

## 4. Documentación del Funcionamiento y Código Fuente

### Flujo de Ejecución Real (Máquina de Estados)

Tras deshabilitar las abstracciones visuales de asincronía en el descompilador de dnSpy (`View -> Options -> Decompiler -> Uncheck Async Methods`), se logró exponer la lógica interna de la clase oculta `Form1.<textBox1_KeyPress>d__7` en su método principal **`MoveNext()`**:

```csharp
// 1. Captura de Entrada (Input)
// Toma los caracteres del usuario en el prompt y los convierte a Base64 hashes simétricos.
this.<a896>5__2 = form.A46(form.textBox1.Text);

// 2. Petición WAN Externa (Prueba de conectividad)
// Realiza un HTTP GET síncrono a la infraestructura de GitHub apuntando a un archivo HEX único.
string a791 = awaiter.GetResult().Trim();

// 3. Sanitización de Payload (Decodificación)
// El método A435 traduce el buffer hexadecimal puro descargado hacia cadenas de texto plano.
string a792 = form.A435(a791);

// 4. Ofuscación de Comparación
// Convierte el texto plano legítimo a Base64 para realizar una validación de firmas opacas.
string a793 = form.A46(a792);

// 5. Bifurcación Lógica de Control
if (this.<a896>5__2 == a793) {
    // Al cumplirse la igualdad, rompe el bucle e invoca la alerta de éxito
}

```

### El Engaño de la Aplicación (*The Cryptographic Decoy*)

```
[ FLUJO DE DISTRACCIÓN - BANNER TELNET ]
Execution ID ──> Algoritmo A31 (ROT13 + Reverse) ──> Servidor Local Pistas (Humo)

[ FLUJO REAL DE VALIDACIÓN - EXTRACCIÓN RAM ]
GitHub Payload (HEX) ──> Método A435 ──> Texto Plano (Contraseña Real) ──> Base64 Match

```

* **La Trampa:** El creador del laboratorio implementó un servidor local de Telnet que exigía decodificar un algoritmo encadenado de `HEX -> Base64 -> Reverse -> ROT13`. Sin embargo, el análisis del código fuente demostró que la contraseña real **nunca pasó por ROT13**. Era un señuelo para desviar la atención del analista.

---

## 5. Auditoría de Red y Topología Local

### Entorno de Simulación

* **Host de Análisis:** Workstation Windows con interfaz Loopback activa (`127.0.0.1`)
* **Capturador de Paquetes:** Wireshark configurado sobre el adaptador *NPcap Loopback Adapter*
* **Servidor Externo:** `raw.githubusercontent.com` (Puerto 443 via TLSv1.3)

### Topología del Tráfico

```
                      ┌──────────────────────────────┐
                      │    raw.githubusercontent.com │ (Servidor Remoto Cifrado)
                      └──────────────┬───────────────┘
                                     │
                                 [Port 443] TLSv1.3 (HTTPS)
                                     │
                      ┌──────────────┴───────────────┐
                      │    WinFormsApp1.exe Process  │ (Target Host)
                      └──────────────┬───────────────┘
                                     │
                            [Puerto Dinámico] TCP Local
                                     │
                      ┌──────────────┴───────────────┐
                      │    Telnet / Sockets Client   │ (Interfaz Loopback)
                      └──────────────────────────────┘

```

---

## 6. Proceso de Extracción Forense de la Credencial

| Paso | Acción Técnica | Resultado en Pantalla / Estado |
| --- | --- | --- |
| **1** | Enumeración de red remota | Wireshark captura peticiones HTTPS cifradas salientes hacia GitHub. |
| **2** | Identificación del Execution ID | Se detecta el ID dinámico mutando en cada reinicio (ej. `32D71993BB-52519`), enlazado al PID de Windows. |
| **3** | Configuración de Debugger | Se inyecta un *Breakpoint* dinámico en dnSpy en la instrucción: `string a793 = form.A46(a792);` |
| **4** | Interrupción en Caliente | Se introduce un input dummy en la interfaz y se pulsa Enter. El proceso se congela en la RAM. |
| **5** | Volcado de Variables (*Dump*) | La pestaña **Locals** revela el contenido en texto claro de la variable **`a792`**, evadiendo toda la ofuscación. |

### Evidencia del Volcado en RAM

* **Variable Inspeccionada:** `Form1.<textBox1_KeyPress>d__7.a792`
* **Contraseña Definitiva Recuperada:**

```text
It never happens all at once. Its slow. Its methodical. Its exhausting.

```

* *[Insertar aquí Captura image_6b1807.png mostrando el breakpoint amarillo y el string en Locals]*
* *[Insertar aquí Captura image_6b13aa.png mostrando la alerta flotante de "Contraseña correcta" en la App]*

---

## 7. Apéndice: Cuestionario de Control de CyberOps

Para cumplir estrictamente con los lineamientos del entregable, se anexan las respuestas estructuradas al cuestionario base:

1. **¿Qué puertos utiliza el programa?** Utiliza el puerto remoto de salida **443 (HTTPS)** para descargar la credencial y un **puerto TCP dinámico aleatorio** (asignado por el kernel de Windows en el rango efímero) para exponer el servidor local de pistas.
2. **¿Cómo realiza el programa la prueba de conectividad?** Inicia un objeto asíncrono `HttpClient` y realiza una solicitud web estructurada de tipo **HTTP GET** hacia el repositorio raw de GitHub del profesor.
3. **Enumera los encodings encontrados y dónde fueron utilizados.** * **HEX:** En el payload remoto descargado de internet.
* **Base64:** En el almacenamiento interno del banner y en el hashing de comparación del input de texto.
* **ROT13 & Reverse String:** Únicamente dentro del método secundario `A31` para dar formato de despiste al texto del servidor local Telnet.


4. **¿Qué tráfico genera el programa?** Tráfico WAN saliente cifrado TLSv1.3 e intercambio de tramas de sockets en texto plano a través de la interfaz local `localhost`.
5. **¿Cómo se conectó al servicio de pistas?** Se identificó el puerto de escucha dinámico filtrando las conexiones activas del proceso en la terminal de comandos y se levantó un canal interactivo mediante el cliente nativo de **Telnet**.

---

**Laboratorio completado por Edwin De Paula (2024-2415)** **ITLA - Seguridad de Redes**

```
