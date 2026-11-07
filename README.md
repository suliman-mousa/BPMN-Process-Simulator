# BPMN Process Simulation Engine

A browser-based simulation environment that makes BPMN process execution visible as it runs: tokens move through the model, configurable timing drives the run, and every element entry and exit is captured, measured and persisted for analysis.

The engine is built on top of the open-source [`bpmn-js`](https://github.com/bpmn-io/bpmn-js) modeler and the [`bpmn-js-token-simulation`](https://github.com/bpmn-io/bpmn-js-token-simulation) kernel by bpmn.io. The simulation kernel, token animation and base interaction modules come from that upstream project; the capabilities below were designed and built for this project.

## My contribution

| Area | What was built |
|---|---|
| **XML-driven runtime configuration** | Simulation parameters (intervals, timeouts, iteration limits) are read from BPMN extension properties and applied through context-pad actions, so behaviour changes without code changes. Bounded iterations, timer cleanup on stop/reset, and restoration of original values after a run. |
| **Event capture pipeline** | A `MutationObserver`-based observer on the simulation log normalizes each element entry/exit into a structured event and buffers it client-side (`logEvent`). |
| **Simulation persistence** | Session registration (`sendSimulationData`) and per-event submission (`sendSimulationEntryLog`) to a backend REST API, coordinated by `executeSimulationData`, so every run is stored with its events. |
| **Execution measurement** | Recorded entry/exit timestamps feed performance indicators: flow duration, waiting time per element, throughput, and bottleneck points. |
| **Architecture documentation** | A full Software Architecture Description ([`ARCHITECTURE.md`](./ARCHITECTURE.md), ISO/IEC/IEEE 42010) and C4-style, sequence and state diagrams ([`DIAGRAMS.md`](./DIAGRAMS.md)). |

**Planned extensions:** ERP integration and AI-assisted process analysis on top of the stored simulation events.

---

## Upstream module reference: Simulation Support / Scripting

The `simulation-support` module provides programmatic control and runtime introspection capabilities for token simulation in [`bpmn-js-token-simulation`](https://github.com/bpmn-io/bpmn-js-token-simulation).

It enables automated interaction with BPMN simulation instances, execution-flow observation, and synchronization with simulation events. These capabilities are useful for integration testing, behavioral verification, and reproducible simulation scenarios.

![Simulation Screen Capture](./screencapture.gif)

### Features

- **Execution Trace Verification** — Record and assert simulation execution traces.
- **Simulation Control** — Enable simulation and control execution through pause and resume interactions.
- **Lifecycle Observation** — Await element-entry and element-exit events.
- **Event Triggering** — Trigger simulation events and continue execution programmatically.
- **Automated Testing Support** — Coordinate test execution with simulation state transitions.

### Prerequisites

- A configured `bpmn-js` modeler instance.
- The `bpmn-js-token-simulation` package installed and available to the application.
- A BPMN diagram containing the elements referenced by the simulation script.

Element IDs used by the simulation support API must correspond to IDs in the loaded BPMN diagram.

### Installation and Configuration

Register the `SimulationSupportModule` through the modeler's `additionalModules` configuration.

```javascript
import BpmnModeler from 'bpmn-js/lib/Modeler';
import SimulationSupportModule from
  'bpmn-js-token-simulation/lib/simulation-support';

const modeler = new BpmnModeler({
  container: '#canvas',
  additionalModules: [
    SimulationSupportModule
  ]
});
```

After module registration, retrieve the `simulationSupport` service from the modeler's dependency injection container:

```javascript
const simulationSupport = modeler.get('simulationSupport');
```

The service provides the control and observation APIs documented in the following sections.

### API Usage

#### Enable Simulation

Enable simulation before interacting with simulation elements:

```javascript
simulationSupport.toggleSimulation(true);
```

#### Trigger a Simulation Element

Use `triggerElement(elementId)` to interact with a simulation element:

```javascript
simulationSupport.triggerElement('StartEvent_1');
```

The effect of triggering an element depends on the simulation state and the element's behavior.

#### Await Element Entry

Use `elementEnter(elementId)` to synchronize subsequent operations with entry into a specific element:

```javascript
await simulationSupport.elementEnter('UserTask_1');
```

This pattern is useful when a test must wait for the simulation to reach a particular point before continuing.

#### Await Element Exit

Use `elementExit(elementId)` to synchronize subsequent operations with the element-exit event:

```javascript
await simulationSupport.elementExit('UserTask_1');
```

#### Pause and Resume Execution

The following example illustrates the control sequence using the API demonstrated by the module:

```javascript
// Enable simulation.
simulationSupport.toggleSimulation(true);

// Start the simulation.
simulationSupport.triggerElement('StartEvent_1');

// Wait until the user task is entered.
await simulationSupport.elementEnter('UserTask_1');

// Perform assertions or other test operations here.

// Trigger the element to continue execution.
simulationSupport.triggerElement('UserTask_1');
```

The example assumes that the BPMN model contains the specified element IDs and that the simulation reaches the expected execution state.

**Important:** Verify the exact pause/resume semantics against the installed module version. The example demonstrates the documented interaction pattern; it does not establish behavior for every BPMN element or simulation state.

### Automated Testing Guidelines

When integrating simulation support into automated tests, follow these practices:

1. **Use deterministic scenarios.** Prepare a known BPMN model and define the expected execution path.
2. **Synchronize on simulation events.** Prefer the module's event-observation APIs over arbitrary delays.
3. **Assert observable behavior.** Verify the expected element-entry, element-exit, and execution-trace outcomes.
4. **Avoid blocking browser dialogs.** Use test-framework assertions and synchronization instead of `window.alert()`.
5. **Handle asynchronous failures.** Ensure the test runner can detect rejected promises and report failures.
6. **Prevent indefinite waits.** Configure test-level timeouts and cleanup according to the capabilities of the installed API.
7. **Isolate test scenarios.** Reset or recreate the modeler and simulation state when required between tests.

The availability of explicit cancellation, per-operation timeouts, and simulation-reset methods must be verified against the actual API implementation.

### Execution Trace Verification

The module supports recording and asserting execution traces. Use these capabilities to validate the order of simulation events against the expected behavior of a BPMN process.

A complete test should establish:

- The expected sequence of executed elements.
- The expected entry and exit points.
- The conditions under which execution continues.
- The assertion outcome when the observed trace differs from the expected trace.

Refer to the implementation and existing tests for the exact trace-recording and assertion APIs.

### Error Handling and Operational Considerations

Simulation scripts should account for the following conditions:

- An element ID does not exist in the loaded BPMN model.
- The expected element is never reached.
- The simulation is disabled or is in an unexpected state.
- An asynchronous observation fails or does not complete within the test's allowed execution time.
- The simulation model changes between test setup and execution.

Where supported by the implementation, document the resulting errors, timeout behavior, and cleanup requirements explicitly. Do not assume that all these conditions are handled automatically by the module.

### API Reference

The `SimulationSupport` service is implemented in:

[`lib/simulation-support/SimulationSupport.js`](https://github.com/bpmn-io/bpmn-js-token-simulation/blob/master/lib/simulation-support/SimulationSupport.js) (upstream repository)

Consult the implementation and associated tests for the authoritative method signatures, return values, supported element types, and lifecycle behavior.

### Example and Reference

- [Token Simulation Repository](https://github.com/bpmn-io/bpmn-js-token-simulation)
- [Simulation Example on CodeSandbox](https://codesandbox.io/s/amazing-christian-ry9yw)

### Scope

This module provides programmatic control and observation of BPMN token simulation. It is intended to support simulation-driven workflows and automated verification; it should not be treated as a substitute for validating the underlying process model or the correctness of the simulation engine itself.
