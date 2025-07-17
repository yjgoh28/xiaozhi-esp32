# AI Service Architecture - Xiaozhi ESP32

## Overview
This document explains the complete AI service architecture in the xiaozhi-esp32 project. While `application.cc` and `mcp_server.cc` are the main coordinators, the AI services are actually distributed across dozens of files working together to provide a complete AI interaction experience.

## AI Service Architecture Distribution

### **`application.cc`** - Main AI Orchestrator 🧠
**Role**: Central coordinator for AI interactions
- **Audio streaming** (microphone → AI, AI → speakers)
- **Conversation state management** (listening, speaking, idle)
- **Message routing** (determines what type of AI message it is)
- **Protocol coordination** (WebSocket/MQTT communication)

### **`mcp_server.cc`** - Tool Execution Only 🔧
**Role**: Handles tool calls from AI (device control commands)
- **Tool registration** and **tool execution**
- **JSON-RPC message parsing**
- **Device capability exposure**

### **But AI Services Are Actually Distributed Across Multiple Files:**

## 1. **Communication Layer** 🌐

### **`protocols/websocket_protocol.cc`**
- **Actual network communication** with AI backend
- **Message serialization/deserialization**
- **Connection management**
- **Binary audio streaming**

### **`protocols/mqtt_protocol.cc`**
- **Alternative communication protocol**
- **Same AI message handling, different transport**

### **`protocols/protocol.cc`**
- **Base protocol implementation**
- **Session management**
- **Message formatting**

## 2. **Audio Processing (Critical for AI)** 🎵

### **`audio_processing/afe_audio_processor.cc`**
- **Audio enhancement** for better AI recognition
- **Voice Activity Detection** (tells AI when user is speaking)
- **Echo cancellation** for AI conversation quality
- **Noise suppression** for cleaner AI input

### **`audio_processing/afe_wake_word.cc`**
- **Wake word detection** ("小智") - triggers AI interaction
- **Offline AI processing** for privacy
- **Audio preprocessing** for AI services

### **`audio_codecs/audio_codec.cc`**
- **Hardware audio interface** for AI communication
- **Real-time audio capture** and **playback**
- **Sample rate conversion** for AI requirements

## 3. **Display Integration** 🖥️

### **`display/display.cc`**
- **AI conversation display** (shows chat messages)
- **Emotion visualization** (AI emotion responses)
- **Status updates** (AI connection status)

## 4. **Board-Specific AI Tools** 🤖

### **`boards/otto-robot/otto_controller.cc`**
- **Robot-specific AI tools** (movement, gestures)
- **Hardware control** for AI commands
- **Status reporting** to AI

### **`boards/esp-hi/esp_hi.cc`**
- **Quadruped robot AI tools**
- **LED control** for AI responses
- **Camera integration** for AI vision

### **`boards/sensecap-watcher/sensecap_watcher.cc`**
- **Vision AI integration** (SSCMA camera AI)
- **Advanced camera controls** for AI

## 5. **Background Processing** ⚙️

### **`background_task.cc`**
- **Asynchronous AI task processing**
- **Audio encoding/decoding** for AI
- **Non-blocking tool execution**

## Complete AI Service Flow

```
User Voice Input
      ↓
audio_codec.cc (hardware capture)
      ↓
afe_audio_processor.cc (enhancement)
      ↓
application.cc (encoding & state management)
      ↓
websocket_protocol.cc (send to AI backend)
      ↓
[AI Backend Processing]
      ↓
websocket_protocol.cc (receive AI response)
      ↓
application.cc (message routing)
      ↓
┌─────────────────────────────────────────────────────────┐
│ Message Type Routing:                                   │
│ • "tts" → Audio playback                               │
│ • "stt" → Display user message                         │
│ • "llm" → Update emotions                              │
│ • "mcp" → mcp_server.cc (tool execution)              │
└─────────────────────────────────────────────────────────┘
      ↓
display.cc (show results) + audio_codec.cc (play TTS)
```

## Detailed Component Breakdown

### **Central Coordination** (`application.cc`)
```cpp
// Main AI message routing
protocol_->OnIncomingJson([this, display](const cJSON* root) {
    auto type = cJSON_GetObjectItem(root, "type");
    
    if (strcmp(type->valuestring, "tts") == 0) {
        // Handle TTS responses from AI
        auto state = cJSON_GetObjectItem(root, "state");
        if (strcmp(state->valuestring, "start") == 0) {
            SetDeviceState(kDeviceStateSpeaking);
        } else if (strcmp(state->valuestring, "sentence_start") == 0) {
            auto text = cJSON_GetObjectItem(root, "text");
            display->SetChatMessage("assistant", text->valuestring);
        }
    } else if (strcmp(type->valuestring, "stt") == 0) {
        // Handle STT results
        auto text = cJSON_GetObjectItem(root, "text");
        display->SetChatMessage("user", text->valuestring);
    } else if (strcmp(type->valuestring, "llm") == 0) {
        // Handle LLM emotion responses
        auto emotion = cJSON_GetObjectItem(root, "emotion");
        display->SetEmotion(emotion->valuestring);
    } else if (strcmp(type->valuestring, "mcp") == 0) {
        // Handle MCP tool calls
        auto payload = cJSON_GetObjectItem(root, "payload");
        McpServer::GetInstance().ParseMessage(payload);
    }
});
```

