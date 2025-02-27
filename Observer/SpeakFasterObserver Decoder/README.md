## Processing a raw Observer data session

### Python environment

Python version 3.8+ is required. You can check your Python version with the
following command:

```sh
python --version
```

It is highly recommended that you do Python development in a virtualenv.

If `virtualenv` is not on your path, it may be because it hasn't been installed.
To install virtualenv on a Debian or Ubuntu machine, do:

```sh
sudo apt-get install python3-virtualenv
```

Once virtualenv is installed, you can create a virtualenv by running a command
like:

```sh
virtualenv -p python3 /home/cais/speakfaster-venv
```

The "/home/cais/speakfaster-venv" here is just an example. You should
use your own folder of choice. Once the virtualenv is created, you can
activate the virtualenv by running command:

```sh
source /home/cais/speakfaster-venv/bin/activate
```

In install the required dependencies in the virtualenv, do:

```sh
pip install -r requirements.txt
```

The requirements.txt is available in the same directory as this README file.
It lists all the required Python packages.

### Installing Python dependencies ffmpeg

ffmpeg is required for video processing. To install ffmpeg on Linux, do

```sh
sudo apt-get install ffmpeg
```

To install ffmpeg on Windows, download an ffmpeg build for Windows from
https://github.com/BtbN/FFmpeg-Builds/releases, then unzip the binary
files to an appropriate folder. Add the folder, e.g.,
"C:\Program Files\ffmpeg-n4.4-80-gbf87bdd3f6-win64-lgpl\bin", to the
Path environment variable.

To install ffmpeg on Mac, you can use homebrew:

```sh
brew install ffmpeg
```

### Setting up Google Cloud credentials

The pre-processing script uses Google Cloud Speech-to-Text API for automatic speech
recognition (ASR) and speaker diarization. You should create or obtain a JSON key file
associated with your Google Cloud account and put it at a secure location under
your home directory (e.g., "/home/cais/keys/my-google-cloud-key.json"). Then
modify your .bashrc or .zshrc (depending on what type of Linux shell is used) file
by adding a line that defines the environment variable:

```sh
export GOOGLE_APPLICATION_CREDENTIALS=/home/cais/keys/my-google-cloud-key.json
```

(Actual path may differ). Save the file. To make the change take effect, either
open a new Linux terminal or do `source ~/.bashrc` or `source ~/.zshrc`.

### Pre-processing raw data for ELAN

The following command processes a raw data session folder from the Observer with
the keypresses protobuf file along with audio recordings and image files.
It extracts audio event labels and ASR transcripts (with tentative speaker IDs).
These audio-based labels are merged with the keypress data as a single merged.tsv file,
which can be loaded into ELAN.

These other files are also generated by the script and can be loaded into ELAN:
- concatenated_audio.wav: The concatenated audio file.
- A screenshots.mp4 video file that is the result of stitching together the
  screenshot .jpg files based on their timestamps, or in case the screenshot
  .jpg files are unavailable:
  - A dummy screenshots video file

```sh
python elan_format_raw.py \
    --gcs_bucket_name="${GCS_BUCKET_NAME_FOR_TEMP_AUDIO}" \
    /home/cais/sf_observer_data/session_3_with_screenshots/ \
    US/Eastern
```

The flag `--gcs_bucket_name` points to the Google Cloud Storage (GCS) bucket
that holds temporary audio files for speech recognition. Make sure you have
write permissions to the bucket. The default value of this flag is for
development purposes only and is unlikely to work for you.

In case screenshot image files are missing from an input directory, use the
`--dummy_vide_frame_image_path` flag to let the script generate a static
dummy video:

```sh
python elan_format_raw.py \
    --gcs_bucket_name="${GCS_BUCKET_NAME_FOR_TEMP_AUDIO}" \
    /home/cais/sf_observer_data/session_2_with_keypresses/ \
    US/Eastern \
    --dummy_video_frame_image_path="${HOME}/SpeakFaster/Observer/SpeakFasterObserver Decoder/testdata/generic_windows_desktop.jpg"
```

### Postprocessing curation result

Based on the Data Curation Playbook, you should export a TSV file named
`curated.tsv` to the data-session directory at the end of the manual curation
process. Once this file has been exported, use the `elan_process_curated.tsv`
script to perform quality check and final conversion on the file. Example
command line:

```sh
python elan_process_curated.py \
    /home/cais/sf_observer_data/session_5_practice_conversation_2 \
    path/to/speaker_map.json
```

The first argument is the path to directory where the `curated.tsv` is located.
The second argument is the path to a JSON file which is expected to contain a
field `realname_to_pseudonym`, which maps speakers' real names
to their respective pseudonyms. The JSON is also used by Observer for online
speaker recognition. So it may have other fields such as `id_to_realname` and
fields related to Azure speaker recognition configurations. These other fields
are unused by the `elan_process_curated.py` script.

```json
{
  "azure_subscription_key": "<REDACTED>",
  "azure_endpoint": "https://westus.api.cognitive.microsoft.com",
  "id_to_realname": {
    "c15e9704-bedf-4cab-9425-7aeedf7f0f79": "Sean",
    "86b5bca5-903b-4e1a-853e-1ec6e3d1aad0": "Sherry"
  },
  "realname_to_pseudonym": {
    "Sean": "User001",
    "Sherry": "Partner001"
  }
}
```

