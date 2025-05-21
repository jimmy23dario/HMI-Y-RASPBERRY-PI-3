# HMI-Y-RASPBERRY-PI-3
#APP PARA PANTALLA HMI POR SERIAL CON ARDUINO NANO PARA VEHICULO ELECTRICO
import os
import pygame
import sys
import math
import serial
import time
from datetime import datetime
from serial.tools import list_ports

# Inicialización de Pygame
pygame.init()
screen = pygame.display.set_mode((1024, 600))
pygame.display.set_caption("Panel Automotriz 7\" - Monitor en Tiempo Real")
pygame.mouse.set_visible(True)
# Cargar ícono de advertencia del motor
motor_icon = pygame.image.load("motor_warning1.png").convert_alpha()
motor_icon = pygame.transform.scale(motor_icon, (100, 100))
# Colores
BLACK = (0, 0, 0)
WHITE = (255, 255, 255)
RED = (255, 50, 50)
GREEN = (50, 255, 50)
ORANGE = (255, 165, 0)
DARK_GRAY = (30, 30, 40)
BLUE = (50, 50, 255)

# Fuentes
font_xlarge = pygame.font.SysFont('Arial', 120)
font_large = pygame.font.SysFont('Arial', 50)
font_medium = pygame.font.SysFont('Arial', 40)
font_small = pygame.font.SysFont('Arial', 20)
font_tiny = pygame.font.SysFont('Arial', 30)

# Variables de sensores
sensor_data = {
    'rpm': 0,
    'temperature': 0,
    'voltage': 0,
    'speed': 0,
    'distance': 0.0,
    'ignition': False,
    'last_update': 0
}


def find_arduino_port():
    if os.path.exists('/dev/ttyACM0'):
        try:
            test_ser = serial.Serial('/dev/ttyACM0', timeout=1)
            test_ser.close()
            return '/dev/ttyACM0'
        except:
            pass

    ports = list_ports.comports()
    for port in ports:
        if ('Arduino' in port.description or
                'USB' in port.description or
                'ACM' in port.device or
                'Serial' in port.description):
            return port.device

    common_ports = ['/dev/ttyUSB0', '/dev/ttyUSB1', '/dev/ttyACM1']
    for port in common_ports:
        if os.path.exists(port):
            try:
                test_ser = serial.Serial(port, timeout=1)
                test_ser.close()
                return port
            except:
                continue
    return None


def init_serial_connection():
    max_retries = 5
    retry_delay = 2
    for attempt in range(max_retries):
        port = find_arduino_port()
        if port is None:
            port = '/dev/ttyACM0'
        try:
            ser = serial.Serial(port=port, baudrate=115200, timeout=1)
            time.sleep(2)
            ser.reset_input_buffer()
            return ser
        except serial.SerialException:
            time.sleep(retry_delay)
    return None


def read_sensor_data(ser):
    try:
        if ser.in_waiting > 0:
            line = ser.readline().decode('utf-8').strip()
            if line and not line.startswith(("RPM", "Sistema")):
                try:
                    parts = line.split(',')
                    if len(parts) >= 6:
                        sensor_data['rpm'] = float(parts[0])
                        sensor_data['speed'] = float(parts[1])
                        sensor_data['distance'] = float(parts[2])
                        sensor_data['temperature'] = float(parts[3])
                        sensor_data['voltage'] = float(parts[4])
                        sensor_data['ignition'] = int(parts[5]) > 0

                        sensor_data['last_update'] = time.time()
                        return True
                except ValueError:
                    pass
    except:
        pass
    return False


def draw_tachometer(x, y, size, rpm):
    max_rpm = 8000
    center = (x, y)
    radius = size // 2
    pygame.draw.circle(screen, WHITE, center, radius, 6)

    for angle in range(-135, 136, 27):
        rad_angle = math.radians(angle)
        start = (center[0] + (radius - 20) * math.cos(rad_angle),
                 center[1] + (radius - 20) * math.sin(rad_angle))
        end = (center[0] + radius * math.cos(rad_angle),
               center[1] + radius * math.sin(rad_angle))
        pygame.draw.line(screen, WHITE, start, end, 3)
        rpm_val = int((angle + 135) * (max_rpm / 270))
        text = font_tiny.render(str(rpm_val), True, WHITE)
        text_rect = text.get_rect(center=(center[0] + (radius - 50) * math.cos(rad_angle),
                                          center[1] + (radius - 50) * math.sin(rad_angle)))
        screen.blit(text, text_rect)

    needle_angle = -135 + (rpm / max_rpm) * 270
    needle_angle = max(-135, min(135, needle_angle))
    rad_angle = math.radians(needle_angle)
    end = (center[0] + (radius - 30) * math.cos(rad_angle),
           center[1] + (radius - 30) * math.sin(rad_angle))
    pygame.draw.line(screen, RED, center, end, 6)

    rpm_text = font_medium.render(f"{min(rpm, max_rpm):.0f}", True, WHITE)
    rpm_rect = rpm_text.get_rect(center=center)
    screen.blit(rpm_text, rpm_rect)

