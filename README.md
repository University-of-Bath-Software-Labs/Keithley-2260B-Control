# Keithley 2260B Control

A LabVIEW application for controlling a Keithley 2260B programmable DC power supply. The application configures the instrument, repeatedly enables and disables its output using user-defined on/off periods, continuously reads the actual voltage and current, and optionally records timestamped measurements to CSV.

## Features

- Connects to a Keithley 2260B through a selectable VISA resource
- Supports configurable serial communication settings
- Configures the output voltage setpoint and current limit
- Cycles the power-supply output on and off until the user stops the program
- Supports independent on time, off time, and measurement interval settings
- Displays live output state, voltage, and current
- Optionally records timestamped voltage and current measurements to CSV
- Automatically creates a `Results` folder if one does not exist
- Automatically names each results file using the exact timestamp at which **Start** is pressed
- Disables controls according to the current stage of operation to prevent settings changing during a test
- Safely disables the power-supply output before closing the instrument session

## Application Workflow

The front panel guides the user through a fixed sequence:

1. Configure the instrument connection and test settings.
2. Press **Confirm Inputs** to initialise and configure the Keithley 2260B.
3. Press **Start** to begin output cycling, measurement acquisition, and optional recording.
4. Press **Stop** to disable the output and close the application safely.

### Initial State

When the application opens:

- Instrument and test settings are enabled.
- **Confirm Inputs** is enabled.
- **Start** and **Stop** are disabled.
- Live status indicators are inactive.

### Configured State

After **Confirm Inputs** is pressed:

- The Keithley connection is initialised.
- The voltage setpoint and current limit are configured.
- The power-supply output remains disabled.
- Instrument and test settings are disabled to prevent changes.
- **Confirm Inputs** is disabled.
- **Start** becomes the only available action.

### Running State

After **Start** is pressed:

- The output is enabled immediately.
- The output remains enabled for **On Time (s)**.
- The output is disabled for **Off Time (s)**.
- The on/off cycle repeats until **Stop** is pressed.
- Actual voltage and current are read at the configured **Measurement Interval (s)**.
- Live voltage, current, output state, and recording state are displayed.
- **Stop** is enabled and all configuration controls remain disabled.

## Front Panel Controls

### Instrument Setup

- **VISA resource name**: Selects the VISA address of the Keithley 2260B.
- **Serial Configuration**: Sets baud rate, flow control, parity, data bits, and stop bits.

### Test Settings

- **Voltage Setpoint (V)**: Requested power-supply output voltage.
- **Current Limit (A)**: Maximum permitted output current.
- **On Time (s)**: Length of each output-enabled period.
- **Off Time (s)**: Length of each output-disabled period.
- **Measurement Interval (s)**: Time between voltage and current readings.
- **Confirm Inputs**: Initialises and configures the instrument using the entered settings.

### Data Recording

- **File Path**: Read-only indicator showing the active results-file location.
- **Record?**: Enables or disables measurement recording for the experiment.
- **Recording?**: Indicates whether measurements are currently being written to file.

### Live Status

- **Output On?**: Indicates the commanded output state.
- **Actual Voltage (V)**: Displays the measured output voltage.
- **Actual Current (A)**: Displays the measured output current.

## Data Recording

Recording is selected before the experiment starts using **Record?**.

When recording is enabled, each CSV row contains:

```text
Timestamp,Voltage (V),Current (A)
```

Example:

```text
15:20:13.12,9.998000,0.000000
15:20:13.37,9.998000,0.000000
15:20:13.62,0.000000,0.000000
```

### File Naming

The application creates the file when **Start** is pressed. The filename contains the exact experiment start timestamp, for example:

```text
Results11Sep2026 15h20m13s.csv
```

### Save Location

Files are stored in:

```text
<Application Directory>/Results/
```

The `Results` folder is created automatically if it does not already exist. The active file path is displayed on the front panel and is not user-editable.

## Requirements

### Development Environment

- LabVIEW development environment compatible with the project source
- NI-VISA
- Keithley 2260B Native LabVIEW Instrument Driver version 1.0.0 or a compatible replacement
- A supported Keithley 2260B programmable DC power supply
- A configured serial/VISA connection to the instrument

The official Keithley driver is available from Tektronix:

<https://www.tek.com/en/support/software/driver/2260b-30-36-software-1>

### Built Application

To run the compiled application on a computer without the LabVIEW development environment, install:

- The matching LabVIEW Runtime Engine for the version used to build the executable
- NI-VISA Runtime
- Any additional driver components included in or required by the build

## How to Run in the LabVIEW Development Environment

1. Clone or download this repository.
2. Install NI-VISA and the Keithley 2260B LabVIEW instrument driver.
3. Connect the Keithley 2260B and confirm that its VISA resource is available.
4. Open the project file:

   ```text
   Keithley 2260B Control.lvproj
   ```

5. Open:

   ```text
   Main.vi
   ```

6. Press the LabVIEW **Run** arrow.
7. Select the VISA resource and enter the required serial and test settings.
8. Select **Record?** if CSV logging is required.
9. Press **Confirm Inputs**.
10. Press **Start**.
11. Press **Stop** to end the test and close the application safely.

> Always use **Stop** to finish a test. This disables the Keithley output, closes the VISA session, and finalises the results file.

## Build and Run the Executable

### Build the Application

1. Open `Keithley 2260B Control.lvproj`.
2. In Project Explorer, expand **Build Specifications**.
3. Right-click the application build specification, for example:

   ```text
   Keithley 2260B Control
   ```

4. Select **Build**.
5. When the build completes, open the configured build-output directory.
6. Run:

   ```text
   Keithley 2260B Control.exe
   ```

The `Results` directory is created alongside the running application when the first recorded experiment starts.

### Recommended Build Specification Settings

Use `Main.vi` as the startup VI and include all required project VIs, typedefs, channel-wire dependencies, and Keithley driver dependencies.

Recommended executable behaviour:

- Show the front panel when launched
- Use the application title `Keithley 2260B Control`
- Hide the LabVIEW toolbar, Run button, Abort button, and Run Continuously button
- Prevent arbitrary front-panel resizing if controls are not configured to scale
- Retain scrollbars as a fallback for smaller displays
- Close the application after the front panel closes
- Build the executable and its support files into one dedicated application directory

## Architecture

The application uses an event-driven producer/consumer design with parallel loops:

- **User Events loop**: Handles front-panel actions and broadcasts application commands using an Event Messenger channel.
- **Control loop**: Owns the Keithley VISA session, configures the power supply, controls timed output switching, and reads voltage/current measurements.
- **File loop**: Receives measurements through a stream channel and writes valid samples to CSV when recording is enabled.

The Control loop uses retained absolute timing values for:

- The next output-state transition
- The next voltage/current measurement

This keeps output timing and measurement acquisition independent while allowing user events to remain responsive.

## Safe Shutdown

When **Stop** is pressed, the application:

1. Broadcasts the exit command to each loop.
2. Commands the Keithley output off.
3. Stops measurement acquisition.
4. Finalises and closes the results file if recording is enabled.
5. Closes the Keithley VISA session.
6. Exits the application loops cleanly.

## Notes

- Confirm all voltage and current settings are appropriate for the connected equipment before starting a test.
- The software timing is controlled by Windows and LabVIEW and is not intended for deterministic, safety-critical switching.
- The Keithley output is explicitly disabled during configuration and shutdown.
- Configuration values cannot be changed after **Confirm Inputs** is pressed.
- The experiment must be stopped and the application restarted to use a different configuration.

## Support

If you need help or wish to modify the application for another test setup, contact the **Electronics & Software Labs** at the University of Bath.
