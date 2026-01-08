# Pinscope Component Definition & Requirements DSL

This document defines:

- How Pinscope models components and pins
- How semantic pins are separated from physical package pins
- How pins are described using orthogonal attributes
- How pin-level requirements are written using the compact ASCII DSL
- How conditional behavior is expressed using patterns

This is intended for engineers authoring component definitions.

## 1. Core Philosophy

Pinscope treats a schematic like a program.

- **Component YAML** describes facts and intent
- **DSL rules** describe enforceable requirements
- **The engine** decides how validation is performed

**If a pin requires something to function correctly, it must be expressed as a DSL rule — not prose.**

## 2. Semantic Pins vs Physical Package Pins

Pinscope strictly separates:

- **Semantic pins** → what a pin means electrically
- **Package pins** → where the pin exists physically

This allows one component definition to support multiple packages (QFN, BGA, WLCSP) without duplicating rules.

## ## 3. Canonical Component Structure

```yaml
component: <part_number>
category: <category>

packages:
  <package_name>:
    pins:
      <pin_uid>: [<package_pin_id>, ...]

pins:
  <pin_uid>:
    names: [string, ...]
    role: <role>
    direction: <direction>
    voltage: <voltage_spec>
    rules: [ <dsl_rule>, ... ]
    tags: [string, ...]
    description: string
```

## 4. Packages and Pin Mapping

### 4.1 Package Pins

Package pin identifiers are **strings**, not numbers.

Supports:

- Numeric pins (`"1"`, `"12"`)
- BGA coordinates (`"A1"`, `"C7"`)
- Alphanumeric connector pins

Multiple physical pins may map to a single semantic pin UID.

#### Example: LGA / QFN Package

```yaml
packages:
  lga8:
    pins:
      power.vdd.main: ["1"]
      analog.cap.ref: ["2"]
      power.gnd.ref: ["3"]
      power.vddio.main: ["4"]
      signal.int2.out: ["5"]
      signal.int1.out: ["6"]
      signal.i2c.sda: ["7"]
      signal.i2c.scl: ["8"]
```

#### Example: BGA Package

```yaml
packages:
  bga64:
    pins:
      power.vdd.main: ["A1", "A2", "B1", "B2"]
      power.gnd.ref: ["C3", "C4", "D3", "D4"]
```

## 5. Semantic Pin UID

### 5.1 What a UID Represents

A pin UID represents electrical meaning:

- It is stable across packages and symbols
- Rules reference pin UIDs only

#### UID Naming Convention

```
<domain>.<function>.<qualifier>
```

Examples:

- `power.vdd.main`
- `power.gnd.ref`
- `signal.i2c.sda`
- `config.boot0.strap`

## 6. Orthogonal Pin Attributes

Pins are modeled using independent fields.  
**Do not overload a single field to encode multiple meanings.**

### 6.1 `role`

What the pin is for electrically.

**Allowed values:**

- `power`
- `ground`
- `io`
- `analog`
- `clock`
- `config`
- `protection`

### 6.2 `direction`

Signal directionality.

**Allowed values:**

- `input`
- `output`
- `bidirectional`
- `passive`

### 6.3 `voltage`

Voltage metadata (not identity).

```yaml
voltage:
  nominal: 3.3
  abs_min: 1.95
  abs_max: 3.6
```

**Notes:**

- `nominal` expresses intended operating voltage
- `abs_min` / `abs_max` protect silicon
- Voltage expressions may reference pin UIDs (engine-resolved)

## 7. Requirements DSL (Compact, ASCII-Only)

Pin requirements are written as single-line DSL instructions under `rules:`.

### Design Goals

- One rule = one responsibility
- ASCII only
- No control flow
- Deterministic and explainable
- `!` indicates firm (ERROR)
- No `!` indicates flexible (WARNING)

## 8. DSL General Syntax

```
<kind>(<arg1>, <arg2>, ...) [!]
```

- `kind` is lowercase
- Arguments are positional where possible
- Whitespace is ignored
- Units are parsed by the engine
- `!` suffix means firm requirement

## 9. Capacitor Requirement: `cap`

### Syntax

```
cap(<value>[+], [purpose=<decoupling|bulk|ref>], [max_dist=<distance>])
```

### Semantics

- Default `purpose`: `decoupling`
- `+` means **at least** (minimum total capacitance)
- Absence of `!` → flexible

### Examples

**Firm decoupling capacitor:**

```yaml
cap(0.1u)!
```

**Bulk capacitor (minimum):**

```yaml
cap(1u+)
```

**Firm bulk capacitor with distance constraint:**

```yaml
cap(10u+, purpose=bulk, max_dist=10mm)!
```

## 10. Pull Resistor Requirement: `pull`

### Syntax

```
pull(<up|down>, <target_pin_uid>, <resistance>)
```

### Target Resolution (Important)

DSL targets must reference **semantic pin UIDs**, not schematic net names.

This guarantees stability across:

- Packages
- Variants
- Symbol naming differences

### Examples

Firm I2C pull-up:

pull(up, power.vddio.main, <=10k)!


**Recommended pull-down:**

```yaml
pull(down, power.gnd.ref, <=100k)
```

