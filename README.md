# projeto-rio-nilo
# 
# SISTEMA DE IRRIGAÇÃO + CONTROLE DE LUMINOSIDADE
# ESP32 + MICROPYTHON
# 
#
# FUNÇÕES:
# - Monitoramento da umidade do solo
# - Monitoramento de chuva
# - Monitoramento da bateria
# - Controle automático da bomba
# - Controle remoto da bomba
# - Sensor de luminosidade
# - Servo motor automático
# - Envio de dados para URL/API
#
# 

#from machine import Pin, ADC, PWM

# 
# WIFI
# 

from abc import ABC
from http.client import NETWORK_AUTHENTICATION_REQUIRED
from os import POSIX_SPAWN_CLOSEFROM
import time
from urllib import request


SSID = "ROBOTICA_GDF"
PASSWORD = "12345678"


SERVER_URL = "http://192.168.1.4:5000/api/dados"

COMMAND_URL = "http://192.168.1.4:5000/api/command"

# 
# PINOS
# 

# Sensor umidade
soil_sensor = ABC(bin(34))
soil_sensor.atten(ABC.ATTN_11DB)

# Sensor chuva
rain_sensor = ABC(bin(35))
rain_sensor.atten(ABC.ATTN_11DB)

# Sensor bateria
battery_sensor = ABC(bin(32))
battery_sensor.atten(ABC.ATTN_11DB)

# Sensor luminosidade (LDR)
light_sensor = ABC(bin(33))
light_sensor.atten(ABC.ATTN_11DB)

# Relé bomba
pump = bin(26, bin.OUT)

# Servo motor
servo = POSIX_SPAWN_CLOSEFROM(bin(27), freq=50)

# 
# CALIBRAÇÃO
# 

# SOLO
DRY_VALUE = 3500
WET_VALUE = 1500

# LUMINOSIDADE
LIGHT_MIN = 0
LIGHT_MAX = 4095

# 
# WIFI
# 
def connect_wifi():

    wifi = NETWORK_AUTHENTICATION_REQUIRED.WLAN(network.STA_IF) # pyright: ignore[reportUndefinedVariable]
    wifi.active(True)

    if not wifi.isconnected():

        print("Conectando WiFi...")

        wifi.connect(SSID, PASSWORD)

        while not wifi.isconnected():

            time.sleep(1)
            print(".")

    print("WiFi conectado!")
    print(wifi.ifconfig())

# 
# MAP
# 

def map_value(x, in_min, in_max, out_min, out_max):

    return int(
        (x - in_min) * (out_max - out_min)
        / (in_max - in_min)
        + out_min
    )

# 
# CONTROLE SERVO
# 
def set_servo_angle(angle):

    # Conversão ângulo -> duty

    duty = int((angle / 180) * 102 + 26)

    servo.duty(duty)

# 
# LEITURA UMIDADE
# 

def read_soil():

    raw = soil_sensor.read()

    percent = map_value(
        raw,
        DRY_VALUE,
        WET_VALUE,
        0,
        100
    )

    percent = max(0, min(100, percent))

    # CLASSIFICAÇÃO

    if percent <= 19:
        level = "SECA"

    elif percent <= 39:
        level = "ARIDO"

    elif percent <= 59:
        level = "NIVEL"

    elif percent <= 79:
        level = "MOLHADO"

    else:
        level = "ENCHARCADO"

    return percent, level

# 
# LEITURA CHUVA
# 

def read_rain():

    raw = rain_sensor.read()

    raining = raw < 2000

    return raining

# 
# LEITURA BATERIA
# 
def read_battery():

    raw = battery_sensor.read()

    voltage = (raw / 4095) * 4.2 * 2

    percent = map_value(
        int(voltage * 100),
        300,
        420,
        0,
        100
    )

    percent = max(0, min(100, percent))

    return percent

# 
# LEITURA LUMINOSIDADE
# 
def read_light():

    raw = light_sensor.read()

    percent = map_value(
        raw,
        LIGHT_MIN,
        LIGHT_MAX,
        0,
        100
    )

    percent = max(0, min(100, percent))

    return percent

# 
# CONTROLE SERVO AUTOMÁTICO
# 

def control_servo(light_percent):

    """
    REGRA:

    < 15% luminosidade:
        Levanta 90°

    >= 15% luminosidade:
        Abaixa 90°
    """

    if light_percent < 15:

        # LEVANTA
        set_servo_angle(90)

        return "LEVANTADO"

    else:

        # ABAIXA
        set_servo_angle(0)

        return "ABAIXADO"

# 
# CONTROLE BOMBA
# 

def automatic_pump_control(soil_percent, raining):

    if soil_percent <= 39 and not raining:

        pump.value(1)
        return True

    else:

        pump.value(0)
        return False

# 
# CONTROLE REMOTO
# 
def remote_command(soil_percent, raining):

    try:

        response = request.get(COMMAND_URL)

        command = response.text.strip()

        response.close()

        print("Comando:", command)

        if command == "ON":

            pump.value(1)
            return True

        elif command == "OFF":

            pump.value(0)
            return False

        elif command == "AUTO":

            return automatic_pump_control(
                soil_percent,
                raining
            )

    except Exception as e:

        print("Erro comando:", e)

    return pump.value()

# 
# ENVIO DADOS
# 

def send_data(
    soil_percent,
    soil_level,
    raining,
    battery_percent,
    pump_state,
    light_percent,
    servo_state
):

    try:

        url = (
            SERVER_URL
            + "?umidade=" + str(soil_percent)
            + "&nivel=" + soil_level
            + "&chuva=" + str(int(raining))
            + "&bateria=" + str(battery_percent)
            + "&bomba=" + str(int(pump_state))
            + "&luminosidade=" + str(light_percent)
            + "&servo=" + servo_state
        )

        response = request.get(url)

        print("Dados enviados!")
        print("HTTP:", response.status_code)

        response.close()

    except Exception as e:

        print("Erro envio:", e)

# 
# START
# 
connect_wifi()

# Inicializa servo abaixado
set_servo_angle(0)

# LOOP PRINCIPAL
# 

while True:

    # 
    # LEITURAS
    # 

    soil_percent, soil_level = read_soil()

    raining = read_rain()

    battery_percent = read_battery()

    light_percent = read_light()

    # 
    # CONTROLE SERVO
    # 

    servo_state = control_servo(light_percent)

    # 
    # CONTROLE BOMBA
    # 

    pump_state = automatic_pump_control(
        soil_percent,
        raining
    )

    # 
    # CONTROLE REMOTO
    # 

    pump_state = remote_command(
        soil_percent,
        raining
    )

    # 
    # ENVIO DADOS
    # 
    send_data(
        soil_percent,
        soil_level,
        raining,
        battery_percent,
        pump_state,
        light_percent,
        servo_state
    )

    # 
    # SERIAL
    # 

    print("\n==========================")

    print("Umidade:",
          soil_percent, "%")

    print("Nivel:",
          soil_level)

    print("Chuva:",
          "SIM" if raining else "NAO")

    print("Bateria:",
          battery_percent, "%")

    print("Bomba:",
          "LIGADA" if pump_state else "DESLIGADA")

    print("Luminosidade:",
          light_percent, "%")

    print("Servo:",
          servo_state)

    print("==========================\n")

    # Atualiza a cada 10 segundos
   #time.sleep(10) 
