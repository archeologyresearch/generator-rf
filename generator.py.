import subprocess
import sys
import os
import time

# --- FUNKCJA DO INSTALACJI BIBLIOTEK ---
def install(package):
    print(f"Instaluję: {package}...")
    subprocess.check_call([sys.executable, "-m", "pip", "install", package])

# --- SPRAWDŹ I ZAINSTALUJ BRAKUJĄCE BIBLIOTEKI ---
required = ["vosk", "pyaudio", "pyserial"]
missing = []

for lib in required:
    try:
        __import__(lib)
    except ImportError:
        missing.append(lib)

if missing:
    print(f"Brakuje bibliotek: {missing}")
    for lib in missing:
        install(lib)
    print("\nZainstalowano. Uruchom skrypt ponownie.")
    sys.exit(0)

# --- DOPIERO TERAZ IMPORTUJEMY RESZTĘ ---
import zipfile
import urllib.request
import vosk
import pyaudio
import json
import serial
import serial.tools.list_ports

# --- KONFIGURACJA ---
MODEL_URL = "https://alphacephei.com/vosk/models/vosk-model-small-pl-0.22.zip"
MODEL_ZIP = "vosk-model-small-pl-0.22.zip"
MODEL_DIR = "vosk-model-small-pl-0.22"
BAUDRATE = 115200      # Prędkość zgodna z generatorem
# --------------------

def find_serial_port():
    """Automatycznie znajduje port COM podłączonego adaptera FT232."""
    ports = list(serial.tools.list_ports.comports())
    if not ports:
        print("Nie znaleziono żadnych portów COM.")
        return None

    print("Dostępne porty COM:")
    for i, port in enumerate(ports):
        print(f"  [{i}] {port.device} - {port.description}")

    # Preferuj porty z opisem zawierającym 'FT232', 'USB Serial' lub 'USB'
    for port in ports:
        desc = port.description.lower()
        if "ft232" in desc or "usb serial" in desc or "usb" in desc:
            print(f"Automatycznie wybrano: {port.device}")
            return port.device

    # Jeśli nie znaleziono preferowanego, wybierz pierwszy z listy
    print(f"Wybrano pierwszy dostępny port: {ports[0].device}")
    return ports[0].device

def ensure_model():
    """Sprawdza, czy model istnieje. Jeśli nie – pobiera i rozpakowuje."""
    if os.path.isdir(MODEL_DIR):
        print(f"Model już istnieje: {MODEL_DIR}")
        return MODEL_DIR
    if not os.path.isfile(MODEL_ZIP):
        print("Pobieram model Vosk (może to potrwać kilka minut)...")
        urllib.request.urlretrieve(MODEL_URL, MODEL_ZIP)
        print("Pobrano.")
    print("Rozpakowuję model...")
    with zipfile.ZipFile(MODEL_ZIP, 'r') as zip_ref:
        zip_ref.extractall(".")
    print(f"Rozpakowano do: {MODEL_DIR}")
    return MODEL_DIR

def text_to_bits(text):
    """Zamienia tekst na ciąg bitów (8-bitowy ASCII)."""
    return ''.join(format(ord(char), '08b') for char in text)

def main():
    # 1. Znajdź port COM
    port = find_serial_port()
    if not port:
        print("Nie można kontynuować bez portu COM. Podłącz FT232 i spróbuj ponownie.")
        return

    # 2. Przygotuj model Vosk
    model_path = ensure_model()
    model = vosk.Model(model_path)
    rec = vosk.KaldiRecognizer(model, 16000)

    # 3. Otwórz mikrofon
    p = pyaudio.PyAudio()
    stream = p.open(format=pyaudio.paInt16, channels=1, rate=16000,
                    input=True, frames_per_buffer=4096)

    # 4. Otwórz port szeregowy
    try:
        ser = serial.Serial(port, BAUDRATE, timeout=2)
        time.sleep(1)
        print(f"Połączono z {port} @ {BAUDRATE} bps")
    except serial.SerialException as e:
        print(f"Błąd portu {port}: {e}")
        print("Sprawdź, czy FT232 jest podłączony i czy port nie jest zajęty.")
        stream.stop_stream()
        stream.close()
        p.terminate()
        return

    print("Mów teraz... (Ctrl+C aby zakończyć)")

    # 5. Główna pętla: mowa → tekst → bity → UART
    try:
        while True:
            data = stream.read(4096)
            if rec.AcceptWaveform(data):
                result = json.loads(rec.Result())
                tekst = result.get("text", "").strip()
                if tekst:
                    bity = text_to_bits(tekst)
                    print(f"\nRozpoznano: {tekst}")
                    print(f"Bity ({len(bity)}): {bity}")
                    ser.write((bity + '\n').encode())
                    print("Wysłano do generatora.")
    except KeyboardInterrupt:
        print("\nZakończono.")
    finally:
        stream.stop_stream()
        stream.close()
        p.terminate()
        ser.close()

if __name__ == "__main__":
    main()
