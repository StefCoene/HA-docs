# Home Assistant: entity & device identity

Documentation on the identity concepts Home Assistant uses for the Velbus
integration — `unique_id`, `entity_id`, `serial`, device/entity registries — and
who sets each value, when, and who can overwrite it.

Background for the property `unique_id` migration in HA core
([PR #176343](https://github.com/home-assistant/core/pull/176343)).

See also [channel_naming.md](channel_naming.md) for how channel names are resolved.

---

## The two registries

Home Assistant persists two central registries in `.storage/`:

| Registry            | Holds                                   | Velbus example                          |
|---------------------|-----------------------------------------|-----------------------------------------|
| **Device Registry** | Physical/logical devices                | A module, or a sub-device such as a button |
| **Entity Registry** | Individual entities (usable things)     | A sensor, a switch, a select            |

Each entity usually belongs to one device. The migration rewrites values in the
**entity** registry, but previously also read from the **device** registry — which
was the reviewer's concern.

---

## Key concepts

### `unique_id` (entity)

- **Purpose:** the *permanent, internal* identity of an entity. HA uses it to
  recognise an entity across restarts. **Not** visible in the UI.
- **Who sets it:** the **integration**, in code (Velbus `entity.py`):
  ```python
  self._attr_unique_id = f"{serial}-{channel.get_channel_number()}"   # or -{property_key}
  ```
- **When:** once, on first creation of the entity. Afterwards it is **frozen** in the
  registry.
- **Why it matters:** if the code later changes the format (this PR: `-0` →
  `-SelectedProgram`), the new code value no longer matches what is stored → HA sees a
  **new** entity → the old one is orphaned and loses its history, name, area, etc.
  **This is why a migration exists**: it rewrites the stored `unique_id` so it matches
  again.

### `entity_id` (e.g. `sensor.living_room_temperature`)

- **Purpose:** the *visible, addressable* name used in the UI, automations and YAML.
- **Who sets it:** HA generates it from the name, but **the user can rename it**.
- **Difference from `unique_id`:** `entity_id` is for humans and may change; `unique_id`
  is for the machine and stays fixed. HA links them through the registry.

### `serial` / `serial_number` (device)

- **Purpose:** a unique serial for the physical device. Velbus derives it from the
  module address/memory.
- **Two forms that collide in this PR:**
  1. **Inside the `unique_id`** — the serial the integration embedded in `{serial}-...`
     at creation time. Frozen, reliable.
  2. **`device.serial_number` in the Device Registry** — a field on the device object.
- **Who can overwrite it:** a device may be managed by **multiple integrations** at once
  (they share a device via matching identifiers). Another integration can then
  **overwrite** `device.serial_number`. A migration built on that could produce a *wrong*
  serial in the new `unique_id` → history lost anyway. Parsing the serial **from the
  `unique_id` itself** uses the frozen value that nothing else can touch.

### `original_name` vs `name` (entity)

- **`original_name`** — the name the **integration** assigned at creation (for a property
  e.g. `"Light value"`). Fixed.
- **`name`** — the override the **user** optionally set. `None` if the user changed
  nothing.
- The migration filters on `original_name` ("is this a property entity?") precisely
  because it is the *integration* name, not something the user happened to type.

### `identifiers` and `via_device_id` (device)

- **`identifiers`** — the set of `(domain, id)` pairs an integration uses to claim "its"
  device, e.g. `("velbus", "1")`. Domain-scoped, so another integration cannot hijack
  Velbus's own identifier.
- **`via_device_id`** — points to a "parent" device, telling HA that a button is a
  *sub-device* of a module. The old migration used this to skip sub-devices — no longer
  needed now the format itself is the discriminator.

---

## How Velbus assembles the pieces

Regular channel:
```
unique_id = f"{serial}-{channel_number}"      # e.g. "qwerty1234567-3"   (channel is always >= 1)
```
Property (module-wide, always channel 0):
```
unique_id = f"{serial}-{property_key}"        # e.g. "qwerty1234567-SelectedProgram"
```
Legacy program select (was a channel before velbus-aio #160):
```
unique_id = f"{serial}-{channel_number}-program_select"   # e.g. "qwerty1234567-5-program_select"
```

Because real channels are **always >= 1**, a `-0` suffix and the `-program_select` suffix
belong exclusively to properties → safe to recognise without the device registry.

---

## Why the migration is needed (upgrade scenario)

A user moving from the old to the new integration version:

| Moment                          | Entity registry holds                     | Code generates       | Result                                   |
|---------------------------------|-------------------------------------------|----------------------|------------------------------------------|
| Old version                     | `qwerty-0` (light value sensor)           | `qwerty-0`           | match → all good                         |
| New version, **without** migration | `qwerty-0`                             | `qwerty-LightValue`  | **mismatch** → old orphaned, new empty   |
| New version, **with** migration | `qwerty-0` → rewritten to `qwerty-LightValue` | `qwerty-LightValue` | match → history/name/area preserved      |

The migration deliberately runs **before** the bus scan, so the registry names are already
correct when the entities are rebuilt.

---

## The essence in one sentence

> A `unique_id` is a **promise you must not break**. Change how the code builds it, and a
> migration must carry the *old stored* values over to the new format — otherwise HA
> "forgets" every entity. And because the `unique_id` is the one value the integration fully
> controls and freezes, parsing the serial *from it* is more reliable than reading a shared
> device field.

---

## Relevant files

| File / location                          | Role                                                        |
|------------------------------------------|-------------------------------------------------------------|
| HA core `velbus/entity.py`               | Builds `unique_id` from `serial` + channel/property key     |
| HA core `velbus/__init__.py`             | `_migrate_property_unique_ids()` — rewrites stored ids       |
| HA `.storage/core.entity_registry`       | Stored `unique_id`, `entity_id`, `original_name`, `name`     |
| HA `.storage/core.device_registry`       | Stored `serial_number`, `identifiers`, `via_device_id`       |
| velbus-aio `properties.py` / `channels.py` | `get_channel_number()`, `get_property_key()`, `get_name()` |
