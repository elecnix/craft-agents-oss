# Speech Recognition Recommendations for Craft Agents

## Technical stack summary

- Runtime and tooling: Bun, Vite, esbuild, TypeScript.
- Desktop shell: Electron main/renderer split with IPC via `window.electronAPI`.
- UI: React 18, Jotai state, Tailwind CSS, shadcn/ui components.
- Message flow: UI input composes a message, applies badges/attachments, then calls `window.electronAPI.sendMessage` to stream into the session system.

Relevant code paths:

- Chat input surface and composition logic: @/Users/nicolas/Source/craft-agents-oss/apps/electron/src/renderer/components/app-shell/input/FreeFormInput.tsx#148-879.
- Message dispatch and optimistic UI update: @/Users/nicolas/Source/craft-agents-oss/apps/electron/src/renderer/App.tsx#674-832.

## Architecture mindset and principles inferred

- **Renderer-first interaction model:** Input state and UX behaviors live in the renderer, with the main process acting as a capability gateway for file storage, IPC, and session processing.
- **Explicit user intent gates:** Most actions are triggered by user events and UI affordances (menu commands, buttons, keyboard shortcuts), which aligns with microphone permission requirements.
- **Session-centric state:** Messages are scoped by session IDs, with optimistic UI updates and background persistence.
- **Pluggable sources and add-ons:** The app already integrates multiple external sources via a common abstraction, suggesting a preference for modular integrations.
- **Progressive enhancement:** UI supports fallbacks and mode switching; this is compatible with a feature that gracefully downgrades when speech is unavailable.

## Integration surfaces for speech recognition

- **Primary UI integration:** Add a microphone action in `FreeFormInput` (near attachments and send/stop controls) and stream transcript into the `input` state. This aligns with the current input orchestration and debounced draft persistence.
- **Session submission path:** Use the existing `onSubmit` path so transcripts participate in badges, attachments, permissions, and session options without special casing.
- **IPC boundary:** If a provider requires API keys or proxying, locate credentials in the main process and expose a minimal IPC method for short-lived tokens or signing. The renderer should not hold long-lived keys.
- **UX considerations:** Provide “hold to talk” and “toggle to dictate” modes, show interim transcript, and allow quick cancel/clear before sending.

## Recommendations (ordered by fit)

### 1) Web Speech API with a React wrapper

#### Why it fits (Web Speech API)

- Electron’s renderer is Chromium-based, so the Web Speech API is the lowest friction option with minimal dependencies.
- The UI can run in the renderer without IPC or external services.
- A React wrapper (such as `react-speech-recognition`) handles transcript lifecycle and provides quick wiring to the existing input state.

#### Setup and use (Web Speech API)

- Web Speech API provides the `SpeechRecognition` interface and supports interim results and continuous mode. It also includes `processLocally` for on-device preference when supported by the browser.
- `react-speech-recognition` uses a hook (`useSpeechRecognition`) and a global `SpeechRecognition` controller to start/stop listening.

#### Integration notes (Web Speech API)

- Insert transcript into `FreeFormInput` state via `setInput`, respecting existing debounced draft updates.
- Use interim results to render gray preview text appended to the existing input until a final result arrives.
- Ensure microphone activation is always tied to a user action (button press or keyboard shortcut) to satisfy browser permission policies.

#### References (Web Speech API)