### **Audio Processing Pipeline**
```cpp
// Audio input processing (application.cc)
audio_processor_->OnOutput([this](std::vector<int16_t>&& data) {
    background_task_->Schedule([this, data = std::move(data)]() mutable {
        opus_encoder_->Encode(std::move(data), [this](std::vector<uint8_t>&& opus) {
            AudioStreamPacket packet;
            packet.payload = std::move(opus);
            audio_send_queue_.emplace_back(std::move(packet));
            xEventGroupSetBits(event_group_, SEND_AUDIO_EVENT);
        });
    });
});

// Audio output processing (application.cc)
protocol_->OnIncomingAudio([this](AudioStreamPacket&& packet) {
    std::lock_guard<std::mutex> lock(mutex_);
    if (device_state_ == kDeviceStateSpeaking) {
        audio_decode_queue_.emplace_back(std::move(packet));
    }
});
```

### **MCP Tool Registration** (`mcp_server.cc`)
```cpp
// Common tools available to all AI instances
AddTool("self.get_device_status",
    "Provides the real-time information of the device",
    PropertyList(),
    [&board](const PropertyList& properties) -> ReturnValue {
        return board.GetDeviceStatusJson();
    });

AddTool("self.audio_speaker.set_volume", 
    "Set the volume of the audio speaker",
    PropertyList({Property("volume", kPropertyTypeInteger, 0, 100)}), 
    [&board](const PropertyList& properties) -> ReturnValue {
        auto codec = board.GetAudioCodec();
        codec->SetOutputVolume(properties["volume"].value<int>());
        return true;
    });
```

### **Board-Specific AI Tools** (Example: Otto Robot)
```cpp
// Otto robot AI tools (otto_controller.cc)
mcp_server.AddTool("self.otto.walk_forward", 
    "行走。steps: 行走步数(1-100); speed: 行走速度(500-1500); direction: 行走方向(-1=后退, 1=前进)",
    PropertyList({
        Property("steps", kPropertyTypeInteger, 3, 1, 100),
        Property("speed", kPropertyTypeInteger, 1000, 500, 1500),
        Property("direction", kPropertyTypeInteger, 1, -1, 1),
        Property("arm_swing", kPropertyTypeInteger, 0, 0, 1)
    }),
    [this](const PropertyList& properties) -> ReturnValue {
        int steps = properties["steps"].value<int>();
        int speed = properties["speed"].value<int>();
        int direction = properties["direction"].value<int>();
        int arm_swing = properties["arm_swing"].value<int>();
        QueueAction(ACTION_WALK, steps, speed, direction, arm_swing);
        return true;
    });
```

## AI Service State Management

### **Device States** (`application.cc`)
```cpp
enum DeviceState {
    kDeviceStateUnknown,
    kDeviceStateStarting,
    kDeviceStateWifiConfiguring,
    kDeviceStateIdle,           // Ready for AI interaction
    kDeviceStateConnecting,     // Connecting to AI backend
    kDeviceStateListening,      // Sending audio to AI
    kDeviceStateSpeaking,       // Playing AI response
    kDeviceStateUpgrading,
    kDeviceStateActivating,
    kDeviceStateAudioTesting,
    kDeviceStateFatalError
};
```

### **State Transitions for AI Interaction**
```cpp
void Application::SetDeviceState(DeviceState state) {
    switch (state) {
        case kDeviceStateIdle:
            display->SetStatus("待机");
            audio_processor_->Stop();
            wake_word_->StartDetection();  // Wait for "小智"
            break;
        case kDeviceStateListening:
            display->SetStatus("聆听中");
            protocol_->SendStartListening(listening_mode_);
            audio_processor_->Start();  // Send audio to AI
            wake_word_->StopDetection();
            break;
        case kDeviceStateSpeaking:
            display->SetStatus("回答中");
            audio_processor_->Stop();
            ResetDecoder();  // Prepare for AI audio
            break;
    }
}
```

## Protocol Layer Details

### **WebSocket Protocol** (`websocket_protocol.cc`)
```cpp
// Sending audio to AI backend
bool WebsocketProtocol::SendAudio(const AudioStreamPacket& packet) {
    if (!audio_ws_client_) return false;
    
    // Create session message with audio payload
    cJSON* root = cJSON_CreateObject();
    cJSON_AddStringToObject(root, "session_id", session_id_.c_str());
    cJSON_AddStringToObject(root, "type", "audio");
    
    // Add binary audio data
    std::string encoded = base64_encode(packet.payload.data(), packet.payload.size());
    cJSON_AddStringToObject(root, "data", encoded.c_str());
    
    // Send to AI backend
    char* json_string = cJSON_Print(root);
    bool result = audio_ws_client_->send(json_string) == 0;
    
    free(json_string);
    cJSON_Delete(root);
    return result;
}
```