(The UUIDs, names, and pseudonyms in the sample JSON above are just examples.)

The `elan_process_curated.py` script is able to find the following types of
possible errors in `curated.tsv` (an incomplete list):

- Duplicate real names or pseudonyms in the `speaker_map.json` file provided.
- Incorrect # of columns
- Incorrect tier names
- tEnd value less than tBegin value in any row
- Real names in `[Speaker:${RealName}]` or `[SpeakerTTS:${RealName}]` tags
  that are not found in the `speaker_map.tsv` file provided.
- A row of the `SpeechTranscript` tier contains no speaker tag such as
  `[Speaker:Sherry]` at the end.
- Incorrect keypress redaction time range format.
- Keypress redaction time ranges that are not found in the Keypresses tier of
  the `curated.tsv` file.

Additionally, the postprocessing script generates a JSON file named
`curated_processed.json` to accompany the TSV file. This file contains metadata such
as timestamps and statistics of the curated speech utterances (e.g., automatically
extracted part-of-speech tags, token count, token lenghts, etc.) prior to masking
out the redacted sections.

When you run into these errors, go back to ELAN, fix the problem and re-export
the `curated.tsv` file and re-run the `elan_process_curated.tsv`. Fix all problem
until the script says "Success..." and exports a file in the same directory named
`curated_processed.tsv`. This new TSV file is ready for data ingestion.

## Individual Pre- and Post-processing Steps

NOTE: The aforementioned `elan_format_raw.py` and `elan_process_curated.py`
scripts should automatically
take care of the pre- and post-processing. The info in this section is relevant
only if you plan to perform individual aspects of the pre- or post-processing
yourself.

### Audio Event Classification

We use [YAMNet](https://tfhub.dev/google/lite-model/yamnet/tflite/1)
to extract audio event labels from input audio files.

Command line example:

```sh
python extract_audio_events.py testdata/test_audio_1.wav /tmp/audio_events.tsv
```

### Visual Object Detection

We use [SSD on MobileNetV2](https://tfhub.dev/tensorflow/ssd_mobilenet_v2/fpnlite_640x640/1) to detect visual objects in images captures from camera(s).

Command line example for an input video file (e.g., an .mp4 file):

```sh
python detect_objects.py \
    --input_video_path testdata/test_video_1.mp4 \
    --output_tsv_path /tmp/visual_objects.tsv
```

Command line example for a series of image files specified by a glob pattern:

```sh
python detect_objects.py \
    --input_image_glob 'testdata/pic*-standard-size.jpg' \
    --frame_rate 2 \
    --output_tsv_path /tmp/visual_objects.tsv
```

Note the test images in the testdata/ folder are under the CC0 (public domain)
license and are obtained from the URLs such as:
- https://search.creativecommons.org/photos/10590078-2f13-4caf-b96d-5d1db14eccd4
- https://search.creativecommons.org/photos/832045ea-53f3-4a3d-9c35-9b51f9add43d

### Automatic Speech Recognition (ASR, Speech-to-text) on audio files

The SpeakFaster Observer writes audio data to .flac files that are approximately
60 second each in length. To transcribe a consecutive series of such .flac files,
find the path to the first .flac file in the series and feed it to the audio_asr.py
script. Additionally, provide path to the  output .tsv file. For example:

```sh
python audio_asr.py data/20210710T095258428-MicWaveIn.flac /tmp/speech_transcript.tsv
```

The script automatically finds consecutive audio files in the same directory as the
first audio file based on the length of each audio file and the timestamps in the
file names.

To perform ASR and speaker diarization at the same time, use the `--speaker_count`
argument. For example:

```sh
python audio_asr.py \
    --speaker_count=2 \
    data/20210710T095258428-MicWaveIn.flac /tmp/speech_transcript.tsv
```

The speaker count must be known beforehand. In the .tsv file, the `Content`
column with contain the speaker index (e.g., "Speaker 2") appended to the
transcripts.

## Running unit tests in this folder

Use:

```sh
./run_tests.sh
```

## Speaker ID enrollment and profile management

We use Azure Cognitive Service's cloud speech API for real-time and offline speaker
ID. The script in this directory `speaker_id_profiles.py` allows you to enroll
new speakers by using their voice samples in the format of WAV file, list
enrolled speakers, and delete existing enrolled speakers.

To enroll a new speaker voice, make sure you have a mono (single-channel) WAV file
with a sample rate 16000 Hz which contains at least 20 seconds of the speaker's
voice sample. Then do:

```sh
python speaker_id_profile.py enroll \
    --azure_subscription_key="${AZURE_SUBSCRIPTION_KEY}" \
    --wav_path=/path/to/my/voice_sample.wav
```

The console printout will contain the profile ID of the new speaker voice.

To list all enrolled voices, do:

```sh
python speaker_id_profile.py list \
    --azure_subscription_key="${AZURE_SUBSCRIPTION_KEY}"
```

To delete an existing enrolled voice, do:

```sh
python speaker_id_profile.py delete \
    --azure_subscription_key="${AZURE_SUBSCRIPTION_KEY}" \
    --profile_id="${PROFILE_ID_TO_DELETE}"
```

If the response status code is 200, it means the deletion is successful. You can
list the enrolled voices again to confirm that.