- Web Speech API usage and configuration: [MDN Using the Web Speech API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Speech_API/Using_the_Web_Speech_API)
- `SpeechRecognition` API surface: [MDN SpeechRecognition](https://developer.mozilla.org/en-US/docs/Web/API/SpeechRecognition)
- `processLocally` behavior: [MDN SpeechRecognition processLocally](https://developer.mozilla.org/en-US/docs/Web/API/SpeechRecognition/processLocally)
- React wrapper overview: [react-speech-recognition README](https://github.com/JamesBrill/react-speech-recognition#readme)

#### Risks (Web Speech API)

- Browser support is limited and differs across platforms. The renderer should detect support and present a fallback option.

### 2) Deepgram JS SDK (streaming WebSocket)

#### Why it fits (Deepgram)

- Provides streaming transcription with direct browser WebSocket support, which maps to the live input experience.
- Avoids local compute and improves accuracy on diverse audio.

#### Setup and use (Deepgram)

- The SDK supports direct WebSocket connections from the browser for live transcription.
- For REST APIs, a proxy is required due to CORS. Live transcription does not require a proxy.

#### Integration notes (Deepgram)

- Use renderer microphone capture (`getUserMedia`) and stream audio chunks to the WebSocket.
- Generate short-lived tokens in the main process to avoid exposing API keys in the renderer.
- Offer a toggle for “cloud transcription” vs “local transcription,” with clear privacy messaging.

#### References (Deepgram)

- SDK install and usage overview: [Deepgram JS SDK README](https://github.com/deepgram/deepgram-js-sdk#readme)
- Browser WebSocket capabilities: [Deepgram JS SDK browser usage](https://github.com/deepgram/deepgram-js-sdk#readme)
- Live streaming guide: [Deepgram live streaming audio](https://developers.deepgram.com/docs/live-streaming-audio)

#### Risks (Deepgram)

- Requires network connectivity and an API key. Token handling must be stored in the main process.

### 3) Azure Speech SDK via Web Speech polyfill

#### Why it fits (Azure)

- The `web-speech-cognitive-services` ponyfill plugs into Web Speech API semantics, keeping the UI logic consistent.
- Works with the existing Web Speech API style integration and `react-speech-recognition` if desired.

#### Setup and use (Azure)

- Install `microsoft-cognitiveservices-speech-sdk` and a ponyfill library, then provide subscription and region credentials.

#### Integration notes (Azure)

- Similar integration to Web Speech API, but credentials should be fetched from the main process.
- This option is useful if you want a consistent experience across platforms while keeping the Web Speech API style in the renderer.

#### References (Azure)

- Azure Speech SDK install and example: [Microsoft Cognitive Services Speech SDK for JavaScript](https://learn.microsoft.com/en-us/javascript/api/overview/azure/microsoft-cognitiveservices-speech-sdk-readme?view=azure-node-latest)

#### Risks (Azure)

- Requires account management and billing. Expect higher latency than local recognition.

### 4) Whisper.cpp (WASM in renderer)

#### Why it fits (Whisper.cpp)

- Fully local transcription with no network dependency.
- Strong accuracy for many languages.

#### Setup and use (Whisper.cpp)

- The WebAssembly demo loads a Whisper model into the browser and runs inference locally. Models are large, and the demo’s performance is limited to small models.

#### Integration notes (Whisper.cpp)

- Bundle the WASM artifacts and models into the Electron app or load from a local resource directory.
- Manage model selection and storage in the main process; inform users of storage costs.
- Use chunked audio to update interim transcript, then finalize once Whisper completes per segment.

#### References (Whisper.cpp)

- Whisper WASM usage and constraints: [whisper.cpp WASM README](https://raw.githubusercontent.com/ggml-org/whisper.cpp/master/examples/whisper.wasm/README.md)

#### Risks (Whisper.cpp)

- Large model sizes and high CPU usage. Not ideal for continuous dictation without careful tuning.

### 5) Vosk (WASM browser ports like `vosk-browser` or Vosklet)

#### Why it fits (Vosk)

- Offline transcription with smaller model sizes than Whisper.
- Streaming results with partial transcripts.

#### Setup and use (Vosk)

- `vosk-browser` provides a Web Worker-based wrapper for Vosk and exposes partial and final results.
- Vosklet is a smaller wrapper that bundles Vosk in a lightweight package with model caching.

#### Integration notes (Vosk)

- Add a small on-device model pack to app resources; allow users to download additional language models.
- Run recognition in a worker to avoid blocking the UI thread.

#### References (Vosk)

- `vosk-browser` usage and worker model: [vosk-browser README](https://raw.githubusercontent.com/schlawg/vosk-browser/master/README.md)
- Vosklet overview and usage: [Vosklet README](https://raw.githubusercontent.com/msqr1/Vosklet/main/README.md)
- Vosk capabilities and language coverage: [Vosk documentation](https://alphacephei.com/vosk/)

#### Risks (Vosk)

- WASM builds and model management add packaging complexity. Accuracy is below Whisper for some languages.

## UX integration notes

- **Controls:** Add a mic button near the send button. Provide hold-to-talk and toggle modes; show live waveform or mic active indicator.
- **Transcript handling:** Display interim text in a lighter style and merge into the input once final results arrive.
- **Editing:** Treat dictated text as editable and keep consistent cursor behaviors with `RichTextInput`.
- **Privacy messaging:** Show a short notice when a cloud provider is selected, with a link to settings.
- **Fallback:** When speech recognition is not supported, hide the mic control and show a tooltip or settings note.

## Suggested implementation order

1. Implement a Web Speech API-based prototype in the renderer to validate UX.
2. Add provider selection (local vs cloud), with credentials handled in the main process.
3. Introduce a cloud provider if accuracy or language breadth needs exceed the Web Speech API.
4. Consider offline models (Whisper or Vosk) as an advanced option for privacy-focused users.
