# stm32h7_freertos_network

A summer internship project building a FreeRTOS-based networking and telemetry stack on the STM32H723 Nucleo-144 development board, targeting AWS IoT Core via MQTT over Ethernet.

This repository is the working home for the project: all firmware, configuration, documentation, and the final written report live here.

## Project overview

The goal of this project is to build a working, well-documented embedded networking stack that can:

1. Boot FreeRTOS on a NUCLEO-H723ZG
2. Drive three LEDs on independent timers (green every 5 s, blue every 60 s, red every 300 s) and maintain per-LED flash counters
3. Bring up Ethernet, DHCP, and TCP/IP via LwIP
4. Connect to a local Mosquitto MQTT broker and publish the LED flash counters as JSON telemetry every 5 minutes (synchronized to the red LED flash)
5. Connect to AWS IoT Core with TLS mutual authentication, using **Just-in-Time Provisioning (JITP)** from a FaunaLabs-operated device CA so that any future device flashed with a correctly-signed certificate can auto-register itself on first connection
6. Demonstrate that the JITP flow actually works by provisioning at least two distinct device identities

The work is structured so that the upper layers (MQTT, TLS, application logic, AWS configuration) are portable to future FaunaLabs hardware that uses a different network transport. This repository is the reference implementation those future projects will start from.

## Why this project matters to FaunaLabs

FaunaLabs is developing the FaunaTag v3, a cetacean acoustic biologging platform. v3 will need to send telemetry to AWS IoT Core for fleet management, deployment coordination, and post-deployment data offload. The production hardware uses an STM32H7A3 microcontroller paired with an ST67W611M1 wireless module, but bringing up that wireless module in parallel with bringing up the entire application-layer networking stack would conflate two hard problems.

This project separates the concerns. The intern brings up the full networking and AWS stack on a well-supported Ethernet-equipped development board. When that is working, the team has a reference implementation of the entire stack above the physical transport. The production firmware will then swap in the ST67W611M1 transport driver underneath, reusing the upper layers.

Concretely, the following pieces of this project transfer directly to FaunaTag v3 firmware:

- FreeRTOS task architecture and interprocess communication patterns
- AWS IoT Core account configuration, Thing registry, and certificate management
- **Fleet provisioning via JITP** — solving a problem that prior FaunaLabs biologging firmware (tagOS/soundOS) never cracked properly
- Topic naming conventions and message schemas
- mbedTLS configuration for AWS endpoint authentication
- coreMQTT integration
- Device shadow document schemas
- Reconnect and resilience logic
- Application-level telemetry batching and formatting

## Deliverables

By the end of the internship, this repository will contain:

1. Working firmware that boots on a NUCLEO-H723ZG, runs a concurrent-timer LED heartbeat, connects to a local Mosquitto broker, publishes structured JSON telemetry every 5 minutes, and (ideally) extends the same behavior to AWS IoT Core via JITP
2. A local test environment: Mosquitto broker configuration and instructions for running it on a development laptop
3. AWS IoT Core configuration documented as infrastructure-as-code where possible (Terraform, CloudFormation, or at minimum step-by-step written instructions with screenshots)
4. A FaunaLabs device CA setup, with scripts and documentation for generating and signing new device certificates that will provision themselves via JITP
5. A written report, 10 to 15 pages, covering the architecture, the integration work, what worked, what did not, measured performance (throughput, latency, memory and CPU utilization), and recommendations for the FaunaTag v3 team
6. Source-level documentation: every major module has a header comment explaining its purpose, and the top-level `docs/` directory explains the architecture for a reader who is new to the codebase
7. A demonstration, at end of internship, showing the full stack working end-to-end

## Hardware and software

### Hardware

- NUCLEO-H723ZG development board (provided by mentor)
- Development laptop (intern provides)
- Ethernet cable and access to a local network
- Optional stretch: a second Nucleo board to demonstrate device-to-device messaging via AWS

### Software

- STM32CubeIDE or a CMake-based toolchain
- STM32CubeMX for peripheral configuration
- STM32CubeH7 firmware package (includes LwIP middleware and FreeRTOS)
- coreMQTT library (AWS, open source, included in AWS FreeRTOS SDK)
- mbedTLS (included in STM32Cube middleware)
- Mosquitto broker for local development
- AWS account (FaunaLabs will provision a scoped sub-account)
- Git, GitHub, and a reasonable working knowledge of version control

