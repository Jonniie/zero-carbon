# JéGO Zero Carbon — Setup Instructions

## Prerequisites

- Python 3.10 or higher
- `pip` package manager

## Installation

1. **Clone or navigate to the project directory:**
   ```bash
   cd /zero-carbon
   ```

2. **Create a virtual environment:**
   ```bash
   python -m venv jego-intern-env
   ```

3. **Activate the environment:**
   ```bash
   # On Linux/macOS
   source jego-intern-env/bin/activate

   # On Windows
   jego-intern-env\Scripts\activate
   ```

4. **Install dependencies:**
   ```bash
   pip install -r requirements.txt
   ```

## Data File

Ensure the telematics dataset is present in the project root:

```
jego telematics mock dataset.json
```

This file contains vehicle telemetry records including GPS coordinates, speed, battery state of charge (SoC), and trip events.

## Running the Notebook

1. **Start JupyterLab:**
   ```bash
   jupyter lab
   ```

2. **Open `script.ipynb`** in the Jupyter interface.

3. **Run all cells** (Cell → Run All) to execute the full analysis pipeline.

## Output

- **Map visualization**: `jego_fleet_corridors.html` — interactive Folium map showing vehicle routes
- **Analysis results**: Displayed inline in the notebook (trip summaries, carbon metrics, efficiency scores)

## Dependencies

| Package | Purpose |
|---------|---------|
| pandas | Data processing |
| numpy | Numerical operations |
| folium | Interactive mapping |
| osmnx | OpenStreetNet integration |
| jupyterlab | Notebook environment |
| scikit-learn | Anomaly detection (Part 4) |