### **MQTT Protocol** (`mqtt_protocol.cc`)
```cpp
// Alternative transport for AI communication
bool MqttProtocol::SendAudio(const AudioStreamPacket& packet) {
    if (!mqtt_client_) return false;
    
    // Publish audio to AI backend via MQTT
    std::string topic = "xiaozhi/audio/" + session_id_;
    return esp_mqtt_client_publish(mqtt_client_, topic.c_str(), 
                                 (const char*)packet.payload.data(), 
                                 packet.payload.size(), 0, 0) != -1;
}
```

## Audio Processing Architecture

### **Audio Enhancement** (`afe_audio_processor.cc`)
```cpp
// Audio preprocessing for better AI recognition
void AfeAudioProcessor::ProcessAudio(const std::vector<int16_t>& input, 
                                   std::vector<int16_t>& output) {
    // Apply noise suppression
    esp_afe_sr_data_t* afe_data = esp_afe_sr_process(afe_handle_, 
                                                    input.data(), 
                                                    input.size());
    
    // Apply voice activity detection
    bool voice_detected = afe_data->vad_state == AFE_VAD_SPEECH;
    
    // Notify AI system of voice activity
    if (voice_detected != last_vad_state_) {
        OnVadStateChange(voice_detected);
        last_vad_state_ = voice_detected;
    }
    
    // Output enhanced audio for AI
    output.assign(afe_data->data, afe_data->data + afe_data->len);
}
```

### **Wake Word Detection** (`afe_wake_word.cc`)
```cpp
// Offline AI processing for privacy
void AfeWakeWord::ProcessAudio(const std::vector<int16_t>& audio) {
    if (!detection_running_) return;
    
    // Use ESP-SR AI model for wake word detection
    int wake_word_result = esp_afe_sr_detect(afe_handle_, audio.data());
    
    if (wake_word_result == ESP_AFE_SR_RESULT_WAKE_WORD) {
        ESP_LOGI(TAG, "Wake word detected: 小智");
        
        // Trigger AI interaction
        OnWakeWordDetected("小智");
        
        // Prepare audio context for AI
        EncodeWakeWordData();
    }
}
```

## Vision AI Integration

### **Camera AI Tools** (`sensecap_watcher.cc`)
```cpp
// Vision AI integration
mcp_server.AddTool("self.camera.take_photo",
    "Take a photo and explain it with AI vision",
    PropertyList({Property("question", kPropertyTypeString)}),
    [this](const PropertyList& properties) -> ReturnValue {
        auto question = properties["question"].value<std::string>();
        
        // Capture image
        if (!camera_->Capture()) {
            return "{\"success\": false, \"message\": \"Failed to capture photo\"}";
        }
        
        // Send to AI vision service
        std::string ai_response = camera_->Explain(question);
        return ai_response;
    });
```

## Key Architecture Insights

### **Distributed AI Services**
The AI functionality is distributed across:
- **8+ audio processing files** (real-time AI audio handling)
- **3+ protocol files** (AI communication)
- **70+ board files** (AI tool implementations)
- **4+ display files** (AI interaction visualization)
- **Background processing** (async AI operations)

### **Design Principles**
1. **Modularity** - Different boards, different AI tools
2. **Real-time Performance** - Audio can't be blocked by tool execution
3. **Hardware Abstraction** - Same AI interface, different hardware
4. **Maintainability** - Each component has specific AI-related responsibility

### **AI Service Layers**
```
┌─────────────────────────────────────────────────────────┐
│ Application Layer (application.cc)                      │
│ - Conversation orchestration                            │
│ - State management                                      │
│ - Message routing                                       │
├─────────────────────────────────────────────────────────┤
│ Protocol Layer (websocket_protocol.cc, mqtt_protocol.cc)│
│ - Network communication                                 │
│ - Message serialization                                 │
│ - Session management                                    │
├─────────────────────────────────────────────────────────┤
│ Processing Layer (audio_processing/, mcp_server.cc)     │
│ - Audio enhancement                                     │
│ - Tool execution                                        │
│ - Background tasks                                      │
├─────────────────────────────────────────────────────────┤
│ Hardware Layer (boards/, audio_codecs/, display/)       │
│ - Device-specific AI tools                             │
│ - Audio capture/playback                               │
│ - Visual feedback                                      │
└─────────────────────────────────────────────────────────┘
```

## Summary

While **`application.cc`** serves as the **central nervous system** coordinating all AI services, and **`mcp_server.cc`** handles AI tool execution, the complete AI service architecture spans across **dozens of files** working together. Each component has a specific role in providing:

- **Real-time voice interaction**
- **Intelligent device control**
- **Visual AI feedback**
- **Hardware-specific AI capabilities**
- **Robust communication with AI backends**

This distributed architecture ensures high performance, maintainability, and extensibility for the AI-powered IoT system.