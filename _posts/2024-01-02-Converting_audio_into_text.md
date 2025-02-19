---
title: Converting Audio into Text (with Predict Punctuation)
description: >-
  Machine Learning Project
author: Cronus
date: 2024-01-02 10:43:00 
categories: [Project, Edu Helper]
tags: [Machine Learning]
pin: false
image:
  path: /assets/img/ML/convert.jpeg
  width: 300
  height: 300
---


EDU\_Siri program has a function that converts the instructor's voice into a text file and saves it. I thought I'd write down some of the things I learned and struggled with while implementing this feature.

I can search "speech\_recognition" library when googling. Initially, I decided to use this module, but there were two factors hat made it unsuitable for development.

1.  While googling about "speech\_recognition", the majority of codes run as the following: Input voice using mic and these input values are converted into text. But my program doesn't utilize the inputting system using a mic, so I needed to come up with another algorithm.
2.  "speech\_recognition" module can convert the voice in a video into a text. However, the module has a size limit. I was unable to use this module, due to its ability to only transcribe the speech to a text for up to 100MB.
3.  If you check a result with speech\_recognition, you can see that it does not recognize characters such as . , ? !. I need to pass a speech recognized text to a summarization function, which would create a matrix based on punctuation marks such as dots. That is, I can't use "speech\_recognition" without a punctation mark. -> This is the biggest reason for not using "speech\_recognition".

I finally choose VOSK Model.(the solution code of converting voice into text without a size limit). It's voice recognition tool-kit. It was pretty hard to search references while developing.

_vosk model download : [https://alphacephei.com/vosk/models](https://alphacephei.com/vosk/models)_

I chose the lightest option, vosk-model-small-en-us-0.15 for lightweighting. Let's check the code.

#### **1\. Convert Video into Audio file**

Because the program starts with receiving a video file, it converts the video file to an audio file before speech recognition. "FRAME\_RATE" decides smapling rate, which is related to sound quality. Google told me that 16000 is the most optimized value, so I used it. "Channels" is related to features of voice. A value of 1 is the one-dimensional sound we typically hear ( a value of 2 seems to have a similar effect to surround sound). 

```python
# Convert Video to audio file
clip = mp.VideoFileClip(video_path)
clip.audio.write_audiofile(speech_path)

FRAME_RATE = 16000
CHANNELS = 1
```

#### **2\. Basic Setup  Code**

Define model that I will use and setting value. During development phase, I set "Words(True)" to check that sound file is exactly converted into text file.

```python
# set up
model = Model(model_name='vosk-model-small-en-us-0.15')
rec = KaldiRecognizer(model, FRAME_RATE)
rec.SetWords(True)
```

You can see both the completed sentence and the individual words translated by voice by setting "SetWords(True)".

```python
speech = AudioSegment.from_mp3(speech_path) # Load file
speech = speech.set_channels(CHANNELS)
speech = speech.set_frame_rate(FRAME_RATE)

rec.AcceptWaveform(speech.raw_data)
result = rec.Result()
```

Now load the file, run it through the speech recognition parser to get the speech raw data and return it in the result variable. The result value is like this.

![1.jpeg](/assets/img/ML/convert(1).jpeg)

Because the result I only want to know is translated sound text, get value of text by using json.loads funciton.

```python
text = json.loads(result)['text']
```

#### **3\. Prediction of Punctuation**

If you read the above text value, it will print out fine as a complete sentence, but there are no characters such as . , ? !. Without these punctuation marks, the function that summarize text can't configure appropriate matrix. That is, text summarization function becomes useless. So I need a solution of predicting punctuation marks.

[https://alphacephei.com/vosk/models/vosk-recasepunc-en-0.22.zip](https://alphacephei.com/vosk/models/vosk-recasepunc-en-0.22.zip)

"vosk-recasepunc" predicts the punctuation of sentences and puts marks such as . , ? ! in appropriate places. You can see two way to use this when goolgling. In my case, using the subprocess module to mindlessly run the recasepunc.py file worked well, so that's what I chose to do.

```python
cased = subprocess.check_output('python recasepunc/recasepunc.py predict recasepunc/checkpoint', shell=True, text=True, input=text)
cased = cased.replace(" ' ", "'").replace(" ? ", "? ").replace(" ! ", "! ")
```

#### **4\. Result**

```
So if you look at recent results from several different leading speech groups, Microsoft showed that this kind of deep neural network 
when used to see coasting model and // (The rest is omitted)
```

#### **Full Code**

```python
def video_to_text(video_name):
    # Set path of video and speech file
    base_path = os.getcwd()
    video_path = base_path + "\\video\\" + str(video_name)
    speech_file = str(video_name).split('.')[0]
    speech_file = speech_file + ".wav"
    speech_path = base_path + "\\speech\\" + str(speech_file)

    # Convert Video to sound file
    clip = mp.VideoFileClip(video_path)
    clip.audio.write_audiofile(speech_path)

    FRAME_RATE = 16000
    CHANNELS = 1

    try:
        model = Model(model_name='vosk-model-small-en-us-0.15')
        rec = KaldiRecognizer(model, FRAME_RATE)
        rec.SetWords(True)
        print("\n\n#############    Now, I'm on my work.. It takes a few minutes.  #################")
        print("##################      It's okay to ignore warnings!      ##########################")
        print("####  If the program is still stuck after enough time has passed, press Enter.  #####")
        speech = AudioSegment.from_mp3(speech_path)
        speech = speech.set_channels(CHANNELS)
        speech = speech.set_frame_rate(FRAME_RATE)

        rec.AcceptWaveform(speech.raw_data)
        result = rec.Result()
        text = json.loads(result)['text']
        cased = subprocess.check_output('python recasepunc/recasepunc.py predict recasepunc/checkpoint', shell=True, text=True, input=text)
        cased = cased.replace(" ' ", "'").replace(" ? ", "? ").replace(" ! ", "! ")
        with open('speech_result.txt',mode ='a') as file: 
            file.write("\n==============================================\n")
            file.write("Content: \n") 
            file.write(str(cased)) 
            print("+===========================================+")
            print("| Converting is done! (Video Sound -> Text) |")
            print("+===========================================+")
    
    except Exception as e:
        error("Error occurred during converting video sound to text! The file is probably an unsupported format")
```