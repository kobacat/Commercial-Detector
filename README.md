This tool processes video files to automatically identify and extract commercials and bumpers.

It uses:
- Speech-to-text (Whisper)
- Speaker diarization (pyannote)
- Visual cues (black frame detection)
- LLMs (OpenAI GPT or Ollama) for segmentation logic

## Requirements
- A computer that can run moderate ML workloads (though we use ChatGPT for the LLM, we still run Whisper and PyAnnote locally)
- Python 3.10.x - 3.12.x (tested most with 3.12.3)
- ffmpeg installed and accessible from the command line
- OpenAI API access
    - Alternatively, you can use ollama, but I found its results very poor in comparison.
    - My usage costs were under $0.10 for over an hour of commercials, which was acceptable for my purposes.
- Access to two huggingface projects by pyannote:
    - [Speaker Diarization](https://huggingface.co/pyannote/speaker-diarization-3.1)
    - [Segmentation](https://huggingface.co/pyannote/segmentation-3.0)
- A .env file with the following environment variables:
    ```env
    OPENAI_API_KEY=your_openai_key
    HUGGINGFACE_TOKEN=your_huggingface_token
    ```

## Installation

```sh
pip install git+https://github.com/matthew-c-lee/Commercial-Detector.git
```

Then you should be able to call it with:

```sh
detect-commercials
```

Upon running it for the first time, it will ask for your OpenAI API Token and a HuggingFace token to download the models. It will store these locally in a .env file.

## Command line usage
```sh
detect-commercials <video_file_or_directory> [options]
```

| Option | Description |
|-|-|
| video_file (required) | Path to a video file (e.g., .mp4) or a directory of videos |
| --output_dir (required) | Directory to save extracted clips |
| --reprocess | Forces reprocessing even if the video is already cached |
| --override_cache | Ignores the entire cache and processes everything from scratch |
| --use_cached_llm_response	| Uses cached LLM results if available (mainly for testing) |

## How it works

Before it processes any of the video files, it will check them by hash to see if they've been processed before (info is stored in sqlite db) and by default skip it if so.

It will also give a rough estimate how many input tokens will be used in our API call to OpenAI, and therefore how much money the job is expected to cost. It will prompt you asking if you wish to proceed given the cost.

1. Audio Extraction: Audio is extracted from the video via ffmpeg.
2. Transcription: The audio is transcribed using OpenAI's Whisper model.
3. Diarization: Speaker identification is performed using PyAnnote + HuggingFace.
4. Visual Cue Detection: Black frames are detected to help identify transitions.
5. Segmentation:
    - Segments are chunked.
    - Prompts are sent to an LLM (gpt-4o or ollama) asking it to identify commercials/bumps.
    - Segments are labeled using timestamps.
6. Snapping to closest "fade to black" time:
    - After the LLM returns labeled segments like:
        ```
        trix_cereal [00:01:05 - 00:01:55]
        bump [00:01:55 - 00:02:05]
        ```
    - We refine the timestamps by aligning them with fade-to-black moments in the video. 
    - The LLM gets us most of the way there, but it's not perfect for exact timings of the segment boundaries. This way, we can get the best of both worlds.

7. Output:
    - Clips of each labeled segment (e.g., trix_cereal.mp4, bump_1.mp4, etc.) are exported into the --output_dir.
