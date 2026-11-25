# J.A.R.V.I.S
# Copyright (c) 2026 Hazem Shbel. All rights reserved.
# Proprietary and confidential. Copying, distributing, or modifying this
# file without written permission from the author is prohibited.

"""
JARVIS Assistant - Background App (Step 6: cloned voice added)
------------------------------------------------------------------
Runs a wake-word listener as a background app with a system tray icon.

- A control window shows the conversation, lets you type commands and has
  buttons for common actions. Close it and JARVIS shrinks to a floating orb;
  double-click the orb or use the tray icon to bring the window back.
  Start with `python jarvis_app.py --orb` to begin as just the orb.
- Right-click the tray icon to open the window, switch the microphone or quit.
- Microphone input uses `sounddevice` instead of PyAudio, so it installs
  on any recent Python version without a C++ compiler.
- Speech-to-text still uses Google's free web recognizer through the
  SpeechRecognition package (needs internet).

Install everything with the SAME Python you run the script with:
    python -m pip install SpeechRecognition sounddevice pystray pillow pyttsx3

JARVIS only does what is on the allowed lists in the "Abilities" section below
(apps, websites, volume, time/date, web search). He cannot run anything else.

JARVIS talks back in the voice from your reference clip (jarvis_voice.wav),
copied on your own computer with Pocket TTS. The one-time setup is in the
"Your own voice" settings below. Check it any time with:
    python jarvis_app.py --test-voice
If the cloned voice isn't set up (or fails), he falls back to a calm British
neural voice (edge-tts, needs internet), and after that to pyttsx3 (offline,
uses the voices installed on your computer).

Optional extras:
    python -m pip install pocket-tts     (the cloned voice)
    python -m pip install edge-tts       (the backup neural voice)
Run `python jarvis_app.py --voices` to list the offline fallback voices.
"""

import os
import re
import math
import subprocess
import sys
import time
import ctypes
import datetime
import webbrowser
import urllib.parse
import threading
import collections
import importlib
import queue

try:
    import tkinter as tk
except ImportError:
    tk = None  # HUD overlay is skipped if Tk isn't part of this Python install

# ---------------------------------------------------------------------------
# Dependency check: tells you the exact command for the Python that is
# actually running this file (fixes the "installed it, still missing" loop).
# ---------------------------------------------------------------------------
REQUIRED = {
    "speech_recognition": "SpeechRecognition",
    "sounddevice": "sounddevice",
    "pystray": "pystray",
    "PIL": "pillow",
    "pyttsx3": "pyttsx3",
    "numpy": "numpy",
}
if sys.version_info >= (3, 13):
    REQUIRED["audioop"] = "audioop-lts"  # audioop was removed from Python 3.13


def check_dependencies():
    missing = []
    for module, pip_name in REQUIRED.items():
        try:
            importlib.import_module(module)
        except Exception as e:  # ImportError, or OSError if PortAudio is missing
            missing.append((pip_name, e))
    if missing:
        names = " ".join(name for name, _ in missing)
        print(f"Python in use: {sys.executable}")
        print("Missing or broken packages:")
        for name, err in missing:
            print(f"  - {name}: {err}")
        print("\nInstall them with this exact command:\n")
        print(f'  "{sys.executable}" -m pip install {names}\n')
        sys.exit(1)


check_dependencies()

import audioop  # noqa: E402
np = importlib.import_module("numpy")  # noqa: E402
import sounddevice as sd  # noqa: E402
import speech_recognition as sr  # noqa: E402
import pyttsx3  # noqa: E402
import pystray  # noqa: E402
from PIL import Image, ImageDraw  # noqa: E402

# ---------------------------------------------------------------------------
# Settings you can tweak
# ---------------------------------------------------------------------------
WAKE_WORD = "jarvis"

SAMPLE_RATE = 16000          # Hz, good for speech recognition
SAMPLE_WIDTH = 2             # bytes per sample (int16)
BLOCK_FRAMES = 1024          # frames read from the mic at a time (~64 ms)
CALIBRATION_BLOCKS = 8       # ~0.5 s of room noise measured before listening
PREROLL_BLOCKS = 5           # audio kept from just before speech starts
SILENCE_BLOCKS = 12          # ~0.8 s of quiet ends a phrase
MIN_PHRASE_BLOCKS = 5        # ignore blips shorter than ~0.3 s
NOISE_MULTIPLIER = 1.8       # speech must be this much louder than the room
MIN_THRESHOLD = 50           # lowest volume that counts as speech; lower this
                             # if JARVIS doesn't hear you, raise it if it
                             # triggers on background noise

# --- Voice (text-to-speech) settings ---
# Main voice: Microsoft neural voice through edge-tts (free, needs internet).
# Other British male options: "en-GB-ThomasNeural". Try them and keep your pick.
EDGE_VOICE = "en-GB-RyanNeural"
EDGE_RATE = "-5%"            # slightly slower = calmer, more butler-like
EDGE_PITCH = "-3Hz"          # slightly deeper

VOICE_RATE = 175             # words per minute; lower = slower, calmer
VOICE_VOLUME = 1.0           # 0.0 to 1.0
# JARVIS uses the first installed voice whose name/id contains one of these
# words (checked in order). Run `python jarvis_app.py --voices` to see yours.
# On Windows, "George" is the British male voice (add it in Settings >
# Time & language > Speech > Add voices).
VOICE_HINTS = ["george", "ryan", "daniel", "en-gb", "british"]

# --- Your own voice (voice cloning) ---
# JARVIS copies the voice from a short, clean recording (5-15 seconds of one
# person talking). It runs on your computer with Pocket TTS, so after a
# one-time download it works offline. One-time setup:
#   1. python -m pip install pocket-tts
#   2. Make a free Hugging Face account, open
#      https://huggingface.co/kyutai/pocket-tts and accept the terms. (They
#      say to only clone voices you have permission to use.)
#   3. python -c "from huggingface_hub import login; login()"
#      and paste an access token from https://huggingface.co/settings/tokens
#   4. Keep the voice clip next to this script, then test it with:
#      python jarvis_app.py --test-voice
# Anything missing or broken? JARVIS just falls back to the Edge voice above.
CLONE_VOICE = True                  # False = never use the cloned voice
VOICE_SAMPLES = ("jarvis_voice.wav", "J_A_R_V_I_S_2_0.mp3")   # first one found is used
CLONE_LOAD_SECONDS = 300            # how long the very first start may wait for the download
CLONE_PREBUFFER = 0.4               # seconds of speech made before playback starts (smoothness)
PRONUNCIATION = {"JARVIS": "Jarvis"}   # spell words the way they should be said out loud

