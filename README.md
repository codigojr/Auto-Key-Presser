import tkinter as tk
from tkinter import ttk, filedialog, messagebox
import cv2
import numpy as np
import pyautogui
import threading
import time
import keyboard
import json
from PIL import Image
import webbrowser


class AutoKeyPresserImproved(tk.Tk):
    reference_image_path = None

    def __init__(self):
        super().__init__()
        self.toggle_key_pressed = threading.Event()
        self.toggle_lock = threading.Lock()

        self.title("Auto Key Presser - Gamer Edition (Mejorado)")
        self.geometry("550x1140")
        self.configure(bg="#2c2c2c")

        self.init_variables()
        self.create_widgets()
        self.set_keyboard_hotkeys()

        self.should_press = threading.Event()
        self.mouse_button_down = threading.Event()
        self.hp_bar_x1 = None
        self.hp_bar_y1 = None
        self.hp_bar_x2 = None
        self.hp_bar_y2 = None
        self.potion_area_x1 = None
        self.potion_area_y1 = None
        self.potion_x1 = None
        self.potion_y1 = None

        # Añadimos esta línea al final
        self.protocol("WM_DELETE_WINDOW", self.on_closing)

    def init_variables(self):
        self.key_to_press = [tk.StringVar(self) for _ in range(5)]
        self.frequency = [tk.DoubleVar(self) for _ in range(5)]
        self.use_key = [tk.BooleanVar(self, value=False) for _ in range(5)]
        # Nueva variable para secuencia de clics
        self.left_mouse_click = tk.BooleanVar(self, value=False)
        self.use_mouse_left = tk.BooleanVar(self, value=False)
        self.use_mouse_right = tk.BooleanVar(self, value=False)
        self.mouse_key = tk.StringVar(self, value="x")
        self.hp_key = tk.StringVar(self)
        self.hp_level = tk.DoubleVar(self, value=80)
        self.monitor_hp_var = tk.BooleanVar(self, value=False)
        self.threads = []
        self.message = tk.StringVar(
            self, value="Bienvenido a Auto Key Presser (Mejorado)!")

    def create_widgets(self):
        title_label = tk.Label(self, text="Auto Key Presser (Mejorado)",
                               bg="#2c2c2c", fg="white", font=("Arial", 18))
        title_label.pack(pady=10)

        self.create_mouse_frame()
        self.create_key_frames()
        self.create_hp_settings_frame()
        self.create_config_buttons()  # Botones de guardar y cargar configuración
        self.create_control_buttons()

    def create_mouse_frame(self):
        mouse_frame = ttk.LabelFrame(
            self, text="Configuración del Mouse", padding=(10, 5))
        mouse_frame.pack(pady=10, fill=tk.X, padx=10)

        ttk.Checkbutton(mouse_frame, text="Left", variable=self.use_mouse_left).grid(
            row=1, column=0, padx=5, sticky=tk.W)
        ttk.Label(mouse_frame, text="Intervalo (s):").grid(
            row=1, column=1, sticky=tk.W)
        self.left_mouse_interval = tk.DoubleVar(value=1)
        ttk.Entry(mouse_frame, textvariable=self.left_mouse_interval,
                  width=5).grid(row=1, column=2, padx=5)
        ttk.Checkbutton(mouse_frame, text="Left Clicker", variable=self.left_mouse_click).grid(
            row=1, column=3, padx=5, sticky=tk.W)

        ttk.Checkbutton(mouse_frame, text="Right", variable=self.use_mouse_right).grid(
            row=2, column=0, padx=5, sticky=tk.W)
        ttk.Label(mouse_frame, text="Intervalo (s):").grid(
            row=2, column=1, sticky=tk.W)
        self.right_mouse_interval = tk.DoubleVar(value=1)
        ttk.Entry(mouse_frame, textvariable=self.right_mouse_interval,
                  width=5).grid(row=2, column=2, padx=5)

        self.mouse_status_label = ttk.Label(
            mouse_frame, text="Estado: Inactivo")
        self.mouse_status_label.grid(row=4, column=0, columnspan=3, pady=5)

    def press_mouse_button_repeatedly(self, button, interval):
        print(f"Thread started for mouse button {button}")
        while self.should_press.is_set():
            if self.mouse_button_down.is_set():
                if button == "left" and self.left_mouse_click.get():  # Condicional para manejar la secuencia de clics
                    print(f"Clicking mouse button {button}")
                    pyautogui.click()
                    # Hace clics rápidos con un intervalo de 0.01 segundos
                    time.sleep(0.01)
                else:
                    print(f"Clicking mouse button {button}")
                    pyautogui.mouseDown(button=button)
                    time.sleep(interval)
                    pyautogui.mouseUp(button=button)
                time.sleep(interval)
            else:
                time.sleep(0.1)
        print(f"Thread ended for mouse button {button}")

    def create_key_frames(self):
        for i in range(5):
            frame = ttk.LabelFrame(
                self, text=f"Configuración de Tecla {i+1}", padding=(10, 5))
            frame.pack(pady=10, fill=tk.X, padx=10)

            use_button = ttk.Checkbutton(
                frame, text="Usar", variable=self.use_key[i])
            use_button.grid(row=i, column=0, sticky="w", padx=5)

            ttk.Label(frame, text=f"Key:").grid(row=i, column=1)
            entry_key = ttk.Entry(
                frame, textvariable=self.key_to_press[i], width=5)
            entry_key.grid(row=i, column=2, padx=5)

            ttk.Label(frame, text=f"Freq (s):").grid(row=i, column=3)
            entry_freq = ttk.Entry(
                frame, textvariable=self.frequency[i], width=5)
            entry_freq.grid(row=i, column=4, padx=5)

    def create_hp_settings_frame(self):
        hp_frame = ttk.LabelFrame(
            self, text="Configuración de HP", padding=(10, 5))
        hp_frame.pack(pady=10, fill=tk.X, padx=10)

        ttk.Button(hp_frame, text="Cargar imagen de referencia",
                   command=self.load_reference_image).grid(row=0, column=0, columnspan=2, pady=5)

        ttk.Label(hp_frame, text="HP Key:").grid(row=1, column=0, pady=5)
        entry = ttk.Entry(hp_frame, textvariable=self.hp_key, width=10)
        entry.grid(row=1, column=1, pady=5, padx=5)

        ttk.Label(hp_frame, text="HP Bar Image Similarity:").grid(
            row=2, column=0, columnspan=2, pady=5)
        self.hp_scale = ttk.Scale(
            hp_frame, from_=0, to=100, length=300, orient=tk.HORIZONTAL, variable=self.hp_level)
        self.hp_scale.grid(row=3, column=0, columnspan=2, pady=5)

        self.hp_checkbox = ttk.Checkbutton(
            hp_frame, text="Monitor HP", variable=self.monitor_hp_var)

        self.hp_checkbox.grid(row=4, column=0, columnspan=2, pady=5)

        self.similarity_label = ttk.Label(
            hp_frame, text=f"Similitud: {self.hp_level.get():.2f}%")
        self.similarity_label.grid(row=5, column=0, columnspan=2, pady=5)

        self.hp_scale.config(command=self.update_similarity_label)

    def create_config_buttons(self):
        frame = ttk.Frame(self)
        frame.pack(pady=10, fill=tk.X)

        ttk.Button(frame, text="Guardar Configuración",
                   command=self.save_config).grid(row=0, column=0, padx=5)
        ttk.Button(frame, text="Cargar Configuración",
                   command=self.load_config).grid(row=0, column=1, padx=5)

    def create_control_buttons(self):
        frame = ttk.Frame(self)
        frame.pack(pady=10, fill=tk.X)

        ttk.Button(frame, text="Start", command=self.start).grid(
            row=0, column=0, padx=5)
        ttk.Button(frame, text="Stop", command=self.stop).grid(
            row=0, column=1, padx=5)
        ttk.Button(frame, text="Salir", command=self.exit).grid(
            row=0, column=2, padx=5)

        self.message_label = ttk.Label(self, textvariable=self.message)
        self.message_label.pack(pady=10)

    def set_keyboard_hotkeys(self):
        self.hooks = []
        t = threading.Thread(target=self.check_hotkeys)
        t.start()
        threading.Thread(target=self.monitor_toggle_event).start()

    def monitor_toggle_event(self):
        while True:
            self.toggle_key_pressed.wait()  # Esperar hasta que el evento se active
            self.toggle_all_presses()  # Cambia el estado de should_press
            self.toggle_key_pressed.clear()  # Limpiar el evento para la próxima detección

    def check_hotkeys(self):
        while True:
            if keyboard.is_pressed('f1'):
                self.set_hp_area1()
                time.sleep(0.2)
            elif keyboard.is_pressed('f2'):
                self.set_hp_area2()
                time.sleep(0.2)
            elif keyboard.is_pressed('f3'):
                print("F3 pressed!")  # Diagnóstico
                if not self.toggle_key_pressed.is_set():
                    self.toggle_key_pressed.set()
                    print("toggle_key_pressed set to True!")  # Diagnóstico
                else:
                    self.toggle_key_pressed.clear()
                    print("toggle_key_pressed set to False!")  # Diagnóstico
                time.sleep(0.2)
            elif keyboard.is_pressed('f4'):
                self.toggle_mouse_press()
                time.sleep(0.2)

    def on_closing(self):
        for hook in self.hooks:
            keyboard.remove_hotkey(hook)
        self.exit()

    def set_hp_area1(self):
        self.hp_bar_x1, self.hp_bar_y1 = pyautogui.position()

    def set_hp_area2(self):
        x2, y2 = pyautogui.position()

        # Ordenamos las coordenadas para asegurarnos de que x1/y1 siempre sean las coordenadas superiores izquierdas
        self.hp_bar_x1, self.hp_bar_x2 = sorted([self.hp_bar_x1, x2])
        self.hp_bar_y1, self.hp_bar_y2 = sorted([self.hp_bar_y1, y2])

        try:
            screenshot = pyautogui.screenshot(region=(self.hp_bar_x1, self.hp_bar_y1,
                                                      self.hp_bar_x2 - self.hp_bar_x1,
                                                      self.hp_bar_y2 - self.hp_bar_y1))
            screenshot.save("hp_bar_reference.png")
            self.message.set(
                f"Área de HP establecida y guardada como hp_bar_reference.png: {(self.hp_bar_x1, self.hp_bar_y1)} a {(self.hp_bar_x2, self.hp_bar_y2)}")
        except Exception as e:
            self.message.set(
                f"Error al capturar y guardar la imagen: {str(e)}")

    def toggle_all_presses(self):
        print("Entering toggle_all_presses")
        with self.toggle_lock:
            if self.should_press.is_set():
                print("Stopping due to should_press being set.")
                self.stop()
                self.after(0, self.message.set, "AutoPresser Desactivado")
            else:
                print("Starting due to should_press not being set.")
                self.start()
                self.after(0, self.message.set, "AutoPresser Activado")

        time.sleep(0.5)
        print("Exiting toggle_all_presses")

    def toggle_mouse_press(self):
        print("Entering toggle_mouse_press")
        if self.mouse_button_down.is_set():
            self.mouse_button_down.clear()
            self.mouse_status_label.config(text="Estado: Inactivo")
        else:
            self.mouse_button_down.set()
            self.mouse_status_label.config(text="Estado: Activo")
        print("Exiting toggle_mouse_press")

    def print_threads_status(self):

        for thread in threading.enumerate():
            print(
                f"Thread Name: {thread.name}, Thread Status: {thread.is_alive()}")

    def stop(self):
        self.should_press.clear()
        self.mouse_button_down.clear()
        self.message.set("AutoPresser Desactivado")

    def exit(self):
        self.stop()
        self.destroy()

    def load_reference_image(self):
        file_path = filedialog.askopenfilename(title="Selecciona una imagen de referencia", filetypes=[
            ("PNG files", "*.png"), ("JPEG files", "*.jpg"), ("All files", "*.*")])
        if file_path:
            self.reference_image_path = file_path
            self.message.set(
                f"Imagen de referencia cargada desde {file_path}")

    def press_key_repeatedly(self, key, interval):
        print(f"Thread started for key {key}")
        while self.should_press.is_set():
            print(f"Pressing key {key}")
            keyboard.press(key)
            time.sleep(0.05)
            keyboard.release(key)
            time.sleep(interval)
        print(f"Thread ended for key {key}")

    def press_mouse_button_repeatedly(self, button, interval):
        print(f"Thread started for mouse button {button}")
        while self.should_press.is_set():
            if self.mouse_button_down.is_set():
                print(f"Clicking mouse button {button}")
                pyautogui.mouseDown(button=button)
                time.sleep(interval)
                pyautogui.mouseUp(button=button)
            else:
                time.sleep(0.1)
        print(f"Thread ended for mouse button {button}")

    def start(self):
        print("Starting...")
        if not self.should_press.is_set():
            self.should_press.set()
            self.after(0, self.message.set, "AutoPresser Activado")
            for i in range(5):
                if self.use_key[i].get():
                    t = threading.Thread(target=self.press_key_repeatedly, args=(
                        self.key_to_press[i].get(), self.frequency[i].get()))
                    self.threads.append(t)
                    t.start()

            if self.use_mouse_left.get():
                t = threading.Thread(target=self.press_mouse_button_repeatedly, args=(
                    "left", self.left_mouse_interval.get()))
                self.threads.append(t)
                t.start()
                self.toggle_mouse_press()

            if self.use_mouse_right.get():
                t = threading.Thread(target=self.press_mouse_button_repeatedly, args=(
                    "right", self.right_mouse_interval.get()))
                self.threads.append(t)
                t.start()
                self.toggle_mouse_press()

            if self.monitor_hp_var.get():
                # Cambia aquí a monitor_hp
                t = threading.Thread(target=self.monitor_hp)
                self.threads.append(t)
                t.start()

    def update_similarity_label(self, _=None):
        self.similarity_label.config(
            text=f"Similitud: {self.hp_level.get():.2f}%")

        # Esta es la línea que vas a agregar:
        print(f"Actualizando label con similitud: {self.hp_level.get():.2f}%")

    def monitor_hp_bar(self):
        try:
            if not self.hp_key.get():
                self.message.set("¡Tecla de HP no definida!")
                return

            if None in [self.hp_bar_x1, self.hp_bar_y1, self.hp_bar_x2, self.hp_bar_y2]:
                self.message.set("¡Área de HP no definida!")
                return

            # Elige un punto en el centro de la barra de HP
            center_x = (self.hp_bar_x1 + self.hp_bar_x2) // 2
            center_y = (self.hp_bar_y1 + self.hp_bar_y2) // 2

            while self.should_press.is_set():
                # Obtiene el color del punto central
                color = pyautogui.pixel(center_x, center_y)

                # Si el color está en el rango rojo (puedes ajustar esto según el color exacto que indica salud baja en tu juego)
                if 120 <= color[0] <= 255 and 0 <= color[1] <= 50 and 0 <= color[2] <= 50:
                    keyboard.press(self.hp_key.get())
                    time.sleep(0.05)
                    keyboard.release(self.hp_key.get())
                    self.message.set("HP bajo, presionando tecla de curación")

                time.sleep(0.5)

        except Exception as e:
            self.message.set(f"Error al monitorear HP: {str(e)}")

    def set_potion_area_start(self):
        self.potion_area_x1, self.potion_area_y1 = pyautogui.position()
        self.potion_x1, self.potion_y1 = pyautogui.position()
        self.message.set(
            f"Posición inicial de poción establecida en: {self.potion_x1, self.potion_y1}")

    def color_in_range(self, color, target, threshold=15):
        """Comprueba si un color está en un cierto rango."""
        return all(abs(color[i] - target[i]) <= threshold for i in range(3))

    def set_potion_area_end(self):
        x2, y2 = pyautogui.position()
        screen_width, screen_height = pyautogui.size()

        # Asegurarse de que self.potion_x1 y self.potion_y1 contengan las coordenadas más pequeñas
        left = min(self.potion_x1, x2)
        top = min(self.potion_y1, y2)
        right = max(self.potion_x1, x2)
        bottom = max(self.potion_y1, y2)

        # Garantizar que las coordenadas estén dentro de los límites de la pantalla
        left = max(0, left)
        top = max(0, top)
        right = min(screen_width, right)
        bottom = min(screen_height, bottom)

        # Toma una captura del área seleccionada
        try:
            screenshot = pyautogui.screenshot(
                region=(left, top, right - left, bottom - top))
            screenshot_path = "potion_area_reference.png"
            screenshot.save(screenshot_path)
            self.message.set(
                f"Área de poción capturada y guardada como: {screenshot_path}")
        except Exception as e:
            self.message.set(f"Error al capturar la imagen: {str(e)}")

        # Detecta el color central del área
        center_x = (left + right) // 2
        center_y = (top + bottom) // 2
        color = pyautogui.pixel(center_x, center_y)
        self.message.set(f"Color detectado en el centro del área: {color}")

    def monitor_hp(self):
        # Monitorea el HP y realiza acciones según su nivel.
        if not self.reference_image_path:
            self.message.set("Imagen de referencia no cargada.")
            return

        reference_image = cv2.imread(
            self.reference_image_path, cv2.IMREAD_GRAYSCALE)
        while self.should_press.is_set():
            try:
                screenshot = pyautogui.screenshot(region=(self.hp_bar_x1, self.hp_bar_y1,
                                                          self.hp_bar_x2 - self.hp_bar_x1,
                                                          self.hp_bar_y2 - self.hp_bar_y1))
                screenshot_np = np.array(screenshot)
                gray_screenshot = cv2.cvtColor(
                    screenshot_np, cv2.COLOR_RGB2GRAY)

                # Calcula la similitud
                res = cv2.matchTemplate(
                    gray_screenshot, reference_image, cv2.TM_CCOEFF_NORMED)
                _, max_val, _, _ = cv2.minMaxLoc(res)

                if max_val * 100 < self.hp_level.get():
                    keyboard.press(self.hp_key.get())
                    time.sleep(0.05)
                    keyboard.release(self.hp_key.get())
                    self.message.set("HP bajo, presionando tecla de curación")

                time.sleep(0.2)  # Intervalo de monitoreo.

            except Exception as e:
                self.message.set(f"Error al monitorear HP: {str(e)}")

    def save_config(self):
        config = {
            "mouse_left": self.use_mouse_left.get(),
            "mouse_right": self.use_mouse_right.get(),
            "left_mouse_interval": self.left_mouse_interval.get(),
            "right_mouse_interval": self.right_mouse_interval.get(),
            "keys": [self.key_to_press[i].get() for i in range(5)],
            "frequencies": [self.frequency[i].get() for i in range(5)],
            "use_keys": [self.use_key[i].get() for i in range(5)],
            "reference_image_path": self.reference_image_path,
            "hp_key": self.hp_key.get(),
            "hp_level": self.hp_level.get(),
            "monitor_hp": self.monitor_hp_var.get(),  # Aquí hice el cambio
            "hp_bar": (self.hp_bar_x1, self.hp_bar_y1, self.hp_bar_x2, self.hp_bar_y2)
        }
        with open("config.json", "w") as file:
            json.dump(config, file)
        messagebox.showinfo("Información", "Configuración guardada.")

    def load_config(self):
        try:
            with open("config.json", "r") as file:
                config = json.load(file)
                self.use_mouse_left.set(config["mouse_left"])
                self.use_mouse_right.set(config["mouse_right"])
                self.left_mouse_interval.set(config["left_mouse_interval"])
                self.right_mouse_interval.set(config["right_mouse_interval"])
                for i in range(5):
                    self.key_to_press[i].set(config["keys"][i])
                    self.frequency[i].set(config["frequencies"][i])
                    self.use_key[i].set(config["use_keys"][i])
                self.reference_image_path = config["reference_image_path"]
                self.hp_key.set(config["hp_key"])
                self.hp_level.set(config["hp_level"])
                # Aquí hice el cambio
                self.monitor_hp_var.set(config["monitor_hp"])
                self.hp_bar_x1, self.hp_bar_y1, self.hp_bar_x2, self.hp_bar_y2 = config[
                    "hp_bar"]
            messagebox.showinfo("Información", "Configuración cargada.")
        except Exception as e:
            messagebox.showerror("Error", str(e))


if __name__ == "__main__":
    app = AutoKeyPresserImproved()
    app.mainloop()