## 11. Example: Complete Component Definition

```yaml
component: MPL3115A2
category: Sensor

packages:
  lga8:
    pins:
      power.vdd.main: ["1"]
      analog.cap.ref: ["2"]
      power.gnd.ref: ["3"]
      power.vddio.main: ["4"]
      signal.int2.out: ["5"]
      signal.int1.out: ["6"]
      signal.i2c.sda: ["7"]
      signal.i2c.scl: ["8"]

pins:
  power.vdd.main:
    names: ["VDD"]
    role: power
    direction: input
    voltage:
      nominal: 3.3
      abs_min: 1.95
      abs_max: 3.6
    rules:
      - cap(0.1u)!
    description: Main power supply

  signal.i2c.sda:
    names: ["SDA"]
    role: io
    direction: bidirectional
    rules:
      - pull(up, power.vddio.main, <=10k)!
    tags: [i2c_sda]
    description: I2C data line
```

## 12. What the Pin DSL Does NOT Support (By Design)

The pin-level DSL does not support:

- Conditionals
- Cross-component logic
- Expressions
- Loops
- Variables

**Conditional behavior belongs in patterns, not pin rules.**

## 13. Engine Interpretation Model (Summary)

1. Detect package used in schematic
2. Resolve physical package pins → pin UIDs
3. Attach semantic pin metadata
4. Parse and compile DSL rules
5. Search schematic for evidence
6. Emit diagnostics with:
   - Rule text
   - Pin UID
   - Missing or invalid evidence

## 14. Guiding Principles

- **Packages** define where a pin is
- **Pins** define what a pin is
- **Descriptions** are for humans
- **Rules** are for engines
- Pin rules are unconditional
- Keep the DSL boring

> A rule should look like something you'd write on a whiteboard during a design review.

## 15. Patterns: Conditional and Cross-Pin Requirements

(unchanged — your section here is already correct and aligned)

Pins declare needs. Patterns decide numbers.

## 15. Patterns: Conditional and Cross-Pin Requirements

Pinscope separates **pin-local invariants** from **conditional, schematic-level logic** using **patterns**.

Patterns allow Pinscope to express datasheet rules that depend on:
- Other components
- Topology
- Operating modes
- Component values

without polluting the pin-level DSL.

---

### 15.1 Why Patterns Exist

Some requirements cannot be expressed as unconditional pin rules.

Examples:
- “Use 1 µF bulk capacitance **if** the output inductor is ≥10 µH”
- “If this pin is used in mode X, require Y elsewhere”
- “If a topology appears, enforce additional constraints”

These rules are:
- Conditional
- Cross-pin or cross-component
- Context-dependent

Such logic **must not** live inside `pins.rules`.

---

### 15.2 Rule Layering Model

Pinscope evaluates requirements in the following order:

1. Pin rules (unconditional, per pin)
2. Part-scoped patterns (conditional, unique to the part)
3. Global patterns (conditional, reusable)
4. Engine defaults (heuristic, non-authoritative)

**Pins declare invariants.  
Patterns declare implications.**

---

### 15.3 What Is a Pattern?

A **pattern** is a conditional rule evaluated over the schematic graph.

A pattern consists of:
- A **condition** (`when`)
- One or more **actions** (`then`)
- A **scope** (pin, net, or component)

Patterns may:
- Introduce new requirements
- Bind symbolic (deferred) requirements
- Upgrade severity of existing requirements

Patterns reuse the **same DSL requirement atoms** as pin rules  
(e.g. `cap(...)`, `pull(...)`).

---

### 15.4 Where Patterns Live

#### Part-scoped patterns

Used when the conditional behavior is intrinsic to a specific part.

- Stored **inside the component YAML**
- Defined under a top-level `patterns:` section
- Scoped using `this_component: true`

#### Global patterns

Used when the rule applies across:
- Multiple parts
- Interfaces
- Common topologies

- Stored in **engine-owned pattern packs**
- Not embedded in component YAML
- Enable/disable per project

---

### 15.5 Symbolic (Deferred) Requirements

Some pin requirements are:
- Always relevant to a pin
- But conditionally defined in value

Pinscope models this using **symbolic requirements**.

#### Key idea

> **Pins declare that a conditional requirement exists.  
> Patterns decide when it applies and how it is resolved.**

---

### 15.6 Declaring a Symbolic Requirement (Pin Section)

```yaml
pins:
  power.vout.main:
    role: power
    direction: output
    rules:
      - cap(out_bulk)

```

Notes:
1. out_bulk is a symbolic requirement key
2. It does not specify a value
3. It does not enforce anything by itself
4. It declares intent

### 15.7 Binding a Symbolic Requirement (Pattern Section)

```yaml
patterns:
  - name: vout_bulk_depends_on_inductor
    scope: net
    when:
      - this_component: true
      - exists:
          component: inductor
          on_net: power.sw.node
          value: ">=10uH"
    then:
      - bind: out_bulk
        require: cap(47u+)
        severity: firm
```

This reads as:
If this component has an inductor ≥10 µH on the SW node,
then the symbolic requirement out_bulk resolves to cap(47u+).