# ---------------------------------------------------------------------------
# Abilities: JARVIS can ONLY do what is listed here. Add or remove freely.
# ---------------------------------------------------------------------------
# "open <name>" launches these programs (Windows). Name -> what to launch.
ALLOWED_APPS = {
    "notepad": "notepad.exe",
    "calculator": "calc.exe",
    "paint": "mspaint.exe",
    "file explorer": "explorer.exe",
    "explorer": "explorer.exe",
    "task manager": "taskmgr.exe",
    "settings": "ms-settings:",
    "chrome": "chrome",
    "edge": "msedge",
    "spotify": "spotify:",
}

# "close <name>" can close ONLY these (Windows process names, tried in order).
# File Explorer is deliberately missing: closing it would kill your taskbar.
CLOSABLE_PROCESSES = {
    "notepad": ["notepad.exe"],
    "calculator": ["CalculatorApp.exe", "Calculator.exe", "calc.exe"],
    "paint": ["mspaint.exe"],
    "task manager": ["Taskmgr.exe"],
    "settings": ["SystemSettings.exe"],
    "chrome": ["chrome.exe"],
    "edge": ["msedge.exe"],
    "spotify": ["Spotify.exe"],
}

OPEN_VERBS = ("open", "launch", "start", "run")
CLOSE_VERBS = ("close", "quit", "exit", "stop", "kill", "shut")

# "open <name>" opens these websites in your default browser.
ALLOWED_SITES = {
    "youtube": "https://www.youtube.com",
    "google": "https://www.google.com",
    "gmail": "https://mail.google.com",
    "github": "https://github.com",
    "maps": "https://maps.google.com",
}

# Shared state between the listener thread and the tray icon
state = {"status": "starting", "last_command": None, "quit_after_speaking": False,
         "color_key": "idle", "mic_enabled": True, "voice_on": True}
ui_events = queue.Queue()   # background threads post here; the window reads it


def log_chat(who, text):
    """Add a line to the conversation shown in the window (who: 'you' or 'jarvis')."""
    ui_events.put(("chat", (who, text)))
icon_ref = {"icon": None}
stop_event = threading.Event()

# JARVIS colour palette (matches the cyan/gold HUD look used elsewhere)
HUD_CYAN = (45, 226, 255, 255)    # idle / listening
HUD_GOLD = (242, 182, 50, 255)    # wake word heard / speaking
HUD_RED = (224, 52, 58, 255)      # error
HUD_GREY = (123, 139, 147, 255)   # microphone switched off


def make_icon_image(color):
    """Generate a soft glowing ring icon in JARVIS's cyan/gold HUD style."""
    scale = 4
    size = 64 * scale
    img = Image.new("RGBA", (size, size), (0, 0, 0, 0))
    draw = ImageDraw.Draw(img)
    cx = cy = size / 2
    r_outer, r_inner = size * 0.44, size * 0.27
    rgb = color[:3]
    for i in range(6, 0, -1):               # soft outward glow
        alpha = int(45 * (i / 6))
        rr = r_outer + i * 5
        draw.ellipse([cx - rr, cy - rr, cx + rr, cy + rr],
                     outline=rgb + (alpha,), width=5)
    draw.ellipse([cx - r_outer, cy - r_outer, cx + r_outer, cy + r_outer],
                 outline=color, width=int(size * 0.05))
    draw.ellipse([cx - r_inner, cy - r_inner, cx + r_inner, cy + r_inner], fill=color)
    return img.resize((64, 64), Image.LANCZOS)


ICON_IDLE = make_icon_image(HUD_CYAN)        # cyan = listening
ICON_ACTIVE = make_icon_image(HUD_GOLD)      # gold = wake word heard
ICON_ERROR = make_icon_image(HUD_RED)        # red = error
ICON_SPEAKING = make_icon_image(HUD_GOLD)    # gold = JARVIS talking
ICON_MUTED = make_icon_image(HUD_GREY)       # grey = microphone off

def set_status(text, image=None):
    state["status"] = text
    if image is not None:
        if image is ICON_IDLE:
            state["color_key"] = "idle"
        elif image is ICON_ACTIVE:
            state["color_key"] = "active"
        elif image is ICON_ERROR:
            state["color_key"] = "error"
        elif image is ICON_SPEAKING:
            state["color_key"] = "speaking"
        elif image is ICON_MUTED:
            state["color_key"] = "muted"
    icon = icon_ref["icon"]
    if icon is not None and image is not None:
        icon.icon = image


# ---------------------------------------------------------------------------
# Voice output (text-to-speech)
# ---------------------------------------------------------------------------
speak_lock = threading.Lock()


def pick_voice_id(engine):
    """Return the id of the best matching installed voice, or None."""
    voices = engine.getProperty("voices")
    for hint in VOICE_HINTS:
        for v in voices:
            if hint in f"{v.name} {v.id}".lower():
                return v.id
    return None


def list_voices():
    engine = pyttsx3.init()
    print("Installed voices:")
    for v in engine.getProperty("voices"):
        print(f"  - {v.name}   (id: {v.id})")
    engine.stop()


def play_mp3_windows(path):
    """Play an mp3 with Windows' built-in media engine (no extra packages)."""
    import ctypes
    mci = ctypes.windll.winmm.mciSendStringW
    alias = "jarvis_voice"
    mci(f'open "{path}" type mpegvideo alias {alias}', None, 0, 0)
    mci(f"play {alias} wait", None, 0, 0)
    mci(f"close {alias}", None, 0, 0)


def speak_edge(text):
    """Speak with the edge-tts neural voice. Raises if unavailable."""
    import asyncio
    import os
    import tempfile
    import edge_tts

    fd, path = tempfile.mkstemp(suffix=".mp3")
    os.close(fd)
    try:
        asyncio.run(
            edge_tts.Communicate(
                text, EDGE_VOICE, rate=EDGE_RATE, pitch=EDGE_PITCH
            ).save(path)
        )
        play_mp3_windows(path)
    finally:
        try:
            os.remove(path)
        except OSError:
            pass


def speak_offline(text):
    """Speak with the offline pyttsx3 voice."""
    try:  # Windows: speech engine needs COM initialised in this thread
        import pythoncom  # type: ignore
        pythoncom.CoInitialize()
    except ImportError:
        pass
    engine = pyttsx3.init()
    voice_id = pick_voice_id(engine)
    if voice_id:
        engine.setProperty("voice", voice_id)
    engine.setProperty("rate", VOICE_RATE)
    engine.setProperty("volume", VOICE_VOLUME)
    engine.say(text)
    engine.runAndWait()
    engine.stop()


# ---------------------------------------------------------------------------
# Cloned voice (Pocket TTS): speaks in the voice of your reference clip
# ---------------------------------------------------------------------------
APP_DIR = os.path.dirname(os.path.abspath(__file__))


def find_voice_sample():
    """Full path of the first reference clip found next to this script, or None."""
    for name in VOICE_SAMPLES:
        path = os.path.join(APP_DIR, name)
        if os.path.isfile(path):
            return path
    return None


