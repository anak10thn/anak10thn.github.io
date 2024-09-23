# Audio Processing: Converting Stereo to Mono, PCM 16-bit, and Resampling to 16kHz

In the world of audio processing, converting audio files to a specific format is a common task. Whether you're preparing audio for machine learning models, optimizing for streaming, or ensuring compatibility with various devices, understanding how to manipulate audio files is crucial. In this blog post, we'll walk through the steps to convert stereo audio to mono, convert it to PCM 16-bit little-endian format, and resample it to 16kHz. We'll also provide a practical implementation using Node.js and the wav and sox-audio libraries.

## Step-by-Step Guide

### 1. Convert Stereo to Mono

If the input audio is in stereo, the first step is to convert it to mono. This can be achieved by averaging the left and right channels:

$$ x_{\text{mono}}(t) = \frac{\text{Left}(t) + \text{Right}(t)}{2} $$

### 2. Convert to PCM 16-bit Little-Endian

Next, we convert the mono audio to PCM 16-bit little-endian format. This involves scaling the audio samples to the 16-bit range and rounding them:

$$ \text{PCM}_{\text{mono}}(t) = \text{round}(x_{\text{mono}}(t) \times 32767) $$

### 3. Resample to 16kHz

Finally, we resample the audio to a 16kHz sampling rate. This step ensures that the audio is compatible with systems that require a specific sampling rate:

$$ y[n] = \text{PCM}_{\text{mono}}\left(\frac{n \cdot F_s}{F_d}\right) $$

### Formula

$$ y[n] = \text{round}\left(\left(\frac{\text{Left}\left(\frac{n \cdot F_s}{F_d}\right) + \text{Right}\left(\frac{n \cdot F_s}{F_d}\right)}{2}\right) \times 32767\right) $$

### Explanation

- **Stereo to Mono**: $\frac{\text{Left}(t) + \text{Right}(t)}{2}$
- **Resampling**: $x\left(\frac{n \cdot F_s}{F_d}\right)$
- **PCM Conversion**: $\text{round}(x(t) \times 32767)$

## Implementation with Node.js, wav, and sox-audio
Now, let's see how we can implement this process using Node.js and the wav and sox-audio libraries.

### Prerequisites

Make sure you have Node.js installed on your machine. Then, install the necessary packages:

```bash
npm install fs wav sox-audio
```

### Code Implementation

Here's a function to process the audio file:

```javascript
const fs = require('fs');
const wav = require('wav');
const SoxCommand = require('sox-audio');

function processAudio(inputPath, outputPath, callback) {
  // Read the input WAV file
  const reader = new wav.Reader();

  reader.on('format', (format) => {
    // Check if the audio is stereo
    if (format.channels === 2) {
      // Convert stereo to mono
      reader.on('data', (data) => {
        const monoData = Buffer.alloc(data.length / 2);
        for (let i = 0; i < data.length; i += 4) {
          const left = data.readInt16LE(i);
          const right = data.readInt16LE(i + 2);
          const mono = Math.round((left + right) / 2);
          monoData.writeInt16LE(mono, i / 2);
        }
        fs.writeFileSync('temp_mono.wav', monoData);
      });
    } else {
      fs.writeFileSync('temp_mono.wav', fs.readFileSync(inputPath));
    }

    // Resample to 16kHz using Sox
    const command = SoxCommand()
      .input('temp_mono.wav')
      .output(outputPath)
      .outputSampleRate(16000)
      .outputEncoding('signed-integer', 16)
      .on('end', () => {
        console.log('Audio processing finished.');
        callback(null, outputPath);
      })
      .on('error', (err) => {
        console.error('Error processing audio:', err);
        callback(err);
      });

    command.run();
  });

  fs.createReadStream(inputPath).pipe(reader);
}
```

### Usage

To use the `processAudio` function, simply call it with the input and output file paths:

```javascript
processAudio('input.mp3', 'output.wav', (err, outputPath) => {
  if (err) {
    console.error('Failed to process audio:', err);
  } else {
    console.log('Processed audio saved to:', outputPath);
  }
});
```

## Conclusion

In this blog post, we've covered the steps to convert stereo audio to mono, convert it to PCM 16-bit little-endian format, and resample it to 16kHz. We've also provided a practical implementation using Node.js and the wav and sox-audio libraries. This process is essential for various applications, including audio preprocessing for machine learning, optimizing audio for streaming, and ensuring compatibility with different devices. Happy coding!
