# Multi Task Learning Speaker Embeddings
Code for the paper: "Leveraging speaker attribute information using multi task learning for speaker verification and diarization" submitted to ICASSP 2021.

Along with experimental code, this repo covers webscraping age information for lawyers in SCOTUS, and nationality for Wikipedia celebrities.

# Contents

- Requirements
- SCOTUS Data Preparation
- VoxCeleb Data Preparation
- Experiments

# Requirements

This was tested on python 3.6.8

- General requirements:
    - numpy
    - tqdm
    - sklearn
    - scipy
- Web-scraping:
    - beautifulsoup4
    - wikipedia
    - wptools
    - Levenshtein
    - python-dateutil
- Experiments:
    - Kaldi
    - torch
    - kaldi_io

You will also need access to the avvo.com API, along with a custom google search API key, with instructions of how to obtain/set these up in the SCOTUS Data Preparation section.

# SCOTUS Data Preparation

## Obtaining case information

This work utilizes the information available via the [Oyez project](https://www.oyez.org/about-audio), which provides audio files and transcriptions for US Supreme Court oral arguments.

The Oyez project has an API, an example of which can be seen here: https://api.oyez.org/cases?per_page=100

Thankfully, another GitHub repo exists which has already scraped the information provided by the API for each case and auto-updates every week: https://github.com/walkerdb/supreme_court_transcripts. This ~3.5GB repo will be used to extract the cases that we want audio and transcripts for.

The following command will clone this repo (specifically my fork, which doesn't update with new cases, to ensure consistency with the number of cases that I have tested my code with):

```sh
git clone https://github.com/cvqluu/supreme_court_transcripts
```

The location of this repository which contains the case information will be referred to `$SCOTUS_CASE_REPO` throughout the rest of the SCOTUS section. Of course, if you want to obtain the most up to date recordings, then you should clone the original repo by walkerdb.

## Downloading Audio and Filtering Cases

Now that `$SCOTUS_CASE_REPO` exists and has been cloned from GitHub, we can now make us of the MTL-Speaker-Embeddings repo:

```sh
git clone https://github.com/cvqluu/MTL-Speaker-Embeddings
cd MTL-Speaker-Embeddings/scotus_data_prep
```

which puts us inside the `scotus_data_prep` folder, which is where the data prep scripts for SCOTUS are all present. 

Next, we want to run the first step, which downloads mp3 files and filters the cases which we can use. This takes in as input the location of the case JSONs in `$SCOTUS_CASE_REPO` and then produces a data folder (of which the location is up to you) which we will call `$BASE_OUTFOLDER`, which will store all the SCOTUS data to be used later on.

```sh
python step1_downloadmp3.py --case-folder $SCOTUS_CASE_REPO/oyez/cases --base-outfolder $BASE_OUTFOLDER
```

This parses through the cases in `$SCOTUS_CASE_REPO` and eliminates cases argued before October 2005, as this is when the Supreme Court started recording digitally, instead of using a reel-to-reel taping system. The taped recordings were excluded as there were a number of problems and defects with these recordings as detailed here: https://www.oyez.org/about-audio. This script also eliminates cases where audio can't be found, and also ones where the transcription has a lot of invalid speaker turns (0 or less duration speaker turns).

The outcome should be a file structure in `$BASE_OUTFOLDER` something like so:

```
$BASE_OUTFOLDER
├── speaker_ids.json
├── audio
|   ├── 2007.07-330.mp3
|   └── ...
└── transcripts
    ├── 2007.07-330.json
    └── ...
```

Sometimes, downloading audio files may fail, but succeed on subsequent attempts. This script is safe to run multiple times, and will try and re-download files that are missing (while skipping ones that already have been processed).

If you have used my fork for `$SCOTUS_CASE_REPO` then you should end up with 2035 mp3s/jsons in the audio and transcripts folders respectively, although this may not be consistentent, depending on the availability of every mp3.
 
## Web scraping

Next, we want to scrape the approximate DoB of each speaker found in the recordings downloaded, which is stored in `$BASE_OUTFOLDER/speaker_ids.json`.

First of all, we need to set up and obtain a few things in order to fill out `local_info.py` inside of `scotus_data_prep`.

### Avvo API

Instructions on how to obtain access to the Avvo API are here: http://avvo.github.io/api-doc/

Once this is set up, you can fill out `local_info.py` with your own `AVVO_API_ACCESS_TOKEN`. (Note: `AVVO_CSE_ID` is covered in the next section)

### Custom Google Search

A custom google search will be needed, which can be set up here:

https://cse.google.com/cse/all

You will want to click `Add` to create a new search engine, and under `Sites to search`, you will want to enter `avvo.com`, with the Language specified as English.

After clicking Create, this should succesfully create this custom search. The control panel will take you to where you can find the `Search Engine ID`, which you can fill into `AVVO_CSE_ID` in `local_info.py`.

You will also need a google API key, which can be set up here: https://developers.google.com/maps/documentation/javascript/get-api-key

Once this is obtained, fill out `GOOGLE_API_KEY` in `local_info.py`.

### Running the webscraper

```sh
python step2_scrape_dob.py --base_outfolder $BASE_OUTFOLDER
```

This will go through the names in `speaker_ids.json` and try and scrape their dates of birth (DoB) from the following sources:

- [Wikipedia](https://en.wikipedia.org/)
- [Supreme court clerkship graduation dates](https://en.wikipedia.org/wiki/List_of_law_clerks_of_the_Supreme_Court_of_the_United_States_(Chief_Justice))
- JUSTIA.com
- Avvo.com

For the final three sources, the year discovered for graduation/admission to practice law will be subtracted by 25 to obtain an approximate date of birth. The dates of birth, and additional information about the source of each DoB are stored in pickled dictionaries in `$BASE_OUTFOLDER`:

```
$BASE_OUTFOLDER
├── speaker_ids.json
├── dobs.p
├── dobs_info.p
├── audio
|   └── ...
└── transcripts
    └── ...
```

Like `step1_downloadmp3.py`, this script is also safe to re-run, and will try to re-scrape names for which no DoB has been found. If you wish to skip the names which have been attemped, and are present in the `dobs.p` dictionary, the step2 script can be run with a `--skip-attempted` flag.

## Prepping data for feature extraction

```sh
python step3_prepdata.py --base_outfolder $BASE_OUTFOLDER
```

This prepares verification and diarization data folders that are ready for Kaldi to extract features from. As a result, the recordings are changed into the consistent naming format `YEAR-XYZ`, and this mapping from orignal names to new names is stored in a JSON file.

Verification/training data is made by splitting up long utterances into non-overlapping 10s segments, with minimum length 4s. Utterances are named in a consistent fashion, and the age (in days) of each speaker at the time of each utterance is calculated, and placed into an `utt2age` file.

Diarization data is made by splitting into 1.5s segments with 0.75s shift. A `ref.rttm` file is also created.

This should result in the following file structure (in addition to what was previously shown):

```
$BASE_OUTFOLDER
├── ...
├── orec_recid_mapping.json
├── recid_orec_mapping.json
├── veri_data
|   ├── utt2spk
|   ├── spk2utt
|   ├── utt2age
|   ├── wav.scp
|   ├── segments
|   ├── real_utt2spk
|   └── real_spk2utt
└── diar_data
    ├── utt2spk
    ├── spk2utt
    ├── ref.rttm
    ├── wav.scp
    ├── segments
    ├── real_utt2spk
    └── real_spk2utt

```

The `real_{utt2spk|spk2utt}` are there as Kaldi feature extraction insists on speaker ids being the prefix to an utterance name - these will be sorted out later on.

## Feature extraction

We will use Kaldi to extract the features. This kaldi feature extraction script is found in `step4_extract_feats.sh`.

You will need to edit at the top of this script and fill in your own $BASE_OUTFOLDER

```sh
#!/bin/bash

step=0
nj=20
base_outfolder=/PATH/TO/BASE_OUTFOLDER <--- Edit here

...
```

We used the `egs/voxceleb/v2` recipe folder to carry out this script (although the only thing specific to this recipe is the `conf/mfcc.conf`).

To carry this out, we will assume you have Kaldi installed at `$KALDI_ROOT`

```sh
cp step4_extract_feats.sh $KALDI_ROOT/egs/voxceleb/v2/
cd $KALDI_ROOT/egs/voxceleb/v2
source path.sh
bash step4_extract_feats.sh
```

## Making train/test splits

After changing back your working directory to `scotus_data_prep`, we can run the final stage script.

```sh
python step5_trim_split_data.py --base_outfolder $BASE_OUTFOLDER --train-proportion 0.8 --pos-per-spk 12
```

This trims down the data folders to match what features have been extracted, and splits the recordings into train and test according to `--train-proportion`. It also makes a verification task for test set utterances, excluding speakers seen in the training set, generating trials of pairs of utterances to compare.

The `--pos-per-speaker` option determines how many positive trials there are per test set speaker. If too high a value is selected, this may error out as it cannot select enough utterances, so if that occurs, try lowering this value.

This should yield the following (once again in addition to what was produced before):

```
$BASE_OUTFOLDER
├── ...
├── veri_data_nosil
|   ├── train
|   |   ├── utt2spk
|   |   ├── spk2utt
|   |   ├── utt2age
|   |   ├── feats.scp
|   |   └── utts
|   ├── test
|   |   ├── veri_pairs
|   |   └── ...
|   └── ...
└── diar_data_nosil
    ├── train
    |   ├── ref.rttm
    |   └── ...
    ├── test
    |   ├── ref.rttm
    |   └── ...
    └── ...
```

# VoxCeleb Data Preparation

TODO

# Experiments

TODO