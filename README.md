# Kai-AI
# === PHASE 1: IMPORTS & SETTINGS ===
import os
import subprocess
import pyttsx3
import speech_recognition as sr
import sqlite3
import datetime
import time
import threading
import shutil
import platform
import webbrowser
import zipfile
import requests
import json
import psutil
import pygame
import pyperclip
import keyboard

from PIL import Image
import fitz  # PyMuPDF
import pytesseract

from watchdog.observers import Observer
from watchdog.events import FileSystemEventHandler
from tkinter import Tk, Label, Text

# === CONFIG & SETTINGS ===
SETTINGS_FILE = "kai_settings.json"
JOURNAL_DB = "kai_journal.db"
GOAL_DB = "kai_goals.db"
KAI_PERSONALITY = "default"
LLM_MODE = "ollama"
CUSTOM_API_KEY = ""
CUSTOM_API_URL = ""
session_reminders = []
AUDIO_MAP = {
    "hype": "audio/hype.mp3",
    "calm": "audio/calm.mp3",
    "low": "audio/low.mp3"
}
WATCH_FOLDER = os.path.expanduser("~/Downloads")

def load_settings():
    if not os.path.exists(SETTINGS_FILE):
        return {"mode": "mentor"}
    with open(SETTINGS_FILE, "r") as f:
        return json.load(f)

def save_settings(settings):
    with open(SETTINGS_FILE, "w") as f:
        json.dump(settings, f)

settings = load_settings()
pygame.init()
pygame.mixer.init()
# === VOICE OUTPUT ===
def speak(text):
    engine = pyttsx3.init()
    voices = engine.getProperty('voices')
    mode = settings.get("mode", "mentor")

    if mode == "chill":
        engine.setProperty('rate', 160)
        engine.setProperty('voice', voices[1].id if len(voices) > 1 else voices[0].id)
    elif mode == "professional":
        engine.setProperty('rate', 190)
        engine.setProperty('voice', voices[0].id)
    elif mode == "hype":
        engine.setProperty('rate', 210)
    else:
        engine.setProperty('rate', 170)  # mentor

    print("Kai:", text)
    log_transcript("Kai", text)
    engine.say(text)
    engine.runAndWait()

# === MEMORY LOGGER ===
def remember_task(task_text):
    conn = sqlite3.connect('kai_memory.db')
    c = conn.cursor()
    c.execute('CREATE TABLE IF NOT EXISTS memory (timestamp TEXT, task TEXT)')
    timestamp = time.strftime("%Y-%m-%d %H:%M:%S")
    c.execute('INSERT INTO memory (timestamp, task) VALUES (?, ?)', (timestamp, task_text))
    conn.commit()
    conn.close()

# === TRANSCRIPT LOGGER ===
def log_transcript(speaker, message):
    with open("kai_transcript.txt", "a", encoding="utf-8") as f:
        f.write(f"{speaker}: {message}\n")

# === EMOTION DETECTION BASED ON AUDIO ENERGY ===
def analyze_emotion(audio_data):
    raw_data = audio_data.get_raw_data()
    energy = sum(abs(x - 128) for x in raw_data) / len(raw_data)
    if energy > 50:
        return "hype"
    elif 30 < energy <= 50:
        return "normal"
    elif 10 < energy <= 30:
        return "calm"
    else:
        return "low"

def respond_to_mood(mood):
    if mood == "hype":
        speak("You're bringing the energy! Let's ride that wave.")
    elif mood == "calm":
        speak("Nice and steady—I’m right here with you.")
    elif mood == "low":
        speak("Sounds like you might need a breather. Want to take a short break or hear something inspiring?")

# === LISTENING WITH MOOD ANALYSIS ===
def listen_and_analyze():
    recognizer = sr.Recognizer()
    mic = sr.Microphone()
    with mic as source:
        print("🎧 Listening (mood-aware)...")
        recognizer.adjust_for_ambient_noise(source)
        audio = recognizer.listen(source)

    try:
        user_text = recognizer.recognize_google(audio)
        detected_mood = analyze_emotion(audio)
        print(f"You: {user_text} | Mood: {detected_mood}")
        log_transcript("You", user_text)
        respond_to_mood(detected_mood)
        return user_text
    except sr.UnknownValueError:
        speak("I couldn’t catch that, want to try again?")
        return ""
# === FILE OPERATIONS ===
def open_file(filepath):
    if os.path.exists(filepath):
        speak(f"Opening file: {filepath}")
        os.startfile(filepath)
        remember_task(f"Opened file: {filepath}")
    else:
        speak("That file was not found.")
        remember_task(f"Missing file: {filepath}")