def pick_output_device():
    """
    Return the MME output device index if available, else None (default).
    Same idea as the microphone: MME is the most forgiving Windows audio driver.
    """
    try:
        for api in sd.query_hostapis():
            if "mme" in api["name"].lower():
                dev = api.get("default_output_device", -1)
                if dev is not None and dev >= 0:
                    return dev
    except Exception as e:
        print(f"[Audio] couldn't inspect host APIs, using the default speakers ({e})")
    return None


def explain_load_error(e):
    """Turn a Pocket TTS loading error into (short reason, what to do about it)."""
    text = f"{type(e).__name__} {e}".lower()
    if any(w in text for w in ("gated", "401", "403", "unauthorized", "forbidden",
                               "authenticat", "log in", "login")):
        return ("Hugging Face access needed",
                "One-time setup: accept the terms at https://huggingface.co/kyutai/pocket-tts, then run:\n"
                f'  "{sys.executable}" -c "from huggingface_hub import login; login()"\n'
                "and paste an access token from https://huggingface.co/settings/tokens")
    if any(w in text for w in ("connection", "timed out", "timeout", "offline",
                               "getaddrinfo", "name resolution", "network")):
        return ("couldn't download the voice model",
                "The first start needs internet to download it. Check your connection and restart.")
    if any(w in text for w in ("winerror 126", "winerror 1114", "dll")):
        return (f"{type(e).__name__}: {e}",
                "PyTorch couldn't start. Install the latest Microsoft Visual C++ "
                "Redistributable (x64) from Microsoft, then restart.")
    return (f"{type(e).__name__}: {e}",
            "Run `python jarvis_app.py --test-voice` in a console to see the full details.")


class ClonedVoice:
    """
    JARVIS's own voice. Loads Pocket TTS once in the background, copies the
    voice from your reference clip, then turns text into speech on demand.
    Nothing here is required: if it can't load, `error` says why and speak()
    (further down) falls back to the Edge voice.
    """

    def __init__(self):
        self.model = None
        self.voice_state = None
        self.output_device = None
        self.error = None                   # short reason it isn't available
        self.done = threading.Event()       # set once loading has finished (worked or not)
        self._started_at = None
        self._start_lock = threading.Lock()

    @property
    def ready(self):
        return self.model is not None and self.voice_state is not None

    def start_loading(self):
        """Begin loading in the background. Safe to call more than once."""
        with self._start_lock:
            if self._started_at is not None:
                return
            self._started_at = time.monotonic()
        threading.Thread(target=self._load, daemon=True).start()

    def wait_ready(self):
        """
        Wait for loading to finish (at most CLONE_LOAD_SECONDS after it began).
        True means the cloned voice can be used right now.
        """
        if not CLONE_VOICE:
            return False
        self.start_loading()
        left = CLONE_LOAD_SECONDS - (time.monotonic() - self._started_at)
        self.done.wait(max(0.0, left))
        return self.ready

    def _fail(self, short, fix=None):
        self.error = short
        print(f"[Voice] cloned voice unavailable: {short}. Using the backup voice.")
        if fix:
            print(f"[Voice] {fix}")
        log_chat("jarvis", f"(My cloned voice isn't available: {short}. Using the backup voice.)")

    def _load(self):
        try:
            sample = find_voice_sample()
            if sample is None:
                return self._fail("voice clip not found",
                                  "Put one of these files next to jarvis_app.py: "
                                  + ", ".join(VOICE_SAMPLES))
            try:
                # This is an optional dependency, so load it dynamically to
                # avoid a static unresolved-import warning when it isn't installed.
                import importlib
                pocket_tts = importlib.import_module("pocket_tts")
                TTSModel = pocket_tts.TTSModel
                export_model_state = pocket_tts.export_model_state
            except ImportError:
                return self._fail("pocket-tts isn't installed",
                                  f'Install it with:  "{sys.executable}" -m pip install pocket-tts')
            print("[Voice] loading the JARVIS voice (the first run downloads the model)...")
            model = TTSModel.load_model()
            voice = self._voice_for(model, sample, export_model_state)
            self.output_device = pick_output_device()
            self.model, self.voice_state = model, voice
            print(f"[Voice] JARVIS voice ready (copied from {os.path.basename(sample)})")
        except Exception as e:
            self._fail(*explain_load_error(e))
        finally:
            self.done.set()

    def _voice_for(self, model, sample, export_model_state):
        """
        Get the voice from the clip. Reading the clip is the slow part, so the
        result is saved next to it and reused on later starts (it is redone
        automatically if you replace the clip or update pocket-tts).
        """
        try:
            from importlib import metadata
            version = metadata.version("pocket-tts")
        except Exception:
            version = "x"
        cache = f"{os.path.splitext(sample)[0]}.pocket-{version}.safetensors"
        if os.path.isfile(cache) and os.path.getmtime(cache) >= os.path.getmtime(sample):
            try:
                return model.get_state_for_audio_prompt(cache)
            except Exception as e:
                print(f"[Voice] the saved voice file couldn't be read ({e}); rebuilding it")
        voice = model.get_state_for_audio_prompt(sample)
        try:
            export_model_state(voice, cache)
        except Exception as e:
            print(f"[Voice] couldn't save the voice file ({e}); it will be rebuilt next time")
        return voice

    def _open_output(self, rate):
        """Open the speakers: Windows' MME driver first, then whatever the default is."""
        last_error = None
        for device in dict.fromkeys((self.output_device, None)):
            try:
                out = sd.OutputStream(samplerate=rate, channels=1, dtype="float32",
                                      device=device, latency="high")
                out.start()
                return out
            except Exception as e:
                last_error = e
        raise last_error

    def speak(self, text):
        """
        Say `text` in the cloned voice and wait until it has finished playing.
        Sound starts after about CLONE_PREBUFFER seconds of speech have been
        made, then keeps playing while the rest is still being generated.
        """
        for word, said in PRONUNCIATION.items():
            text = re.sub(r"\b" + re.escape(word) + r"\b", said, text)
        rate = self.model.sample_rate
        gain = max(0.0, min(1.0, VOICE_VOLUME))
        began = time.monotonic()
        first_sound = None
        played = 0                          # samples handed to the speakers so far
        out = self._open_output(rate)
        try:
            waiting, waiting_len = [], 0    # speech made before playback starts
            for chunk in self.model.generate_audio_stream(self.voice_state, text,
                                                          copy_state=True):
                audio = chunk.detach().cpu().numpy().astype("float32").reshape(-1) * gain
                audio = np.clip(audio, -1.0, 1.0)
                if played == 0:
                    waiting.append(audio)
                    waiting_len += len(audio)
                    if waiting_len < CLONE_PREBUFFER * rate:
                        continue
                    audio, waiting, waiting_len = np.concatenate(waiting), [], 0
                if first_sound is None:
                    first_sound = time.monotonic() - began
                out.write(audio.reshape(-1, 1))
                played += len(audio)
            if waiting:                     # a short sentence that never filled the buffer
                audio = np.concatenate(waiting)
                first_sound = time.monotonic() - began
                out.write(audio.reshape(-1, 1))
                played += len(audio)
            out.write(np.zeros((int(rate * 0.2), 1), dtype="float32"))   # silent tail
            out.stop()                      # waits until the last sound has been played
        except Exception as e:
            if played == 0:
                raise                       # nothing was heard: let speak() try the backup voice
            print(f"[Voice] cloned voice stopped part-way ({e})")
        finally:
            out.close()
        if first_sound is not None:
            print(f"[Voice] {played / rate:.1f}s of speech, first sound after {first_sound:.2f}s")


