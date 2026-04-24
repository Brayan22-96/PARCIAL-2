# PARCIAL-2

### Diferencia entre NetFlow y sFlow + escenario de uso

La diferencia fundamental entre NetFlow y sFlow radica en la forma en que recolectan la información del tráfico de red.

NetFlow realiza un análisis completo de los flujos, registrando información detallada de cada conexión (basado en la 5-tupla: IP origen, IP destino, puerto origen, puerto destino y protocolo). Esto permite obtener métricas precisas como número de paquetes, bytes transmitidos y duración del flujo. Sin embargo, este nivel de detalle implica un mayor consumo de CPU y memoria en el dispositivo de red.

Por otro lado, sFlow utiliza un método de muestreo estadístico, donde solo se analiza un subconjunto de los paquetes que circulan por la red. Esto reduce significativamente la carga en el dispositivo, pero a costa de perder precisión en los datos.

En un enlace de 100 Gbps, elegiría sFlow sobre NetFlow para detectar “top talkers”, ya que:

- Permite escalar mejor en enlaces de alta velocidad.
- Reduce la sobrecarga en el router o switch.
- Proporciona una visión suficientemente representativa del tráfico para identificar los principales generadores de datos.

### Campos de la 5-tupla en NetFlow

La 5-tupla en NetFlow está compuesta por los siguientes campos:
- Dirección IP de origen
- Dirección IP de destino
- Puerto de origen
- Puerto de destino
- Protocolo (TCP, UDP, ICMP, etc.)

Si se desea medir el consumo de ancho de banda por aplicación (por ejemplo, distinguir entre HTTP y SSH), el collector debe inspeccionar principalmente:

Los puertos de origen y destino

Esto se debe a que muchas aplicaciones utilizan puertos bien conocidos, por ejemplo:

- HTTP → puerto 80
- HTTPS → puerto 443
- SSH → puerto 22

### Interpretación de los datos (IP Accounting)

Dada la tabla:

| Source        | Destination   | Packets | Bytes  |
|---------------|--------------|--------:|-------:|
| 192.168.1.10  | 10.0.0.5     | 1500    | 120000 |
| 192.168.1.10  | 10.0.0.8     | 800     | 64000  |
| 10.0.0.5      | 192.168.1.10 | 50      | 4000   |

Se observa que:

- Desde 192.168.1.10 hacia 10.0.0.5 hay 1500 paquetes, mientras que en sentido contrario solo hay 50 paquetes.
- Esto indica una asimetría extrema en el tráfico.

Esto sugiere que:

- 192.168.1.10 está enviando una gran cantidad de datos, pero recibe muy poca respuesta.
- Puede tratarse de:
- Tráfico tipo cliente → servidor (ej. subida de datos, streaming, backup)
- Un posible envío masivo sin respuesta (ej. escaneo o intento de ataque)
- Problemas de red (pérdida de paquetes o mala configuración)

Conclusión:
Existe una asimetría significativa en el flujo, donde el tráfico de salida es mucho mayor que el de retorno, lo cual podría indicar un comportamiento anómalo o una aplicación que genera tráfico unidireccional.

## 2.b Arquitectura de Monitoreo – Estación de Trenes

### Descripción general
Se diseña una arquitectura distribuida basada en contenedores Docker, donde cada cámara ejecuta un modelo YOLO especializado. Los resultados (video y metadata) se envían a servidores centrales redundantes, garantizando alta disponibilidad, calidad de servicio (QoS) y monitoreo mediante NetFlow/IP Accounting.

### Componentes de la arquitectura
Contenedores (Docker)

Cada uno en la red 10.0.0.0/24:

| Contenedor | Función                          | IP         |
|------------|----------------------------------|------------|
| C1         | YOLO + OCR (placas)              | 10.0.0.11  |
| C2         | Conteo de parqueadero            | 10.0.0.12  |
| C3         | Detección de personas            | 10.0.0.13  |
| C4         | Detección de animales            | 10.0.0.14  |
| C5         | Objetos perdidos (maletas, etc.) | 10.0.0.15  |

### Máquinas Virtuales
| VM   | Función                | IP         |
|------|----------------------|------------|
| VM1  | Colector principal   | 10.0.0.100 |
| VM2  | Respaldo redundante  | 10.0.0.101 |

### Red y Conectividad
- Switch virtual: Open vSwitch o Linux Bridge  
- Enlaces redundantes hacia VM1 y VM2  
- Subred: 10.0.0.0/24  
- Fuente de video: RTSP (cámaras o archivos)
<img width="2730" height="768" alt="mermaid-diagram (2)" src="https://github.com/user-attachments/assets/8ce256fb-9bac-453c-8511-b73364a37781" />

### Tipo de Tráfico
| Tipo de dato | Protocolo | Motivo                    |
|--------------|----------|---------------------------|
| Video        | UDP      | Baja latencia             |
| Metadata     | TCP      | Entrega confiable         |

### Throughput por Contenedor
| Tipo       | Cálculo                        | Resultado |
|------------|--------------------------------|----------|
| Video      | 30 fps × 50 KB × 8             | 12 Mbps  |
| Metadata   | 200 bytes × 10 × 8             | 0.016 Mbps |
| **Total**  | Video + Metadata               | 12.016 Mbps |

### Throughput Total (Sistema)
| Elemento        | Valor        |
|----------------|-------------|
| Contenedores   | 5           |
| Total sistema  | ≈ 60.08 Mbps |

### Ejemplo de 5-Tuple (NetFlow)
| Campo        | Valor        |
|--------------|-------------|
| IP origen    | 10.0.0.11   |
| IP destino   | 10.0.0.100  |
| Puerto origen| 5000        |
| Puerto destino| 9000       |
| Protocolo    | UDP         |

### IP Accounting (Monitoreo)
iptables -L -v -n
- Permite identificar qué IP (contenedor) envía más tráfico observando la columna de bytes acumulados.

### Jitter y Solución
- Uso de jitter buffer en el receptor  
- Sincronización por timestamps  
- Aplicación de QoS para priorizar video