def get_last_opened_file():
    try:
        conn = sqlite3.connect('kai_memory.db')
        c = conn.cursor()
        c.execute("SELECT task FROM memory WHERE task LIKE 'Opened file:%' ORDER BY timestamp DESC LIMIT 1")
        row = c.fetchone()
        conn.close()
        if row:
            return row[0].replace("Opened file:", "").strip()
    except Exception as e:
        print("File recall error:", e)
    return None

def summarize_file(filepath):
    if filepath.endswith(".txt") and os.path.exists(filepath):
        with open(filepath, "r", encoding="utf-8") as f:
            lines = f.read().splitlines()
        summary = " ".join(lines[:3]) if len(lines) > 3 else "\n".join(lines)
        speak("Here’s a quick summary:")
        speak(summary)
        remember_task(f"Summarized file: {filepath}")
    else:
        speak("I can only summarize text files right now.")

def save_copy(filepath):
    if not os.path.exists(filepath):
        speak("Can’t find that file to save.")
        return
    name, ext = os.path.splitext(filepath)
    copy_path = f"{name}_copy{ext}"
    try:
        shutil.copy(filepath, copy_path)
        speak(f"I’ve saved a backup copy as {copy_path}")
        remember_task(f"Saved file copy: {copy_path}")
    except Exception as e:
        speak("Couldn’t create a copy.")
        print("File copy error:", e)

# === SYSTEM STATUS ===
def get_battery_status():
    battery = psutil.sensors_battery()
    if not battery:
        return "Battery info not available."
    return f"Battery is at {battery.percent}% and {'charging' if battery.power_plugged else 'not charging'}."

def get_cpu_status():
    return f"CPU is running at {psutil.cpu_percent(interval=1)}% usage."

def get_disk_usage():
    total, used, free = shutil.disk_usage("/")
    return f"Disk: {used // (2**30)}GB used of {total // (2**30)}GB. {free // (2**30)}GB free."

def get_system_info():
    return f"You’re on {platform.system()} version {platform.release()}."

def summarize_system_health():
    speak("Running full system scan...")
    try:
        summary = f"{get_system_info()}. {get_battery_status()}. {get_cpu_status()}. {get_disk_usage()}."
        print("=== SYSTEM STATUS ===\n" + summary)
        speak(summary)
        remember_task("System health check performed")
    except Exception as e:
        speak("Couldn’t complete the system check.")
        print("System scan error:", e)
        remember_task("System health scan failed.")

def suggest_next_step():
    hour = datetime.datetime.now().hour
    if hour >= 21:
        speak("It’s late. Want to back up files or plan tomorrow?")
    elif 17 <= hour < 21:
        speak("Evening. Need help logging work or relaxing the system?")
    elif 12 <= hour < 14:
        speak("Midday. Should I summarize tasks or open work files?")
    elif hour < 9:
        speak("Morning. Want your agenda or a health check?")
    else:
        speak("I’m ready. Say the word.")
    remember_task("Kai offered time-based suggestion")

# === MEMORY TASKS + FOLLOWUPS ===
def get_recent_tasks(limit=5):
    try:
        conn = sqlite3.connect('kai_memory.db')
        c = conn.cursor()
        c.execute('SELECT timestamp, task FROM memory ORDER BY timestamp DESC LIMIT ?', (limit,))
        rows = c.fetchall()
        conn.close()
        if not rows:
            speak("I don’t remember anything yet.")
            return
        speak(f"Your last {limit} events:")
        for t, task in rows:
            speak(f"{t}: {task}")
    except Exception as e:
        speak("Couldn’t read memory.")
        print("Memory error:", e)

def follow_up_from_action(task):
    t = task.lower()
    if "file" in t and "opened" in t:
        speak("Want me to summarize that or save a copy?")
    elif "system check" in t:
        speak("Should I optimize, free space, or rescan?")
    elif "shut down" in t:
        speak("Want me to close apps, sync memory, or sign out?")
    else:
        speak("Need to follow that up with anything?")

# === LLM ENGINE ROUTER ===
def ask_kai(prompt):
    try:
        result = subprocess.run(['ollama', 'run', 'mistral'], input=prompt.encode(), stdout=subprocess.PIPE)
        output = result.stdout.decode().strip()
        speak(output)
        remember_task(f"Kai LLM response to: {prompt}")
        return output
    except Exception as e:
        speak("Language model not responding.")
        print("LLM error:", e)
        return ""

def set_llm_mode(mode):
    global LLM_MODE
    LLM_MODE = mode.lower()
    speak(f"LLM set to {LLM_MODE}")

def call_openai(prompt):
    try:
        headers = {"Authorization": f"Bearer {CUSTOM_API_KEY}", "Content-Type": "application/json"}
        data = {"model": "gpt-3.5-turbo", "messages": [{"role": "user", "content": prompt}]}
        res = requests.post("https://api.openai.com/v1/chat/completions", json=data, headers=headers)
        out = res.json()["choices"][0]["message"]["content"]
        speak(out)
        return out
    except Exception as e:
        speak("OpenAI call failed.")
        print("OpenAI error:", e)
        return ""