voice_clone = ClonedVoice()


def test_cloned_voice():
    """`python jarvis_app.py --test-voice`: load the cloned voice and say one line."""
    print(f"Python in use: {sys.executable}")
    print(f"Looking for the voice clip next to this script: {', '.join(VOICE_SAMPLES)}")
    if not CLONE_VOICE:
        print("CLONE_VOICE is False near the top of this file. Switch it to True to test.")
        return
    voice_clone.start_loading()
    voice_clone.done.wait()
    if not voice_clone.ready:
        print("\nThe cloned voice isn't available yet. See the message above.")
        return
    print("Speaking a test line...")
    voice_clone.speak("Good evening, sir. JARVIS online, and at your service.")
    print("Done. If that sounded right, start JARVIS normally.")


def speak(text):
    """
    Say `text` out loud and wait until it finishes.

    This blocks on purpose: the microphone is closed while JARVIS talks, so
    he never hears (and reacts to) his own voice.
    A fresh engine is created per call because reusing one across threads is
    a known source of hangs with pyttsx3.

    Voice order: your cloned voice, then the Edge neural voice, then the
    offline voice. The first one that works is used.
    """
    if not text:
        return
    print(f"[JARVIS] {text}")
    log_chat("jarvis", text)
    if not state["voice_on"]:
        return
    with speak_lock:
        previous = state["status"]
        if CLONE_VOICE and not voice_clone.done.is_set():
            set_status("loading the JARVIS voice (the first run downloads it)...")
        use_clone = voice_clone.wait_ready()
        set_status("speaking", ICON_SPEAKING)
        try:
            spoken = False
            if use_clone:
                try:
                    voice_clone.speak(text)
                    spoken = True
                except Exception as e:
                    print(f"[Voice] cloned voice failed ({e}); using the backup voice")
            if not spoken:
                try:
                    speak_edge(text)
                except Exception as e:  # not installed, offline, etc.
                    print(f"[Voice] neural voice unavailable ({e}); using offline voice")
                    speak_offline(text)
        except Exception as e:
            print(f"[Voice error] {e}")
        finally:
            if not state["mic_enabled"]:
                set_status("microphone off", ICON_MUTED)
            elif previous == "microphone off":
                set_status(f"listening for '{WAKE_WORD}'", ICON_IDLE)
            else:
                set_status(previous, ICON_IDLE)


def press_key(vk_code, times=1):
    """Tap a keyboard key (used for the media/volume keys)."""
    for _ in range(times):
        ctypes.windll.user32.keybd_event(vk_code, 0, 0, 0)
        ctypes.windll.user32.keybd_event(vk_code, 0, 2, 0)  # key up


VK_VOLUME_MUTE, VK_VOLUME_DOWN, VK_VOLUME_UP = 0xAD, 0xAE, 0xAF

FILLER_WORDS = {"the", "please", "can", "you", "could", "would", "for", "me",
                "up", "a", "to", "my", "app", "application", "program", "down"}

ABILITY_PHRASES = (
    "what can you do", "what do you do", "what are you able", "your abilities",
    "your capabilities", "what can i ask", "what can you help", "what are your",
)


def find_target(clean, names):
    """Return the longest allowed name that appears as whole words in `clean`."""
    for name in sorted(names, key=len, reverse=True):
        if re.search(r"\b" + re.escape(name) + r"\b", clean):
            return name
    return None


def close_app(name):
    """Politely ask the app to close (like clicking X). True if it was running."""
    closed = False
    for proc in CLOSABLE_PROCESSES[name]:
        result = subprocess.run(
            ["taskkill", "/IM", proc],
            capture_output=True,
            creationflags=0x08000000,  # don't flash a console window
        )
        if result.returncode == 0:
            closed = True
    return closed


def describe_abilities():
    """What JARVIS says when (and only when) you ask what he can do."""
    apps = ", ".join(a for a in ALLOWED_APPS if a != "explorer")
    closable = ", ".join(CLOSABLE_PROCESSES)
    sites = ", ".join(ALLOWED_SITES)
    return (
        f"I can open these apps: {apps}. "
        f"I can close these: {closable}. "
        f"I can open these websites: {sites}. "
        "I can search Google or YouTube, tell you the time and date, "
        "and turn the volume up, down or mute it."
    )


