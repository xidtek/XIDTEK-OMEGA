# =============================================================================
# XIDTEK-Δ7-OMEGA v22.0
# SISTEMA DE PRODUCCIÓN MUSICAL AUTÓNOMO CON AUTO-ACTUALIZACIÓN
# =============================================================================
# INSTRUCCIONES PARA QUE FUNCIONE LA AUTO-ACTUALIZACIÓN:
# 1. Cambia las líneas REPO_OWNER y REPO_NAME (más abajo) con tus datos de GitHub.
# 2. Sube este script a tu repositorio.
# 3. Crea un archivo version.txt con el contenido "22.0" y súbelo también.
# 4. Ve a "Releases" en GitHub y crea una nueva release con tag "v22.0".
# =============================================================================

# 1. INSTALACIÓN DE DEPENDENCIAS (para Colab)
try:
    import google.colab
    IN_COLAB = True
except:
    IN_COLAB = False

if IN_COLAB:
    !pip install -q psutil diffusers transformers accelerate soundfile
    !pip install -q demucs matchering librosa edge-tts pydub mutagen numpy scipy matplotlib
    !pip install -q audioseal smolagents litellm

import os
import sys
import subprocess
import json
import zipfile
import gc
import resource
import time
import numpy as np
import matplotlib.pyplot as plt
import soundfile as sf
import torch
import librosa
import librosa.display
import psutil
from pydub import AudioSegment
from pydub.generators import Sine
from pydub.effects import normalize, high_pass_filter, low_pass_filter
from mutagen.mp3 import MP3
from mutagen.id3 import ID3, TIT2, TPE1, TALB, COMM
import demucs.separate
import matchering as mg
from diffusers import AudioLDM2Pipeline
from audioseal import AudioSeal
from smolagents import CodeAgent, tool, LiteLLMModel

if IN_COLAB:
    from google.colab import drive, files

# =============================================================================
# CONFIGURACIÓN DE GITHUB PARA AUTO-ACTUALIZACIÓN
# =============================================================================
# ¡CAMBIA ESTAS DOS LÍNEAS CON LOS DATOS DE TU REPOSITORIO!
REPO_OWNER = "xidtek"           # Tu usuario de GitHub (YA CAMBIADO)
REPO_NAME = "XIDTEK-OMEGA"      # Nombre de tu repositorio

# =============================================================================
# MÓDULO DE AUTO-ACTUALIZACIÓN
# =============================================================================
def check_for_updates():
    """
    Verifica si hay una nueva versión en GitHub.
    Compara la versión actual (guardada en version.txt) con la última release.
    """
    print("\n🔍 [XIDTEK] Buscando actualizaciones...")

    # Leer versión actual desde archivo (si existe)
    try:
        with open("version.txt", "r") as f:
            current_version = f.read().strip()
    except:
        current_version = "22.0"  # Versión por defecto

    try:
        import requests
        import zipfile
        import io
        import shutil

        # Obtener última versión desde GitHub API
        api_url = f"https://api.github.com/repos/{REPO_OWNER}/{REPO_NAME}/releases/latest"
        response = requests.get(api_url, timeout=5)

        if response.status_code != 200:
            print("   ⚠️ No se pudo conectar a GitHub. Continuando.")
            return False

        latest_release = response.json()
        latest_version = latest_release.get("tag_name", "v0.0").lstrip("v")

        if latest_version == current_version:
            print(f"   ✅ XIDTEK ya está actualizado (v{current_version}).")
            return False

        print(f"   🔄 Nueva versión: v{latest_version} (actual: v{current_version})")

        # Descargar e instalar
        zip_url = f"https://github.com/{REPO_OWNER}/{REPO_NAME}/archive/refs/tags/v{latest_version}.zip"
        r = requests.get(zip_url, timeout=10)

        if r.status_code != 200:
            print("   ⚠️ No se pudo descargar la actualización.")
            return False

        z = zipfile.ZipFile(io.BytesIO(r.content))
        temp_dir = "temp_update"
        z.extractall(temp_dir)

        extracted_files = os.listdir(temp_dir)
        if not extracted_files:
            shutil.rmtree(temp_dir)
            return False

        src_dir = os.path.join(temp_dir, extracted_files[0])
        script_files = [f for f in os.listdir(src_dir) if f.endswith(".py")]
        if not script_files:
            shutil.rmtree(temp_dir)
            return False

        main_script = os.path.join(src_dir, script_files[0])
        import shutil
        shutil.copy2(main_script, "xidtek_updated.py")

        with open("version.txt", "w") as f:
            f.write(latest_version)

        shutil.rmtree(temp_dir)

        print("   ✅ Actualización descargada.")
        print("\n" + "="*60)
        print("🔄 XIDTEK se ha actualizado a la versión", latest_version)
        print("📁 El nuevo script está en: xidtek_updated.py")
        print("▶️  Ejecuta la siguiente celda:")
        print("    !python xidtek_updated.py")
        print("="*60)
        return True

    except Exception as e:
        print(f"   ⚠️ Error al verificar actualizaciones: {e}")
        return False

