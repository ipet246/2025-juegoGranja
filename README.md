"""
Este es un mini juego de granja que hice donde podés plantar y cosechar.
Cada vez que cosechás, ganás monedas, y cuando juntás cierta cantidad 
(el juego lo hace cada 5 monedas), se envía un mensaje automático al email 
que la persona ingresó al empezar. 
También guarda los datos del jugador en un archivo JSON para que, si vuelve a entrar,
se mantengan sus monedas y su información. 
Lo armé de forma simple con pygame y usando Gmail para mandar las notificaciones.
"""

import pygame, time, sys
import json, os, re
from email.message import EmailMessage
import smtplib



GMAIL_REMITENTE = "ambarlopez444@gmail.com"
APP_PASSWORD = "v i g p o ps v k y k f c p c h"


def enviar_correo_gmail(destinatario_email, asunto_texto, mensaje_texto):
    """Envía un correo automáticamente sin pedir contraseña al usuario."""

    email = EmailMessage()
    email["From"] = GMAIL_REMITENTE
    email["To"] = destinatario_email
    email["Subject"] = asunto_texto
    email.set_content(mensaje_texto)

    try:
        smtp = smtplib.SMTP_SSL("smtp.gmail.com", 465)
        smtp.login(GMAIL_REMITENTE, APP_PASSWORD)
        smtp.send_message(email)
        smtp.quit()
        print(f"Correo enviado a {destinatario_email}")

    except Exception as e:
        print(f"ERROR al enviar correo: {e}")


# ============================================================
# =================  BASE DE DATOS JSON  ======================
# ============================================================

FILE = "database.json"

# Crear archivo si no existe
if not os.path.exists(FILE):
    with open(FILE, "w") as f:
        json.dump({}, f)


def cargar_datos():
    with open(FILE, "r") as f:
        return json.load(f)


def guardar_datos(data):
    with open(FILE, "w") as f:
        json.dump(data, f, indent=4)


def validar_email(email):
    patron = r"^[\w\.-]+@[\w\.-]+\.\w+$"
    return re.match(patron, email) is not None


def registrar_jugador(nombre, email):
    datos = cargar_datos()

    if nombre in datos:
        return False  # ya existe

    datos[nombre] = {"email": email, "coins": 0}
    guardar_datos(datos)
    return True


def actualizar_monedas(nombre, coins):
    datos = cargar_datos()
    datos[nombre]["coins"] = coins
    guardar_datos(datos)


# ============================================================
# ===================  JUEGO DE LA GRANJA  ====================
# ============================================================

pygame.init()
WIDTH, HEIGHT = 600, 400
screen = pygame.display.set_mode((WIDTH, HEIGHT))
pygame.display.set_caption("Mini Granja Simple")
FONT = pygame.font.SysFont(None, 28)

# Colores
BROWN = (139, 69, 19)
GREEN = (34, 139, 34)
RED = (200, 50, 50)
WHITE = (255,255,255)

SIZE = 100
GROW_TIME = 5  # segundos


def pedir_texto(mensaje):
    texto = ""
    clock = pygame.time.Clock()

    while True:
        screen.fill((50, 50, 50))
        msg_render = FONT.render(mensaje, True, WHITE)
        screen.blit(msg_render, (20, 20))

        box = pygame.Rect(20, 80, 400, 40)
        pygame.draw.rect(screen, WHITE, box, 2)

        input_render = FONT.render(texto, True, WHITE)
        screen.blit(input_render, (25, 85))

        pygame.display.flip()

        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                pygame.quit()
                sys.exit()

            if event.type == pygame.KEYDOWN:
                if event.key == pygame.K_RETURN:
                    return texto
                elif event.key == pygame.K_BACKSPACE:
                    texto = texto[:-1]
                else:
                    texto += event.unicode

        clock.tick(30)


class Plot:
    def __init__(self, x, y):
        self.rect = pygame.Rect(x, y, SIZE, SIZE)
        self.state = "empty"
        self.time_planted = None

    def update(self):
        if self.state == "growing":
            if time.time() - self.time_planted >= GROW_TIME:
                self.state = "ready"

    def click(self):
        if self.state == "empty":
            self.state = "growing"
            self.time_planted = time.time()
        elif self.state == "ready":
            self.state = "empty"
            return 1
        return 0

    def draw(self, screen):
        color = (
            BROWN if self.state == "empty"
            else GREEN if self.state == "growing"
            else RED
        )
        pygame.draw.rect(screen, color, self.rect)


def main():
    nombre = pedir_texto("Ingrese nombre del personaje:")

    # VALIDAR EMAIL
    while True:
        email = pedir_texto("Ingrese su email válido:")
        if validar_email(email):
            break
        else:
            pedir_texto("Email inválido. ENTER para continuar.")

    # REGISTRO
    if not registrar_jugador(nombre, email):
        pedir_texto("Jugador existente. ENTER para cargar datos.")

    datos = cargar_datos()
    coins = datos[nombre]["coins"]

    clock = pygame.time.Clock()

    plots = []
    for r in range(2):
        for c in range(3):
            plots.append(Plot(70 + c*(SIZE+20), 80 + r*(SIZE+20)))

    running = True
    while running:
        clock.tick(30)
        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                actualizar_monedas(nombre, coins)
                running = False

            elif event.type == pygame.MOUSEBUTTONDOWN and event.button == 1:
                for p in plots:
                    if p.rect.collidepoint(event.pos):
                        antes = coins
                        coins += p.click()

                        # ENVÍO AUTOMÁTICO CADA 5 MONEDAS
                        if coins != antes and coins % 5 == 0:
                            asunto = "¡Nuevo logro en tu granja!"
                            mensaje = f"Hola {nombre}, ¡felicidades! Llegaste a {coins} monedas."
                            enviar_correo_gmail(email, asunto, mensaje)

        for p in plots:
            p.update()

        screen.fill((100,200,255))
        text = FONT.render(f"{nombre} - Monedas: {coins}", True, WHITE)
        screen.blit(text, (20,20))

        for p in plots:
            p.draw(screen)

        pygame.display.flip()

    pygame.quit()
    sys.exit()


if __name__ == "__main__":
    main()


