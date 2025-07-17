# Xiaozhi ESP32 - Important Files Guide

## Overview
This document provides a comprehensive guide to the key files in the xiaozhi-esp32 project, focusing on AI flow and agentic patterns. The project implements an ESP32-based AI chatbot with voice interaction and device control capabilities using the Model Context Protocol (MCP).

## Core Architecture

### AI Communication Channels
The system uses **two separate communication channels**:

1. **Primary AI Communication** (Non-MCP)
   - Voice interactions (ASR/TTS)
   - Chat messages
   - Emotion control
   - Uses traditional JSON messages

2. **MCP Protocol** (Tool Calls Only)
   - Device control commands
   - Hardware function calls
   - Uses JSON-RPC 2.0 format

## Key Files Analysis

### 1. Main Application Logic

#### `main/application.cc` 🎯 **MOST IMPORTANT**
**The central orchestrator of the entire AI interaction system**

**Key Functions:**
- **Audio Streaming** (lines 481-486, 606-643)
  ```cpp
  // Receiving audio from LLM/TTS service
  protocol_->OnIncomingAudio([this](AudioStreamPacket&& packet) {
      audio_decode_queue_.emplace_back(std::move(packet));
  });
  
  // Sending audio to LLM/ASR service
  audio_processor_->OnOutput([this](std::vector<int16_t>&& data) {
      opus_encoder_->Encode(std::move(data), [this](std::vector<uint8_t>&& opus) {
          audio_send_queue_.emplace_back(std::move(packet));
      });
  });
  ```

- **Main Conversation Flow** (lines 511-600)
  ```cpp
  protocol_->OnIncomingJson([this, display](const cJSON* root) {
      auto type = cJSON_GetObjectItem(root, "type");
      
      if (strcmp(type->valuestring, "tts") == 0) {
          // Handle TTS responses from LLM
      } else if (strcmp(type->valuestring, "stt") == 0) {
          // Handle STT results
      } else if (strcmp(type->valuestring, "llm") == 0) {
          // Handle LLM emotion responses
      } else if (strcmp(type->valuestring, "mcp") == 0) {
          // Handle MCP tool calls
      }
  });
  ```

- **AI State Machine** (lines 966-1037)
  - Device states: `idle`, `listening`, `speaking`, `connecting`
  - Voice-triggered activation with wake word detection
  - Streaming audio processing (ASR → LLM → TTS)

- **Audio Processing Loop** (lines 792-906)
  ```cpp
  void Application::AudioLoop() {
      while (true) {
          OnAudioInput();   // Process microphone input
          OnAudioOutput();  // Process speaker output
      }
  }
  ```

#### `main/application.h`
**Header file defining the Application class structure**
- State definitions and enums
- Audio processing components
- Protocol interfaces

### 2. MCP (Model Context Protocol) Implementation

#### `main/mcp_server.cc` 🤖 **AGENTIC CORE**
**The heart of the agentic system - enables AI to discover and control device capabilities**

**Key Features:**
- **Tool Registration System**
  ```cpp
  McpServer::GetInstance().AddTool(
      "self.audio_speaker.set_volume",
      "Set the volume of the audio speaker",
      PropertyList({Property("volume", kPropertyTypeInteger, 0, 100)}),
      [&board](const PropertyList& properties) -> ReturnValue {
          auto codec = board.GetAudioCodec();
          codec->SetOutputVolume(properties["volume"].value<int>());
          return true;
      }
  );
  ```

- **Common System Tools**
  - `self.get_device_status` - Device status query
  - `self.audio_speaker.set_volume` - Audio control
  - `self.screen.set_brightness` - Display control
  - `self.screen.set_theme` - Theme switching
  - `self.camera.take_photo` - Vision AI integration

- **System Prompts** (Tool descriptions that guide AI)
  ```cpp
  "Provides the real-time information of the device, including the current status of the audio speaker, screen, battery, network, etc.\n"
  "Use this tool for: \n"
  "1. Answering questions about current condition\n"
  "2. As the first step to control the device"
  ```

#### `main/mcp_server.h`
**MCP server interface definitions**
- Tool class definitions
- Property system for type-safe parameters
- JSON-RPC message structures

### 3. Communication Protocols

