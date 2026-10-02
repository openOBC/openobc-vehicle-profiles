# openobc-vehicle-profiles

Source of truth for OpenOBC vehicle compatibility. The OpenOBC phone app fetches
`catalog.json` from this repository at runtime (anonymously, via
`raw.githubusercontent.com`) to show which vehicles are supported and let the
user select the profile matching their car. The selected profile is then written
to the OpenOBC device over BLE.

## Files

- `catalog.json` — the catalog the app fetches.
- `schema/catalog.schema.json` — JSON Schema for the catalog; CI validates
  every change against it, so a structurally invalid catalog cannot land on
  `main`.

## Catalog format `openobc-vehicle-catalog@1`

Top level:

| Field | Description |
|---|---|
| `format` | Must be exactly `openobc-vehicle-catalog@1`. Consumers reject unknown versions. |
| `profiles` | Array of profile entries (at least one). |

Each profile entry:

| Field | Type | Description |
|---|---|---|
| `id` | string | Stable identifier used by the app (kebab-case). |
| `profileId` | 0–254 | Numeric identifier understood by the firmware's vehicle profile registry. **This is the value sent to the device.** `255` is reserved. |
| `manufacturer` | string | e.g. `nissan`. |
| `model` | string | e.g. `tiida`. |
| `variant` | string | Trim identifier, snake_case (e.g. `abs`, `non_abs`). |
| `displayName` | string | Human-readable name shown in the picker. |
| `description` | string | One-line description shown under the name. |
| `minFw` | semver | Minimum firmware version able to activate this profile. Entries with `minFw` above the connected device's firmware version are shown as disabled in the app. |
| `supportedFeatures` | array | Feature chips shown in the picker: objects `{ "id": snake_case, "name": string }`. |

### How `profileId` relates to the firmware

The firmware ships a **vehicle profile registry** that maps each numeric
`profileId` to a compiled-in decoder (see
`src/vehicle_profiles/registry.h` in the firmware repo). The catalog entry's
`profileId` **must match** the firmware registry id exactly — the app sends this
number to the device via the `vehicleProfile` app-config field, and the device
rejects ids it does not know.

Adding a vehicle whose decode logic already exists in a firmware release
requires only a new catalog entry here (with `minFw` set to the first firmware
release that includes the corresponding registry id). Adding decode logic for a
brand-new vehicle requires a firmware change; update this catalog in the same
release so both stay in lockstep.

## Authoring checklist

1. Add the entry to `catalog.json` — follow the field table above.
2. Ensure `profileId` matches the firmware registry and no other entry uses it.
3. Set `minFw` to the first firmware release that knows this `profileId`.
4. CI must pass (schema validation). Verify locally with:
   ```sh
   pip install jsonschema
   python -c "import json, jsonschema; jsonschema.validate(json.load(open('catalog.json')), json.load(open('schema/catalog.schema.json')))"
   ```
5. Anonymous fetch check after merge:
   `https://raw.githubusercontent.com/openOBC/openobc-vehicle-profiles/main/catalog.json`