### FaunaLabs-provided resources

- NUCLEO-H723ZG board
- AWS account access with an IAM user scoped to development permissions
- A subdomain on faunalabs.org for the AWS IoT Core endpoint, if useful
- Weekly 1:1 mentoring time with Dave
- Async support via Slack or email between 1:1s

## Milestones

The project is structured as a six-phase sequence across roughly ten weeks. Each phase has a clearly verifiable milestone, and each phase must be completed before the next begins. The transition from local MQTT (Phase 4) to AWS IoT Core (Phase 5) is gated on full success of all prior phases. If Phase 5 is not reached, that is not a failed summer — Phases 1 through 4 deliver a working, documented, verifiable networking stack that has real value to the FaunaTag v3 effort.

### Phase 1: Foundations (roughly weeks 1 to 2)

- Set up STM32CubeIDE, clone STM32CubeH7, build and flash the default NUCLEO-H723ZG blink example
- Read through the first five chapters of the FreeRTOS kernel reference manual
- Generate a CubeMX project with FreeRTOS middleware enabled
- Create two FreeRTOS tasks that communicate via a queue: one task reads a simulated sensor value on a 1 Hz tick, another task prints received values to the debug UART
- **Milestone**: Tasks, queues, and semaphores are understood and working. UART debug output is visible in a terminal emulator.

### Phase 2: LED heartbeat (roughly week 2 to 3)

This phase introduces multiple concurrent FreeRTOS timers and provides a visible, always-on indication that the system is running. It also establishes the counters that will later drive telemetry.

- Configure the three Nucleo user LEDs (green, blue/yellow, red) as GPIO outputs
- Create three FreeRTOS software timers (or dedicated tasks) that flash the LEDs on the following schedule:
  - **Green**: 250 ms flash every 5 seconds
  - **Blue**: 250 ms flash every 60 seconds
  - **Red**: 250 ms flash every 300 seconds (5 minutes)
- Maintain three monotonically increasing counters (`green_flash_count`, `blue_flash_count`, `red_flash_count`) that increment each time their respective LED flashes. These counters are shared state and must be accessed safely from the tasks or timers that read them.
- **Milestone**: All three LEDs are flashing on their correct schedules, observed over a full 5-minute window. The counters are queryable from a debug shell or visible in UART debug output at each flash event.

Implementation note: the Nucleo-H723ZG's blue-equivalent is PE1 (the yellow LED); label it as blue in the code and documentation to match the naming in this brief.

### Phase 3: Ethernet and local MQTT (roughly weeks 3 to 4)

- Enable LwIP and the STM32 Ethernet HAL driver
- Get DHCP working and verify with ping from the development laptop
- Implement a simple TCP echo server and verify with netcat
- Install Mosquitto on the development laptop
- Integrate coreMQTT, connect to the local Mosquitto broker, publish and subscribe to a topic
- **Milestone**: The Nucleo publishes a test message to the local Mosquitto broker and receives a subscription echo. `mosquitto_sub` on the laptop shows the message.

### Phase 4: Telemetry publish on red LED flash (roughly weeks 4 to 5)

This phase wires together Phases 2 and 3. Every time the red LED flashes (i.e. every 5 minutes), the device publishes a JSON telemetry message to the local Mosquitto broker containing the current values of the LED flash counters. This produces a low-rate, verifiable telemetry stream that is easy to inspect in broker logs to confirm end-to-end behavior over long windows.

- Create a telemetry task (or use an existing task) that wakes on red-LED-flash events, reads the current counter values, formats them as JSON, and publishes the message via coreMQTT
- The message schema is:

```json
{
  "device_id": "stm32h7-nucleo-001",
  "uptime_seconds": 12345,
  "red_flash_count": 4,
  "green_flash_count": 240,
  "blue_flash_count": 20
}
```

- Publish to topic `faunalabs/dev/<device_id>/telemetry`
- Run the Nucleo continuously for at least 30 minutes and verify the broker receives 6 or more messages with monotonically increasing counters
- **Milestone**: A 30-minute Mosquitto log shows one message per 5-minute interval, counters are monotonically increasing, and the JSON parses cleanly.

### Phase 5: AWS IoT Core with fleet provisioning (roughly weeks 5 to 8)

