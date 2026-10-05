# Real-time Voice Interaction Chatbot Migration Guide

This guide details the architecture, components, and steps required to migrate the real-time voice interaction chatbot from the current project to a new Android application.

## 1. System Overview

The chatbot is a hybrid system designed for low-latency voice interaction. It primarily uses **OpenAI's Realtime API** (via WebSocket) for fluid, conversational voice-to-voice interaction. It also includes a robust fallback mechanism using a local knowledge base (`ChatbotDatabase`) and standard HTTP-based LLM completion (`ChatGPTService`) if the realtime connection is unavailable or for specific tasks.

### Key Features
- **Real-time Voice Streaming:** Bi-directional audio streaming using WebSockets (`wss://api.openai.com/v1/realtime`).
- **Interruptibility:** The user can interrupt the bot, and the system handles the audio stream accordingly.
- **Tool Calling:** The bot can trigger client-side actions (e.g., `navigate_to_model`) based on conversation context.
- **Local Fallback:** A regex-based local database provides instant responses for common queries even offline.
- **Reactive UI:** A `VideoView` displays a robot face (e.g., looping video) that reacts to speech states.

## 2. Core Components

### A. Frontend (Android)

1.  **`ChatbotDialogFragment.java`** (`app/src/main/java/com/keenon/peanut/sample/chatGBT/`)
    -   **Role:** The main controller and UI container (fullscreen DialogFragment).
    -   **Responsibilities:**
        -   Manages the lifecycle of the voice session.
        -   Handles UI state (Listening vs. Speaking).
        -   Integrates `OpenAIRealtimeClient` and `RealtimeAudioMgr`.
        -   Executes tool calls (e.g., navigation logic).
        -   Manages the "Robot Face" video playback.

2.  **`OpenAIRealtimeClient.java`** (`app/src/main/java/com/keenon/peanut/sample/util/`)
    -   **Role:** The WebSocket client for OpenAI.
    -   **Responsibilities:**
        -   Connects to `wss://api.openai.com/v1/realtime`.
        -   Sends session updates (instructions, tools, voice settings).
        -   Streams audio data (Base64 encoded PCM) to the server.
        -   Receives audio deltas and text transcripts from the server.
        -   Parses server events (e.g., `response.audio.delta`, `response.function_call_arguments.done`).

3.  **`RealtimeAudioMgr.java`** (`app/src/main/java/com/keenon/peanut/sample/util/`)
    -   **Role:** Low-level audio I/O manager.
    -   **Responsibilities:**
        -   **Recording:** Captures raw audio from the microphone (PCM 16-bit, 24kHz or 16kHz resampled).
        -   **Playback:** Plays received PCM audio data using `AudioTrack`.
        -   **Echo Suppression:** Simple logic to pause recording/input processing while the bot is speaking.

4.  **`ChatGPTConfig.java`** (`app/src/main/java/com/keenon/peanut/sample/util/`)
    -   **Role:** Configuration manager backed by `SharedPreferences`.
    -   **Responsibilities:** Stores API keys, model selection (`gpt-4o-realtime-preview`), and provider settings (OpenAI vs n8n).

5.  **`ChatbotDatabase.java`** (`app/src/main/java/com/keenon/peanut/sample/util/`)
    -   **Role:** Local knowledge base.
    -   **Responsibilities:** Provides hardcoded responses for specific keywords (English/French) to ensure basic functionality without API costs or internet.

### B. Backend / External
-   **OpenAI Realtime API:** The primary intelligence engine.
-   **`webhook_server.py`:** A simple Python Flask server used for testing or receiving webhook notifications (e.g., from an n8n workflow).

## 3. Data Flow

1.  **Initialization:** `ChatbotDialogFragment` opens -> checks permissions -> initializes `OpenAIRealtimeClient`.
2.  **Session Setup:** Client connects -> Sends `session.update` with instructions (from `knowledge_base.txt`) and tool definitions.
3.  **User Speaks:**
    -   `RealtimeAudioMgr` captures audio -> Base64 encodes it.
    -   `OpenAIRealtimeClient` sends `input_audio_buffer.append` events.