#### `main/protocols/websocket_protocol.cc`
**WebSocket communication with LLM backend**
- Real-time bidirectional communication
- JSON message handling
- Binary audio streaming
- Connection management

#### `main/protocols/mqtt_protocol.cc`
**MQTT communication with LLM backend**
- Alternative to WebSocket
- Same message format, different transport
- Better for certain network conditions

#### `main/protocols/protocol.cc`
**Base protocol class**
- Common functionality for both WebSocket and MQTT
- Message serialization/deserialization
- Session management

### 4. Audio Processing Components

#### `main/audio_processing/afe_audio_processor.cc`
**Audio Front-End processing**
- Noise suppression and echo cancellation
- Voice Activity Detection (VAD)
- Audio enhancement for better AI recognition
- Feeds processed audio to conversation system

#### `main/audio_processing/afe_wake_word.cc`
**Wake word detection system**
- Detects "小智" (Xiaozhi) wake word
- Triggers conversation start
- Integrates with main conversation flow
- Offline processing for privacy

#### `main/audio_codecs/audio_codec.cc`
**Hardware audio interface**
- Microphone input capture
- Speaker output playback
- Sample rate conversion
- Hardware abstraction layer

### 5. Board-Specific Implementations

#### `main/boards/esp-hi/esp_hi.cc`
**ESP-Hi robot board implementation**
- **Robot Control Tools**
  ```cpp
  // Basic robot control
  mcp_server.AddTool("self.dog.basic_control", 
      "机器人的基础动作。forward: 向前移动\nbackward: 向后移动\nturn_left: 向左转",
      PropertyList({Property("action", kPropertyTypeString)}),
      [this](const PropertyList& properties) -> ReturnValue {
          const std::string& action = properties["action"].value<std::string>();
          if (action == "forward") {
              servo_dog_ctrl_send(DOG_STATE_FORWARD, NULL);
          }
          return true;
      });
  
  // LED control
  mcp_server.AddTool("self.light.set_rgb", "设置RGB颜色", 
      PropertyList({
          Property("r", kPropertyTypeInteger, 0, 255),
          Property("g", kPropertyTypeInteger, 0, 255),
          Property("b", kPropertyTypeInteger, 0, 255)
      }),
      [this](const PropertyList& properties) -> ReturnValue {
          int r = properties["r"].value<int>();
          int g = properties["g"].value<int>();
          int b = properties["b"].value<int>();
          SetLedColor(r, g, b);
          return true;
      });
  ```

#### `main/boards/otto-robot/otto_controller.cc`
**Otto robot implementation**
- **Movement Control**
  ```cpp
  mcp_server.AddTool("self.otto.walk_forward", 
      "行走。steps: 行走步数(1-100); speed: 行走速度(500-1500); direction: 行走方向(-1=后退, 1=前进)",
      PropertyList({
          Property("steps", kPropertyTypeInteger, 3, 1, 100),
          Property("speed", kPropertyTypeInteger, 1000, 500, 1500),
          Property("direction", kPropertyTypeInteger, 1, -1, 1)
      }),
      [this](const PropertyList& properties) -> ReturnValue {
          int steps = properties["steps"].value<int>();
          int speed = properties["speed"].value<int>();
          int direction = properties["direction"].value<int>();
          QueueAction(ACTION_WALK, steps, speed, direction, 0);
          return true;
      });
  ```

#### `main/boards/sensecap-watcher/sensecap_watcher.cc`
**SenseCAP Watcher implementation**
- Vision AI integration
- SSCMA (SenseCAP Smart Camera AI) support
- Advanced camera control

### 6. Display and User Interface

#### `main/display/display.cc`
**Display system interface**
- Status bar updates
- Chat message display
- Emotion visualization
- Theme management

#### `main/display/lcd_display.cc`
**LCD display implementation**
- Hardware-specific LCD control
- Graphics rendering
- Touch interface support

### 7. Hardware Abstraction

#### `main/boards/common/board.cc`
**Hardware abstraction layer**
- Board initialization
- Component management
- Power management
- Hardware status reporting

#### `main/boards/common/wifi_board.cc`
**WiFi connectivity**
- Network connection management
- WiFi configuration
- Network status monitoring

### 8. Documentation Files

