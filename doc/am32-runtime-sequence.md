# AM32 Runtime Sequence Diagram

This document gives a high-level analysis of the AM32 firmware runtime flow and a sequence diagram for the main control path.

The core source modules involved are:

- `Src/main.c`: startup, control loop, commutation orchestration
- `Src/signal.c`: input detection and DMA input decoding
- `Src/dshot.c`: DShot frame decode and DShot telemetry packaging
- `Src/kiss_telemetry.c`: serial telemetry packet formatting
- `Src/functions.c`: utility helpers like `map()`, delays, and CRC
- `Src/sounds.c`: startup and status tones

## Project Analysis

AM32 is structured like a small real-time motor-control kernel around a few repeating responsibilities:

- input acquisition: DMA captures signal timing, then `signal.c` classifies and decodes it
- input normalization: `main.c` converts raw input into `adjusted_input`, manages arming, reverse, brake, and startup rules
- motor commutation: `main.c` uses either polling or interrupt-driven zero-cross detection to advance the motor phases
- protection and control: a high-frequency routine applies PID-based current limiting, stall protection, speed control, and voltage / thermal protection
- telemetry and UX: the firmware can package telemetry replies and play tones for startup, calibration, and commands

In practical terms, the runtime loop is:

1. initialize hardware and load persistent settings
2. capture incoming control signal
3. decode protocol and compute a usable input value
4. decide whether to arm, brake, start, or continue running
5. drive the commutation loop
6. periodically apply protection logic and send telemetry

## Sequence Diagram

```mermaid
sequenceDiagram
    autonumber
    participant FC as Flight Controller / Receiver
    participant SIG as signal.c
    participant DSH as dshot.c
    participant MAIN as main.c
    participant CTRL as tenKhzRoutine()
    participant COMM as Commutation Core
    participant TEL as Telemetry
    participant SND as sounds.c

    MAIN->>MAIN: initAfterJump()
    MAIN->>MAIN: checkDeviceInfo()
    MAIN->>MAIN: initCorePeripherals()
    MAIN->>MAIN: enableCorePeripherals()
    MAIN->>MAIN: loadEEpromSettings()
    MAIN->>SND: playStartupTune() or playBrushedStartupTune()
    MAIN->>SIG: receiveDshotDma()

    loop Runtime loop
        FC->>SIG: input pulses / DShot frame via DMA
        SIG->>SIG: transfercomplete()

        alt Input type not locked yet
            SIG->>SIG: detectInput()
            SIG->>SIG: checkDshot() / checkServo()
            SIG->>SIG: receiveDshotDma()
        else DShot input
            SIG->>DSH: computeDshotDMA()
            DSH-->>MAIN: newinput / dshotcommand / telemetry flags
            SIG->>SIG: receiveDshotDma()
        else Servo input
            SIG->>SIG: computeServoInput()
            SIG->>SIG: receiveDshotDma()
        end

        MAIN->>MAIN: processDshot() or setInput()
        alt DShot processing path
            MAIN->>DSH: computeDshotDMA()
            opt DShot telemetry response
                MAIN->>DSH: make_dshot_package(e_com_time)
            end
        end

        MAIN->>MAIN: setInput()
        alt Startup / run request
            MAIN->>MAIN: startMotor()
            MAIN->>COMM: commutate()
            MAIN->>COMM: enableCompInterrupts()
        else Stop / brake / tone request
            MAIN->>COMM: allOff() / fullBrake() / proportionalBrake()
            opt Command tone
                MAIN->>SND: playDefaultTone() / playChangedTone() / playBeaconTune3() / playInputTune2()
            end
        end

        MAIN->>CTRL: tenKhzRoutine()

        par Control and protection
            CTRL->>CTRL: doPidCalculations()
            CTRL->>CTRL: voltage / current / thermal checks
            CTRL->>CTRL: duty ramp and limit updates
        and Polling startup / recovery path
            alt old_routine and running
                CTRL->>MAIN: getBemfState()
                CTRL->>MAIN: zcfoundroutine()
                MAIN->>COMM: commutate()
            end
        and Sine mode path
            alt stepper_sine active
                CTRL->>MAIN: advanceincrement()
                opt Transition back to BEMF mode
                    MAIN->>COMM: commutate()
                end
            end
        end

        opt Interrupt-driven commutation
            COMM-->>MAIN: interruptRoutine()
            MAIN->>COMM: maskPhaseInterrupts()
            MAIN->>COMM: SET_AND_ENABLE_COM_INT(waitTime)
            COMM-->>MAIN: PeriodElapsedCallback()
            MAIN->>COMM: commutate()
            MAIN->>COMM: enableCompInterrupts()
        end

        opt Serial telemetry request
            MAIN->>TEL: makeTelemPackage()
            TEL-->>MAIN: telemetry bytes
            MAIN->>TEL: send_telem_DMA()
        end

        opt ESC info response
            MAIN->>TEL: makeInfoPacket()
            MAIN->>TEL: send_telem_DMA()
        end

        opt Brushed build
            CTRL->>MAIN: runBrushedLoop()
        end
    end
```

## Notes

- `signal.c` is the entry point for incoming control data.
- `main.c` owns the system state machine and decides what the motor should do next.
- `tenKhzRoutine()` is the periodic control hub where protection and duty-cycle control are applied.
- commutation can happen in two modes:
  - polling mode during startup or recovery
  - interrupt-driven mode after zero-cross timing becomes stable
- telemetry is a side path and does not replace the main motor-control path.

## Related Documentation

- Detailed `main.c` call graph: [`doc/main-c-analysis.md`](C:\Users\user\Documents\Github\AM32\doc\main-c-analysis.md)