def call_custom_llm(prompt):
    try:
        res = requests.post(CUSTOM_API_URL, json={"prompt": prompt})
        out = res.json()["response"]
        speak(out)
        return out
    except Exception as e:
        speak("Custom LLM failed.")
        print("Custom error:", e)
        return ""

def ask_llm_api(prompt):
    if LLM_MODE == "openai":
        return call_openai(prompt)
    elif LLM_MODE == "custom":
        return call_custom_llm(prompt)
    else:
        return ask_kai(prompt)
# === TIME & REMINDERS ===
def get_current_time():
    now = datetime.datetime.now()
    return now.strftime("It's %I:%M %p on %A, %B %d, %Y.")

def tell_time():
    now = get_current_time()
    speak(now)
    remember_task(f"Told time: {now}")

def set_verbal_reminder(message, target_hour):
    now = datetime.datetime.now()
    target_time = now.replace(hour=target_hour, minute=0, second=0, microsecond=0)
    if target_time < now:
        target_time += datetime.timedelta(days=1)
    delay = (target_time - now).total_seconds()

    def alert():
        speak(f"Hey Mark, reminder: {message}")
        remember_task(f"Reminder at {target_hour}: {message}")

    t = threading.Timer(delay, alert)
    t.start()
    session_reminders.append((target_time, message))
    speak(f"Got it. I’ll remind you at {target_time.strftime('%I:%M %p')}.")

def parse_reminder_command(text):
    import re
    pattern = r"remind me at (\d{1,2}) (am|pm)?(?: to| that)? (.+)"
    match = re.search(pattern, text.lower())
    if match:
        hour = int(match.group(1))
        suffix = match.group(2)
        message = match.group(3)

        if suffix == "pm" and hour < 12:
            hour += 12
        elif suffix == "am" and hour == 12:
            hour = 0
        set_verbal_reminder(message, hour)
    else:
        speak("Try saying: remind me at 4 pm to check email.")

# === JOURNALING ===
def detect_sentiment(text):
    t = text.lower()
    if any(w in t for w in ["sad", "tired", "anxious", "lonely", "stressed", "angry"]):
        return "low"
    elif any(w in t for w in ["happy", "excited", "grateful", "amazing", "blessed"]):
        return "hype"
    elif any(w in t for w in ["calm", "peaceful", "relaxed", "steady"]):
        return "calm"
    return "neutral"

def log_journal_entry(text, mood="neutral"):
    try:
        conn = sqlite3.connect(JOURNAL_DB)
        c = conn.cursor()
        c.execute('CREATE TABLE IF NOT EXISTS journal (timestamp TEXT, mood TEXT, entry TEXT)')
        ts = time.strftime("%Y-%m-%d %H:%M:%S")
        c.execute('INSERT INTO journal VALUES (?, ?, ?)', (ts, mood, text))
        conn.commit()
        conn.close()
        speak("Got it. I’ve saved your journal entry.")
    except Exception as e:
        speak("Something went wrong saving your journal.")
        print("Journal error:", e)

def start_voice_journal():
    speak("Alright, I’m listening. Share your thoughts.")
    entry = listen_and_analyze()
    mood = detect_sentiment(entry)
    log_journal_entry(entry, mood)
    if mood == "low":
        speak("Want to hear something positive or take a break?")
    elif mood == "hype":
        speak("You sound fired up. Let’s keep that energy going!")
    elif mood == "calm":
        speak("Smooth energy—journaling helps maintain it.")

def recall_journal_entries(limit=5):
    try:
        conn = sqlite3.connect(JOURNAL_DB)
        c = conn.cursor()
        c.execute('SELECT timestamp, mood, entry FROM journal ORDER BY timestamp DESC LIMIT ?', (limit,))
        rows = c.fetchall()
        conn.close()
        if not rows:
            speak("You haven’t logged any entries yet.")
            return
        speak(f"Here are your last {limit} reflections:")
        for ts, mood, entry in rows:
            snippet = entry[:60] + ("..." if len(entry) > 60 else "")
            speak(f"{ts} — mood {mood}. You said: {snippet}")
    except Exception as e:
        speak("Couldn’t load your journal.")
        print("Journal read error:", e)

# === MUSIC + BREATHING SUPPORT ===
def play_mood_music(mood):
    path = AUDIO_MAP.get(mood)
    if not path or not os.path.exists(path):
        speak("No track found for that mood.")
        return
    try:
        pygame.mixer.music.load(path)
        pygame.mixer.music.play()
        speak(f"Playing something {mood} for you.")
        remember_task(f"Played {mood} track")
    except Exception as e:
        speak("Couldn’t play music.")
        print("Music error:", e)

def stop_music():
    pygame.mixer.music.stop()
    speak("Music stopped.")

