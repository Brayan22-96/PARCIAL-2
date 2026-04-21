# PARCIAL-2

## Diferencia entre NetFlow y sFlow
Diferencia FundamentalNetFlow (Cisco): Es un protocolo basado en el estado de los flujos. El router examina todos los paquetes que pasan y los agrupa en "flujos" (basados en IP origen/destino, puertos, etc.). Consume más CPU y memoria porque mantiene una tabla de flujos activa en el dispositivo.sFlow (Flujo muestreado): Es un protocolo de muestreo estadístico. No analiza todos los paquetes ni mantiene estados; simplemente toma uno de cada $N$ paquetes (por ejemplo, 1 de cada 1000) y envía la cabecera directamente al colector. Es mucho más ligero para el hardware.Escenario de 100 GbpsPara detectar "top talkers" en un enlace de 100 Gbps, elegiría sFlow. A esas velocidades tan altas, procesar cada paquete para actualizar una tabla de flujos (NetFlow) podría saturar el plano de control (CPU) del router o switch. sFlow permite tener visibilidad del tráfico masivo con un impacto mínimo en el rendimiento del equipo.

## La 5-tuple de NetFlow y Medición por Aplicación
Los 5 campos que definen un flujo único son:

IP de Origen

IP de Destino

Puerto de Origen

Puerto de Destino

Protocolo de Capa 4 (TCP, UDP, ICMP, etc.)

Identificación de aplicaciones:
Si deseas medir el consumo por aplicación (HTTP vs. SSH), el colector debe inspeccionar los Puertos de Destino (y a veces los de origen).

HTTP: Generalmente puerto 80 (o 443 para HTTPS).

SHH: Puerto 22.

## Interpretación de IP Accounting
Análisis de la Tabla
De 192.168.1.10 a 10.0.0.5: Se han enviado 1500 paquetes (120.000 Bytes).

De 10.0.0.5 a 192.168.1.10: Se han recibido solo 50 paquetes (4.000 Bytes).

Interpretación de la Asimetría Extrema
Una asimetría tan marcada (muchos paquetes saliendo y casi ninguno regresando) suele indicar tres posibles escenarios:

Tráfico Unidireccional Masivo: El host está realizando un respaldo de datos o una carga (upload) hacia el servidor, y solo recibe ACKs (confirmaciones) de vuelta.192.168.1.10

Ataque de Denegación de Servicio (DoS): El host podría estar realizando un escaneo de puertos o un ataque de inundación (Flooding) contra la IP .10.0.0.5

Enrutamiento Asimétrico: Los paquetes de ida pasan por este router, pero los paquetes de regreso están tomando una ruta física distinta, por lo que el router no los contabiliza.

| Source | Destination | Packets | Bytes |
| :--- | :--- | :--- | :--- |
| 192.168.1.10 | 10.0.0.5 | 1500 | 120000 |
| 10.0.0.5 | 192.168.1.10 | 50 | 4000 |


graph TD
    %% Nodo de Procesamiento
    subgraph "NODO 1: CONTENEDOR (YOLOv8)"
        A[Cámara / Video Stream] -->|Inferencia| B[Python Script]
        B -->|JSON Over Network| C{eth0: 172.17.0.2}
        
        note1[<b>Comandos Clave:</b><br/>- docker build -t yolo-app .<br/>- docker run --net app-network]
    end

    %% Nodo de Red
    subgraph "NODO 2: VM GATEWAY (softflowd)"
        C --> D[Bridge Virtual: docker0]
        D --> E{softflowd}
        
        E -->|NetFlow Export| F[Dest: 192.168.1.50:9995]
        
        note2[<b>Comandos Clave:</b><br/>- softflowd -i docker0 -n 192.168.1.50:9995<br/>- iptables -A FORWARD -c]
    end

    %% Nodo de Visualización
    subgraph "NODO 3: DASHBOARD (Colab/Streamlit)"
        F --> G[Colector de Flujos]
        G --> H[Procesamiento con Pandas]
        H --> I[Dashboard: Alta Visibilidad]
        
        note3[<b>Comandos Clave:</b><br/>- pip install streamlit ultralytics<br/>- streamlit run app.py]
    end

    %% Conexiones
    C -.->|Muestreo| E