def draw_speedometer(x, y, size, speed):
    max_speed = 180  # Velocidad máxima en km/h
    center = (x, y)
    radius = size // 2

    pygame.draw.circle(screen, BLUE, center, radius, 6)

def draw_status_panel():
    temp = sensor_data['temperature']
    volt = sensor_data['voltage']
    speed = sensor_data['speed']
    dist = sensor_data['distance']
    ignition = sensor_data['ignition']

    temp_color = ORANGE if temp > 90 else GREEN
    volt_color = RED if volt < 45 else ORANGE if volt <= 68 or volt > 76 else GREEN

    ignition_color = GREEN if ignition else RED

    screen.blit(font_large.render(f"{temp:.1f}°C", True, temp_color), (40, 30))
    pygame.draw.rect(screen, WHITE, (30, 20, 280, 100), 2)

    screen.blit(font_large.render(f"{volt:.1f}V", True, volt_color), (40, 150))
    pygame.draw.rect(screen, WHITE, (30, 140, 280, 100), 2)

    screen.blit(font_large.render(f"{speed:.1f} km/h", True, BLUE), (40, 270))
    pygame.draw.rect(screen, WHITE, (30, 260, 280, 100), 2)

    screen.blit(font_large.render(f"{dist:.2f} m", True, WHITE), (40, 390))
    pygame.draw.rect(screen, WHITE, (30, 380, 280, 100), 2)

    screen.blit(font_large.render("ON" if ignition else "OFF", True, ignition_color), (100, 500))


def draw_status_bar():
    last = time.time() - sensor_data['last_update']
    update_color = GREEN if last < 2 else ORANGE if last < 5 else RED

    pygame.draw.rect(screen, DARK_GRAY, (0, 550, 1024, 50))
    pygame.draw.rect(screen, update_color, (10, 560, 20, 30))

    status = font_small.render(
        f"Sistema {'OPERATIVO' if sensor_data['ignition'] else 'APAGADO'} | "
        f"Temp: {sensor_data['temperature']:.1f}°C | "
        f"Batería: {sensor_data['voltage']:.1f}V | "
        f"Velocidad: {sensor_data['speed']:.1f} km/h | "
        f"Distancia: {sensor_data['distance']:.2f} km | "
        f"RPM: {sensor_data['rpm']:.0f}", True, WHITE)
    screen.blit(status, (40, 560))

    show_motor_warning = (
        sensor_data['temperature'] > 90 or
        sensor_data['voltage'] < 45 or
        sensor_data['voltage'] > 76 or
        not sensor_data['ignition']
    )
    if show_motor_warning:
        screen.blit(motor_icon, (350, 90))


def draw_connection_status():
    last_update = time.time() - sensor_data['last_update']
    if last_update > 5:
        screen.blit(font_medium.render("SIN CONEXIÓN AL ARDUINO", True, RED), (350, 10))
    elif last_update > 2:
        screen.blit(font_small.render("Conectando...", True, ORANGE), (450, 10))


def main():
    ser = init_serial_connection()
    if ser is None:
        screen.fill(DARK_GRAY)
        screen.blit(font_large.render("ARDUINO NO CONECTADO", True, RED), (150, 200))
        screen.blit(font_medium.render("Conecte el Arduino y reinicie", True, WHITE), (200, 300))
        screen.blit(font_small.render("Puertos disponibles:", True, WHITE), (200, 400))
        y = 450
        for p in list_ports.comports():
            screen.blit(font_tiny.render(f"{p.device}: {p.description}", True, WHITE), (220, y))
            y += 30
        pygame.display.flip()
        time.sleep(10)
        pygame.quit()
        sys.exit()

    clock = pygame.time.Clock()
    last_data_time = time.time()
    last_distance_update = time.time()

    running = True
    while running:
        current_time = time.time()
        delta_time = current_time - last_distance_update
        if delta_time > 0.1:
            speed = sensor_data['speed']
            if speed > 0:
                sensor_data['distance'] += (speed * delta_time) / 3600.0  # km/h to km
            last_distance_update = current_time

        for event in pygame.event.get():
            if event.type == pygame.QUIT or (event.type == pygame.KEYDOWN and event.key == pygame.K_ESCAPE):
                running = False

        if current_time - last_data_time > 0.1:
            if read_sensor_data(ser):
                last_data_time = current_time

        screen.fill(DARK_GRAY)
        draw_status_panel()
        draw_tachometer(700, 300, 400, sensor_data['rpm'])
        draw_status_bar()
        draw_connection_status()
        pygame.display.flip()
        clock.tick(30)




    ser.close()
    pygame.quit()
    sys.exit()


if __name__ == "__main__":
    main()