def breathing_coach():
    speak("Let’s breathe together. Inhale... two... three... four...")
    time.sleep(4)
    speak("Hold... two... three...")
    time.sleep(3)
    speak("Exhale... two... three... four...")
    time.sleep(4)
# === VISUAL OCR READER ===
def ocr_image(image_path):
    try:
        img = Image.open(image_path)
        return pytesseract.image_to_string(img)
    except Exception as e:
        speak("Couldn’t read the image.")
        print("OCR error:", e)
        return ""

def read_pdf(pdf_path):
    try:
        doc = fitz.open(pdf_path)
        text = ""
        for page in doc:
            text += page.get_text()
        doc.close()
        return text
    except Exception as e:
        speak("PDF reading failed.")
        print("PDF error:", e)
        return ""

def read_visual_file(filepath):
    if not os.path.exists(filepath):
        speak("That file doesn’t exist.")
        return

    ext = os.path.splitext(filepath)[1].lower()
    if ext in [".png", ".jpg", ".jpeg", ".bmp"]:
        text = ocr_image(filepath)
        if text:
            speak("Here's what I found in the image:")
            speak(text[:300])
            remember_task(f"Read image {filepath}")
    elif ext == ".pdf":
        text = read_pdf(filepath)
        if text:
            speak("Summary from the PDF:")
            speak(text[:400])
            remember_task(f"Read PDF {filepath}")
    else:
        speak("I only read PDFs and image files right now.")

# === FOLDER MONITOR ===
class KaiFolderMonitor(FileSystemEventHandler):
    def on_created(self, event):
        if not event.is_directory:
            filepath = event.src_path
            ext = os.path.splitext(filepath)[1].lower()
            speak(f"New file detected: {os.path.basename(filepath)}")
            remember_task(f"Detected new file: {filepath}")
            if ext in [".pdf", ".png", ".jpg", ".jpeg", ".txt"]:
                speak("Want me to read it?")
                reply = listen_and_analyze().lower()
                if "yes" in reply:
                    read_visual_file(filepath)
                else:
                    speak("Okay. Logged but won’t process.")

def start_folder_monitor():
    event_handler = KaiFolderMonitor()
    observer = Observer()
    observer.schedule(event_handler, path=WATCH_FOLDER, recursive=False)
    t = threading.Thread(target=observer.start, daemon=True)
    t.start()
    speak(f"Watching {WATCH_FOLDER} for new files.")
    remember_task(f"Started watching {WATCH_FOLDER}")

# === INTERPRET COMMANDS ===
def interpret_contextual_command(text):
    text = text.lower()
    if "open file" in text:
        path = text.replace("open file", "").strip()
        open_file(path)
    elif "summarize last file" in text:
        path = get_last_opened_file()
        if path:
            summarize_file(path)
        else:
            speak("No recent file found.")
    elif "remind me at" in text:
        parse_reminder_command(text)
    elif "what time is it" in text or "tell me the time" in text:
        tell_time()
    elif "start journal" in text:
        start_voice_journal()
    elif "show journal" in text:
        recall_journal_entries()
    elif "system check" in text:
        summarize_system_health()
    elif "suggest something" in text or "what should i do" in text:
        suggest_next_step()
    elif "play mood music" in text:
        speak("Which mood—hype, calm, or low?")
        mood = listen_and_analyze().lower()
        play_mood_music(mood)
    elif "stop music" in text:
        stop_music()
    elif "help me relax" in text:
        play_mood_music("calm")
        breathing_coach()
    elif "log a goal" in text:
        speak("What’s your goal?")
        g = listen_and_analyze()
        log_goal(g)
    elif "track habit" in text:
        speak("Name the habit.")
        h = listen_and_analyze()
        log_habit(h)
    elif "my habits" in text or "check habits" in text:
        check_habits()
    elif "kai summary" in text or "show dashboard" in text:
        kai_summary()
    elif "read" in text and "pdf" in text:
        speak("Give the full path.")
        path = listen_and_analyze()
        read_visual_file(path)
    elif "move file" in text:
        speak("Source path?")
        src = listen_and_analyze()
        speak("Destination folder?")
        tgt = listen_and_analyze()
        move_and_rename_file(src, tgt)
    elif "export memory" in text:
        export_memory_json()
    elif "import memory" in text:
        speak("Path to .json file?")
        p = listen_and_analyze()
        import_memory_json(p)
    else:
        ask_llm_api(text)

# === MAIN ENGINE ===
def main():
    speak(f"Kai is online in {settings['mode']} mode.")
    summarize_system_health()
    suggest_next_step()
    start_folder_monitor()
    while True:
        command = listen_and_analyze().lower()
        interpret_contextual_command(command)

if __name__ == "__main__":
    t1 = threading.Thread(target=main)
    t1.start()
