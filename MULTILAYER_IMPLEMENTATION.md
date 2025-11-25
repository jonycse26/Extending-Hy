# Multi-Layer Satellite Constellation Implementation

This document describes the implementation of multi-layer satellite constellation support (LEO + MEO) in Hypatia.

## Overview

The implementation extends Hypatia to support multi-layered satellite constellations with:
- **LEO (Low Earth Orbit) shell**: Handles ground station connections and local routing
- **MEO (Medium Earth Orbit) shell**: Acts as backhaul to relieve LEO ISLs for long-distance or multi-hop paths

## Key Components

### 1. Cross-Layer ISL Generation (`satgenpy/satgen/isls/generate_cross_layer_isls.py`)

Generates inter-satellite links between LEO and MEO satellites. The function `generate_cross_layer_isls_simple()` creates connections based on orbital geometry, connecting each LEO satellite to the nearest MEO satellite(s).

**Usage:**
```python
from satgen.isls import generate_cross_layer_isls_simple

generate_cross_layer_isls_simple(
    output_filename_isls,
    leo_num_orbits,
    leo_num_sats_per_orbit,
    leo_idx_offset,
    meo_num_orbits,
    meo_num_sats_per_orbit,
    meo_idx_offset,
    cross_layer_connections_per_leo=1
)
```

### 2. Multi-Layer Routing Algorithm (`satgenpy/satgen/dynamic_state/algorithm_multilayer_meo_backhaul.py`)

Implements routing for multi-layer constellations with the following behavior:
- **Ground stations** connect only to LEO satellites
- **LEO satellites** can forward traffic via MEO satellites when:
  - Distance > `meo_backhaul_distance_threshold_m` (default: 10,000 km), OR
  - Path requires > `meo_backhaul_hop_threshold` satellite hops (default: 3)
- **MEO satellites** act as backhaul to relieve LEO ISLs

**Algorithm Name Format:**
```
algorithm_multilayer_meo_backhaul:leo_num:meo_num:distance_threshold:hop_threshold
```

Example:
```
algorithm_multilayer_meo_backhaul:1584:48:10000000:3
```
- 1584 LEO satellites
- 48 MEO satellites
- 10,000,000 m (10,000 km) distance threshold
- 3 hop threshold

### 3. Multi-Layer Helper (`paper/satellite_networks_state/main_helper_multilayer.py`)

Extended helper class that supports generating multi-layer constellations with separate LEO and MEO shells.

**Key Features:**
- Generates TLEs for both LEO and MEO shells
- Combines TLEs into a single file with proper indexing
- Generates ISLs within each shell (LEO-LEO, MEO-MEO)
- Optionally generates cross-layer ISLs (LEO-MEO)
- Supports different ISL configurations:
  - `isls_plus_grid`: Only intra-shell ISLs
  - `isls_plus_grid_with_cross_layer`: Intra-shell + cross-layer ISLs
  - `isls_none`: No ISLs

### 4. Example Constellation (`paper/satellite_networks_state/main_multilayer_example.py`)

Example multi-layer constellation configuration:
- **LEO shell**: 72 orbits × 22 satellites = 1,584 satellites at ~550 km altitude
- **MEO shell**: 6 orbits × 8 satellites = 48 satellites at ~20,000 km altitude
- **Cross-layer ISLs**: Enabled by default

**Usage:**
```bash
python main_multilayer_example.py [duration (s)] [time step (ms)] \
    [isls_plus_grid_with_cross_layer] [ground_stations_top_100] \
    [algorithm_multilayer_meo_backhaul:1584:48:10000000:3] [num threads]
```

## Evaluation Experiments

Three evaluation experiments are provided in `paper/ns3_experiments/multilayer_evaluation/`:

### Experiment 1: Distance-based MEO Backhaul Usage
Evaluates how often MEO backhaul is used with different distance thresholds (5,000 km, 10,000 km, 15,000 km).

### Experiment 2: Hop-based MEO Backhaul Usage
Evaluates how often MEO backhaul is used with different hop count thresholds (2, 3, 4 hops).

### Experiment 3: Multi-layer vs Single-layer Comparison
Compares multi-layer constellation performance with single-layer LEO constellation.

## Implementation Details

### Satellite Indexing

Satellites are indexed sequentially:
- LEO satellites: 0 to `leo_num_satellites - 1`
- MEO satellites: `leo_num_satellites` to `leo_num_satellites + meo_num_satellites - 1`

### Routing Logic

The routing algorithm:
1. Calculates shortest paths in the combined LEO+MEO network
2. For each source-destination pair, evaluates:
   - Direct LEO path (if source is LEO)
   - Path via MEO backhaul (if thresholds are exceeded)
3. Selects the best path based on distance and hop count
4. Uses MEO backhaul when:
   - Direct LEO path distance > distance threshold, OR
   - Direct LEO path hops > hop threshold

### ISL Generation

ISLs are generated in three stages:
1. **LEO-LEO ISLs**: Plus-grid pattern within LEO shell
2. **MEO-MEO ISLs**: Plus-grid pattern within MEO shell
3. **LEO-MEO ISLs**: Cross-layer connections (optional)

## Files Modified/Created

### New Files:
- `satgenpy/satgen/isls/generate_cross_layer_isls.py`
- `satgenpy/satgen/dynamic_state/algorithm_multilayer_meo_backhaul.py`
- `paper/satellite_networks_state/main_helper_multilayer.py`
- `paper/satellite_networks_state/main_multilayer_example.py`
- `paper/ns3_experiments/multilayer_evaluation/experiment_1_distance_threshold/run_experiment.py`
- `paper/ns3_experiments/multilayer_evaluation/experiment_2_hop_threshold/run_experiment.py`
- `paper/ns3_experiments/multilayer_evaluation/experiment_3_comparison/run_experiment.py`

### Modified Files:
- `satgenpy/satgen/isls/__init__.py` - Added cross-layer ISL generation export
- `satgenpy/satgen/dynamic_state/generate_dynamic_state.py` - Added multi-layer algorithm support

## Testing

To test the implementation:

1. **Generate constellation state:**
   ```bash
   cd paper/satellite_networks_state
   python main_multilayer_example.py 100 1000 isls_plus_grid_with_cross_layer \
       ground_stations_top_100 algorithm_multilayer_meo_backhaul:1584:48:10000000:3 4
   ```

2. **Run evaluation experiments:**
   ```bash
   cd paper/ns3_experiments/multilayer_evaluation/experiment_1_distance_threshold
   python run_experiment.py
   ```

## Future Work

Potential extensions:
- Support for GEO (Geostationary Orbit) satellites
- More sophisticated cross-layer ISL generation based on actual distances
- Dynamic routing that adapts thresholds based on network conditions
- Integration with trace-driven emulation workflows
- Performance optimizations for large-scale constellations

## References

- Hypatia paper: "Exploring the 'Internet from space' with Hypatia" (IMC 2020)
- Multi-layer satellite systems (MLSS) research
- IRIS2 constellation specifications