This phase is gated. **Do not begin Phase 5 until Phases 1 through 4 are 100 percent working and documented.** If Phase 4 is flaky, or the counters ever go backwards, or the JSON occasionally fails to parse, fix those before starting Phase 5. AWS-side debugging is dramatically harder than local debugging, and a wobbly foundation will waste weeks.

The goal of this phase is not just "get one device talking to AWS" but to set up proper fleet-scale device onboarding. Dave's prior biologging work never achieved proper fleet management with AWS IoT Core, and this project is the opportunity to do it right.

**Provisioning approach**: Just-in-Time Provisioning (JITP) with a FaunaLabs-operated CA. The goal is that any new FaunaLabs device, flashed with a certificate signed by the FaunaLabs device CA, can connect to AWS IoT Core for the first time and be auto-registered as a Thing with the correct policy attached, without manual intervention in the AWS console.

**Sub-milestones:**

- **5a**: Set up a FaunaLabs device CA (can be a simple OpenSSL-based CA run locally; no need for a hosted HSM or commercial CA for this project). Register the CA with AWS IoT Core.
- **5b**: Write a provisioning template that describes how a new Thing should be created when a device presents a CA-signed cert for the first time: Thing name from certificate CN, policy attachment, any Thing attributes.
- **5c**: Generate a unique device certificate signed by the FaunaLabs CA, flash it onto the Nucleo alongside the private key (for development only; production key storage is out of scope for this project).
- **5d**: Configure mbedTLS with the device certificate, the FaunaLabs CA cert, and the Amazon Root CA.
- **5e**: Point coreMQTT at the AWS IoT Core endpoint. On first connect, confirm via CloudTrail or the IoT console that the Thing was auto-registered by JITP.
- **5f**: Publish the same JSON telemetry stream from Phase 4, but now to an AWS IoT topic (`faunalabs/dev/<device_id>/telemetry`) instead of the local broker. Verify receipt in the AWS IoT console test client.
- **5g**: Repeat the provisioning test with a second fresh device certificate (even if reflashed on the same Nucleo, with a new CN) to confirm that JITP is actually working and the first success was not a one-off.
- **Milestone**: Two distinct device certs, both provisioned via JITP, both publishing telemetry, both visible in the AWS IoT Core console with correct Thing attributes and policies.

Reference material to study before starting Phase 5:
- AWS IoT Core Developer Guide, specifically the "Provisioning devices" and "Just-in-Time Provisioning" chapters
- AWS IoT Core policy documents and the principle of least privilege
- mbedTLS configuration for embedded TLS clients
- CloudTrail for auditing IoT provisioning events

### Phase 6: Hardening and writeup (roughly weeks 9 to 10)

- Add reconnect logic with exponential backoff (both for MQTT disconnects and for TLS handshake failures)
- Add a watchdog that recovers the firmware from a stuck network state
- Write the final report
- Prepare the demonstration
- Clean up the repository, write the top-level README, document the architecture, and specifically write a `docs/aws-setup.md` that walks another engineer through recreating the AWS-side configuration from scratch
- **Milestone**: Report complete, code clean, repository presentable to a future reader who was not part of the project, AWS-side configuration recreatable by another engineer in under an hour.

## Repository structure

The repository should grow into roughly the following layout over the summer. This is a suggestion; the intern can adjust it in consultation with Dave.

```
stm32h7_freertos_network/
├── README.md                      (this file, eventually expanded)
├── firmware/                      (the STM32 project)
│   ├── Core/                      (CubeMX-generated application code)
│   ├── Drivers/                   (CubeMX-generated HAL)
│   ├── Middlewares/               (FreeRTOS, LwIP, mbedTLS, coreMQTT)
│   ├── Application/               (intern-written modules)
│   │   ├── network/               (transport abstraction)
│   │   ├── mqtt/                  (coreMQTT integration and topic helpers)
│   │   ├── telemetry/             (application-layer message formatting)
│   │   └── config/                (AWS endpoint, cert management)
│   └── STM32H723ZGTX_FLASH.ld
├── aws/                           (AWS IoT configuration)
│   ├── iac/                       (Terraform or CloudFormation, if used)
│   ├── policies/                  (IAM and IoT policy JSON)
│   └── README.md                  (manual setup instructions)
├── tools/                         (laptop-side helpers)
│   ├── mosquitto/                 (broker config for local dev)
│   └── scripts/                   (useful shell or Python tools)
├── docs/
│   ├── architecture.md            (high-level design)
│   ├── aws-setup.md               (AWS configuration walkthrough)
│   ├── porting.md                 (notes on what transfers to FaunaTag v3)
│   └── report.md                  (the final written report)
└── .gitignore
```

