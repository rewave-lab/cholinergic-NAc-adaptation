# Cholinergic NAc adaptation — processed spike data

This repository shares processed spike times rather than the original raw `.dat` recordings. The files documented below are sufficient to inspect the single-unit spike trains, locate the optotagged cells, and reproduce the laser-aligned raster plots.

## Data files

### Spike-time text files

The canonical spike-time files are stored in:

```text
data/LDT/txt/<recording>_time_cluster_shank_sua.txt
```

Each text file contains three rows:

1. spike time in seconds;
2. unit/cluster ID;
3. shank ID.

Each column therefore represents one spike. Load and transpose the file to obtain one spike per table row:

```python
from pathlib import Path

import numpy as np
import pandas as pd

spike_file = Path("data/LDT/txt/LDT_VEH31_time_cluster_shank_sua.txt")
spike_matrix = np.loadtxt(spike_file)

if spike_matrix.shape[0] != 3:
    raise ValueError(f"Expected three rows in {spike_file}, found {spike_matrix.shape}")

spikes = pd.DataFrame(
    spike_matrix.T,
    columns=["time_s", "unit_id", "shank_id"],
)
spikes[["unit_id", "shank_id"]] = spikes[["unit_id", "shank_id"]].astype(int)

print(spikes.head())
print("Units:", sorted(spikes["unit_id"].unique()))
```

To inspect one unit:

```python
import matplotlib.pyplot as plt

unit_id = 215
unit_spikes = spikes.loc[spikes["unit_id"] == unit_id, "time_s"]

fig, ax = plt.subplots(figsize=(10, 2))
ax.eventplot(unit_spikes, lineoffsets=1, linelengths=0.8, color="black")
ax.set(xlabel="Time (s)", yticks=[], title=f"Unit {unit_id}")
plt.tight_layout()
plt.show()
```

## Locating optotagged cells

The cell identities used by the analyses are stored in:

```text
data/LDT_tagged/LDT_tagged.json
```

The JSON has two cohort dictionaries:

- `V_dados`: vehicle-treated animals;
- `C_dados`: cocaine-treated animals.

Within each cohort, cells are grouped using the original internal names:

| JSON name | Display name |
|---|---|
| `CHAT` | ChAT |
| `Vglut` | VGLuT |
| `Vgat` | cGAT |

Each population contains recording names and their tagged unit IDs. For example:

```json
{
  "V_dados": {
    "CHAT": {
      "LDT_VEH31": [213, 215, 226, 171, 178]
    }
  }
}
```

Use the recording name and unit ID together to retrieve the cell identity:

```python
import json
from pathlib import Path

metadata_file = Path("data/LDT_tagged/LDT_tagged.json")
with metadata_file.open(encoding="utf-8") as stream:
    tagged_cells = json.load(stream)

display_names = {"CHAT": "ChAT", "Vglut": "VGLuT", "Vgat": "cGAT"}


def find_tag(recording, unit_id):
    matches = []
    for cohort, populations in tagged_cells.items():
        for population, recordings in populations.items():
            if int(unit_id) in recordings.get(recording, []):
                matches.append(
                    {
                        "cohort": cohort,
                        "population": display_names.get(population, population),
                    }
                )
    return matches


print(find_tag("LDT_VEH31", 215))  # ChAT
print(find_tag("LDT_VEH42", 95))   # VGLuT
print(find_tag("LDT_VEH56", 142))  # cGAT
```

The JSON provides the curated cell identities. The TXT file provides all spikes belonging to the corresponding `unit_id`; it does not contain the biological identity itself.

## Optional: laser-aligned response

Laser-onset sample indices are stored in:

```text
OPT/laser/<recording>_laseron.npy
```

They are required only to reproduce the peri-laser raster and firing-rate plots. At the acquisition rate used here, convert the onset samples to seconds with:

```python
sampling_rate_hz = 30_000
laser_onsets_s = np.load("OPT/laser/LDT_VEH31_laseron.npy") / sampling_rate_hz
```

The complete plotting example is available in [`notebook/plot_tagcell.ipynb`](notebook/plot_tagcell.ipynb).

Three recordings use historical `COCA` names for the laser files:

| JSON/TXT recording | Laser file prefix |
|---|---|
| `LDT_COC31` | `LDT_COCA31` |
| `LDT_COC53` | `LDT_COCA53` |
| `LDT_COC54` | `LDT_COCA54` |

