import network
import time
import socket
import machine
import json

# Definir constantes del TCS34725 que faltaban
ENABLE = 0x00
ATIME = 0x01
CONTROL = 0x0F
CDATA = 0x14

i2c = machine.I2C(0, scl=machine.Pin(22), sda=machine.Pin(21), freq=400000)
TCS34725_ADDR = 0x29
CMD = 0x80

def init_tcs34725():
    i2c.writeto_mem(TCS34725_ADDR, CMD | ENABLE, b'\x03')
    i2c.writeto_mem(TCS34725_ADDR, CMD | ATIME, b'\xEB')
    i2c.writeto_mem(TCS34725_ADDR, CMD | CONTROL, b'\x01')
    time.sleep(0.5)

def get_datetime_formatted():
    t = time.localtime()
    year, month, day, hour, minute, second = t[0:6]
    return f"{day:02d}/{month:02d}/{year} {hour:02d}:{minute:02d}:{second:02d}"

# Función para mostrar menú
def show_menu():
    print("\n" + "=" * 40)
    print("       MENU PRINCIPAL")
    print("=" * 40)
    print(f"Fecha y Hora: {get_datetime_formatted()}")
    print("\nOpciones:")
    print("1. Ver estado de conexión")
    print("2. Ver información del ESP32")
    print("3. Continuar enviando datos")
    print("=" * 40)

    try:
        data = i2c.readfrom_mem(TCS34725_ADDR, CMD | CDATA, 8)

        luminicidad = (data[1] << 8) | data[0]
        rojo = (data[3] << 8) | data[2]
        azul = (data[5] << 8) | data[4]
        verde = (data[7] << 8) | data[6]

        print(f"Lectura TCS34725 -> R:{rojo} G:{verde} B:{azul} C:{luminicidad}")
        return rojo, verde, azul, luminicidad

    except Exception as e:
        print("Error al leer TCS34725:", e)
        return None

# Inicializar sensor
init_tcs34725()
print("Escaneo I2C:", [hex(x) for x in i2c.scan()])

# Conectar a WiFi
wf = network.WLAN(network.STA_IF)
wf.active(True)

print("Conectando a WiFi...")
wf.connect('Nancy', '39798969')

while not wf.isconnected():
    print(".")
    time.sleep(1)

print("Conectado a:", wf.ifconfig())

show_menu()  # Mostrar menú al conectarse

# Crear socket cliente para conectarse a la PC
print("Conectando al servidor en la PC...")
s_pc = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

print("Conectando a la Raspberry Pi...")
s_pi = socket.socket(socket.AF_INET, socket.SOCK_STREAM)#192.168.1.10

try:
    # conectar a la PC (ajusta IP y puerto según tu servidor)
    s_pc.connect(('192.168.1.8', 2024))#15portatil   8 pc
    print("Conectado al PC")

    s_pi.connect(('192.168.1.10', 2024))
    print("Conectado a la Raspberry Pi")
    packet_id = 0   # contador de paquetes
    # enviar datos cada segundo
    while True:
        rgb_data = show_menu()# Usa show_menu para leer sensor
        if rgb_data is not None:
            rojo, azul, verde, luminicidad = rgb_data
            
            # crear estructura JSON con timestamp y ID de paquete
            payload = {
                "packet_id": packet_id,
                "timestamp": int(time.time()) + 946684800,
                "rojo": rojo,
                "azul": azul,
                "verde": verde,
                "luminicidad": luminicidad
            }
            
            # convertir a JSON y enviar
            mensaje = json.dumps(payload) + "\n"
            try:
                s_pc.send(mensaje.encode())
            except:
                print("Error enviando al PC")

            try:
                s_pi.send(mensaje.encode())
            except:
                print("Error enviando a la Raspberry")
        else:
            print("No se pudo leer el sensor")
            
        packet_id += 1
        
        time.sleep(1)  # esperar 1 segundo antes de enviar el siguiente dato
        
