# Scribe iOS Microphone Fix Patch

## Problem

On iOS, when you try to start recording in Obsidian with the Scribe plugin, you get:
> "Failed to access microphone"

Even though:
- System Settings → Privacy → Microphone → Obsidian is **enabled**
- The Obsidian app has microphone permission

This is because iOS blocks `getUserMedia` from webview contexts inside third-party apps, even when system permissions are granted. The browser-level API simply returns an error with no helpful diagnostic.

## Solution

Patch `startRecording()` in `src/audioRecord/audioRecord.ts` to:

1. **Detect iOS** via `navigator.userAgent`
2. **Parse the actual error type** (`NotAllowedError`, `NotFoundError`, `NotReadableError`, etc.)
3. **Show targeted, actionable notices** instead of the generic "Failed to access the microphone"
4. **Recommend the workaround** for iOS: use Voice Memos → then "Transcribe & Summarize Current File" in Scribe

## Changes

### File: `src/audioRecord/audioRecord.ts`

**Before:**
```typescript
} catch (err) {
  new Notice('Scribe: Failed to access the microphone');
  console.error('Error accessing microphone:', err);
  this.mediaRecorder = null;
  this.startTime = null;
  throw err;
}
```

**After:**
```typescript
} catch (err: unknown) {
  const error = err as Error & { name?: string };
  const errorName = error?.name ?? '';
  const isIOS = /iPhone|iPad|iPod/.test(navigator.userAgent);

  if (isIOS) {
    if (errorName === 'NotAllowedError' || errorName === 'PermissionDeniedError') {
      new Notice('Scribe: ⚠️ Microphone access denied. Please go to Settings > Privacy > Microphone > Obsidian and enable it.');
    } else if (errorName === 'NotFoundError' || errorName === 'DevicesNotFoundError') {
      new Notice('Scribe: ⚠️ No microphone found. On iPhone, use the Voice Memos app to record, then transcribe via "Transcribe & Summarize Current File" in Scribe.');
    } else if (errorName === 'NotReadableError' || errorName === 'TrackStartError') {
      new Notice('Scribe: ⚠️ Microphone is already in use. Close other apps using the mic (camera, FaceTime, etc.) and try again.');
    } else {
      new Notice('Scribe: ⚠️ iOS recording error. For best results, record using the Voice Memos app, then use "Transcribe & Summarize Current File" on the audio file.');
    }
  } else {
    if (errorName === 'NotAllowedError' || errorName === 'PermissionDeniedError') {
      new Notice('Scribe: ⚠️ Microphone access denied. Please allow mic access in your system settings and browser permissions.');
    } else if (errorName === 'NotFoundError' || errorName === 'DevicesNotFoundError') {
      new Notice('Scribe: ⚠️ No microphone detected. Please connect a microphone and try again.');
    } else if (errorName === 'NotReadableError' || errorName === 'TrackStartError') {
      new Notice('Scribe: ⚠️ Microphone is already in use by another application. Please close other apps using the microphone.');
    } else {
      new Notice(`Scribe: ⚠️ Failed to access microphone: ${error?.message ?? 'Unknown error'}`);
    }
  }

  console.error('Error accessing microphone:', err);
  this.mediaRecorder = null;
  this.startTime = null;
  throw err;
}
```

## Full Patched File

A complete patched version of `audioRecord.ts` is in this directory:  
`audioRecord.ts` — just replace the original with this file.

## How to Apply

1. **Option A — Install via BRAT (recommended):**
   - Install the BRAT plugin in Obsidian
   - Add `Mikodin/obsidian-scribe` as a beta plugin
   - Replace `src/audioRecord/audioRecord.ts` with the patched version

2. **Option B — Fork the repo:**
   - Fork https://github.com/Mikodin/obsidian-scribe
   - Apply the changes to `src/audioRecord/audioRecord.ts`
   - Submit a PR to the original repo

3. **Option C — For this dev (Mikodin):**
   - Just replace the catch block in `startRecording()` with the improved version above
   - Ship as a patch update

## iOS Workaround (Until Fix is Released)

Until the patch is merged, iOS users can still use Scribe:

1. **Record audio** using the built-in **Voice Memos** app
2. **Export** the recording to Obsidian (via Files app → share to Obsidian, or save to iCloud Drive)
3. **Open the audio file** in Obsidian
4. **Run command** → "Scribe: Transcribe & Summarize Current File"

This bypasses the iOS webview mic limitation entirely.

## Testing

After applying the patch, test each error condition:

| Scenario | Expected Notice |
|----------|----------------|
| iOS + mic permission denied | "Go to Settings > Privacy > Microphone > Obsidian" |
| iOS + no mic found | "Use Voice Memos app, then Transcribe & Summarize" |
| iOS + mic in use by another app | "Close camera, FaceTime, etc." |
| iOS + unknown error | "For best results, record using Voice Memos..." |
| Desktop + permission denied | "Allow mic access in system settings and browser" |
| Desktop + no mic detected | "Please connect a microphone" |
| Desktop + mic in use | "Close other apps using the microphone" |