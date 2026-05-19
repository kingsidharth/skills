# Audio Processing: Silence Detection, Normalization, Compression

Post-capture audio processing for cleaning up recorded audio without altering system output levels.

## Silence Detection

Monitor audio buffers for silence to pause recording, trigger events, or segment output.

### RMS-Based Detection

Calculate Root Mean Square of each buffer and compare against a threshold:

```swift
func rmsLevel(buffer: AVAudioPCMBuffer) -> Float {
    guard let channelData = buffer.floatChannelData else { return 0 }
    let frames = Int(buffer.frameLength)
    let samples = channelData[0]
    
    var sum: Float = 0
    vDSP_measqv(samples, 1, &sum, vDSP_Length(frames))
    return sqrt(sum) // RMS in linear scale
}
```

Typical thresholds (linear scale):
- `< 0.001` — Near-silence (digital silence, very quiet room)
- `< 0.01` — Quiet (ambient noise floor in a typical room)
- `< 0.05` — Low speech / distant speaker

Use a **debounce window** (2–5 seconds of consecutive silence) before triggering "silence detected" to avoid false positives from natural speech pauses.

### vDSP for Efficient Computation

Apple's Accelerate framework (`vDSP`) handles vectorized audio math. Prefer it over manual loops:

```swift
import Accelerate

// Peak level
var peak: Float = 0
vDSP_maxmgv(samples, 1, &peak, vDSP_Length(frames))

// RMS
var meanSquare: Float = 0
vDSP_measqv(samples, 1, &meanSquare, vDSP_Length(frames))
let rms = sqrt(meanSquare)

// Convert to dB
let dB = 20 * log10(max(rms, 1e-10))
```

dB thresholds:
- `-60 dB` — Effective silence
- `-40 dB` — Very quiet
- `-20 dB` — Normal speech

## Level Normalization

Normalize audio levels without touching system volume. Apply in post-processing or real-time on the captured buffer.

### Peak Normalization

Scale all samples so the loudest peak hits a target level (e.g., -1 dBFS):

```swift
var peak: Float = 0
vDSP_maxmgv(samples, 1, &peak, vDSP_Length(frames))

let targetPeak: Float = 0.9 // -0.9 dBFS
if peak > 0 {
    var gain = targetPeak / peak
    vDSP_vsmul(samples, 1, &gain, samples, 1, vDSP_Length(frames))
}
```

Suitable for post-recording normalization. For real-time, use a smoothed gain factor to avoid clicks.

### Loudness Normalization (LUFS)

For broadcast-quality normalization, target -16 LUFS (podcast standard) or -14 LUFS (streaming). Requires measuring integrated loudness over a window. Apple's `AVAudioUnitEQ` or third-party libraries can compute LUFS. More complex than peak normalization but perceptually consistent.

## Dynamic Range Compression

Reduce the gap between loud and quiet parts. Essential for meeting recordings where speakers vary in distance/volume.

### Simple Compressor Logic

```swift
struct Compressor {
    var threshold: Float = 0.3    // Linear amplitude threshold
    var ratio: Float = 4.0        // 4:1 compression above threshold
    var attackTime: Float = 0.005 // 5ms attack
    var releaseTime: Float = 0.05 // 50ms release
    var envelope: Float = 0
    
    mutating func process(_ sample: Float, sampleRate: Float) -> Float {
        let absSample = abs(sample)
        let coeff = absSample > envelope
            ? exp(-1.0 / (attackTime * sampleRate))
            : exp(-1.0 / (releaseTime * sampleRate))
        envelope = coeff * envelope + (1 - coeff) * absSample
        
        if envelope > threshold {
            let excess = envelope - threshold
            let gain = threshold + excess / ratio
            return sample * (gain / envelope)
        }
        return sample
    }
}
```

For production use, prefer `AVAudioUnitEffect` with Apple's built-in `kAudioUnitSubType_DynamicsProcessor` AU, which handles lookahead, knee curves, and makeup gain.

### AVAudioUnitEffect Route

```swift
let dynamicsProcessor = AVAudioUnitEffect(audioComponentDescription: 
    AudioComponentDescription(
        componentType: kAudioUnitType_Effect,
        componentSubType: kAudioUnitSubType_DynamicsProcessor,
        componentManufacturer: kAudioUnitManufacturer_Apple,
        componentFlags: 0, componentFlagsMask: 0
    ))
engine.attach(dynamicsProcessor)
// Connect: inputNode → dynamicsProcessor → mainMixerNode
```

Configure via AudioUnit properties for threshold, ratio, attack, release, makeup gain.

## Noise Gate

Remove low-level noise between speech segments. Complementary to compression:

- Open threshold: `-40 dB` (gate opens when signal exceeds)
- Close threshold: `-50 dB` (gate closes when signal drops below)
- Hold time: `200ms` (gate stays open after signal drops below close threshold)

Implement as an amplitude envelope follower that mutes output when below threshold.

## Processing Pipeline Order

Recommended chain for meeting/speech recording:

1. **Noise gate** — Remove inter-speech noise floor
2. **Compressor** — Even out speaker volume differences
3. **Peak normalization** — Bring overall level to target
4. **High-pass filter** — Roll off below 80Hz (removes rumble, HVAC)

Apply via AVAudioEngine node chain, or offline via vDSP on recorded buffers.

## High-Pass Filter (De-Rumble)

```swift
let eq = AVAudioUnitEQ(numberOfBands: 1)
eq.bands[0].filterType = .highPass
eq.bands[0].frequency = 80 // Hz
eq.bands[0].bypass = false
engine.attach(eq)
```