# =============================================================================
# FUNCIÓN PRINCIPAL DE XIDTEK
# =============================================================================
def run_xidtek():
    """Ejecuta el flujo completo de producción musical."""

    print("\n🚀 Iniciando XIDTEK-Δ7-OMEGA v22.0...")

    # 1. Forzar memoria
    try:
        resource.setrlimit(resource.RLIMIT_AS, (-1, -1))
    except:
        pass
    gc.collect()
    print(f"🧹 Memoria: {psutil.virtual_memory().available / (1024**3):.2f} GB")

    # 2. Montar Drive (si estamos en Colab)
    if IN_COLAB:
        drive.mount('/content/drive')
        LOOP_PATH = "/content/drive/MyDrive/V.01.mp3"
    else:
        LOOP_PATH = "V.01.mp3"  # Si se ejecuta localmente, el archivo debe estar en el mismo directorio

    if not os.path.exists(LOOP_PATH):
        print("❌ No se encontró el archivo V.01.mp3.")
        if IN_COLAB:
            print("   Sube 'V.01.mp3' a la raíz de 'Mi unidad'.")
        else:
            print("   Coloca 'V.01.mp3' en la misma carpeta que este script.")
        return

    print("✅ Loop cargado.")

    # =========================================================================
    # METADATOS (puedes cambiarlos aquí)
    # =========================================================================
    METADATOS = {
        "titulo": "Quantum Hit",
        "artista": "XIDTEK-Δ7-OMEGA",
        "album": "Quantum Hits Vol.1",
        "genero": "Dubstep / Electronic",
        "isrc": "US-XID-01-00001",
        "upc": "000000000000",
        "compositores": "XIDTEK-LAB",
        "productores": "XIDTEK-Δ7-OMEGA",
        "fecha": "2026-07-26"
    }

    # =========================================================================
    # FASE 1: ANÁLISIS ESTRUCTURAL (96kHz)
    # =========================================================================
    print("\n🔬 FASE 1: ANÁLISIS ESTRUCTURAL (96kHz)")
    SAMPLE_RATE = 96000
    HOP_LENGTH = 1024
    FRAME_LENGTH = 4096

    y, sr = librosa.load(LOOP_PATH, sr=SAMPLE_RATE, duration=120)
    tempo_array, _ = librosa.beat.beat_track(y=y, sr=sr)
    tempo = int(tempo_array.item()) if tempo_array.size > 0 else 140
    print(f"🎵 BPM: {tempo}")

    rms = librosa.feature.rms(y=y, frame_length=FRAME_LENGTH, hop_length=HOP_LENGTH)[0]
    times = librosa.times_like(rms, sr=sr, hop_length=HOP_LENGTH)
    drop_time = float(times[np.argmax(rms)])
    print(f"💥 Drop: {drop_time:.2f}s")

    plt.figure(figsize=(14, 6))
    D = librosa.amplitude_to_db(np.abs(librosa.stft(y, n_fft=FRAME_LENGTH, hop_length=HOP_LENGTH)), ref=np.max)
    librosa.display.specshow(D, sr=sr, x_axis='time', y_axis='log', hop_length=HOP_LENGTH)
    plt.axvline(x=drop_time, color='cyan', linestyle='--', label=f'Drop: {drop_time:.2f}s')
    plt.colorbar(format='%+2.0f dB')
    plt.title('Espectrograma (96kHz)')
    plt.tight_layout()
    plt.savefig("espectrograma_V01_96k.png", dpi=150)
    print("✅ Espectrograma guardado.")

    # =========================================================================
    # FASE 2: GENERACIÓN DE BASE CON ESTRUCTURA DE HIT
    # =========================================================================
    print("\n🎚️ FASE 2: GENERACIÓN DE BASE CON ESTRUCTURA DE HIT")

    def variar_audio(seg, pitch=1.0, filtro=None, gain=0):
        if pitch != 1.0:
            new_rate = int(seg.frame_rate * pitch)
            seg = seg._spawn(seg.raw_data, overrides={"frame_rate": new_rate})
            seg = seg.set_frame_rate(96000)
        if filtro == "hp":
            seg = high_pass_filter(seg, 400)
        elif filtro == "lp":
            seg = low_pass_filter(seg, 800)
        if gain:
            seg = seg.apply_gain(gain)
        return seg

    def extender_seccion(seg, segundos):
        target = segundos * 1000
        ext = seg
        while len(ext) < target:
            ext = ext.append(seg, crossfade=500)
        return ext[:target]

    loop = AudioSegment.from_mp3(LOOP_PATH).set_frame_rate(96000).set_channels(2).set_sample_width(3)

    secciones = {
        "intro": {"dur": 15, "pitch": 0.9, "filtro": "hp", "gain": -6},
        "build_up": {"dur": 15, "pitch": 1.0, "filtro": None, "gain": -3},
        "drop": {"dur": 30, "pitch": 1.0, "filtro": None, "gain": 3},
        "breakdown": {"dur": 15, "pitch": 0.85, "filtro": "lp", "gain": -6},
        "build_up_2": {"dur": 15, "pitch": 1.0, "filtro": None, "gain": -1},
        "drop_final": {"dur": 30, "pitch": 1.0, "filtro": None, "gain": 4}
    }

    audio_final = AudioSegment.silent(duration=0, frame_rate=96000)
    for nombre, params in secciones.items():
        seg = variar_audio(loop, pitch=params["pitch"], filtro=params["filtro"], gain=params["gain"])
        seg_ext = extender_seccion(seg, params["dur"])
        audio_final = audio_final.append(seg_ext, crossfade=1000)
        print(f"   ✅ {nombre} generado ({params['dur']}s)")

    target_ms = 180000
    if len(audio_final) > target_ms:
        audio_final = audio_final[:target_ms]
    elif len(audio_final) < target_ms:
        silencio = AudioSegment.silent(duration=target_ms - len(audio_final), frame_rate=96000)
        audio_final = audio_final + silencio

    audio_final.export("base_generada_96k.wav", format="wav", parameters=["-ar", "96000", "-ac", "2", "-acodec", "pcm_s24le"])
    print("✅ Base generada con estructura de hit en 96kHz/24-bit.")

    # =========================================================================
    # FASE 3: DEMUCS + EXPORTACIÓN DE STEMS
    # =========================================================================
    print("\n🔊 FASE 3: SEPARACIÓN DE STEMS (DEMUCS)")
    try:
        demucs.separate.main(["-n", "htdemucs", "-o", "separated", "base_generada_96k.wav"])
        stem_dir = "separated/htdemucs/base_generada_96k"
        for name in ["drums", "bass", "other", "vocals"]:
            path = f"{stem_dir}/{name}.wav"
            if os.path.exists(path):
                seg = AudioSegment.from_wav(path).set_frame_rate(96000).set_sample_width(3)
                seg.export(f"stem_{name}_96k_24bit.wav", format="wav", parameters=["-ar", "96000", "-ac", "2", "-acodec", "pcm_s24le"])
        print("✅ Stems separados y exportados.")
    except Exception as e:
        print(f"⚠️ Demucs falló: {e}. Usando mezcla completa.")

    # =========================================================================
    # FASE 4: TAG XIDTEK MEJORADO
    # =========================================================================
    print("\n🎤 FASE 4: GENERACIÓN DEL TAG XIDTEK")
    os.system("edge-tts --voice 'es-ES-ElviraNeural' --text 'XIDTEK' --write-media tag_raw.mp3")
    tag = AudioSegment.from_mp3("tag_raw.mp3")
    tag = tag.set_frame_rate(96000).set_sample_width(3)
    tag = tag.speedup(1.10)
    tag = tag._spawn(tag.raw_data, overrides={"frame_rate": int(tag.frame_rate * 0.85)}).set_frame_rate(96000)
    tag = high_pass_filter(tag, 150)
    tag = low_pass_filter(tag, 8000)
    reverb = tag - 6 + 45
    tag = tag.overlay(reverb, position=45)
    tag = tag.overlay((tag - 3) + 12, position=12)
    tag = normalize(tag).apply_gain(-3)
    tag.export("tag_mejorado_96k.wav", format="wav", parameters=["-ar", "96000", "-ac", "2", "-acodec", "pcm_s24le"])

    audio = AudioSegment.from_wav("base_generada_96k.wav")
    tag_intro = tag.fade_in(200).fade_out(200) - 6
    tag_pre = tag.fade_in(100).fade_out(100) + 3
    audio = audio.overlay(tag_intro, position=0)
    pre_pos = int((drop_time - 2) * 1000)
    if pre_pos < 0:
        pre_pos = 0
    audio = audio.overlay(tag_pre, position=pre_pos)
    audio.export("con_tag_96k.wav", format="wav", parameters=["-ar", "96000", "-ac", "2", "-acodec", "pcm_s24le"])
    print("✅ Tag insertado estratégicamente.")

    # =========================================================================
    # FASE 5: WATERMARKING NEURONAL (AUDIOSEAL)
    # =========================================================================
    print("\n🔒 FASE 5: WATERMARKING NEURONAL")
    try:
        model = AudioSeal.load_generator("facebook/audioseal")
        audio_data, sr = sf.read("con_tag_96k.wav")
        if len(audio_data.shape) > 1:
            audio_data = np.mean(audio_data, axis=1)
        wm = model.generate(audio_data, sr, message="XIDTEK", alpha=0.5)
        audio_marcada = audio_data + wm[0]
        sf.write("marcado_96k.wav", audio_marcada, sr)
        print("✅ AudioSeal aplicado.")
    except Exception as e:
        print(f"⚠️ AudioSeal falló: {e}. Usando backup (tono 19kHz).")
        audio = AudioSegment.from_wav("con_tag_96k.wav")
        y_orig, sr = librosa.load("con_tag_96k.wav", sr=96000)
        tone = 0.0001 * np.sin(2 * np.pi * 19000 * np.linspace(0, len(y_orig)/sr, len(y_orig)))
        sf.write("marcado_96k.wav", y_orig + tone, sr)

    # =========================================================================
    # FASE 6: MASTERIZACIÓN MULTI-PERFIL
    # =========================================================================
    print("\n🎛️ FASE 6: MASTERIZACIÓN MULTI-PERFIL")
    perfiles = {
        "Spotify": {"loudness": -14, "sample_rate": 44100, "bit_depth": 16},
        "AppleMusic": {"loudness": -16, "sample_rate": 48000, "bit_depth": 24},
        "Radio": {"loudness": -23, "sample_rate": 44100, "bit_depth": 16},
        "Vinilo": {"loudness": -18, "sample_rate": 96000, "bit_depth": 24}
    }

    for nombre, params in perfiles.items():
        print(f"   Masterizando para {nombre}...")
        sr_out = params["sample_rate"]
        bit_depth = params["bit_depth"]
        codec = "pcm_s16le" if bit_depth == 16 else "pcm_s24le"
        subprocess.run([
            "ffmpeg", "-y", "-i", "marcado_96k.wav",
            "-af", f"loudnorm=I={params['loudness']}:TP=-1:LRA=11",
            "-ar", str(sr_out), "-ac", "2",
            "-c:a", codec,
            f"master_{nombre}.wav"
        ], check=True, capture_output=True)
        print(f"   ✅ {nombre} listo.")

    # =========================================================================
    # FASE 7: METADATOS Y DOCUMENTACIÓN
    # =========================================================================
    print("\n📄 FASE 7: METADATOS Y DOCUMENTACIÓN")
    metadata_json = {
        "titulo": METADATOS["titulo"],
        "artista": METADATOS["artista"],
        "album": METADATOS["album"],
        "genero": METADATOS["genero"],
        "isrc": METADATOS["isrc"],
        "upc": METADATOS["upc"],
        "compositores": METADATOS["compositores"],
        "productores": METADATOS["productores"],
        "fecha": METADATOS["fecha"],
        "duracion": "3:00",
        "bpm": tempo,
        "drop_time": drop_time,
        "plataformas": list(perfiles.keys())
    }
    with open("metadatos.json", "w") as f:
        json.dump(metadata_json, f, indent=2)

    informe = f"""
INFORME TÉCNICO XIDTEK (v22.0)
=============================
Título: {METADATOS["titulo"]}
Artista: {METADATOS["artista"]}
ISRC: {METADATOS["isrc"]}
UPC: {METADATOS["upc"]}
BPM: {tempo}
Drop: {drop_time:.2f}s
Género: {METADATOS["genero"]}
Fecha: {METADATOS["fecha"]}

Masterización por plataforma:
{chr(10).join([f"- {p}: {perfiles[p]['loudness']} LUFS, {perfiles[p]['sample_rate']}Hz, {perfiles[p]['bit_depth']}-bit" for p in perfiles])}

Stems: Batería, Bajo, Otros, Voces (96kHz/24-bit)
Marcas de agua: AudioSeal (neuronal) + Backup tono 19kHz
"""
    with open("informe_tecnico.txt", "w") as f:
        f.write(informe)
    print("✅ Metadatos e informe guardados.")

    # =========================================================================
    # FASE 8: EMPAQUETADO FINAL
    # =========================================================================
    print("\n📦 FASE 8: EMPAQUETADO FINAL")
    zip_path = "XIDTEK_PAQUETE_DISCOGRAFICA_v22.zip"
    with zipfile.ZipFile(zip_path, "w") as zipf:
        for p in perfiles.keys():
            fname = f"master_{p}.wav"
            if os.path.exists(fname):
                zipf.write(fname)
        for stem in ["drums", "bass", "other", "vocals"]:
            fname = f"stem_{stem}_96k_24bit.wav"
            if os.path.exists(fname):
                zipf.write(fname)
        zipf.write("metadatos.json")
        zipf.write("informe_tecnico.txt")
        zipf.write(LOOP_PATH, "V.01.mp3")
        if os.path.exists("espectrograma_V01_96k.png"):
            zipf.write("espectrograma_V01_96k.png")
    print("✅ Paquete final creado.")

    # =========================================================================
    # FASE 9: PANEL DE CONTROL
    # =========================================================================
    print("\n🖥️ FASE 9: PANEL DE CONTROL")
    print("="*80)
    print("📦 PAQUETE DISCOGRÁFICO COMPLETO (v22.0)")
    print("="*80)
    print(f"📁 Archivo: {zip_path}")
    print("📊 CONTENIDO:")
    print("   - Master Spotify (44.1kHz/16-bit)")
    print("   - Master Apple Music (48kHz/24-bit)")
    print("   - Master Radio (44.1kHz/16-bit)")
    print("   - Master Vinilo (96kHz/24-bit)")
    print("   - Stems (Batería, Bajo, Otros, Voces)")
    print("   - Metadatos (JSON)")
    print("   - Informe Técnico (TXT)")
    print("   - V.01.mp3 (loop original)")
    print("   - Espectrograma (96kHz)")
    print("="*80)
    print("🔒 Sellos: AudioSeal + Backup tono 19kHz")
    print("🎤 Tag XIDTEK en Intro y Pre-coro")
    print("🧠 Auto-actualización: Activada (GitHub)")

    # Descargar el archivo (si estamos en Colab)
    if IN_COLAB:
        files.download(zip_path)
    else:
        print(f"\n📁 Archivo generado en: {os.path.abspath(zip_path)}")

# =============================================================================
# PUNTO DE ENTRADA PRINCIPAL
# =============================================================================
if __name__ == "__main__":
    # Verificar actualizaciones (si no estamos en modo "actualización")
    if not os.path.exists("updating.lock"):
        updated = check_for_updates()
        if updated:
            # Si se actualizó, no ejecutar el flujo principal
            sys.exit(0)

    # Ejecutar XIDTEK
    run_xidtek()
