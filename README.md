# Real-time Parallel Audio Stream Separation (Browser)

## Problem Statement
Simultaneously capture system audio (via screen capture) and microphone audio in a Chrome browser. The challenge is that the microphone often picks up system audio (echo/bleed), making it difficult to separate the user's voice from system sounds. This project provides a real-time solution to cleanly separate these streams using advanced signal processing.

---

## Root Cause of Echo/Bleed-In
- **Microphone audio** includes both the user's voice and any sound played by the system (if using speakers).
- **System audio** is captured directly from the device output.
- **Echo/bleed** occurs because the microphone physically picks up system sounds from speakers, causing overlap between the two streams.

---

## Solution Architecture

### 1. Stream Capture
- **Microphone:** `navigator.mediaDevices.getUserMedia({ audio: true })`
- **System Audio:** `navigator.mediaDevices.getDisplayMedia({ audio: true, video: true })`
- Streams are checked for audio tracks before processing to ensure reliability.

### 2. Audio Processing Pipeline
- **Web Audio API** is used for real-time audio processing.
- **Processing Modes:**
  - **Adaptive Filtering (NLMS):** Predicts and subtracts system audio from the microphone signal.
  - **Spectral Subtraction:** Removes system audio components in the frequency domain.
  - **Cross-Correlation:** Finds optimal delay between system audio and microphone echo, then cancels echo.
  - **Hybrid:** Combines adaptive filtering and spectral subtraction for robust separation.
- **Noise Gate:** Suppresses low-level background noise after separation.

### 3. Visualization & Metrics
- Real-time waveform visualizations for raw and processed signals.
- Metrics: Echo reduction (%), SNR improvement (dB), processing latency (ms), CPU usage (%).

---

## System Architecture Diagram

```
+-------------------+      +-------------------+      +-------------------+
| Microphone Stream | ---> |  Audio Processing | ---> |  Output/Playback  |
+-------------------+      |   (NLMS, Spectral |      +-------------------+
| System Audio      | ---> |   Subtraction,    |
+-------------------+      |   Cross-Corr,     |
                           |   Hybrid, Noise   |
                           |   Gate)           |
                           +-------------------+
```

---

## Algorithmic Approaches

### 1. Adaptive Filtering (NLMS)
- Uses the system audio as a reference to subtract its presence from the microphone stream.
- Continuously adapts filter weights to minimize echo.

### 2. Spectral Subtraction
- Estimates system audio spectrum and subtracts it from the microphone spectrum.
- Useful for stationary noise and music.

### 3. Cross-Correlation Echo Cancellation
- Finds the best delay between system and mic signals, then subtracts the delayed system audio from the mic.

### 4. Hybrid Mode
- Combines adaptive filtering and spectral subtraction for improved separation.

### 5. Noise Gate
- Suppresses low-level noise after separation for cleaner output.

---

## Code Snippets

### Adaptive Filter (NLMS)
```js
applyAdaptiveFilter(micInput, systemInput, output) {
    const mu = this.params.adaptiveGain;
    const filterLength = 128;
    if (!this.adaptiveWeights) {
        this.adaptiveWeights = new Float32Array(filterLength);
        this.systemHistory = new Float32Array(filterLength);
    }
    for (let i = 0; i < micInput.length; i++) {
        // Shift system audio history
        for (let j = filterLength - 1; j > 0; j--) {
            this.systemHistory[j] = this.systemHistory[j - 1];
        }
        this.systemHistory[0] = systemInput[i];
        // Calculate predicted echo
        let echo = 0;
        for (let j = 0; j < filterLength; j++) {
            echo += this.adaptiveWeights[j] * this.systemHistory[j];
        }
        // Calculate error (desired signal)
        const error = micInput[i] - echo;
        output[i] = error;
        // Update filter weights
        const norm = this.calculateNorm(this.systemHistory) + 1e-10;
        for (let j = 0; j < filterLength; j++) {
            this.adaptiveWeights[j] += (mu * error * this.systemHistory[j]) / norm;
        }
    }
}
```

### Spectral Subtraction
```js
applySpectralSubtraction(micInput, systemInput, output) {
    const alpha = this.params.spectralSubtraction;
    for (let i = 0; i < micInput.length; i++) {
        const estimatedNoise = systemInput[i] * alpha;
        const subtracted = micInput[i] - estimatedNoise;
        const spectralFloor = 0.1 * Math.abs(micInput[i]);
        output[i] = Math.sign(subtracted) * Math.max(Math.abs(subtracted), spectralFloor);
    }
}
```

### Cross-Correlation
```js
applyCrossCorrelation(micInput, systemInput, output) {
    // ...existing code for delay estimation and echo cancellation...
}
```

---

## Decisions & Trade-offs
- **Web Audio API** chosen for browser compatibility and real-time processing.
- **Multiple algorithms** provided for flexibility; users can switch modes based on scenario.
- **Visualization and metrics** help users understand separation quality and performance.
- **Trade-offs:**
  - Adaptive filtering works best for linear echo paths; may struggle with nonlinearities.
  - Spectral subtraction is effective for stationary noise but less so for dynamic signals.
  - Cross-correlation is robust for time-aligned echoes but can be computationally intensive.
  - Hybrid mode balances strengths but increases CPU usage.

---

## Limitations
- **Browser/OS restrictions:** System audio capture may not be available on all platforms or browser versions.
- **Latency:** Real-time processing is subject to browser and hardware constraints.
- **Nonlinear echo paths:** Some acoustic environments may reduce algorithm effectiveness.
- **User experience:** Requires user to select correct tab/window with active audio for system stream.

---

## Usage
1. Open `parallel-audio-separation-demo.html` in Chrome.
2. Click "Start Microphone" and "Start System Audio".
3. Select a tab/window with active audio for system capture.
4. Adjust processing parameters and modes as needed.
5. View real-time visualizations and metrics.

---

## References
- [Web Audio API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Audio_API)
- [NLMS Adaptive Filter](https://en.wikipedia.org/wiki/Least_mean_squares_filter)
- [Spectral Subtraction](https://en.wikipedia.org/wiki/Spectral_subtraction)
- [Cross-Correlation](https://en.wikipedia.org/wiki/Cross-correlation)
- [Chrome getDisplayMedia](https://developer.mozilla.org/en-US/docs/Web/API/MediaDevices/getDisplayMedia)

---

## License
MIT