def respond(command):
    """
    Work out what the user wants, do it if it is on the allowed list, and
    return what JARVIS should say. Anything not recognised is politely refused.

    Step 3 idea: when nothing here matches, send `command` to an LLM and
    return its answer instead of the "can't do that" line at the bottom.
    """
    text = command.lower().strip().rstrip(".?!")
    words = [w for w in text.split() if w not in FILLER_WORDS]
    clean = " ".join(words)

    # --- "what can you do?" (checked first, only answers when asked) ---
    if (any(p in text for p in ABILITY_PHRASES)
            or "abilities" in words or "capabilities" in words or text == "help"):
        return describe_abilities()

    # --- shut JARVIS himself down (closes this app only, not the computer) ---
    if clean in ("goodbye", "quit", "exit", "quit jarvis", "exit jarvis", "goodbye jarvis"):
        state["quit_after_speaking"] = True
        return "Goodbye, sir."

    # --- close an app (allowed list only) ---
    if words and words[0] in CLOSE_VERBS:
        target = find_target(clean, CLOSABLE_PROCESSES)
        if target:
            if close_app(target):
                return f"Closing {target}."
            return f"{target.title()} doesn't seem to be running, sir."
        if find_target(clean, ALLOWED_SITES):
            return "I can't close websites, sir. You can close the tab yourself."
        if find_target(clean, ALLOWED_APPS):
            return "I'm not able to close that one, sir."
        return "I'm not authorised to close that, sir."

    # --- open an app or website (allowed lists only) ---
    if words and words[0] in OPEN_VERBS:
        app = find_target(clean, ALLOWED_APPS)
        if app:
            try:
                os.startfile(ALLOWED_APPS[app])
                return f"Opening {app}."
            except OSError:
                return f"I couldn't open {app}, sir. It may not be installed."
        site = find_target(clean, ALLOWED_SITES)
        if site:
            webbrowser.open(ALLOWED_SITES[site])
            return f"Opening {site}."
        return "I'm not authorised to open that, sir."

    # --- time and date ---
    if "time" in words:
        return datetime.datetime.now().strftime("It is %I:%M %p, sir.").replace(" 0", " ")
    if "date" in words or "today" in words or "day" in words:
        return datetime.datetime.now().strftime("Today is %A, %B %d, sir.")

    # --- volume ---
    if "volume" in words or "mute" in words or "unmute" in words:
        if "mute" in words or "unmute" in words:
            press_key(VK_VOLUME_MUTE)
            return "Done, sir."
        if "lower" in words or "decrease" in words or "quieter" in words or "down" in text:
            press_key(VK_VOLUME_DOWN, 5)
            return "Lowering the volume."
        if "raise" in words or "increase" in words or "louder" in words or "up" in text:
            press_key(VK_VOLUME_UP, 5)
            return "Raising the volume."

    # --- web search ---
    for prefix in ("search youtube", "youtube search"):
        if clean.startswith(prefix):
            query = clean[len(prefix):].strip()
            if query:
                webbrowser.open("https://www.youtube.com/results?search_query="
                                + urllib.parse.quote_plus(query))
                return f"Searching YouTube for {query}."
    for prefix in ("search", "google", "look up"):
        if clean.startswith(prefix):
            query = clean[len(prefix):].strip()
            if query:
                webbrowser.open("https://www.google.com/search?q="
                                + urllib.parse.quote_plus(query))
                return f"Searching for {query}."

    # --- small talk ---
    if any(g in words for g in ("hello", "hi", "hey")):
        return "Hello, sir. How can I help?"
    if "how" in words and "are" in words:
        return "All systems nominal, sir."
    if "thank" in text or "thanks" in text:
        return "Always a pleasure, sir."

    return "I'm afraid I can't do that yet, sir."


# ---------------------------------------------------------------------------
# Microphone recording
# ---------------------------------------------------------------------------
# Picked once at startup. Windows' newer WASAPI audio driver sometimes
# throws "[WinError 50] The request is not supported" when a program opens
# and closes the microphone over and over, which is exactly what JARVIS
# does between phrases. The older MME driver doesn't have that problem, so
# we use it if it's available.
INPUT_DEVICE = None


def pick_input_device():
    """Return the MME input device index if available, else None (default)."""
    try:
        for api in sd.query_hostapis():
            if "mme" in api["name"].lower():
                dev = api.get("default_input_device", -1)
                if dev is not None and dev >= 0:
                    return dev
    except Exception as e:
        print(f"[Audio] couldn't inspect host APIs, using the default mic ({e})")
    return None


def record_phrase(timeout=None, phrase_time_limit=None):
    """
    Record one spoken phrase from the default microphone.

    Waits for speech to start (up to `timeout` seconds if given), then
    records until about 0.8 s of silence or `phrase_time_limit` seconds.
    Returns a speech_recognition.AudioData, or None if nothing was said.
    """
    with sd.RawInputStream(
        samplerate=SAMPLE_RATE,
        channels=1,
        dtype="int16",
        blocksize=BLOCK_FRAMES,
        device=INPUT_DEVICE,
    ) as stream:
        # Measure background noise so the speech threshold adapts to the room
        levels = []
        for _ in range(CALIBRATION_BLOCKS):
            data, _overflow = stream.read(BLOCK_FRAMES)
            levels.append(audioop.rms(bytes(data), SAMPLE_WIDTH))
        ambient = sum(levels) / len(levels)
        threshold = max(ambient * NOISE_MULTIPLIER, MIN_THRESHOLD)

        preroll = collections.deque(maxlen=PREROLL_BLOCKS)
        frames = []
        started = False
        silent_blocks = 0
        wait_start = time.monotonic()
        speech_start = 0.0

        while not stop_event.is_set():
            data, _overflow = stream.read(BLOCK_FRAMES)
            chunk = bytes(data)
            level = audioop.rms(chunk, SAMPLE_WIDTH)
            now = time.monotonic()

            if not started:
                if level > threshold:
                    started = True
                    speech_start = now
                    frames.extend(preroll)
                    frames.append(chunk)
                else:
                    preroll.append(chunk)
                    if timeout is not None and now - wait_start > timeout:
                        return None
            else:
                frames.append(chunk)
                silent_blocks = 0 if level > threshold else silent_blocks + 1
                if silent_blocks >= SILENCE_BLOCKS:
                    break
                if phrase_time_limit and now - speech_start > phrase_time_limit:
                    break

    if not started or len(frames) < MIN_PHRASE_BLOCKS:
        return None
    return sr.AudioData(b"".join(frames), SAMPLE_RATE, SAMPLE_WIDTH)


def transcribe(recognizer, audio):
    """Turn recorded audio into lowercase text, or None if not understood."""
    try:
        return recognizer.recognize_google(audio).lower()
    except sr.UnknownValueError:
        return None
    except sr.RequestError:
        set_status("error: no internet / speech service unreachable", ICON_ERROR)
        time.sleep(2)
        return None


# ---------------------------------------------------------------------------
# Listener thread
# ---------------------------------------------------------------------------
def listener_loop():
    recognizer = sr.Recognizer()

    global INPUT_DEVICE
    INPUT_DEVICE = pick_input_device()
    try:
        sd.check_input_settings(samplerate=SAMPLE_RATE, channels=1, dtype="int16",
                                device=INPUT_DEVICE)
    except Exception as e:
        if INPUT_DEVICE is not None:
            # MME device didn't accept these settings either; fall back to
            # whatever sounddevice considers the default.
            print(f"[Audio] MME device rejected settings ({e}); trying the default mic")
            INPUT_DEVICE = None
            try:
                sd.check_input_settings(samplerate=SAMPLE_RATE, channels=1, dtype="int16")
            except Exception as e2:
                set_status(f"error: no usable microphone ({e2})", ICON_ERROR)
                print(f"[Error] No usable microphone: {e2}")
                return
        else:
            set_status(f"error: no usable microphone ({e})", ICON_ERROR)
            print(f"[Error] No usable microphone: {e}")
            return
    print(f"[Audio] using input device: "
          f"{sd.query_devices(INPUT_DEVICE)['name'] if INPUT_DEVICE is not None else 'system default'}")

    set_status(f"listening for '{WAKE_WORD}'", ICON_IDLE)
    speak("JARVIS online. At your service, sir.")
    set_status(f"listening for '{WAKE_WORD}'", ICON_IDLE)

    while not stop_event.is_set():
        try:
            if not state["mic_enabled"]:      # microphone switched off: stay closed
                time.sleep(0.25)
                continue
            audio = record_phrase()
            if audio is None or not state["mic_enabled"]:
                continue

            heard = transcribe(recognizer, audio)
            if not heard or WAKE_WORD not in heard:
                continue

            set_status("wake word heard - listening for command", ICON_ACTIVE)

            # Handles "jarvis open notepad" said in one breath...
            command = heard.split(WAKE_WORD, 1)[1].strip()

            # ...or "jarvis" followed by a pause and then the command
            if not command:
                speak("Yes, sir?")
                set_status("wake word heard - listening for command", ICON_ACTIVE)
                audio = record_phrase(timeout=6, phrase_time_limit=8)
                command = transcribe(recognizer, audio) if audio else None

            if command:
                state["last_command"] = command
                set_status(f'last command: "{command}"')
                print(f"[Command] {command}")
                log_chat("you", command)
                speak(respond(command))
                if state["quit_after_speaking"]:
                    on_quit(icon_ref["icon"], None)
                    return
            else:
                set_status("didn't catch a command - say the wake word again")
                speak("I didn't catch that, sir.")

            icon_ref["icon"].icon = ICON_IDLE

        except Exception as e:
            set_status(f"error: {e}", ICON_ERROR)
            print(f"[Error] {e}")
            time.sleep(2)


