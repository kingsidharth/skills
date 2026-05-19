# Microphone Capture with Voice Processing

AVAudioEngine's voice processing mode is Apple's recommended path for mic capture with echo cancellation (AEC) and noise suppression. Available on macOS 10.15+, with significant improvements in macOS 14+.

## Voice Processing Mode

Enabling voice processing on AVAudioEngine applies:
- **Acoustic Echo Cancellation (AEC)** — Removes speaker output from mic input. Essential when capturing mic alongside system audio playback.
- **Noise Suppression** — Reduces background noise (fans, keyboard, ambient)
- **Automatic Gain Control (AGC)** — Normalizes input levels. Slight gain reduction is expected and documented by Apple.

Voice processing requires both input and output nodes to be in voice processing mode. Enabling on either node automatically enables on the other.

## Setup

```swift
let engine = AVAudioEngine()

// Must be stopped when enabling voice processing
try engine.inputNode.setVoiceProcessingEnabled(true)
// This implicitly enables it on outputNode too

// Install tap on input node
let inputFormat = engine.inputNode.outputFormat(forBus: 0)
engine.inputNode.installTap(onBus: 0, bufferSize: 4096, format: inputFormat) { buffer, time in
    // buffer contains echo-cancelled, noise-suppressed PCM audio
}

try engine.start()
```

## Critical Constraints

- Voice processing **cannot be enabled dynamically** — engine must be stopped first
- Only available when rendering to a real audio device, not in manual rendering mode
- A slight gain change is expected when voice processing is active — this is by design
- On macOS 13 and earlier, voice processing behavior is less reliable; macOS 14+ is strongly recommended

## Audio Ducking (macOS 14+)

Controls how other audio (media playback, notifications) is attenuated when voice activity is detected:

```swift
var duckingConfig = AVAudioVoiceProcessingOtherAudioDuckingConfiguration(
    enableAdvancedDucking: true,  // Dynamic ducking based on voice activity
    duckingLevel: .mid            // .min, .mid, .max, .default
)
engine.inputNode.voiceProcessingOtherAudioDuckingConfiguration = duckingConfig
```

With `enableAdvancedDucking`: media volume drops when either party talks, rises during silence. Mirrors FaceTime SharePlay behavior.

## Muted Talker Detection (macOS 14+)

Detect when user is speaking while muted — useful for "you're muted" prompts:

```swift
engine.inputNode.setMutedSpeechActivityEventListener { event in
    switch event {
    case .started:
        // User started talking while muted
    case .ended:
        // User stopped talking
    @unknown default: break
    }
}

// Must mute via the API for detection to work
engine.inputNode.isVoiceProcessingInputMuted = true
```

## Alternative: AUVoiceProcessingIO (Lower Level)

For more control, use the `kAudioUnitSubType_VoiceProcessingIO` Audio Unit directly. Same AEC/noise suppression capabilities but requires manual CoreAudio setup. Properties:
- `kAUVoiceIOProperty_VoiceProcessingEnableAGC` — Toggle AGC
- `kAUVoiceIOProperty_MuteOutput` — Mute without stopping
- `kAUVoiceIOProperty_DuckNonVoiceAudio` — Duck other audio

Use this route when AVAudioEngine's abstractions are insufficient, or when integrating with existing CoreAudio pipelines.

## Combining Mic + System Audio

The recommended architecture for a meeting recorder:

1. **System audio**: Core Audio Taps (or ScreenCaptureKit) → separate PCM stream
2. **Mic audio**: AVAudioEngine with voice processing → separate PCM stream with AEC
3. **Merge or keep separate**: Either interleave into a single file with two tracks, or keep as separate files for independent processing

Do NOT route system audio through the same AVAudioEngine instance as mic input — voice processing will treat system audio as "echo" and cancel it from the mic stream. Keep them as independent capture pipelines.

## Format Considerations

AVAudioEngine's input node format is dictated by the hardware. Common: 48kHz, Float32, 1ch (mono when voice processing enabled). For ASR:

```swift
let desiredFormat = AVAudioFormat(commonFormat: .pcmFormatInt16, sampleRate: 16000, channels: 1, interleaved: true)!
let converter = AVAudioConverter(from: inputFormat, to: desiredFormat)!
```

Run conversion in the tap callback or on a dedicated processing queue.
