# LiveKit Overview

The livekit system has 4 components:
1. Livekit server
2. Token server
3. Agent server
4. Web client

<img width="1732" height="937" alt="image" src="https://github.com/user-attachments/assets/9d5ddfbb-2af7-476e-b79c-f5348e8f3ad4" />

## 1. LiveKit Server

-   Hosts rooms (similar to MS Teams rooms).
-   Web clients and agent servers connect to the rooms hosted on livekit server.

## 2. Token Server

- Issues **access tokens** for joining rooms.
- LiveKit doesn't provide a built-in token server for production use. 
- Need to implement custom backend server to generate access tokens (often use authentication providers like **Auth0, Clerk, or Firebase Auth**).

### Flow

1.  Web client / agent server calls the **Token Server** to get an
    access token.
2.  Use that access token to connect to the **LiveKit Server** and
    create/join a room.
3.  Other web clients / agent servers also use **the same access token** to join the
    same room.

## 3. Agent Server

-   A server that runs agents.
-   One server can run multiple sessions.
-   1 session can have many personas, but can only run 1 persona at a time.
  
### Example: Defining Personas

``` python
class Assistant(Agent):
    def __init__(self):
        super().__init__(
            instructions="You are a helpful voice AI assistant..."
        )

class BillingAgent(Agent):
    def __init__(self):
        super().__init__(
            instructions="You handle billing questions..."
        )

class SupportAgent(Agent):
    def __init__(self):
        super().__init__(
            instructions="You handle technical support..."
        )
```

### Persona Switching (Handoff)

- In 1 session, an agent can switch to different persona.
LiveKit supports **handoff** to switch from one agent persona to
another.

``` python
@function_tool
async def transfer_to_billing(self, ctx: RunContext):
    """Transfer to billing department"""
    await self.session.say("Transferring you to billing...")
    
    # Returning a NEW agent instance triggers handoff
    return BillingAgent()
```

- When the function returns a new agent instance, LiveKit automatically
transfers control to that agent.

### Session

- `AgentSession` control the AI provider:

```python
session = AgentSession(
    stt="deepgram/nova-3:multi",
    llm="openai/gpt-4.1-mini",
    tts="cartesia/sonic-3:9626c31c-bec5-4cca-baa8-f8ba9e84c8bc",
    vad=silero.VAD.load(),
    turn_detection=MultilingualModel(),
)
```

-   One Agent Server can run multiple sessions at the same time.

``` python
@server.rtc_session(agent_name="voice-agent")
async def voice_agent(ctx: JobContext):
    session = AgentSession(...)
    await session.start(room=ctx.room, agent=VoiceAssistant())

@server.rtc_session(agent_name="transcriber-agent")
async def transcriber(ctx: JobContext):
    session = AgentSession(...)
    await session.start(room=ctx.room, agent=TranscriberAgent())
```
## 5. Web Client

-   Any client for human to connect to room (React clients, Gradio apps, mobile apps, etc.)
-   Connects to rooms the same way as Agent Servers:
    -   Request access token from Token Server
    -   Use token to join LiveKit room


# Setup LiveKit (Locally, for dev)

## 1. LiveKit Server
Install livekit-server, livekit-cli
```shell
# Install livekit-server
curl -sSL https://get.livekit.io | bash

# Install livekit-cli (lk)
curl -sSL https://get.livekit.io/cli | bash

# Test installation
livekit-server --version
lk --version
```

Run server 
```shell
livekit-server --dev --port 8080 --bind 0.0.0.0
```

## 2. Generate Token 
For dev mode:
```shell
lk token create \
  --api-key devkey \
  --api-secret secret \
  --room my-room \
  --identity user123
```

## 3. Run agent server
Install libraries:
```shell
uv add \
  "livekit-agents[silero,turn-detector]~=1.3" \
  "livekit-plugins-noise-cancellation~=0.2" \
  "python-dotenv"
```
Create `.env`:
```shell
LIVEKIT_API_KEY=devkey
LIVEKIT_API_SECRET=secret
LIVEKIT_URL=ws://localhost:8080
```
Create `agent.py`:
```python
from dotenv import load_dotenv

from livekit import agents, rtc
from livekit.agents import AgentServer,AgentSession, Agent, room_io
from livekit.plugins import noise_cancellation, silero
from livekit.plugins.turn_detector.multilingual import MultilingualModel

load_dotenv(".env")

class Assistant(Agent):
    def __init__(self) -> None:
        super().__init__(
            instructions="""You are a helpful voice AI assistant.
            You eagerly assist users with their questions by providing information from your extensive knowledge.
            Your responses are concise, to the point, and without any complex formatting or punctuation including emojis, asterisks, or other symbols.
            You are curious, friendly, and have a sense of humor.""",
        )

server = AgentServer()

@server.rtc_session(agent_name="my-agent")
async def my_agent(ctx: agents.JobContext):
    session = AgentSession(
        stt="deepgram/nova-3:multi",
        llm="openai/gpt-4.1-mini",
        tts="cartesia/sonic-3:9626c31c-bec5-4cca-baa8-f8ba9e84c8bc",
        vad=silero.VAD.load(),
        turn_detection=MultilingualModel(),
    )

    await session.start(
        room=ctx.room,
        agent=Assistant(),
        room_options=room_io.RoomOptions(
            audio_input=room_io.AudioInputOptions(
                noise_cancellation=lambda params: noise_cancellation.BVCTelephony() if params.participant.kind == rtc.ParticipantKind.PARTICIPANT_KIND_SIP else noise_cancellation.BVC(),
            ),
        ),
    )

    await session.generate_reply(
        instructions="Greet the user and offer your assistance."
    )


if __name__ == "__main__":
    agents.cli.run_app(server)
```