# ---------------------------------------------------------------------------
# Interface: a control window plus a small floating orb
# ---------------------------------------------------------------------------
# The window shows the conversation, lets you type commands, and has buttons
# for the common actions. Closing it (the X) shrinks JARVIS to the floating
# orb; double-click the orb or use the tray icon to bring the window back.
UI = {
    "bg": "#03070d",       # window background
    "panel": "#07121c",    # status panel and conversation
    "field": "#0b1d2b",    # input field and buttons
    "line": "#12384a",     # borders and hover
    "text": "#d6eef5",
    "dim": "#6f93a1",
    "cyan": "#2de2ff",
    "gold": "#f2b632",
    "red": "#e0343a",
    "grey": "#7b8b93",
}
ORB_BG = "#010204"         # colour made see-through behind the floating orb
ORB_SIZE = 150

STATE_COLORS = {"idle": UI["cyan"], "active": UI["gold"], "speaking": UI["gold"],
                "error": UI["red"], "muted": UI["grey"]}
STATE_SPEEDS = {"idle": 1.0, "active": 4.0, "speaking": 3.0, "error": 0.5, "muted": 0.2}
STATE_TITLES = {"idle": "Listening", "active": "Heard you", "speaking": "Speaking",
                "error": "Needs attention", "muted": "Microphone off"}

FONT_UI = "Segoe UI"
FONT_HUD = "Bahnschrift"   # Windows 10+; Tk falls back to a default if missing

# Buttons under the conversation: (label, what JARVIS is told)
QUICK_ACTIONS = [
    ("Time", "what time is it"),
    ("Date", "what is the date"),
    ("Notepad", "open notepad"),
    ("Chrome", "open chrome"),
    ("YouTube", "open youtube"),
    ("Louder", "volume up"),
    ("Quieter", "volume down"),
    ("Mute", "mute"),
    ("Help", "what can you do"),
]

PLACEHOLDER = "Type a command, for example: open notepad"


def draw_hud(c, size, angle, key, bg, label=None):
    """Draw the animated JARVIS ring on canvas `c`, scaled to `size` pixels."""
    c.delete("all")
    k = size / ORB_SIZE
    cx = cy = size / 2
    color = STATE_COLORS.get(key, STATE_COLORS["idle"])
    speed = STATE_SPEEDS.get(key, 1.0)
    thin, thick = max(1, round(k)), max(2, round(2 * k))
    r_ring, r_tick, r_arc = 58 * k, 50 * k, 44 * k

    c.create_oval(cx - r_ring, cy - r_ring, cx + r_ring, cy + r_ring,
                  outline=color, width=thin)
    for i in range(16):
        a = math.radians(i * 22.5 + angle * speed)
        c.create_line(cx + r_tick * math.cos(a), cy + r_tick * math.sin(a),
                      cx + r_ring * math.cos(a), cy + r_ring * math.sin(a),
                      fill=color, width=thick)
    start = (angle * speed * 2) % 360
    for offset in (0, 180):
        c.create_arc(cx - r_arc, cy - r_arc, cx + r_arc, cy + r_arc,
                     start=start + offset, extent=70,
                     outline=color, width=thick, style="arc")

    swing = (5 if key == "speaking" else 3) * k     # the core breathes harder when he talks
    pulse = 16 * k + swing * math.sin(angle / 8)
    c.create_oval(cx - pulse, cy - pulse, cx + pulse, cy + pulse,
                  fill=color, outline="")
    c.create_text(cx, cy + 2 * k, text="J", fill=bg,
                  font=(FONT_HUD, int(14 * k), "bold"))
    if label:
        c.create_text(cx, size - 10, text=label, fill=color, font=("Consolas", 7))


class JarvisApp:
    """The JARVIS window and floating orb. Must run on the main thread."""

    BIG = 260

    def __init__(self, start_as_orb=False):
        self.root = tk.Tk()
        self.root.withdraw()            # invisible anchor; the window and orb below are what you see
        self.angle = 0.0
        self.history, self.history_pos = [], 0
        self.placeholder_on = False
        self.title_bar_done = False
        self._shown = None              # last values pushed to the side panel
        self._origin = (0, 0)
        self._build_window()
        self._build_orb()
        if start_as_orb:
            self.hide_window()
        else:
            self.show_window()

    # ----- building the window ------------------------------------------
    def _button(self, parent, text, command, fg=None):
        fg = fg or UI["cyan"]
        b = tk.Button(parent, text=text, command=command, bg=UI["field"], fg=fg,
                      activebackground=UI["line"], activeforeground=fg,
                      relief="flat", bd=0, padx=12, pady=6, cursor="hand2",
                      font=(FONT_UI, 10), highlightthickness=1,
                      highlightbackground=UI["field"], highlightcolor=UI["cyan"])
        b.bind("<Enter>", lambda e: b.config(bg=UI["line"]))
        b.bind("<Leave>", lambda e: b.config(bg=UI["field"]))
        return b

    def _build_window(self):
        win = self.win = tk.Toplevel(self.root)
        win.title("JARVIS")
        win.geometry("920x600")
        win.minsize(800, 520)
        win.configure(bg=UI["bg"])
        win.protocol("WM_DELETE_WINDOW", self.hide_window)
        win.columnconfigure(1, weight=1)
        win.rowconfigure(0, weight=1)
        try:
            from PIL import ImageTk
            self._icon_img = ImageTk.PhotoImage(ICON_IDLE)
            win.iconphoto(True, self._icon_img)
        except Exception:
            pass

        # --- left: live status ---
        side = tk.Frame(win, bg=UI["panel"], width=300,
                        highlightthickness=1, highlightbackground=UI["line"])
        side.grid(row=0, column=0, sticky="ns", padx=(16, 8), pady=16)
        side.pack_propagate(False)

        self.big = tk.Canvas(side, width=self.BIG, height=self.BIG,
                             bg=UI["panel"], highlightthickness=0)
        self.big.pack(pady=(24, 4))
        self.state_title = tk.Label(side, text="Starting", bg=UI["panel"],
                                    fg=UI["cyan"], font=(FONT_HUD, 18, "bold"))
        self.state_title.pack()
        self.state_detail = tk.Label(side, text="", bg=UI["panel"], fg=UI["dim"],
                                     font=(FONT_UI, 10), wraplength=250,
                                     justify="center")
        self.state_detail.pack(pady=(4, 16), padx=16)

        self.mic_btn = self._button(side, "", self._on_mic)
        self.mic_btn.pack(fill="x", padx=24, pady=(0, 8))
        self.voice_btn = self._button(side, "", self._on_voice)
        self.voice_btn.pack(fill="x", padx=24)

        tk.Label(side, text="Closing this window keeps JARVIS running.\n"
                            "Double-click the orb to bring it back.",
                 bg=UI["panel"], fg=UI["dim"], font=(FONT_UI, 9),
                 wraplength=250, justify="center").pack(side="bottom", pady=(0, 16))
        self._button(side, "Quit JARVIS", self._on_quit, fg=UI["red"]).pack(
            side="bottom", fill="x", padx=24, pady=(0, 10))

        # --- right: conversation, quick actions, input ---
        main = tk.Frame(win, bg=UI["bg"])
        main.grid(row=0, column=1, sticky="nsew", padx=(8, 16), pady=16)
        main.rowconfigure(0, weight=1)
        main.columnconfigure(0, weight=1)

        log_frame = tk.Frame(main, bg=UI["panel"], highlightthickness=1,
                             highlightbackground=UI["line"])
        log_frame.grid(row=0, column=0, sticky="nsew")
        log_frame.rowconfigure(0, weight=1)
        log_frame.columnconfigure(0, weight=1)
        self.log = tk.Text(log_frame, bg=UI["panel"], fg=UI["text"], bd=0,
                           wrap="word", font=(FONT_UI, 11), padx=18, pady=14,
                           cursor="arrow", state="disabled", highlightthickness=0,
                           tabs=(72,))
        self.log.grid(row=0, column=0, sticky="nsew")
        self.log.tag_configure("row", lmargin1=0, lmargin2=72, spacing3=10)
        self.log.tag_configure("you", foreground=UI["gold"], font=(FONT_UI, 10, "bold"))
        self.log.tag_configure("jarvis", foreground=UI["cyan"], font=(FONT_UI, 10, "bold"))
        self.log.tag_configure("body", foreground=UI["text"])
        self.log.tag_configure("hint", foreground=UI["dim"], font=(FONT_UI, 11, "italic"))
        self.log.config(state="normal")
        self.log.insert("end", f"Say “{WAKE_WORD.title()}” followed by a command, "
                               "or type one below.", "hint")
        self.log.config(state="disabled")

        chips = tk.Frame(main, bg=UI["bg"])
        chips.grid(row=1, column=0, sticky="ew", pady=(10, 0))
        for label, command in QUICK_ACTIONS:
            self._button(chips, label, lambda c=command: self.send(c)).pack(
                side="left", padx=(0, 6))

        row = tk.Frame(main, bg=UI["bg"])
        row.grid(row=2, column=0, sticky="ew", pady=(10, 0))
        row.columnconfigure(0, weight=1)
        self.entry = tk.Entry(row, bg=UI["field"], fg=UI["text"],
                              insertbackground=UI["cyan"], relief="flat",
                              font=(FONT_UI, 12), highlightthickness=1,
                              highlightbackground=UI["line"],
                              highlightcolor=UI["cyan"])
        self.entry.grid(row=0, column=0, sticky="ew", ipady=8)
        self._button(row, "Send", self._on_send, fg=UI["gold"]).grid(
            row=0, column=1, sticky="ns", padx=(8, 0))
        self.entry.bind("<Return>", lambda e: self._on_send())
        self.entry.bind("<Up>", lambda e: self._history_step(-1))
        self.entry.bind("<Down>", lambda e: self._history_step(1))
        self.entry.bind("<FocusIn>", self._focus_in)
        self.entry.bind("<FocusOut>", self._focus_out)
        self._show_placeholder()
        win.focus_set()

    def _build_orb(self):
        orb = self.orb = tk.Toplevel(self.root)
        orb.overrideredirect(True)
        orb.attributes("-topmost", True)
        orb.configure(bg=ORB_BG)
        try:
            orb.attributes("-transparentcolor", ORB_BG)  # Windows only
        except tk.TclError:
            pass
        sw, sh = orb.winfo_screenwidth(), orb.winfo_screenheight()
        orb.geometry(f"{ORB_SIZE}x{ORB_SIZE}+{sw - ORB_SIZE - 24}+{sh - ORB_SIZE - 90}")
        self.orb_canvas = tk.Canvas(orb, width=ORB_SIZE, height=ORB_SIZE,
                                    bg=ORB_BG, highlightthickness=0)
        self.orb_canvas.pack()
        self.orb_canvas.bind("<ButtonPress-1>", self._start_drag)
        self.orb_canvas.bind("<B1-Motion>", self._drag)
        self.orb_canvas.bind("<Double-Button-1>", lambda e: self.show_window())
        self.orb_canvas.bind("<Button-3>", lambda e: self.orb.withdraw())  # tray icon still brings it back
        orb.withdraw()

    # ----- showing and hiding -------------------------------------------
    def show_window(self):
        self.orb.withdraw()
        self.win.deiconify()
        self.win.lift()
        self.win.focus_force()
        self.window_open = True
        if not self.title_bar_done:
            self.title_bar_done = True
            self._dark_title_bar()

    def hide_window(self):
        self.win.withdraw()
        self.orb.deiconify()
        self.orb.attributes("-topmost", True)
        self.window_open = False

    def _dark_title_bar(self):
        """Ask Windows 10/11 for a dark title bar to match the window."""
        if sys.platform != "win32":
            return
        try:
            self.win.update_idletasks()
            hwnd = ctypes.windll.user32.GetParent(self.win.winfo_id())
            value = ctypes.c_int(1)
            for attribute in (20, 19):   # 20 on current Windows, 19 on older builds
                if ctypes.windll.dwmapi.DwmSetWindowAttribute(
                        hwnd, attribute, ctypes.byref(value), ctypes.sizeof(value)) == 0:
                    break
        except Exception:
            pass

    def _start_drag(self, event):
        self._origin = (event.x, event.y)

    def _drag(self, event):
        x = self.orb.winfo_x() + (event.x - self._origin[0])
        y = self.orb.winfo_y() + (event.y - self._origin[1])
        self.orb.geometry(f"+{x}+{y}")

    # ----- input field ---------------------------------------------------
    def _show_placeholder(self):
        self.entry.delete(0, "end")
        self.entry.insert(0, PLACEHOLDER)
        self.entry.config(fg=UI["dim"])
        self.placeholder_on = True

    def _focus_in(self, _event):
        if self.placeholder_on:
            self.entry.delete(0, "end")
            self.entry.config(fg=UI["text"])
            self.placeholder_on = False

    def _focus_out(self, _event):
        if not self.entry.get():
            self._show_placeholder()

    def _set_entry(self, text):
        self.placeholder_on = False
        self.entry.config(fg=UI["text"])
        self.entry.delete(0, "end")
        self.entry.insert(0, text)

    def _history_step(self, delta):
        if self.history:
            self.history_pos = max(0, min(len(self.history), self.history_pos + delta))
            self._set_entry(self.history[self.history_pos]
                            if self.history_pos < len(self.history) else "")
        return "break"

    def _on_send(self):
        text = "" if self.placeholder_on else self.entry.get()
        self.send(text)
        if text.strip():
            self._set_entry("")

    def send(self, text):
        """Run a typed or button command exactly as if it had been spoken."""
        text = (text or "").strip()
        if not text:
            return
        if not self.history or self.history[-1] != text:
            self.history.append(text)
        self.history_pos = len(self.history)
        run_text_command(text)

    # ----- side panel buttons ---------------------------------------------
    def _on_mic(self):
        toggle_mic()

    def _on_voice(self):
        state["voice_on"] = not state["voice_on"]

    def _on_quit(self):
        quit_app()

    # ----- keeping the screen in step with JARVIS --------------------------
    def _append(self, who, text):
        hint = self.log.tag_ranges("hint")
        self.log.config(state="normal")
        if hint:
            self.log.delete(hint[0], hint[-1])
        self.log.insert("end", ("You" if who == "you" else "JARVIS") + "\t", ("row", who))
        self.log.insert("end", text + "\n", ("row", "body"))
        lines = int(self.log.index("end-1c").split(".")[0])
        if lines > 500:                                  # keep the log from growing forever
            self.log.delete("1.0", f"{lines - 400}.0")
        self.log.config(state="disabled")
        self.log.see("end")

    def _drain_events(self):
        try:
            while True:
                kind, payload = ui_events.get_nowait()
                if kind == "chat":
                    self._append(*payload)
                elif kind == "show":
                    self.show_window()
        except queue.Empty:
            pass

    def _refresh_panel(self, key):
        mic, voice = state["mic_enabled"], state["voice_on"]
        detail = state.get("status", "")
        if key == "idle" and detail.startswith("listening for"):
            detail = f"Say “{WAKE_WORD}” to give a command."
        elif detail.startswith("error: "):
            detail = detail[len("error: "):]
        snapshot = (key, detail, mic, voice)
        if snapshot == self._shown:
            return
        self._shown = snapshot
        color = STATE_COLORS.get(key, UI["cyan"])
        self.state_title.config(text=STATE_TITLES.get(key, "Listening"), fg=color)
        self.state_detail.config(text=detail)
        for btn, on, label in ((self.mic_btn, mic, "Microphone"),
                               (self.voice_btn, voice, "Voice replies")):
            fg = UI["cyan"] if on else UI["grey"]
            btn.config(text=f"{label}: {'on' if on else 'off'}", fg=fg,
                       activeforeground=fg)

    def _tick(self):
        if stop_event.is_set():
            self.root.destroy()
            return
        self._drain_events()
        self.angle = (self.angle + 2) % 3600
        key = state.get("color_key", "idle")
        if self.window_open:
            draw_hud(self.big, self.BIG, self.angle, key, UI["panel"])
            self._refresh_panel(key)
        else:
            draw_hud(self.orb_canvas, ORB_SIZE, self.angle, key, ORB_BG,
                     label=state.get("status", "")[:34].upper())
        self.root.after(40, self._tick)

    def run(self):
        self._tick()
        self.root.mainloop()


