# Replion

Replion plays back recorded coding agent sessions so that live demos run predictably.

## Language

### Sources

**Harness**:
A coding agent tool whose sessions Replion can play back, such as Claude Code or GitHub Copilot CLI.
_Avoid_: Agent, tool, CLI

**Adapter**:
The translator that turns one Harness's stored session files into Sessions.
_Avoid_: Parser, importer, plugin

**Session**:
Replion's harness-agnostic record of a single coding agent conversation, as the Player consumes it. A resumed conversation is still one Session.
_Avoid_: Transcript, log, recording, conversation

**Session Title**:
The name shown for a Session: the Harness's own title when it has one, otherwise the shortened first prompt.
_Avoid_: Name, label, summary

### Events

**Event**:
A single moment in a Session that the Player shows at its own time.
_Avoid_: Message, entry, record, step

**Prompt**:
An Event holding what the presenter typed to the agent, including slash commands but not what they expand into.
_Avoid_: User message, input, query

**Reasoning**:
An Event holding the agent's thinking.
_Avoid_: Thinking, thoughts

**Reply**:
An Event holding text the agent wrote to the presenter.
_Avoid_: Output, response, answer, assistant message

**Tool Call**:
An Event holding one tool the agent used, together with its result. A subagent appears as a single Tool Call.
_Avoid_: Tool use, action, function call

**Compaction**:
An Event marking where the Harness summarized earlier conversation to save context. Its summary is the only thing Replion calls a "summary".
_Avoid_: Checkpoint

**Interruption**:
An Event marking where the presenter stopped the agent partway through its work.
_Avoid_: Abort, cancel

**Notification**:
An Event where the Harness reports something to the agent without the presenter typing it, such as a background agent finishing.
_Avoid_: Task notification, system message

### Choosing

**Session List**:
Every Session from every Harness, newest first, that the presenter chooses from.
_Avoid_: Library, browser, index

**Session Entry**:
One row in the Session List: a Session's Title, Harness, project folder and time, without its Events.
_Avoid_: Summary, listing, row

**Star**:
A lasting mark the presenter puts on a Session while preparing a demo, so the Session List can show only starred Sessions.
_Avoid_: Favorite, bookmark, pin

### Playback

**Player**:
The part of Replion that animates a Session for an audience.
_Avoid_: Viewer, replayer

**Stop Point**:
A place in a Session where playback waits for the presenter to continue. It comes right after each Prompt.
_Avoid_: Breakpoint, pause

**Speed**:
The factor that shrinks the recorded gaps between Events during playback. The presenter can change it at any time.
_Avoid_: Rate, tempo

**Max Pause**:
The longest wait the Player allows between two Events, however long the recorded gap was.
_Avoid_: Timeout, gap limit

**Theme**:
The presenter's choice of colors for the Player, used to fit the venue.
_Avoid_: Skin, style

**Session Totals**:
The figures shown once a Session has finished playing, such as duration, agent working time, tokens, and the number of Prompts and Tool Calls.
_Avoid_: Summary, stats, report
