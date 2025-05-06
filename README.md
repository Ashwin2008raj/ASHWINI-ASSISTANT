#import sys
import pyttsx3
import speech_recognition as sr
import webbrowser
import subprocess
import requests
import datetime
import os
import random
import pyautogui
import wikipedia
import pygame

from PyQt5.QtWidgets import *
from PyQt5.QtCore import Qt, QTimer, QUrl
from PyQt5.QtGui import QPalette, QColor
from PyQt5.QtMultimedia import QMediaPlayer, QMediaContent
from PyQt5.QtMultimediaWidgets import QVideoWidget
from threading import Thread

# ========== TTS Setup ==========
engine = pyttsx3.init()
voices = engine.getProperty('voices')
engine.setProperty('voice', voices[1].id)
engine.setProperty('rate', 150)

recognizer = sr.Recognizer()
WEATHER_API_KEY = "709b914e635beec6f421b723ecd0ccda"

# ========== Speak ==========
def speak(text):
    engine.say(text)
    engine.runAndWait()

# ========== Listen ==========
def listen_command():
    with sr.Microphone() as source:
        recognizer.adjust_for_ambient_noise(source, duration=0.5)
        print("Listening...")
        try:
            audio = recognizer.listen(source, timeout=3, phrase_time_limit=5)
            command = recognizer.recognize_google(audio).lower()
            print(f"Recognized: {command}")
            return command
        except:
            return ""

# ========== AI Integration ==========
def ask_ai(question):
    try:
        response = requests.post("http://localhost:11434/api/generate", json={
            "model": "llama3",
            "prompt": question,
            "stream": False
        })
        return response.json().get("response", "Sorry, no reply.")
    except Exception as e:
        return str(e)

# ========== Internet Features ==========
def get_weather(city):
    url = f"https://api.openweathermap.org/data/2.5/weather?q={city}&appid={WEATHER_API_KEY}&units=metric"
    try:
        res = requests.get(url, timeout=4)
        data = res.json()
        if data.get("cod") != 200:
            return f"No weather data for {city}."
        temp = data['main']['temp']
        desc = data['weather'][0]['description']
        return f"{city}: {desc}, {temp}°C"
    except:
        return "Weather service unavailable."

def get_simple_weather():
    try:
        res = requests.get("https://wttr.in/Pondicherry?format=3", timeout=4)
        return res.text
    except:
        return "Simple weather unavailable."

def get_joke():
    try:
        res = requests.get("https://official-joke-api.appspot.com/random_joke", timeout=4).json()
        return f"{res['setup']} ... {res['punchline']}"
    except:
        return "No joke right now."

def get_fact():
    try:
        res = requests.get("https://uselessfacts.jsph.pl/random.json?language=en", timeout=4).json()
        return res['text']
    except:
        return "No fact found."

def wiki_summary(query):
    try:
        return wikipedia.summary(query, sentences=2)
    except:
        return "Wikipedia has no info."

# ========== Message Reader ==========
def read_messages():
    try:
        with open("messages.txt", "r") as f:
            content = f.read()
            if content.strip():
                return content.strip()
            else:
                return "There are no messages."
    except:
        return "Could not read messages."

# ========== GUI ==========
class AshwiniGUI(QWidget):
    def __init__(self):
        super().__init__()
        self.setWindowTitle("ASHWINI 1.0")
        self.setGeometry(100, 100, 1000, 600)
        self.setStyleSheet("background: transparent;")

        self.videoWidget = QVideoWidget(self)
        self.mediaPlayer = QMediaPlayer(None, QMediaPlayer.VideoSurface)
        self.mediaPlayer.setVideoOutput(self.videoWidget)
        self.mediaPlayer.setMedia(QMediaContent(QUrl.fromLocalFile("background.mp4")))
        self.mediaPlayer.setVolume(0)
        self.mediaPlayer.play()
        self.mediaPlayer.mediaStatusChanged.connect(self.loop_video)

        self.panel = QWidget(self)
        self.panel.setGeometry(250, 150, 500, 300)
        self.panel.setStyleSheet("""
            background-color: rgba(0, 0, 0, 120);
            border: 2px solid cyan;
            border-radius: 20px;
        """)

        self.label = QLabel("ASHWINI: Listening...", self.panel)
        self.label.setStyleSheet("""
            color: cyan;
            font-size: 24px;
            font-weight: bold;
            qproperty-alignment: AlignCenter;
        """)
        self.label.setAlignment(Qt.AlignCenter)

        self.wave = QProgressBar(self.panel)
        self.wave.setMaximum(100)
        self.wave.setTextVisible(False)
        self.wave.setStyleSheet("QProgressBar::chunk { background-color: cyan; }")

        layout = QVBoxLayout(self.panel)
        layout.addWidget(self.label)
        layout.addWidget(self.wave)

        self.timer = QTimer()
        self.timer.timeout.connect(self.animate_waveform)
        self.timer.start(150)

        self.show()

    def update_text(self, text):
        self.label.setText(f"ASHWINI: {text}")

    def animate_waveform(self):
        self.wave.setValue(random.randint(10, 100))

    def loop_video(self, status):
        if status == QMediaPlayer.EndOfMedia:
            self.mediaPlayer.setPosition(0)
            self.mediaPlayer.play()