#### `docs/mcp-protocol.md` 📚 **PROTOCOL REFERENCE**
**Complete MCP protocol documentation**
- Message format specifications
- Interaction flow diagrams
- JSON-RPC 2.0 implementation details
- Tool call examples

#### `docs/mcp-usage.md` 📖 **USAGE GUIDE**
**Practical MCP usage examples**
- Tool registration patterns
- Common control scenarios
- JSON message examples
- Best practices

#### `docs/websocket.md`
**WebSocket communication protocol**
- Session management
- Message formats
- Connection handling

## Message Flow Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    LLM/AI Backend                           │
├─────────────────────────────────────────────────────────────┤
│ 1. Voice Processing (ASR/TTS)     ←→ WebSocket/MQTT        │
│ 2. Chat Messages                  ←→ JSON Messages         │
│ 3. Emotion Control               ←→ JSON Messages         │
│ 4. Tool Calls                    ←→ MCP Protocol          │
└─────────────────────────────────────────────────────────────┘
                           │
                           │ All wrapped in session messages
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                    ESP32 Device                             │
│                                                             │
│ • Main conversation: Traditional JSON                       │
│ • Device control: MCP tool calls                          │
│ • Audio streaming: Binary OPUS data                        │
└─────────────────────────────────────────────────────────────┘
```

## Key Message Formats

### Traditional JSON Messages
```json
{
  "type": "tts",
  "state": "sentence_start",
  "text": "Hello, how can I help you?"
}

{
  "type": "stt",
  "text": "Turn on the lights"
}

{
  "type": "llm",
  "emotion": "happy"
}
```

### MCP Tool Calls
```json
{
  "type": "mcp",
  "payload": {
    "jsonrpc": "2.0",
    "method": "tools/call",
    "params": {
      "name": "self.audio_speaker.set_volume",
      "arguments": {"volume": 50}
    },
    "id": 2
  }
}
```

### MCP Tool Response
```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "content": [{"type": "text", "text": "true"}],
    "isError": false
  }
}
```

## Development Workflow

### Adding New Device Capabilities

1. **Register MCP Tools** in board-specific files
   ```cpp
   mcp_server.AddTool("self.device.new_function", 
       "Description for AI understanding",
       PropertyList({Property("param", kPropertyTypeInteger, 0, 100)}),
       [this](const PropertyList& properties) -> ReturnValue {
           // Implementation
           return true;
       });
   ```

2. **Update Device Status** in `board.cc`
   ```cpp
   std::string GetDeviceStatusJson() {
       // Return JSON with current device state
   }
   ```

3. **Handle Hardware** in board-specific implementation
   - Initialize hardware components
   - Implement control logic
   - Manage hardware resources

### Testing AI Interactions

1. **Monitor Debug Output**
   ```cpp
   ESP_LOGI(TAG, ">> %s", user_message);  // User speech
   ESP_LOGI(TAG, "<< %s", ai_response);   // AI response
   ```

2. **Test Tool Calls** via MCP protocol
   - Use `tools/list` to discover capabilities
   - Use `tools/call` to test functionality

3. **Verify Audio Flow**
   - Check audio input/output levels
   - Monitor OPUS encoding/decoding
   - Verify TTS playback quality

## Configuration Files

### `main/Kconfig.projbuild`
**Build configuration**
- Protocol selection (MCP vs legacy)
- Audio processing options
- Hardware feature flags

### `main/boards/*/config.json`
**Board-specific configuration**
- Hardware pin mappings
- Audio codec settings
- Display parameters

## Summary

The xiaozhi-esp32 project implements a sophisticated AI interaction system with these key characteristics:

1. **Dual Communication Channels**: Traditional JSON for conversation, MCP for device control
2. **Agentic Architecture**: AI can discover and invoke device capabilities dynamically
3. **Real-time Audio Processing**: Streaming ASR/TTS with voice activity detection
4. **Modular Design**: Easy to add new devices and capabilities
5. **Hardware Abstraction**: Supports 70+ different ESP32 boards
6. **Type-Safe Tool System**: Robust parameter validation and error handling

The **`main/application.cc`** file is the central orchestrator, while **`main/mcp_server.cc`** enables the agentic capabilities that make this system truly intelligent and adaptable.