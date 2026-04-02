# Velbus-aio: Channel naming

Documentation on how channel names are determined in velbus-aio.

See also [module_spec_json.md](module_spec_json.md) for an overview of all JSON spec fields.

---

## Overview

There are two ways channel names are determined:

1. **Live connection** — names are retrieved via Velbus bus messages
2. **VLP import** — names are read from the VLP file via memory addresses

---

## 1. Live connection

### Step 1 — Default name from JSON spec

When channels are created (`_load_default_channels()` in `module.py`), each channel receives the default name from the module spec JSON:

```json
// module_spec/22.json (VMB7IN)
"Channels": {
  "01": { "Name": "Input 1", "Type": "ButtonCounter", "Editable": "yes", "Subdevice": "no" },
  "02": { "Name": "Input 2", ... }
}
```

This name is only a fallback and will be overwritten once the real name is received.

### Step 2 — Requesting names via messages

After creating the channels, `_request_channel_name()` sends a `ChannelNameRequestMessage` (0xEF) to the module.

The module responds per channel in 3 consecutive messages of 6 characters each:

| Message          | Command | Content     |
|------------------|---------|-------------|
| ChannelNamePart1 | 0xF0    | Chars 1-6   |
| ChannelNamePart2 | 0xF1    | Chars 7-12  |
| ChannelNamePart3 | 0xF2    | Chars 13-18 |

The 3 parts are assembled via `channel.set_name_part(part, name)`. Once part 3 is received, the channel is marked as loaded (`set_loaded(True)`).

The names received this way are the names configured in **Velbuslink** and stored in the module memory. Channels that have not been renamed return their default name (e.g. `Push button 4`).

### Cache

After loading, names are stored in `.storage/velbuscache-{entry_id}/{address}.json`:

```json
{
  "name": "VMB7IN name",
  "channels": {
    "1": {"name": "GSC teller", "loaded": true},
    "2": {"name": "Water", "loaded": true},
    "6": {"name": "TrekSchakelaar", "loaded": true}
  }
}
```

On the next startup, names are loaded from cache — the `ChannelNameRequestMessage` messages are no longer sent.

---

## 2. VLP import

When importing a `.vlp` file, `vlp_reader.py` reads the channel names directly from the binary memory of the VLP file.

The JSON spec contains memory address ranges per channel under `Memory.Channels`:

```json
// module_spec/22.json (VMB7IN)
"Memory": {
  "Channels": {
    "01": "0000-000F",
    "02": "0010-001F"
  }
}
```

`_get_channel_name()` in `vlp_reader.py`:
1. Look up the address range in `Memory.Channels`
2. Read the bytes at those addresses from the VLP memory block
3. Remove `FF` bytes (padding)
4. Decode as ASCII string

---

## Comparison

| Aspect            | Live connection                         | VLP import                                |
|-------------------|-----------------------------------------|-------------------------------------------|
| Source            | Module memory via Velbus bus            | VLP file (offline)                        |
| Messages          | 0xEF (request) + 0xF0/F1/F2 (response) | None — directly from binary memory        |
| Used by           | `module.py` → `_request_channel_name`  | `vlp_reader.py` → `_get_channel_name`    |
| `Memory.Channels` | **Not used**                            | **Used** (address ranges)                 |
| Cache             | Yes, after first load                   | No                                        |

---

## Relevant files

| File                 | Function                                              |
|----------------------|-------------------------------------------------------|
| `module.py`          | `_load_default_channels()`, `_request_channel_name()` |
| `vlp_reader.py`      | `_get_channel_name()`                                 |
| `module_spec/*.json` | Default names, `Memory.Channels` address ranges       |
| `channels.py`        | `set_name_part()`, `set_loaded()`                     |
