Creating a focus experiment using PsychoPy with an EEG Muse headband is a fantastic way to explore attention, cognitive performance, or meditation states. The Muse headband is a consumer-grade EEG device that measures brain activity via four dry electrodes (two on the forehead, two behind the ears), sampling at 256 Hz. It’s portable, Bluetooth-enabled, and pairs with tools like PsychoPy for real-time data collection. Below, I’ll outline a step-by-step approach to design and implement a focus experiment, leveraging PsychoPy’s capabilities and the Muse’s EEG streaming.

---

### Experiment Concept: Measuring Focus During a Cognitive Task
Let’s design an experiment to assess focus by having participants perform a simple attention task (e.g., a Flanker task) while recording EEG data from the Muse headband. The Flanker task involves identifying the direction of a central arrow flanked by distracting arrows (congruent: →→→→→ or incongruent: ←←→←←). We’ll measure reaction times, accuracy, and EEG signals (e.g., alpha and beta waves) to correlate brain activity with focus levels.

#### Goals:
1. Quantify focus through task performance (reaction time and accuracy).
2. Analyze EEG data to identify brainwave patterns (e.g., beta for focus, alpha for relaxation) during high- and low-focus states.
3. Test how focus changes over time or under distraction.

---

### Step 1: Prerequisites
- **Hardware**: Muse headband (Muse 2 or Muse S), a computer with Bluetooth.
- **Software**: 
  - PsychoPy (latest version, installed via pip or standalone installer).
  - Python (3.6+ recommended for compatibility).
  - `muse-lsl` library (for streaming Muse EEG data).
  - LSL (Lab Streaming Layer) for syncing EEG with PsychoPy events.
- **Setup**: Ensure the Muse is charged, paired via Bluetooth, and the electrodes fit snugly on the participant’s head.

#### Install Dependencies
1. Install PsychoPy:
   ```bash
   pip install psychopy
   ```
2. Install `muse-lsl`:
   ```bash
   pip install muse-lsl
   ```
3. Verify LSL is installed (bundled with `pylsl`):
   ```bash
   pip install pylsl
   ```

---

### Step 2: Experiment Design in PsychoPy
PsychoPy’s Builder interface simplifies creating the Flanker task, while Python scripting integrates EEG streaming.

#### Task Structure
1. **Welcome Screen**: Instructions (e.g., “Press the left arrow key if the center arrow points left, right key if it points right”).
2. **Trials Loop**: 
   - 50 trials (25 congruent, 25 incongruent, randomized).
   - Stimulus: Text component displaying arrow strings (e.g., “→→→→→” or “←←→←←”).
   - Duration: 500 ms stimulus presentation, 1500 ms response window.
   - Record: Keypress (left/right) and reaction time.
3. **Feedback**: Correct/incorrect response shown briefly (optional).
4. **Markers**: Send event markers (e.g., “congruent_start”, “response”) to LSL for EEG synchronization.

#### Building in PsychoPy Builder
1. **New Experiment**: Open PsychoPy Builder, create a flow with:
   - Text component for instructions.
   - Loop around a trial routine.
2. **Trial Routine**:
   - Add a Text component for the arrow stimulus.
   - Add a Keyboard component to capture responses.
   - Add a Code component for LSL markers (see below).
3. **Conditions File**: Create a `.csv` file (e.g., `conditions.csv`) with columns:
   ```
   stimulus,condition
   "→→→→→","congruent"
   "←←→←←","incongruent"
   ...
   ```
   Link this to the Loop properties.

#### Adding LSL Markers
In the Code component:
- **Begin Experiment** tab:
  ```python
  from pylsl import StreamOutlet, StreamInfo
  info = StreamInfo('PsychoPyMarkers', 'Markers', 1, 0, 'string', 'myuid34234')
  outlet = StreamOutlet(info)
  ```
- **Begin Routine** tab:
  ```python
  outlet.push_sample([condition + "_start"])  # e.g., "congruent_start"
  ```
- **End Routine** tab:
  ```python
  outlet.push_sample(["response"])
  ```