4.  **Processing (Server):** OpenAI VAD (Voice Activity Detection) detects speech end -> generates response.
5.  **Bot Responds:**
    -   Server sends `response.audio.delta` events.
    -   `OpenAIRealtimeClient` decodes Base64 -> `RealtimeAudioMgr` plays audio via `AudioTrack`.
    -   UI updates to "Speaking" state (Face video plays).
6.  **Tool Execution (Optional):**
    -   Server sends `response.function_call_arguments.done`.
    -   Client executes logic (e.g., `performNavigation`).
    -   Client sends `conversation.item.create` (function output) back to server.

## 4. Migration Steps

### Step 1: Dependencies
Add the following to your module-level `build.gradle` (usually `app/build.gradle`):

```gradle
dependencies {
    // Networking
    implementation 'com.squareup.okhttp3:okhttp:4.10.0'
    implementation 'com.squareup.okio:okio:3.2.0'
    
    // JSON Parsing
    implementation 'org.json:json:20210307' // Or use Android's native org.json
}
```

### Step 2: Permissions
Add these permissions to `AndroidManifest.xml`:

```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.RECORD_AUDIO" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
```

### Step 3: Assets & Resources
1.  **Knowledge Base:** Copy `app/src/main/assets/knowledge_base.txt` to your new project's `assets` folder. Update the content to match your new showroom/domain.
2.  **Robot Face:** Copy the video file (e.g., `app/src/main/res/raw/smile.mp4`) or use a placeholder image.
3.  **Layouts:** Copy `app/src/main/res/layout/dialog_chatbot.xml`. Ensure you have the necessary drawables (backgrounds, buttons).

### Step 4: Code Integration
1.  **Copy Utility Classes:** Copy the `com.keenon.peanut.sample.util` package (specifically `OpenAIRealtimeClient`, `RealtimeAudioMgr`, `ChatGPTConfig`, `ChatbotDatabase`) to your project.
2.  **Copy UI:** Copy `ChatbotDialogFragment.java`.
    -   **Update Imports:** Change package declarations to match your new project structure.
    -   **Refactor Navigation:** The current code references `Model`, `ModelManager`, and `RobotSessionManager`. You will need to adapt `performNavigation()` and `findModelByName()` to interact with your new application's navigation logic.

### Step 5: Configuration
1.  **Initialize Config:** Somewhere in your app's startup or settings screen, allow the user to input their OpenAI API Key.
    ```java
    ChatGPTConfig.getInstance(context).setApiKey("your-api-key-here");
    ChatGPTConfig.getInstance(context).setModel("gpt-4o-realtime-preview");
    ```

## 5. Troubleshooting & Common Issues

-   **Audio Format:** The Realtime API is picky about audio formats. The `RealtimeAudioMgr` implements specific resampling (16kHz -> 24kHz) if the device doesn't natively support 24kHz capture. Ensure `INPUT_SAMPLE_RATE` and `OUTPUT_SAMPLE_RATE` match the API expectations (PCM 16-bit, 24kHz, Mono).
-   **Echo/Feedback:** The `RealtimeAudioMgr` has a simple software echo suppression check (`if (isPlaying) continue;`). For robust usage in loud environments, consider using Android's `AcousticEchoCanceler` if available on the hardware.
-   **Network Latency:** Voice-to-voice relies heavily on network stability. If the connection drops, the `OpenAIRealtimeClient` will attempt to reconnect or you can handle the `onError` callback to switch to the fallback text-based mode.
-   **Permission Denied:** Ensure you handle runtime permissions for `RECORD_AUDIO` correctly (as seen in `ChatbotDialogFragment.java`).

## 6. Webhook Server (Optional)
The `webhook_server.py` is an optional component. It listens on port 5000 and is useful if you want to forward robot events to an external automation platform like n8n.
-   **Usage:** `python webhook_server.py`
-   **Endpoint:** `POST http://localhost:5000/webhook`

---
*Generated by Gemini for Project Migration*
