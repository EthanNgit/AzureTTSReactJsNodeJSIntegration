# AzureTTS integration with ReactJs NodeJS
Integrating Azure TTS into a React and Node based webapp can be difficult, but here was my solution to this problem. This assumes you already have a working webapp and routes set up. If you do not follow an online tutorial to get started.

## Step 1: Setup Project
Install express and configure your roots, if you need the npm its here.
```
npm install express
```
Then after its configured, again if you need it look it up, in your file that routes the apis (typically index) do the following
```
// Create router
const ttsRouter = require('./routes/tts-call');
// Use router
app.use("/api/tts/", ttsRouter);
```
## Step 2: Setup Microsoft Azure account

Microsoft Azure TTS is free up to 500k characters per month, and then it can incur charges (if you set your account up to do so). Microsoft Azure has plenty of realistic voices, making it ideal for small productions.

### 1. [Sign up](https://azure.microsoft.com/en-us) for your Azure account (you will need to input a card).

### 2. Create an instance

- Head to your portal and click the "More Services" arrow:
  ![image](https://github.com/EthanNgit/AzureTTSReactJsNodeJSIntegration/assets/105979510/2b2951d1-f305-41b4-8927-65b38f82b5ec)

- Click "Speech Services":
  ![image](https://github.com/EthanNgit/AzureTTSReactJsNodeJSIntegration/assets/105979510/b65ce27d-9d21-4acb-9ca0-cd02456c4de5)

- Click "Create":
  ![image](https://github.com/EthanNgit/AzureTTSReactJsNodeJSIntegration/assets/105979510/c7a1c070-2255-4f20-81a5-536ee838b10b)

- Choose the available subscription, create a new resource group (something like "project-one-resource"), then choose a name and region for this instance. If you want to use the free trial (500k per month max), select the free tier, "Free, F(0)".

### 3. Get information

- Get the key and region ready for use later; you will need it.
  ![image](https://github.com/EthanNgit/AzureTTSReactJsNodeJSIntegration/assets/105979510/8f840cf3-1afd-42e5-956e-f10a52f8bfac)

### Side Note: Voices

- If you want to figure out the voice you want now, you can also click the "Voice Studio" button.
  ![image](https://github.com/EthanNgit/AzureTTSReactJsNodeJSIntegration/assets/105979510/3899dfec-e1ee-4a98-ac17-2ad1f0e603ef)

- Scroll down to "Voice Gallery" and click it.
  ![image](https://github.com/EthanNgit/AzureTTSReactJsNodeJSIntegration/assets/105979510/15f54f92-bda2-4f65-bfb6-49e00145239a)

- Here you can go through all languages and voices available and also note the emotion and styles the voices can show. Keep in mind that trying custom lines will use out of your free 500k.
  ![image](https://github.com/EthanNgit/AzureTTSReactJsNodeJSIntegration/assets/105979510/e31522ea-2252-4e2c-b80a-ef7704c5475d)
  
- If you want to use the voice find it [here](https://learn.microsoft.com/en-us/azure/ai-services/speech-service/language-support?tabs=tts) and save the tag for it
<br>

## Step 3: Create Route
Next you can create a file in your backend routes file. I used tts-call.js. Since we are using express you can do the following
```
const express = require("express");
const { PassThrough } = require("nodemailer/lib/xoauth2");

const router = express.Router();

router.post("/", async (req, res) => {
    const { text } = req.body;

    // Enter your key (either key1 or key2) from the azure instance
    const subscriptionKey = "YOUR_SUBSCRIPTION_KEY";
    // Enter the region that you can find from your azure instance (eg eastus)
    const serviceRegion = 'YOUR_REGION';

    try {
        const sdk = await import("microsoft-cognitiveservices-speech-sdk");
        
        const speechConfig = sdk.SpeechConfig.fromSubscription(
            subscriptionKey,
            serviceRegion
        );

        speechConfig.speechSynthesisOutputFormat =
            sdk.SpeechSynthesisOutputFormat.Audio16Khz32KBitRateMonoMp3;

        const synthesizer = new sdk.SpeechSynthesizer(speechConfig, null);

        // Here you can add your voice (eg zh-CN-XiaoxiaoNeural)
        // You will also have to find your language (found in the language options documentations above eg zh-CN)
        // You can also add other modifiers if the voice allows you too
        // There is a list in the documentation marked below for all the modifiers but you add it like so 
        // <voice name="YOUR_TTS_VOICE" modifier="YOUR_MODIFIER_SETTING" modifier2="YOUR_MODIFIER_SETTING">
        synthesizer.speakSsmlAsync(
            `
            <speak version="1.0" xmlns="http://www.w3.org/2001/10/synthesis"
            xmlns:mstts="https://www.w3.org/2001/mstts" xml:lang="YOUR_LANGUAGE">
            <voice name="YOUR_TTS_VOICE">
                    ${text}
            </voice>
            </speak>
            `,
            result => {
                const { audioData} = result;

                synthesizer.close();

                const bufferStream = Buffer.from(audioData);
                res.set('Content-Type', 'audio/mpeg');
                res.send(bufferStream);
            }
        );
    } catch (error) {
        console.error('Error:', error);
        res.status(500).send('Internal Server Error');
    }
});

module.exports = router;

```

> This is the documentation for voice settings [here](https://learn.microsoft.com/en-us/azure/ai-services/speech-service/speech-synthesis-markup-voice)

## Step 4: API usage from the frontend.
Now that it is established in the backend to call to the azure tts api, you can call it from your frontend. Below is simple version (make sure you have axios installed frontend to call apis)
```
    const response = await axios.post('http://localhost:3000/api/tts', { text }, { responseType: 'arraybuffer' });
    
    const audioArrayBuffer = response.data;
    const audioBlob = new Blob([audioArrayBuffer], { type: 'audio/mpeg' });
    const audioUrl = URL.createObjectURL(audioBlob);
    const audioElement = new Audio(audioUrl);

    await audioElement.play();
```
What this does is gets the tts buffer array given your input text, and then creates a blob out of it using mpeg (mp3), then creates a url to be called, then uses a audio element to play it, and then plays it.
Here is a more advanced usage, using a button to play the audio as a React component, you need the font awesome package as well.
```
import { faVolumeHigh } from '@fortawesome/free-solid-svg-icons/faVolumeHigh'
import { FontAwesomeIcon } from '@fortawesome/react-fontawesome'
import axios from 'axios';
import React, { forwardRef, useEffect, useImperativeHandle, useRef, useState } from 'react'

interface TextToSpeechProps {
    text: string;
}

const TTSButton: React.ForwardRefRenderFunction<any, TextToSpeechProps> = ({ text }, ref) => {
  const [audioBuffer, setAudioBuffer] = useState<ArrayBuffer | null>(null);
  const [isPlaying, setIsPlaying] = useState(false);
  const iconRef = useRef<null | SVGSVGElement>(null);
  
    const playAudio = async () => {
      if (!isPlaying) {
        try {
          setIsPlaying(true);
  
          if (audioBuffer) {
            const audioArrayBuffer = audioBuffer;
            const audioBlob = new Blob([audioArrayBuffer], { type: 'audio/mpeg' });
            const audioUrl = URL.createObjectURL(audioBlob);
            const audioElement = new Audio(audioUrl);

            audioElement.addEventListener('ended', () => {
              setIsPlaying(false);
            });

            await audioElement.play();

            console.log("Played saved audio");
          } else {
            const response = await axios.post('http://localhost:3001/api/tts', { text }, { responseType: 'arraybuffer' });
            setAudioBuffer(response.data);
            
            const audioArrayBuffer = response.data;
            const audioBlob = new Blob([audioArrayBuffer], { type: 'audio/mpeg' });
            const audioUrl = URL.createObjectURL(audioBlob);
            const audioElement = new Audio(audioUrl);

            audioElement.addEventListener('ended', () => {
              setIsPlaying(false);
            });

            await audioElement.play();

            console.log("Played new audio");
          }
        } catch (error) {
          console.error('Error playing audio:', error);
        } 
      }
    };

    useEffect(() => {
      return () => {
        setAudioBuffer(null);
        setIsPlaying(false);
      };
    }, [text]);

    useImperativeHandle(ref, () => ({
      forceClick: playAudio,
    }));
  
    return (
      <FontAwesomeIcon icon={faVolumeHigh} className='cn-listen-icon' onClick={playAudio} ref={iconRef} />
    );
  };
  
  export default forwardRef(TTSButton);
```
You can use it like
```
<TTSButton text="TEXT_YOU_WANT_SPOKEN" />
```

## Step 5: Finish
That is all it took, its really simple, and rewarding compared to other tts options avaliable given the generous free trial.

https://github.com/EthanNgit/AzureTTSReactJsNodeJSIntegration/assets/105979510/ad7fd088-d78c-41ec-ab5f-a6c2dd160641