# ---------------------------------------------------------------------------
# Tray icon
# ---------------------------------------------------------------------------
def on_status_click(icon, item):
    print(f"[Status] {state['status']}")


def quit_app():
    stop_event.set()
    icon = icon_ref["icon"]
    if icon is not None:
        icon.stop()


def on_quit(icon, item):
    quit_app()


def toggle_mic():
    """Switch the microphone on or off (off = the mic stream is not opened at all)."""
    state["mic_enabled"] = not state["mic_enabled"]
    if state["mic_enabled"]:
        set_status(f"listening for '{WAKE_WORD}'", ICON_IDLE)
    else:
        set_status("microphone off", ICON_MUTED)


def run_text_command(command):
    """Run a typed or button command through the same path as a spoken one."""
    def work():
        state["last_command"] = command
        log_chat("you", command)
        speak(respond(command))
        if state["quit_after_speaking"]:
            quit_app()
    threading.Thread(target=work, daemon=True).start()


def on_open_window(icon, item):
    ui_events.put(("show", None))


def on_toggle_mic(icon, item):
    toggle_mic()


def build_tray_icon():
    menu = pystray.Menu(
        pystray.MenuItem("Open JARVIS", on_open_window, default=True),
        pystray.MenuItem("Microphone", on_toggle_mic,
                         checked=lambda item: state["mic_enabled"]),
        pystray.MenuItem(
            lambda item: f"Status: {state['status']}", on_status_click, enabled=False
        ),
        pystray.Menu.SEPARATOR,
        pystray.MenuItem("Quit JARVIS", on_quit),
    )
    icon = pystray.Icon("jarvis", ICON_IDLE, "JARVIS Assistant", menu)
    icon_ref["icon"] = icon
    return icon


def main():
    if "--voices" in sys.argv:
        list_voices()
        return
    if "--test-voice" in sys.argv:
        test_cloned_voice()
        return
    sys.stdout.write(
        f"JARVIS running on Python {sys.version.split()[0]} ({sys.executable})\n"
    )
    print("Look for the glowing circle in your system tray (click ^ near the clock).")

    if CLONE_VOICE:
        voice_clone.start_loading()     # load the cloned voice while everything else starts
    threading.Thread(target=listener_loop, daemon=True).start()

    # The tray icon runs in its own thread so the window can use tkinter's
    # main loop, which tkinter requires to run on the main thread.
    icon = build_tray_icon()
    threading.Thread(target=icon.run, daemon=True).start()

    if tk is not None:
        print("The JARVIS window is opening. Close it to shrink JARVIS to the orb "
              "(drag to move, double-click to reopen the window).")
        JarvisApp(start_as_orb="--orb" in sys.argv).run()
    else:
        print("tkinter isn't available in this Python install, "
              "so the window and orb are disabled; the tray icon still works.")
        stop_event.wait()


if __name__ == "__main__":
    main()
