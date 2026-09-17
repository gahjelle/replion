# Decoding mixed-shape JSONL in Gleam

Research for [#3](https://github.com/gahjelle/replion/issues/3). Researched 2026-09-17.

## Question

What's the idiomatic way in Gleam to decode JSONL files where:

- each line is one of many unrelated record shapes, told apart by `type` (and sometimes `subtype`)
- unknown types get skipped instead of failing
- fields like `message.content` can be a string or an array of typed blocks

And how do you read such a file line by line on the Erlang target, including large files?

## Answer in brief

- **Parse one line at a time.** Call `json.parse(line, record_decoder())` on each line and handle each line's `Result` separately. One bad line gives one `Error` and never stops the rest of the file.
- **Pick the shape by `type`.** Decode `type` with `use type_ <- decode.field("type", decode.string)`, then use `case type_ { ... }` to choose the decoder for the rest of the record. Add a catch-all branch, `other -> decode.success(Skipped(other))`, so unknown types come back as a value instead of an error. Handle `subtype` the same way inside a branch.
- **Handle string-or-array with `one_of`.** Use `decode.one_of(decode.string |> decode.map(fn(s) { [Text(s)] }), or: [decode.list(block_decoder())])`, so both forms become `List(Block)`. Block `type`s get the same `case` treatment, with an `OtherBlock` catch-all.
- **Stream the file with `file_streams`.** `simplifile` (v2.7.0) can only read a whole file into memory; it has no line or stream API. `file_streams` (v2.0.0) provides `file_stream.open_read` and `file_stream.read_line`. It opens files with `raw` and a 64 KiB `read_ahead`, which is the setup Erlang's docs recommend for reading lines. Reading the whole file with `simplifile.read` and `string.split(_, "\n")` works too, and was just as fast on today's files (see Measurements).
- **Versions:** gleam_stdlib 1.0.5, gleam_json 3.1.0 (needs **Erlang/OTP 27+**), simplifile 2.7.0, file_streams 2.0.0, Gleam 1.18.1.

## Sources and versions

Versions are the latest that `gleam add` resolved on 2026-09-17. Claims about behaviour come from the package source in `build/packages/` for those versions.

| Package | Version | Where to look |
|---|---|---|
| gleam_stdlib | 1.0.5 | [`gleam/dynamic/decode`](https://hexdocs.pm/gleam_stdlib/gleam/dynamic/decode.html) |
| gleam_json | 3.1.0 | [`gleam/json`](https://hexdocs.pm/gleam_json/gleam/json.html), `src/gleam_json_ffi.erl` |
| simplifile | 2.7.0 | [`simplifile`](https://hexdocs.pm/simplifile/simplifile.html) |
| file_streams | 2.0.0 | [`file_streams/file_stream`](https://hexdocs.pm/file_streams/file_streams/file_stream.html) |
| Erlang/OTP kernel | 27 | [`file:read_line/1`](https://www.erlang.org/docs/27/apps/kernel/file) |

## Findings

### 1. Decoding by `type` in `gleam/dynamic/decode`

- **The `case` pattern is branching.** `decode.field(name, decoder, next)` is `subfield([name], ...)`. It decodes the field and then runs the decoder that `next(value)` returns ([decode.gleam, `subfield`](https://hexdocs.pm/gleam_stdlib/gleam/dynamic/decode.html#subfield)). So `use type_ <- decode.field("type", decode.string)` followed by `case type_ { ... }` really branches: each branch returns a different decoder for the same data.
- **Errors add up.** `subfield` joins the errors from the field decoder and from `next`. If `type` is missing, `next` still runs with a placeholder value (for `decode.string` that is `""`), but the result is still an `Error` that says `Field: Nothing ["type"]`. It does not quietly become `Skipped("")`. The sketch confirmed this.
- **`decode.then(decoder, next)`** does the same job without `use`: it runs `decoder`, then `next(value)`, and keeps the first errors ([`then`](https://hexdocs.pm/gleam_stdlib/gleam/dynamic/decode.html#then)). Use it when you want to peek at `type` with `decode.at(["type"], decode.string)` before calling another decoder.
- **Unknown types:** make the fallback branch `decode.success(Skipped(other))`. Use `decode.failure(placeholder, expected: "...")` only when an unknown value should count as an error.
- **`decode.one_of(first, or: [...])`** tries each decoder in order and returns the first success. If they all fail, it reports the errors from the *first* decoder ([`one_of`](https://hexdocs.pm/gleam_stdlib/gleam/dynamic/decode.html#one_of)). Use `collapse_errors` or `map_errors` if you want a clearer message.
- **Nested structures:** `decode.recursive(fn() { ... })` builds the inner decoder lazily, only when it runs ([`recursive`](https://hexdocs.pm/gleam_stdlib/gleam/dynamic/decode.html#recursive)). You need it when a decoder function would otherwise call itself as soon as it is created. In Claude Code files, `tool_result.content` is itself string-or-array.
- **Nulls:** `decode.optional(inner)` returns `None` for Erlang `null`, `nil` or `undefined` (`gleam_stdlib.erl` `is_null/1`). On OTP 27, `json:decode` turns JSON `null` into `null`. `decode.optional_field(name, default, inner)` gives `default` only when the key is missing, not when it is `null`. For fields like `"isMeta": null` (which occur in real files), combine the two: `optional_field("isMeta", False, optional(bool) |> map(unwrap(_, False)))`.
- **Opaque payloads:** `decode.dynamic` keeps a sub-tree (such as `tool_use.input`) as `Dynamic` without decoding it.

### 2. gleam_json on Erlang

- `json.parse(from: String, using: Decoder(t)) -> Result(t, json.DecodeError)`. On Erlang it turns the string into a `BitArray` and calls `parse_bits` ([json.gleam](https://hexdocs.pm/gleam_json/gleam/json.html#parse)).
- The FFI calls OTP's `json:decode/1`. The Erlang code has a compile-time check that fails with `erlang_otp_27_required` on older OTP releases. **The server needs OTP 27 or newer** (or gleam_json v1.0.1).
- The error type separates syntax errors from shape errors. Bad syntax gives `UnexpectedEndOfInput`, `UnexpectedByte(hex)` or `UnexpectedSequence`. A shape mismatch gives `UnableToDecode(List(decode.DecodeError))`, where each error includes a path.
- Every line is fully parsed into Erlang maps before any decoder runs. A `Skipped` record therefore costs a full JSON parse. Decoding only the fields you need saves building Gleam values, but not parsing.
- gleam_json 3.1.0 works on both targets, so the decoders can go in `shared/` next to the Session codecs (see ADR 0001). Only the file reading is Erlang-specific.

### 3. Reading files line by line on Erlang

- **simplifile 2.7.0** has `read` and `read_bits`, which load the whole file, but no line or stream API. I searched the source for `stream` and `line` and found no public functions.
- **file_streams 2.0.0**:
  - `file_stream.open_read(path)` opens with `[Read, ReadAhead(64 * 1024), Raw]`.
  - `file_stream.read_line(stream) -> Result(String, FileStreamError)` includes the trailing `\n` and returns `Error(Eof)` at the end of the file. In raw mode it calls `file:read_line/1` and converts the result as UTF-8, returning `Error(InvalidUnicode)` on bad bytes.
  - Also provides `close`. `open_read_text` (which takes an encoding) is slower, and its docs say to use `open_read` for UTF-8 lines.
  - `read_line` is "not supported on the JavaScript target". That's fine, since only `server/` reads files.
- **Erlang `file:read_line/1`** docs: "it is inefficient to use it on `raw` files if the file is not opened with option `{read_ahead, Size}` … combining `raw` and `{read_ahead, Size}` is highly recommended when opening a text file for raw line-oriented reading." `open_read` already does this. Lines longer than the read-ahead buffer still work; the buffer only affects speed.

### 4. Real Claude Code files

I checked the shapes against 15 local session files (6,423 lines, 14 MB) with `jq`:

- **Record types:** `assistant`, `user`, `attachment`, `system`, `mode`, `last-prompt`, `permission-mode`, `ai-title`, `agent-name`, `file-history-snapshot`, `queue-operation`, `pr-link` and others. There are more than 20 top-level types, so skipping unknown types is essential.
- **System subtypes:** `turn_duration`, `away_summary`, `local_command`, `informational`, `compact_boundary`, `bridge_status`.
- **`message.content`:** `user` records have 891 arrays and 95 strings. `assistant` records always have an array with exactly one block (`thinking`, `text` or `tool_use`).
- **`tool_result.content`:** 822 strings and 33 arrays.
- **Sizes and nulls:** the longest line was about 185 KB. Some `isMeta` values are `null`.

## Verified sketch

I compiled and ran this in a scratch project: Gleam 1.18.1, OTP 27.3.1, with the package versions above plus `argv`. On all 6,423 real lines it decoded 986 `User`, 1,606 `Assistant`, 109 `System` and 3,722 `Skipped` records with 0 errors. These counts match `jq`. A test file with a bad line gave one `UnexpectedByte`. An `assistant` record with missing fields gave one `UnableToDecode` listing the paths. Neither stopped the rest of the file.

```gleam
import file_streams/file_stream
import file_streams/file_stream_error
import gleam/dynamic/decode.{type Decoder}
import gleam/json
import gleam/option

pub type Block {
  Text(text: String)
  Thinking(thinking: String)
  ToolUse(id: String, name: String, input: decode.Dynamic)
  ToolResult(tool_use_id: String, content: List(Block), is_error: Bool)
  OtherBlock(type_: String)
}

pub type Record {
  User(uuid: String, timestamp: String, is_meta: Bool, content: List(Block))
  Assistant(uuid: String, timestamp: String, message_id: String, content: List(Block))
  System(subtype: String, timestamp: String)
  Skipped(type_: String)
}

/// `content` is either a bare string or an array of typed blocks.
fn content_decoder() -> Decoder(List(Block)) {
  decode.one_of(decode.string |> decode.map(fn(s) { [Text(s)] }), or: [
    decode.list(block_decoder()),
  ])
}

fn block_decoder() -> Decoder(Block) {
  use type_ <- decode.field("type", decode.string)
  case type_ {
    "text" -> {
      use text <- decode.field("text", decode.string)
      decode.success(Text(text))
    }
    "thinking" -> {
      use thinking <- decode.field("thinking", decode.string)
      decode.success(Thinking(thinking))
    }
    "tool_use" -> {
      use id <- decode.field("id", decode.string)
      use name <- decode.field("name", decode.string)
      use input <- decode.field("input", decode.dynamic)
      decode.success(ToolUse(id:, name:, input:))
    }
    "tool_result" -> {
      use tool_use_id <- decode.field("tool_use_id", decode.string)
      use content <- decode.optional_field(
        "content",
        [],
        decode.recursive(content_decoder),
      )
      use is_error <- decode.optional_field("is_error", False, nullable_bool())
      decode.success(ToolResult(tool_use_id:, content:, is_error:))
    }
    other -> decode.success(OtherBlock(other))
  }
}

pub fn record_decoder() -> Decoder(Record) {
  use type_ <- decode.field("type", decode.string)
  case type_ {
    "user" -> {
      use uuid <- decode.field("uuid", decode.string)
      use timestamp <- decode.field("timestamp", decode.string)
      use is_meta <- decode.optional_field("isMeta", False, nullable_bool())
      use content <- decode.subfield(["message", "content"], content_decoder())
      decode.success(User(uuid:, timestamp:, is_meta:, content:))
    }
    "assistant" -> {
      use uuid <- decode.field("uuid", decode.string)
      use timestamp <- decode.field("timestamp", decode.string)
      use message_id <- decode.subfield(["message", "id"], decode.string)
      use content <- decode.subfield(["message", "content"], content_decoder())
      decode.success(Assistant(uuid:, timestamp:, message_id:, content:))
    }
    "system" -> {
      use subtype <- decode.optional_field("subtype", "", decode.string)
      use timestamp <- decode.optional_field("timestamp", "", decode.string)
      decode.success(System(subtype:, timestamp:))
    }
    other -> decode.success(Skipped(other))
  }
}

/// Missing key -> default (optional_field); present-but-null -> default (optional).
fn nullable_bool() -> Decoder(Bool) {
  decode.optional(decode.bool) |> decode.map(option.unwrap(_, False))
}

/// Fold over the lines of a file, holding one line in memory at a time.
pub fn fold_lines(
  path: String,
  from acc: a,
  with f: fn(a, String) -> a,
) -> Result(a, file_stream_error.FileStreamError) {
  case file_stream.open_read(path) {
    Error(e) -> Error(e)
    Ok(stream) -> {
      let result = loop(stream, acc, f)
      let _ = file_stream.close(stream)
      result
    }
  }
}

fn loop(stream, acc, f) {
  case file_stream.read_line(stream) {
    Ok(line) -> loop(stream, f(acc, line), f)
    Error(file_stream_error.Eof) -> Ok(acc)
    Error(e) -> Error(e)
  }
}

// Per line: skip blank lines, then
//   json.parse(string.trim(line), record_decoder())
// and treat each Result on its own.
```

## Measurements

These are informal timings from one run with `gleam run`, so they include starting the BEAM. The input was all 15 files joined together (14 MB, 6,423 lines).

| Approach | Wall time |
|---|---|
| `file_stream.open_read` + `read_line` fold | 1.83 s |
| `simplifile.read` + `string.split("\n")` + fold | 1.87 s |

Speed is about the same, so choose based on memory. Streaming holds one line at a time, while `simplifile.read` holds the whole file plus the split list. Today's biggest session file is 3.3 MB, so either would work. Streaming only matters for much larger files, or for building a Session Entry cheaply by stopping after the first Prompt or `ai-title`, which `read_line` allows and `simplifile.read` does not.

## Implications for Replion

- Put the per-Harness record decoders in the Claude Code Adapter on the server. The Session JSON codecs stay in `shared/`.
- Pin OTP 27+ for `server/` (gleam_json 3 requires it).
- Decode `type` first, then branch with `case`, ending in a catch-all `Skipped` or `OtherBlock` branch. Turn `string | array` content into a list with `one_of`, and use `optional(...)` inside `optional_field` for fields that may be `null`.
- Read with `file_streams`. A bad line should be logged and skipped, never fail the whole Session.
- Assistant records hold one content block each, so the Adapter should group them by `message.id` (or simply turn each block into its own Event: Reasoning, Reply or Tool Call). The decoder does not need to handle this.

## Open questions

- Do other Harnesses (such as the GitHub Copilot CLI) use JSONL too? This research only looked at Claude Code files.
- Can the Session List be built fast enough by reading only the start of each file? That depends on where `ai-title` records appear in the file, which I didn't check here.
