# Pipeline Architecture Design Document 

**Preprocessing bash (functions):**

- **WAV Conversion:**
    cmd1 = f'ffmpeg -y -i "{INPUT_AUDIO}" -c:a pcm_s16le -ar {SAMPLING_RATE} "{WAV_AUDIO}" > "{WAV_AUDIO}.log" 2>&1'
    
- **Silence Removal & Volume Normalization:**
    cmd2 = f'ffmpeg -y -i "{WAV_AUDIO}" -af "silenceremove=start_periods=1:stop_periods=-1:start_threshold=-50dB:stop_threshold=-50dB:start_silence=0.2:stop_silence=0.2, loudnorm" -c:a pcm_s16le -ar {SAMPLING_RATE} "{SILCUT_AUDIO}" > "{SILCUT_AUDIO}.log" 2>&1'
    
- **Speech Normalization:**
    cmd3 = f'ffmpeg -y -i "{SILCUT_AUDIO}" -af "speechnorm=e=50:r=0.0005:l=1" -c:a pcm_s16le -ar {SAMPLING_RATE} "{CPS_AUDIO}" > "{CPS_AUDIO}.log" 2>&1'

**How to execute commands:** if os.system(cmd) != 0:
        raise RuntimeError("FFmpeg failed...")

---

**Transcribing:** **Faster Whisper** —> Batched inference with Silero VAD.

    from faster_whisper import WhisperModel, BatchedInferencePipeline

    model_size = "large-v3"

    model = WhisperModel(model_size, device="cuda", compute_type="float16")
    batched_model = BatchedInferencePipeline(model=model)

    segments_gen, info = batched_model.transcribe(
        CPS_AUDIO, 
        language="it",
        beam_size=5, 
        vad_filter=True,
        vad_parameters=dict(min_silence_duration_ms=500),
        batch_size=16,
    )

    segments = list(segments_gen)

    for segment in segments:
        print("[%.2fs -> %.2fs] %s" % (segment.start, segment.end, segment.text))

---

**Postprocessing:** - **Remove non-ASCII characters:**
    
    def remove_non_ascii_en(text):
        return ''.join(i for i in text if ord(i)<128)
    
    def remove_non_ascii_it(text):
        # Set of both grave and acute accents used in Italian, upper and lowercase
        italian_accents = set("àèéìíòóùúÀÈÉÌÍÒÓÙÚ")
        return ''.join(i for i in text if ord(i) < 128 or i in italian_accents)
    
- **Correct misspellings in each segment:** from autocorrect import Speller
    
    # Initialize the lightweight spell checker for Italian
    spell = Speller(lang='it')
    
    # Example usage inside the segment loop:
    clean_text = remove_non_ascii_it(raw_text)
    corrected_text = spell(clean_text)

---

** Consulted References:**

* A Comparative Study of Speech-to-Text for Italian : https://ceur-ws.org/Vol-4121/Ital-IA_2025_paper_33.pdf
* Enhancing Whisper transcriptions: pre- & post-processing techniques (article) -> https://developers.openai.com/cookbook/examples/whisper_processing_guide
* Enhancing Whisper transcriptions: pre- & post-processing techniques (notebook) -> https://github.com/Azure-Samples/openai/blob/main/Basic_Samples/Whisper/dotnet/csharp/Whisper_processing_guide.ipynb
* Audio Pre-Processings For Better Results in the Transcription Pipeline: https://medium.com/@developerjo0517/audio-pre-processings-for-better-results-in-the-transcription-pipeline-bab1e8f63334
* https://github.com/SYSTRAN/faster-whisper
* https://mobiusml.github.io/batched_whisper_blog/
* https://github.com/EtienneAb3d/WhisperHallu/tree/main