except Exception as e:
    print(f"Error de conexión: {e}")
finally:
    s_pc.close()
    s_pi.close()
    print("Socket cerrado")

#################################################################################################################################################################################################



# -*- coding: utf-8 -*-
"""
Created on Thu Mar  5 22:48:49 2026

@author: smuel
"""

#!/usr/bin/env python3
# -*- coding: utf-8 -*-
from datetime import datetime, timezone
import socket
import json
import sys
import matplotlib.pyplot as plt

HOST = '0.0.0.0'
PORT = 2024

def main():
    server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)

    try:
        server_socket.bind((HOST, PORT))
        server_socket.listen(1)

        print("=" * 60)
        print("   SERVIDOR + GRÁFICA EN TIEMPO REAL TCS34725")
        print("=" * 60)
        print(f"Escuchando en puerto {PORT}...")
        print("Esperando conexión del ESP32...\n")

        client_socket, client_address = server_socket.accept()
        print(f"Cliente conectado desde: {client_address[0]}:{client_address[1]}")
        print("-" * 60)

        plt.ion()
        fig, ax = plt.subplots()
        ax.set_title("Lecturas TCS34725 en tiempo real")
        ax.set_xlabel("Paquete")
        ax.set_ylabel("Valor")
        ax.grid(True)

        x_data = []
        rojo_data = []
        verde_data = []
        azul_data = []
        luminicidad_data = []

        # AQUÍ ESTÁ EL ÚNICO CAMBIO: colores reales para cada línea
        line_rojo, = ax.plot([], [], 'red', label='Rojo', linewidth=2)
        line_verde, = ax.plot([], [], 'green', label='Verde', linewidth=2)
        line_azul, = ax.plot([], [], 'blue', label='Azul', linewidth=2)
        line_lum, = ax.plot([], [], 'gray', label='Luminicidad', linewidth=2)

        ax.legend()

        buffer = ""

        try:
            while True:
                data = client_socket.recv(1024).decode("utf-8")

                if not data:
                    print("\nConexión cerrada por el cliente")
                    break

                buffer += data

                while "\n" in buffer:
                    line, buffer = buffer.split("\n", 1)

                    if line.strip():
                        try:
                            payload = json.loads(line)

                            ts=payload.get("timestamp", "N/A")
                            packet_id = payload.get("packet_id", 0)
                            fecha = datetime.fromtimestamp(ts, tz=timezone.utc)
                            rojo = payload.get("rojo", 0)
                            verde = payload.get("verde", 0)
                            azul = payload.get("azul", 0)
                            luminicidad = payload.get("luminicidad", 0)

                            print(f"Paquete {packet_id} Time S:{fecha} -> R:{rojo} G:{verde} B:{azul} C:{luminicidad}")

                            x_data.append(packet_id)
                            rojo_data.append(rojo)
                            verde_data.append(verde)
                            azul_data.append(azul)
                            luminicidad_data.append(luminicidad)

                            line_rojo.set_data(x_data, rojo_data)
                            line_verde.set_data(x_data, verde_data)
                            line_azul.set_data(x_data, azul_data)
                            line_lum.set_data(x_data, luminicidad_data)

                            ax.relim()
                            ax.autoscale_view()

                            fig.canvas.draw()
                            fig.canvas.flush_events()

                        except json.JSONDecodeError as e:
                            print(f"Error al parsear JSON: {e}")
                            print(f"Datos recibidos: {line}\n")

        except KeyboardInterrupt:
            print("\nServidor interrumpido por el usuario")

        finally:
            client_socket.close()

    except OSError as e:
        print(f"Error al vincular el puerto {PORT}")
        print(f"Detalles: {e}")
        print("Asegúrate de que el puerto no esté en uso")
        sys.exit(1)

    finally:
        server_socket.close()
        print("Servidor cerrado")

if __name__ == "__main__":
    main()