## CI and automation

A minimal GitHub Actions workflow builds the firmware on every push and pull request. The intern is responsible for setting this up in Phase 1 or early Phase 2, before the codebase grows large enough to be painful to bring under CI later. A green CI badge on the README is part of the Phase 6 cleanup.

No AI code review tools are enabled for this project. The intern's learning is better served by writing code, running it on hardware, discovering problems through testing, and discussing them in 1:1s. Embedded firmware behavior depends heavily on context that AI reviewers typically lack (interrupt safety, volatile semantics, DMA coherency, hardware-specific timing).

Branch protection is enabled on `main`: all changes land via pull request, not direct push. Even when Dave is the only reviewer and approvals are quick, the discipline of feature-branch workflow is a career-relevant skill worth establishing from day one.

## Version control and workflow

All work is tracked on this public GitHub repository.

- The intern commits frequently with clear messages. "Fixed a bug" is not an acceptable commit message; "Fix DHCP lease renewal deadlock in ethernet_task" is.
- Weekly 1:1s with Dave include a brief review of that week's commits.
- Any significant design decision is documented in a commit message or, for larger decisions, in a markdown file in `docs/`.
- No AWS credentials, private keys, or device certificates are ever committed. The repository includes a `.gitignore` that excludes common credential patterns, and the intern is responsible for verifying this before the first push.
- Any secrets that need to be referenced in code are loaded from a local configuration file that is excluded from version control. A template file (`config.template.h`) shows the structure without the actual values.

## Supervision model

- Weekly 1:1s with Dave, scheduled at a recurring time
- Async support via Slack or email between 1:1s for unblocking questions
- The intern is expected to work independently between meetings. Getting stuck is normal and expected; the right response is to document what was tried, ask a focused question, and move on to a parallel task while waiting for a response
- "Stuck for more than four hours on the same problem" is a signal to ask for help, not to push through
- The expectation is honest communication about progress. A week where things did not go well is not a failure; a week where things did not go well and nobody found out until the 1:1 is a problem

## Success criteria

This project defines success at two levels: a **baseline success** that must be achieved, and an **ideal success** that includes the AWS IoT Core work.

### Baseline success (Phases 1 through 4)

By the end of the internship, the following things are true:

1. The NUCLEO-H723ZG boots, runs FreeRTOS, and flashes three LEDs on their specified independent schedules (5 s, 60 s, 300 s)
2. The device, when powered on and plugged into an Ethernet-connected network, autonomously connects to a local Mosquitto broker and publishes JSON telemetry every 5 minutes containing monotonically-increasing LED flash counters
3. A 30-minute recording of the Mosquitto broker log shows the expected number of messages with the expected shape
4. The written report is complete, honest about what worked and what did not, and useful to the FaunaTag v3 team
5. The repository is clean, well-organized, and readable
6. The intern can give a 15-minute verbal walkthrough of the full stack (from FreeRTOS scheduling through Ethernet frame arrival to MQTT publish) and explain why each layer exists

### Ideal success (Phases 1 through 6 including AWS IoT Core with JITP)

In addition to all of the above:

7. A FaunaLabs device CA exists, is registered with AWS IoT Core, and can sign device certificates that provision themselves via JITP on first connection
8. At least two distinct device identities have been successfully provisioned via JITP and are publishing telemetry to AWS IoT Core
9. The AWS-side configuration (CA registration, JITP template, IoT policies, IAM roles) is documented well enough in `docs/aws-setup.md` that another engineer could recreate it in under an hour
10. The transport abstraction is clean enough that swapping Ethernet for a different transport would require changes only in one file

### Stretch (beyond Phase 6)

11. Measured throughput, latency, and memory utilization are documented in the final report
12. The intern has identified at least one non-trivial improvement or correction that will affect FaunaTag v3 firmware design
13. Reconnect resilience has been tested under adversarial conditions (cable pull, broker restart, TLS renegotiation)

## About FaunaLabs

FaunaLabs is a 501(c)(3) conservation technology nonprofit developing biologging hardware and software for marine mammal research. More at https://faunalabs.org.

## License

MIT. A `LICENSE` file is included at the root of the repository. All code committed to this repository is released under the MIT License unless explicitly noted otherwise in a per-file header.
