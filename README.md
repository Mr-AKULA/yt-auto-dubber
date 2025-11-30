ПРЕДУПРЕЖДАЮ , НАПИСАНО ПАХАБНО И ЕСТЬ КОСЯКИ , ЕСЛИ БУДУТ ВОПРОСЫ ПИШИТЕ , ОБЯЗАТЕЛЬНО ОТВЕЧУ НО ВОЗМОЖНО НЕ СВОЕВРЕММЕНО.
ЭТО ПЕРВАЯ РАБОЧАЯ ВЕРСИЯ ПРИ НИОБХОДИМОСТИ МОЖНО АВТАМОТИЗИРОВАТЬ И МОЖЕТ ДАЖЕ СДЕЛАТЬ РАСШИРЕНИЕ , НО Я НЕ УВЕРЕН.
---
ссылка на расширение - https://downloadhelper.net/

https://github.com/snakers4/silero-models

Видео-инструкция https://youtu.be/saonPsHSiPs

# 📘 yt-auto-dubber

Автоматизированный Python-конвейер для скачивания, транскрибирования, перевода и озвучивания YouTube-видео с заменой звуковой дорожки.

Поддерживает:

* 🎧 Whisper (ASR)
* 🌐 Перевод текста (LibreTranslate / любой другой бесплатный API)
* 🔊 Silero TTS
* 🎬 FFmpeg (монтаж + замена звука)
* 📦 Автоматическое разбиение текста на части
* 🎚 Генерацию чистой, бесшумной дорожки с паузами

---

## 📦 Функционал

✔ Скачать видео с YouTube
✔ Извлечь аудио
✔ Распознать речь → получить текст
✔ Перевести текст на русский
✔ Озвучить текст голосом Silero
✔ Собрать чистую WAV-дорожку
✔ Вставить её в видео (обрезка, выравнивание, добавление чёрного экрана)

---

# 🚀 Установка

## 1. Установить Python 3.10–3.12

[https://www.python.org/downloads/](https://www.python.org/downloads/)

## 2. Клонировать репозиторий

```bash
git clone https://github.com/yourname/yt-auto-dubber.git
cd yt-auto-dubber
```

## 3. Установить зависимости

```bash
pip install -r requirements.txt
```

## 4. Установить FFmpeg (обязательно)

### Windows

Скачать: [https://www.gyan.dev/ffmpeg/builds/](https://www.gyan.dev/ffmpeg/builds/)
Добавить ffmpeg.exe в PATH.

### Linux

```bash
sudo apt install ffmpeg
```

### macOS

```bash
brew install ffmpeg
```

---

# 📥 1. Скачать YouTube-видео

Скрипт `download_video.py` скачивает файл:

```python
from pytube import YouTube

url = "https://youtube.com/..."
yt = YouTube(url)
yt.streams.filter(file_extension="mp4").first().download("video.mp4")
```

Файл сохраняется как:

```
video.mp4
```

---

# 🎧 2. Извлечь аудио из видео

```bash
ffmpeg -i video.mp4 audio.wav
```

---

# 📝 3. Транскрипция Whisper (получение текста)

```python
import whisper

model = whisper.load_model("medium")  # или small, large
result = model.transcribe("audio.wav", task="translate", language="en")

with open("output_en.txt", "w", encoding="utf-8") as f:
    f.write(result["text"])
```

Выходной файл:

```
output_en.txt
```

---

# 🌐 4. Перевод текста на русский (бесплатный API)

Пакет: `deep_translator`
Используем LibreTranslate.

```python
from deep_translator import LibreTranslateAPI

lt = LibreTranslateAPI(source="en", target="ru")

text = open("output_en.txt", "r", encoding="utf-8").read()

translated = lt.translate(text)

open("output_ru.txt", "w", encoding="utf-8").write(translated)
```

---

# 🔊 5. Генерация озвучки Silero

Файл **tts_generate.py**:

```python
from pydub import AudioSegment
import re
import numpy as np

sample_rate = 48000
speaker = 'xenia'

def split_text_safe(text, max_len=700):
    sentences = re.split(r'(?<=[.!?])\s+', text)
    chunks, current = [], ""
    for s in sentences:
        if len(current) + len(s) + 1 > max_len:
            chunks.append(current.strip())
            current = s
        else:
            current += " " + s
    if current:
        chunks.append(current.strip())
    return chunks

example_text = open("output_ru.txt", "r", encoding="utf-8").read()
chunks = split_text_safe(example_text, 700)

from silero import silero_tts
model = silero_tts()

final_audio = AudioSegment.empty()
silence = AudioSegment.silent(duration=400)

for chunk in chunks:
    audio_chunk = model.apply_tts(
        text=chunk,
        speaker=speaker,
        sample_rate=sample_rate
    )

    audio_np = audio_chunk.cpu().numpy()
    audio_np_int16 = (audio_np * 32767).astype(np.int16)

    seg = AudioSegment(
        audio_np_int16.tobytes(),
        frame_rate=sample_rate,
        sample_width=2,
        channels=1
    )
    final_audio += seg + silence

final_audio.export("output_final_clean.wav", format="wav")
```

Результат:

```
output_final_clean.wav
```

---

# 🎬 6. Замена звуковой дорожки + обрезка + чёрный экран

Обрезаем первые **10 секунд**:

```bash
ffmpeg -i video.mp4 -ss 00:00:10 -c copy video_cut.mp4
```

Заменяем аудио:

```bash
ffmpeg -i video_cut.mp4 -i output_final_clean.wav -map 0:v -map 1:a -c:v copy -shortest video_final.mp4
```

Если аудио длиннее → добавляем чёрный экран:

```bash
ffmpeg -f lavfi -i color=c=black:s=1920x1080:d=5 black.mp4
ffmpeg -i video_cut.mp4 -i black.mp4 -filter_complex "[0:v][1:v]concat=n=2:v=1:a=0" long_video.mp4
```

---

# 📁 Итоговые файлы

```
video.mp4                — оригинальное видео  
audio.wav                — извлечённое аудио  
output_en.txt            — транскрипция Whisper  
output_ru.txt            — перевод LibreTranslate  
output_final_clean.wav   — финальная озвучка  
video_final.mp4          — видео с заменённой звуковой дорожкой
```

---

# 🤖 Автоматический режим (одним скриптом)

Если хочешь — могу собрать полный `main.py`, который будет выполнять весь пайплайн одним нажатием.

---

Хочешь, чтобы я подготовил:
✔ красивое оформление README с таблицами / логотипом?
✔ структуру папок?
✔ requirements.txt?