# ========== Command Execution ==========
def run_commands(command, gui):
    if "youtube" in command:
        speak("Opening YouTube")
        gui.update_text("Opening YouTube")
        webbrowser.open("https://www.youtube.com")

    elif "search" in command:
        query = command.replace("search", "").strip()
        speak(f"Searching {query}")
        gui.update_text(f"Searching {query}")
        webbrowser.open(f"https://www.google.com/search?q={query}")

    elif "weather in" in command:
        city = command.replace("weather in", "").strip()
        result = get_weather(city)
        speak(result)
        gui.update_text(result)

    elif "weather" in command:
        result = get_simple_weather()
        speak(result)
        gui.update_text(result)

    elif "joke" in command:
        joke = get_joke()
        speak(joke)
        gui.update_text(joke)

    elif "fact" in command:
        fact = get_fact()
        speak(fact)
        gui.update_text(fact)

    elif "wikipedia" in command:
        query = command.replace("wikipedia", "").strip()
        result = wiki_summary(query)
        speak(result)
        gui.update_text(result)

    elif "read message" in command or "read messages" in command:
        result = read_messages()
        speak(result)
        gui.update_text(result)

    elif "time" in command:
        now = datetime.datetime.now().strftime("%I:%M %p")
        speak(f"The time is {now}")
        gui.update_text(f"Time: {now}")

    elif "open notepad" in command:
        subprocess.Popen(['notepad.exe'])
        speak("Opening Notepad")
        gui.update_text("Opened Notepad")

    elif "open chrome" in command:
        subprocess.Popen(["C:\\Program Files\\Google\\Chrome\\Application\\chrome.exe"])
        speak("Opening Chrome")
        gui.update_text("Opened Chrome")

    elif "exit" in command or "quit" in command:
        speak("Goodbye!")
        gui.update_text("Exiting...")
        sys.exit()

    elif "move mouse" in command:
        pyautogui.moveTo(100, 100)
        speak("Mouse moved.")
        gui.update_text("Mouse moved")

    elif "click" in command:
        pyautogui.click()
        speak("Clicked.")
        gui.update_text("Mouse clicked")

    elif "scroll down" in command:
        pyautogui.scroll(-500)
        speak("Scrolling down")
        gui.update_text("Scrolled down")

    elif "type" in command:
        text = command.replace("type", "").strip()
        pyautogui.typewrite(text)
        speak(f"Typing {text}")
        gui.update_text(f"Typed: {text}")

    else:
        result = ask_ai(command)
        speak(result)
        gui.update_text(result)

# ========== Wake Word ==========
def listen_for_wake_word(gui):
    with sr.Microphone() as source:
        recognizer.adjust_for_ambient_noise(source)
        while True:
            try:
                audio = recognizer.listen(source, timeout=5)
                text = recognizer.recognize_google(audio).lower()
                if "ashwini" in text:
                    speak("Yes, sir?")
                    gui.update_text("Yes, sir?")
                    break
            except:
                continue

# ========== Main ==========
def main():
    app = QApplication(sys.argv)
    gui = AshwiniGUI()
    speak("Ashwini assistant activated.")

    def start_listening():
        listen_for_wake_word(gui)
        while True:
            command = listen_command()
            if command:
                run_commands(command, gui)

    thread = Thread(target=start_listening)
    thread.start()
    sys.exit(app.exec_())

if __name__ == "__main__":
    main()

