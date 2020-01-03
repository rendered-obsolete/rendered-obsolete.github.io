### Words

The [__Words__ tab](https://rhasspy.readthedocs.io/en/latest/usage/#words-tab) is used to add [custom words](https://rhasspy.readthedocs.io/en/latest/training/#custom-words) to `custom_words.txt`.  When you click __Train__ these are then merged to `dictionary.txt` for [speech-to-text transcription](https://rhasspy.readthedocs.io/en/latest/speech-to-text/):  

![](/assets/rhasspy_words_train.png)

To add new words:  

1. Enter word in __Word__
1. Click __Lookup__ and wait a few seconds
1. Select desired variant from __Pronunciations__ (if available)
1. Click __Add__ to add the word and its phonemes to __Custom Words__
    - __Pronouce__ and __Download__ use eSpeak TTS to convert it to audio and then play it or download the wav, respectively
1. Repeat 1-4 to taste, then __Save Custom Words__ and __Train__
1. After 15-30 seconds (on a Pi3) a `Training completed` pop-up should appear



https://www.youtube.com/watch?v=L5g_t6FAYLY



Uses the [Hermes protocol over MQTT](https://rhasspy.readthedocs.io/en/latest/usage/#mqtt)- just like Snips!