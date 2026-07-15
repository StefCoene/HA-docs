# Velbus-aio: Module spec JSON fields

Documentation of all fields in the module spec JSON files (`module_spec/{type_hex}.json`) and what they are used for.

---

### `Type`

**Value:** string, e.g. `"VMB7IN"`  
**Used in:** `module.py` → `get_type_name()`  
**Purpose:** Returns the human-readable name of the module type for UI and logging.

---

### `Channels`

Object with per channel (key = numbered `"01"`, `"02"`, ...):

#### `Channels.*.Name`
**Value:** string, e.g. `"Input 1"`  
**Used in:** `module.py` → `_load_default_channels()`  
**Purpose:** Default channel name, overwritten by the name from module memory or VLP.

#### `Channels.*.Type`
**Value:** string — one of: `Button`, `ButtonCounter`, `Relay`, `Dimmer`, `Blind`, `Temperature`, `ThermostatChannel`, `Sensor`, `EdgeLit`  
**Used in:**
- `module.py` → `_load_default_channels()` — instantiates the correct channel class
- `module.py` → `_request_module_status()` — determines which status requests to send (Blind/Dimmer/Relay → ModuleStatusRequest, ButtonCounter → CounterStatusRequest)

**Purpose:** Dynamic channel class selection and status request logic.

#### `Channels.*.Editable`
**Value:** `"yes"` / `"no"`  
**Used in:** `module.py` → `_load_default_channels()` — sets the `nameEditable` flag on the channel  
**Purpose:** Indicates whether the channel name can be retrieved from module memory via the protocol. When `"no"`, the default name is kept.

#### `Channels.*.Subdevice`
**Value:** `"yes"` / `"no"`  
**Used in:** `module.py` → `_load_default_channels()` — sets the `subDevice` flag on the channel  
**Purpose:** Determines in HA whether a channel gets a separate sub-device (`yes`) or falls under the main device (`no`).

---

### `Memory`

#### `Memory.ModuleName`
**Value:** hex address ranges, e.g. `"03AC-03EB"` or `"062C-066B;03C0-03FF"`  
**Used in:**
- `module.py` → `__load_memory()` — sends `ReadDataBlockFromMemoryMessage` (0xEC) for each 4-byte block
- `module.py` → `_process_memory_data_block_message()` — assembles the module name from received bytes (0xFF bytes are skipped)

**Purpose:** Location in module memory where the module name is stored.

#### `Memory.Channels`
**Value:** hex address ranges per channel, e.g. `{ "01": "0000-000F", "02": "0010-001F" }`  
**Used in:** `vlp_reader.py` → `_get_channel_name()` — reads channel name from VLP memory  
**Purpose:** Location in module memory where channel names are stored. Only used during VLP import, not during a live bus scan. During a live connection, channel names are retrieved via bus messages: `ChannelNamePart1` (0xF0), `ChannelNamePart2` (0xF1) and `ChannelNamePart3` (0xF2).

#### `Memory.Address`
**Value:** object with hex addresses as keys and `Match` patterns as values  
**Example:**
```json
"03FE": {
  "Match": {
    "1": { "%......01": { "Channel": "01", "SubName": "Unit", "Value": "liter" } }
  }
}
```
**Used in:**
- `module.py` → `__load_memory()` — sends `ReadDataFromMemoryMessage` (0xED) for each address
- `module.py` → `_process_memory_data_message()` — matches bits against patterns and assigns values to channels

**Purpose:** Read module-specific configuration via binary pattern matching (e.g. counter unit for VMB7IN).

---

### `Properties`

**Value:** object with property name as key and `Type` + `Name` as fields  
**Example:** `{ "bus_error_rx": { "Type": "BusErrorRx", "Name": "bus_error_rx" } }`  
**Used in:** `module.py` → `_load_properties()` — dynamically instantiates the correct property class from `properties.py`  
**Purpose:** Module-level properties alongside channels, e.g. selected program, bus error counter, light value.

---

### `AllChannelStatus`

**Value:** hex string, e.g. `"FF"`  
**Used in:**
- `module.py` → `_request_module_status()` — if present: requests status for all channels in one message
- `module.py` → `_request_channel_name()` — if present: requests all channel names at once (`0xFF`)

**Purpose:** Optimisation for modules with many channels — one bulk request instead of per channel.

---

### `Thermostat`

**Value:** `"yes"`  
**Used in:** `module.py` → `_load_default_channels()` — activates thermostat logic on the temperature channel  
**Purpose:** Marks the module as a thermostat.

---

### `ThermostatAddr`

**Value:** integer (0 = no thermostat)  
**Used in:** `module.py` → `_load_default_channels()` — alternative to `Thermostat`, checks whether value != 0  
**Purpose:** Thermostat identification for modules with sub-addresses.

---

### `TemperatureChannel`

**Value:** channel number as string, e.g. `"10"`  
**Used in:** `module.py` → `_load_default_channels()` — sets `thermostat: True` flag on this channel  
**Purpose:** Indicates which channel contains the temperature sensor in thermostat modules.

---

### `ChannelNumbers`

**Value:** mapping object, e.g. `{ "Name": { "Map": { "03": "01", "09": "10" } } }`  
**Used in:** `module.py` → `_translate_channel_name()` — maps protocol channel numbers to software channel numbers  
**Purpose:** Correction for modules with non-linear channel numbering in the protocol.

---

### `sliderScale`

**Value:** integer, e.g. `254`  
**Used in:** `module.py` → `_load_default_channels()` — sets `slider_scale` on Dimmer channel instances  
**Purpose:** Hardware-specific scale value for dimmer channels (0-254 instead of 0-255).

---

## Relevant files

| File                 | Function                                                                                         |
|----------------------|--------------------------------------------------------------------------------------------------|
| `module.py`          | `_load_default_channels()`, `__load_memory()`, `_load_properties()`, `_translate_channel_name()` |
| `vlp_reader.py`      | `_get_channel_name()`                                                                            |
| `channels.py`        | Channel classes                                                                                  |
| `properties.py`      | Property classes (BusErrorRx, SelectedProgram, ...)                                              |
| `module_spec/*.json` | Per-module configuration                                                                         |