Download models and run `agent.py`:
```shell
uv run agent.py download-files

uv run agent.py dev
```
# Agent server advanced 

#### 1. Custom AI provider
Using plugin:

```python
session = AgentSession(
    stt=inference.STT(
        model="assemblyai/universal-streaming", 
        language="en"
    ),
    llm="openai/gpt-4.1-mini",
    tts="cartesia/sonic-3:9626c31c-bec5-4cca-baa8-f8ba9e84c8bc",
    vad=silero.VAD.load(),
    turn_detection=MultilingualModel(),
)
```

To use a custom STT, LLM, or TTS provider without a plugin, we have 2 options:

**Option 1**:  inherit from `tts.TTS`, `llm.LLM`, `stt.STT`. For example

```python
# Example, inhering from tts.TTS

from livekit import rtc
import httpx
from livekit.agents import tts, llm, stt
from typing import AsyncIterable

class CustomTTS(tts.TTS):
    """Custom Text-to-Speech provider"""
    
    def __init__(self, api_url: str, api_key: str, voice: str = "default"):
        self._api_url = api_url
        self._api_key = api_key
        self._voice = voice
        self._client = httpx.AsyncClient()
    
    def synthesize(
        self,
        text: str,
    ) -> AsyncIterable[rtc.AudioFrame]:
        """Convert text to speech audio"""
        
        async def _generate():
            # Call your TTS API
            async with self._client.stream(
                "POST",
                f"{self._api_url}/synthesize",
                headers={"Authorization": f"Bearer {self._api_key}"},
                json={
                    "text": text,
                    "voice": self._voice,
                    "sample_rate": 24000,
                    "format": "pcm"
                }
            ) as response:
                async for chunk in response.aiter_bytes(chunk_size=4800):
                    # Convert bytes to AudioFrame
                    frame = rtc.AudioFrame(
                        data=chunk,
                        sample_rate=24000,
                        num_channels=1,
                        samples_per_channel=len(chunk) // 2
                    )
                    yield frame
        
        return _generate()

# Usage
session = AgentSession(
    tts=CustomTTS(
        api_url="https://your-tts-api.com",
        api_key="your-key",
        voice="vietnamese-female"
    ),
    # ... other config
)
```

**Option 2**: Customize `stt_node()`, `llm_node()`, `tts_node()`

```python
# Example customizing tts_node

class CustomSTTAgent(Agent):
    async def tts_node(
            self,
            text: AsyncIterable[str],
            model_settings: ModelSettings
        ) -> AsyncIterable[rtc.AudioFrame]:
            """Override TTS node to use custom API"""
            
            async for text_chunk in text:
                # Call your custom TTS API
                async for audio_frame in self._call_tts_api(text_chunk):
                    yield audio_frame
        
        async def _call_tts_api(self, text: str) -> AsyncIterable[rtc.AudioFrame]:
            """Call external TTS API and convert to AudioFrames"""
            pass
            

# Usage
session = AgentSession(
    stt="deepgram/nova-2",
    llm="openai/gpt-4o"
    # No TTS - handled by tts_node override
)
```

### 2. Function calling
```python
class MyAgent(Agent):
    @function_tool
    async def get_weather(self, ctx, city):
        return f"Weather in {city}: 32°C, sunny"

```

How it works:

User says: "What's the weather in Hanoi?"
LLM sees the get_weather tool and **decides** to call it

**You DON'T control when tools are called.** 

Can try to force the model to call tool using prompt:

```python
class WeatherAgent(Agent):
    def __init__(self):
        super().__init__(
            instructions="""You are a weather assistant.
            
            IMPORTANT: You MUST use the get_weather tool for ANY weather question.
            NEVER guess or make up weather information.
            Always call get_weather before responding about weather."""
        )
```

### 3. RAG retrieval
Is calling complex tool

```python
class RAGAgent(Agent):
    def __init__(self):
        super().__init__(
            instructions="Answer questions using the knowledge base."
        )
    
    @function_tool
    async def search_knowledge(
        self,
        ctx: RunContext,
        query: str
    ) -> str:
        """Search the knowledge base for relevant information.
        
        Args:
            query: What to search for
        """
        # Call your vector DB API
        async with httpx.AsyncClient() as client:
            response = await client.post(
                "https://your-vectordb.com/search",
                json={
                    "query": query,
                    "top_k": 3
                }
            )
            results = response.json()
            
            # Format results
            context = "\n\n".join([
                f"Document {i+1}: {doc['text']}"
                for i, doc in enumerate(results['documents'])
            ])
            
            return context
```

