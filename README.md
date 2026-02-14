# LiveKit Overview

<img width="1732" height="937" alt="image" src="https://github.com/user-attachments/assets/9d5ddfbb-2af7-476e-b79c-f5348e8f3ad4" />

## 1. LiveKit Server

-   Hosts rooms (similar to MS Teams rooms).
-   Web clients and agent servers connect to the rooms hosted on livekit server.

------------------------------------------------------------------------

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

------------------------------------------------------------------------

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

------------------------------------------------------------------------

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

------------------------------------------------------------------------

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
------------------------------------------------------------------------

## 5. Web Client

-   Any client for human to connect to room (React clients, Gradio apps, mobile apps, etc.)
-   Connects to rooms the same way as Agent Servers:
    -   Request access token from Token Server
    -   Use token to join LiveKit room

------------------------------------------------------------------------

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