---

### Step 3: Streaming EEG from Muse
The Muse headband streams EEG data via Bluetooth using the `muse-lsl` library, which integrates with LSL.

#### Start EEG Streaming
1. Create a separate Python script (`start_eeg_stream.py`) in your experiment folder:
   ```python
   from muselsl import stream, list_muses
   import time

   muses = list_muses()
   if not muses:
       print("No Muse device found. Ensure it’s on and paired.")
   else:
       print("Connecting to", muses[0]['name'])
       stream(muses[0]['address'])  # Streams EEG to LSL
       print("Streaming EEG data. Keep this running during the experiment.")
       while True:
           time.sleep(1)  # Keep script alive
   ```
2. Run this in a terminal before starting the PsychoPy experiment:
   ```bash
   python start_eeg_stream.py
   ```

#### Verify Streaming
- Use an LSL viewer (e.g., `muse-lsl`’s built-in viewer) to confirm EEG data is flowing:
  ```bash
  python -m muselsl.view
  ```

---

### Step 4: Running the Experiment
1. **Preparation**:
   - Fit the Muse headband on the participant (ensure good contact at AF7, AF8, TP9, TP10).
   - Start the EEG streaming script.
2. **Launch PsychoPy**:
   - Open your `.psyexp` file in PsychoPy Builder.
   - Click “Run” to start the experiment.
3. **Data Collection**:
   - PsychoPy logs reaction times and responses in a `.csv` file.
   - EEG data and markers are streamed via LSL.

---

### Step 5: Recording and Analyzing EEG Data
To save EEG data with markers, use an LSL recorder like LabRecorder or a Python script.

#### Simple Recording Script
Create `record_eeg.py`:
```python
from pylsl import StreamInlet, resolve_stream
import pandas as pd
import time

# Resolve EEG and marker streams
print("Looking for streams...")
eeg_streams = resolve_stream('type=EEG')
marker_streams = resolve_stream('type=Markers')

# Create inlets
eeg_inlet = StreamInlet(eeg_streams[0])
marker_inlet = StreamInlet(marker_streams[0])

# Data storage
eeg_data = []
markers = []

# Record for 5 minutes (adjust as needed)
start_time = time.time()
while time.time() - start_time < 300:
    # EEG data
    sample, timestamp = eeg_inlet.pull_sample()
    eeg_data.append([timestamp] + sample)
    # Markers
    marker, marker_time = marker_inlet.pull_sample(timeout=0.0)
    if marker:
        markers.append([marker_time, marker[0]])

# Save to CSV
eeg_df = pd.DataFrame(eeg_data, columns=['timestamp', 'TP9', 'AF7', 'AF8', 'TP10'])
marker_df = pd.DataFrame(markers, columns=['timestamp', 'marker'])
eeg_df.to_csv('eeg_data.csv', index=False)
marker_df.to_csv('markers.csv', index=False)
print("Data saved.")
```

Run this during the experiment:
```bash
python record_eeg.py
```

#### Analysis Ideas
1. **Behavioral**: Compare reaction times and accuracy between congruent and incongruent trials.
2. **EEG**:
   - Use Python libraries like `mne` to process EEG data.
   - Compute power spectral density (PSD) for alpha (8-13 Hz) and beta (13-30 Hz) bands.
   - Align EEG with markers to compare brain activity during focused (fast responses) vs. unfocused (slow/errors) trials.

---

### Step 6: Tips and Troubleshooting
- **Timing**: Muse’s Bluetooth latency is ~50 ms; ensure PsychoPy’s timing is synced (use `core.getTime()` for precision).
- **Signal Quality**: Check Muse fit; noisy data often comes from loose electrodes.
- **Extensions**: Add a baseline (resting state) or manipulate difficulty (e.g., add auditory distractors).

---

This setup gives you a solid foundation for a focus experiment. You’ll get behavioral data from PsychoPy and EEG insights from the Muse, letting you explore how brain activity reflects attention. If you want to tweak the task, dive deeper into analysis, or need help with a specific part, let me know—I can refine it further!
