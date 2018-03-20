# Personal Assistant (Jarvis) in Python

I thought it would be cool to create a **personal assistant** in
[**Python**](https://pythonspot.com/). If you are into movies you may have
heard of Jarvis, an A.I. based character in the Iron Man films. In this
tutorial we will create a [**robot**.](https://pythonspot.com/robotics)

The features I want to have are:

For this tutorial you will need (Ubuntu) Linux,
[**Python**](https://pythonspot.com/) and a working microphone.

### Related courses:

### Video

This is what you'll create:

### Recognize spoken voice

Speech recognition can by done using the Python SpeechRecognition module. We
make use of the [Google Speech API](https://pythonspot.com/speech-recognition-
using-google-speech-api/) because of it's great quality.

### Answer in spoken voice (Text To Speech)

Various [**APIs and programs are available for text to speech
applications**](https://pythonspot.com/speech-engines-with-python-tutorial/).
Espeak and pyttsx work out of the box but sound very robotic. We decided to go
with the Google Text To Speech API, gTTS.

Using it is as simple as:

    
    
    from gtts import gTTS
    import os
    tts = gTTS(text='Hello World', lang='en')
    tts.save("hello.mp3")
    os.system("mpg321 hello.mp3")  
  
---  
  
[![gtts](https://pythonspot-9329.kxcdn.com/wp-
content/uploads/2015/07/gtts.png%20658w,%20https://pythonspot-9329.kxcdn.com
/wp-content/uploads/2015/07/gtts-
300x74.png%20300w)](https://pythonspot-9329.kxcdn.com/wp-
content/uploads/2015/07/gtts.png)

### Complete program

The program below will answer spoken questions.

    
    
    #!/usr/bin/env python3
    # Requires PyAudio and PySpeech.
     
    import speech_recognition as sr
    from time import ctime
    import time
    import os
    from gtts import gTTS
     
    def speak(audioString): print(audioString) tts = gTTS(text=audioString, lang='en') tts.save("audio.mp3") os.system("mpg321 audio.mp3")
     
    def recordAudio(): # Record Audio r = sr.Recognizer() with sr.Microphone() as source: print("Say something!") audio = r.listen(source)
      # Speech recognition using Google Speech Recognition data = "" try: # Uses the default API key # To use another API key: `r.recognize_google(audio, key="GOOGLE_SPEECH_RECOGNITION_API_KEY")` data = r.recognize_google(audio) print("You said: " + data) except sr.UnknownValueError: print("Google Speech Recognition could not understand audio") except sr.RequestError as e: print("Could not request results from Google Speech Recognition service; {0}".format(e))
      return data
     
    def jarvis(data): if "how are you" in data: speak("I am fine")
      if "what time is it" in data: speak(ctime())
      if "where is" in data: data = data.split(" ") location = data[2] speak("Hold on Frank, I will show you where " + location + " is.") os.system("chromium-browser https://www.google.nl/maps/place/" + location + "/&amp;")
     
    # initialization
    time.sleep(2)
    speak("Hi Frank, what can I do for you?")
    while 1: data = recordAudio() jarvis(data)  
  
---  
  
### Related posts:
