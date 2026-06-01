# Ollama-Windows-V.2

> Extends V.1 with local text-to-speech (TTS) and a document viewer — your LLM now speaks its responses and can open files directly from the terminal.

---

## Overview

This guide builds directly on [Ollama-Windows-V.1](https://github.com/quintenlittle/Ollama-Windows-V.1). Complete that setup first, then follow the steps here to add:

- **Text-to-speech** — responses are spoken aloud using a local piper voice model, no cloud, no API
- **Document viewer** — open PDF, DOCX, and text files directly from the terminal using your system's default applications

No RAG or indexing is involved here. That is covered in a separate repo.

---

## Prerequisites

- Completed [Ollama-Windows-V.1](https://github.com/quintenlittle/Ollama-Windows-V.1) setup
- Windows 10 or 11
- Python 3.10 or higher — download from [python.org](https://www.python.org/downloads/) if not already installed
  > During installation check **Add Python to PATH**
- [VLC Media Player](https://www.videolan.org/vlc/) — used for audio playback (free, lightweight)

---

## Part 1 — Text-to-Speech

### Step 1 — Install piper-tts

Open Command Prompt and run:

```
pip install piper-tts
```

Verify it installed:

```
python -m piper --help
```

---

### Step 2 — Download a Voice Model

Create a folder to store your voice models:

```
mkdir C:\ollama-voices
```

Download the Cori voice (British female, high quality — recommended):

```
curl -L "https://huggingface.co/rhasspy/piper-voices/resolve/main/en/en_GB/cori/high/en_GB-cori-high.onnx" -o "C:\ollama-voices\en_GB-cori-high.onnx"
curl -L "https://huggingface.co/rhasspy/piper-voices/resolve/main/en/en_GB/cori/high/en_GB-cori-high.onnx.json" -o "C:\ollama-voices\en_GB-cori-high.onnx.json"
```

> **Both files are required.** The `.onnx` is the voice model and the `.onnx.json` is its config. Always download them as a pair.

Test the voice:

```
echo Hello, I am now speaking. | python -m piper --model C:\ollama-voices\en_GB-cori-high.onnx --output_file C:\ollama-voices\test.wav && C:\ollama-voices\test.wav
```

You should hear the voice play through your speakers.

---

### Step 3 — Update Your .bat File

Open your existing `.bat` file from V.1 and replace the contents with the following — substituting your model name and voice path:

```bat
@echo off
title REBEL
set OLLAMA_NOHISTORY=1

:loop
set /p "USER_INPUT=Input: "
if "%USER_INPUT%"=="" goto loop

for /f "delims=" %%R in ('echo %USER_INPUT% ^| ollama run CUSTOM_MODEL_NAME 2^>nul') do (
    set "RESPONSE=%%R"
    echo Output: %%R
)

echo %RESPONSE% | python -m piper --model C:\ollama-voices\en_GB-cori-high.onnx --output_file C:\ollama-voices\response.wav 2>nul
start /wait "" C:\ollama-voices\response.wav

goto loop
```

> **Note:** Replace `CUSTOM_MODEL_NAME` with your model name from V.1.
> The `start /wait` line plays the audio using your system default WAV player and waits for it to finish before accepting the next input. VLC handles this cleanly.

---

### Step 4 — Set VLC as Default for WAV Files (Recommended)

1. Right-click any `.wav` file → **Open with** → **Choose another app**
2. Select **VLC media player**
3. Check **Always use this app to open .wav files**
4. Click **OK**

This ensures audio plays without a visible window interrupting the terminal.

> **Tip:** For truly silent playback with no window at all, use VLC's command line mode. Replace the `start /wait` line in your `.bat` with:
> ```
> "C:\Program Files\VideoLAN\VLC\vlc.exe" --intf dummy --play-and-exit C:\ollama-voices\response.wav 2>nul
> ```

---

### Changing Voices

Browse available voices at: **https://rhasspy.github.io/piper-samples/**

Each voice entry has a listen button so you can preview before downloading. To swap voices:

1. Download the new `.onnx` and `.onnx.json` pair into `C:\ollama-voices\`
2. Update the `--model` path in your `.bat` file to point at the new `.onnx` file
3. Check the `.onnx.json` file for `"sample_rate"` — most voices are `22050` but some low-quality variants use `16000`. This matters if the voice sounds too fast or too slow

---

## Part 2 — Document Viewer

### Step 5 — Add Open-File Support to Your .bat

This adds a command that lets you open a file by typing its path directly in the terminal. The file opens in your system default application — PDFs in your PDF viewer, DOCX in Word, etc.

Add the following block inside your `.bat` loop, before the `goto loop` line:

```bat
if /i "%USER_INPUT:~0,5%"=="open " (
    set "FILEPATH=%USER_INPUT:~5%"
    start "" "%FILEPATH%"
    goto loop
)
```

**Usage example:**

```
Input: open C:\Users\YourName\Documents\report.pdf
```

The file opens immediately in whatever application is set as default for that file type on your system.

---

### Step 6 — Full Updated .bat File

Here is the complete `.bat` file combining everything from V.1 and V.2:

```bat
@echo off
title REBEL
set OLLAMA_NOHISTORY=1

:loop
set /p "USER_INPUT=Input: "
if "%USER_INPUT%"=="" goto loop

:: Open file command
if /i "%USER_INPUT:~0,5%"=="open " (
    set "FILEPATH=%USER_INPUT:~5%"
    start "" "%FILEPATH%"
    goto loop
)

:: Send to model and speak response
for /f "delims=" %%R in ('echo %USER_INPUT% ^| ollama run CUSTOM_MODEL_NAME 2^>nul') do (
    set "RESPONSE=%%R"
    echo Output: %%R
)

echo %RESPONSE% | python -m piper --model C:\ollama-voices\en_GB-cori-high.onnx --output_file C:\ollama-voices\response.wav 2>nul
"C:\Program Files\VideoLAN\VLC\vlc.exe" --intf dummy --play-and-exit C:\ollama-voices\response.wav 2>nul

goto loop
```

Save, rename to `.bat` if needed, and double-click to launch.

---

## How It Works

| Feature | Detail |
|---------|--------|
| **Text-to-speech** | piper converts the model response to a WAV file locally, VLC plays it silently |
| **Fully offline** | piper runs entirely on-device, no internet required after voice model download |
| **Document viewer** | `open` command uses Windows `start` to launch the file in its default application |
| **Zero history** | `OLLAMA_NOHISTORY=1` carries over from V.1 |
| **No RAG** | This version is chat-only — document indexing and querying is covered separately |

---

## Files Overview

```
C:\
└── modelfile                        # From V.1 — base model and system prompt

C:\ollama-voices\
├── en_GB-cori-high.onnx             # Piper voice model
└── en_GB-cori-high.onnx.json        # Voice config (required alongside .onnx)

Desktop\
└── CUSTOM_MODEL_NAME.bat            # Updated launcher with TTS and file open
```

---

## Troubleshooting

**Voice sounds too fast or too slow**
The voice sample rate doesn't match. Check `"sample_rate"` inside the `.onnx.json` file. If it says `16000` the voice runs at 16kHz — this is normal for `low` quality variants. The bat file plays whatever WAV piper generates so speed is controlled by the model itself, not the playback.

**No audio plays**
Confirm VLC is installed and the path `C:\Program Files\VideoLAN\VLC\vlc.exe` exists. On some systems VLC installs to `C:\Program Files (x86)\VideoLAN\VLC\vlc.exe` — check both.

**python -m piper not found**
Python was not added to PATH during installation. Re-run the Python installer, choose **Modify**, and check **Add Python to environment variables**.

**open command not working**
The file path has spaces — wrap it in quotes: `open "C:\My Documents\my file.pdf"`

---

## Related

- [Ollama-Windows-V.1](https://github.com/quintenlittle/Ollama-Windows-V.1) — Base setup: install Ollama, create a custom model, desktop shortcut
- [Ollama-Linux-V.1](https://github.com/quintenlittle/Ollama-Linux-V.1) — Same base setup for Linux
- [RAG-Technique-V.1](https://github.com/quintenlittle/RAG-Technique-V.1) — Index your personal document library and query it with a local LLM
