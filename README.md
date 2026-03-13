# PDAT Scorr Wrapper

Integrated toolchain for RISC-V processor design, verification, and optimization using DSL-based instruction constraints.

## Overview

This repository integrates three components of the PDAT (Processor Design and Test) ecosystem:

- **PdatDsl** - Domain-Specific Language for specifying RISC-V ISA subsets
- **ScorrPdat** - RTL synthesis optimization and formal equivalence checking
- **CoreSim** - Constrained-random simulation framework with VCD-based analysis

Together, these tools enable:
- Specification of processor ISA constraints
- Automated synthesis optimization targeting silicon (SKY130) or printed electronics (PPDK/EGFET)
- Simulation-based signal correspondence analysis
- Static timing analysis via OpenSTA
- Formal verification of RTL designs

## Prerequisites

### System Requirements
- **Python 3.7+**
- **Git** with submodule support
- **VCS** (Synopsys) - Required for simulation
- **Synlig** - SystemVerilog frontend for Yosys
- **ABC** - Sequential logic optimization tool
- **Yosys** - Open-source synthesis framework

### Optional Tools
- **Z3** - SMT solver (for advanced verification)
- **OpenSTA** - Static timing analysis
- **SKY130 PDK** - For gate-level synthesis (silicon)
- **EGFET PDK (PPDK)** - For gate-level synthesis (printed electronics). See [PrintedComputing/EGFET_PDK](https://github.com/PrintedComputing/EGFET_PDK)

## Supported PDKs

### SKY130 (Silicon)
SkyWater 130nm process. Used as the default PDK for gate-level synthesis via `synth_to_gates.sh`.

### PPDK / EGFET (Printed Electronics)
Electrolyte-gated FET technology for inkjet-printed circuits. Uses a custom genlib-based ABC flow with post-processing to match PPDK liberty pin names (A1/A2 vs A/B for 2-input gates). Synthesis uses `synth_to_gates_ppdk.sh` and timing analysis uses `analyze_timing_ppdk.sh`.

Key differences from SKY130:
- 11 standard cells (INVX1, NAND2X1, NOR2X1, AND2X1, OR2X1, XOR2X1, XNOR2X1, DFFNRX1, DFFX1, LATCHX1, TSBUF)
- No buffer gate; BUF is synthesized as two back-to-back INVX1 cells
- DFF mapping uses `dfflegalize -cell $_DFF_PN0_ 0` (DFFNRX1 with tied-off reset)
- Cell areas are ~100,000x larger than SKY130 (printed electronics feature sizes are several micrometers)
- Operating frequencies in the Hz range (vs. MHz for SKY130)

## Installation

### 1. Clone the Repository

If you haven't already cloned with submodules:

```bash
git clone --recursive <repository-url> PdatScorrWrapper
cd PdatScorrWrapper
```

If you already cloned, initialize the submodules:

```bash
git submodule update --init --recursive
```

This will initialize:
- `PdatDsl/` - DSL parser and code generation
- `ScorrPdat/` - RTL synthesis and optimization
- `CoreSim/` - Simulation framework with Ibex core

### 2. Install PdatDsl

```bash
cd PdatDsl
pip install -e .
cd ..
```

This installs the `pdat-dsl` command-line tool for parsing DSL files and generating SystemVerilog constraints.

**Verify installation:**
```bash
pdat-dsl version
```

### 3. Install ScorrPdat Dependencies

```bash
cd ScorrPdat
pip install -r requirements.txt
cd ..
```

This installs Python dependencies for synthesis scripts and RTL analysis tools.

### 4. Initialize CoreSim Submodules

```bash
cd CoreSim
git submodule update --init --recursive
cd ..
```

This initializes the Ibex core and other processor submodules required for simulation.

**Note:** VCS must be installed and available in your PATH for simulation.

## Usage

### Quick Start Example

```bash
# 1. Create or use an existing DSL specification
cd PdatDsl
cat examples/example_16reg.dsl

# 2. Run synthesis optimization with constraints
cd ../ScorrPdat
./synth_ibex_with_constraints.sh ../PdatDsl/examples/example_16reg.dsl

# 3. Run simulation (requires VCS)
cd ../CoreSim
./run_ibex_random.sh testbenches/ibex/dsls/example_16reg.dsl
```

### Batch Synthesis

ScorrPdat provides batch processing for multiple DSL files:

```bash
cd ScorrPdat
./batch_synth.sh [options]
# Or use the Python batch script:
python batch_synth.py [options]
```

This will process multiple DSL specifications and generate optimized RTL for each.

## Workflow

### 1. Define ISA Constraints (PdatDsl)

Create a DSL file specifying your processor's instruction constraints:

```dsl
# example_spec.dsl
require RV32I
require RV32M
require_registers x0-x15

# Outlaw specific instructions
instruction DIV
instruction REM
```

**Validate the specification:**
```bash
cd PdatDsl
pdat-dsl parse examples/example_spec.dsl
```

### 2. Synthesize with Constraints (ScorrPdat)

Generate optimized RTL with instruction constraints applied:

```bash
cd ScorrPdat
# Full synthesis flow
./synth_ibex_with_constraints.sh ../PdatDsl/examples/example_spec.dsl --gates

# Simplified synthesis (gate-level only, no formal verification)
./synth_core_simplified.sh ../PdatDsl/examples/example_spec.dsl

# Simplified synthesis targeting PPDK (printed electronics)
./synth_core_simplified_ppdk.sh ../PdatDsl/examples/example_spec.dsl
```

**Options:**
- `--gates` - Generate gate-level netlist
- `--3stage` - Enable 3-stage pipeline mode
- `--abc-depth N` - Set k-induction depth for ABC

**Output:** Optimized RTLIL and optionally gate-level Verilog in `output/`

### 3. Run Simulation (CoreSim)

Generate VCD traces for signal correspondence analysis:

```bash
cd CoreSim
./run_ibex_random.sh testbenches/ibex/dsls/example_spec.dsl --constants-only
```

**Options:**
- `--flush-cycles N` - NOP cycles for pipeline flush (default: 100)
- `--random-cycles N` - Random instruction cycles (default: 1000)
- `--3stage` - Enable 3-stage pipeline
- `--constants-only` - Filter to constant signals only

**Output:** VCD files and JSON files in `output/`:
- `ibex_reset_state.vcd` - Initial state
- `ibex_random_sim.vcd` - Full trace
- `initial_state.json` - Initial flip-flop values
- `signal_correspondences.json` - Equivalence classes

### 4. Formal Verification (Optional)

Use the RTL-scorr Yosys plugin (separate repository) with generated JSONs:

```bash
# Example using rtl_scorr plugin
yosys -m rtl_scorr.so -p "
  read_verilog ibex_core.v
  rtl_scorr output/signal_correspondences.json \
            output/initial_state.json \
            -apply-opt -k 2
"
```

## Project Structure

```
PdatScorrWrapper/
├── .gitmodules              # Submodule configuration
├── README.md                # This file
├── PdatDsl/                 # DSL parser and code generation
│   ├── pdat_dsl/           # Python package
│   ├── examples/           # Example DSL files
│   └── README.md
├── ScorrPdat/              # RTL synthesis and optimization
│   ├── scripts/            # Gate synthesis & timing analysis
│   │   ├── synth_to_gates.sh        # SKY130 gate mapping
│   │   ├── synth_to_gates_ppdk.sh   # PPDK gate mapping
│   │   ├── analyze_timing.sh        # SKY130 STA (OpenSTA)
│   │   └── analyze_timing_ppdk.sh   # PPDK STA (OpenSTA)
│   ├── odc/               # ODC (Observability-Don't-Care) analysis
│   ├── rtl_scorr/          # SMT-based verification
│   ├── configs/            # Synthesis configurations
│   ├── synth_ibex_with_constraints.sh  # Full synthesis flow
│   ├── synth_core_simplified.sh        # Simplified (SKY130)
│   ├── synth_core_simplified_ppdk.sh   # Simplified (PPDK)
│   ├── batch_synth.sh
│   ├── batch_synth.py
│   └── README.md
└── CoreSim/                # Simulation framework
    ├── cores/              # Processor submodules (Ibex, etc.)
    ├── testbenches/        # VCS testbenches
    ├── scripts/            # Compilation and utilities
    ├── run_ibex_random.sh
    └── README.md
```

## Common Workflows

### Development Cycle

1. **Edit DSL** - Modify ISA constraints in `PdatDsl/examples/`
2. **Synthesize** - Run `./batch_synth.sh` in ScorrPdat
3. **Simulate** - Run `./run_ibex_random.sh` in CoreSim
4. **Analyze** - Inspect generated JSONs and reports
5. **Iterate** - Refine constraints and repeat

### Adding Custom DSL Specifications

```bash
# Create new DSL file
cd PdatDsl/examples
cat > my_custom_spec.dsl << 'EOF'
require RV32I
require_registers x0-x7
instruction MUL
instruction DIV
EOF

# Validate
pdat-dsl parse my_custom_spec.dsl

# Synthesize
cd ../../ScorrPdat
./synth_ibex_with_constraints.sh ../PdatDsl/examples/my_custom_spec.dsl
```

## Environment Variables

### IBEX_ROOT
Override the default Ibex core location:

```bash
export IBEX_ROOT=/path/to/custom/ibex
```

Default search order:
1. `$IBEX_ROOT` (if set)
2. `../PdatCoreSim/cores/ibex`
3. `../CoreSim/cores/ibex`

## Troubleshooting

### Submodules Not Initialized
```bash
git submodule update --init --recursive
```

### PdatDsl Command Not Found
```bash
cd PdatDsl
pip install -e .
```

### VCS Not Found (CoreSim)
Ensure VCS is installed and in your PATH:
```bash
which vcs
```

### Missing Synlig/ABC (ScorrPdat)
Install Synlig and ABC according to their respective documentation.

### OpenSTA Not Found
OpenSTA is required for static timing analysis. Install from [OpenSTA GitHub](https://github.com/The-OpenROAD-Project/OpenSTA) and ensure `sta` is in your PATH.

## License

This project is licensed under the Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International License (CC BY-NC-SA 4.0).

- **Non-commercial use**: Free to use, modify, and share under the ShareAlike terms
- **Commercial use**: Contact Nathan Bleier (nbleier@umich.edu) for commercial licensing options

See individual submodule LICENSE files for details.

## Related Projects

- **RtlScorr** - Yosys plugin for formal signal correspondence verification
- **Ibex Core** - lowRISC's 2-stage/3-stage RV32IMC processor
- **EGFET PDK** - [PrintedComputing/EGFET_PDK](https://github.com/PrintedComputing/EGFET_PDK) - Standard cell library for printed electronics

## Contributing

Contributions are welcome! Please see individual submodule repositories for contribution guidelines.

## Contact

For questions or commercial licensing:
- Nathan Bleier - nbleier@umich.edu

## References

- [RISC-V ISA Specification](https://riscv.org/technical/specifications/)
- [Yosys Open SYnthesis Suite](https://yosyshq.net/yosys/)
- [lowRISC Ibex Core](https://github.com/lowRISC/ibex)
- [Printed Microprocessors (ISCA 2020)](https://ieeexplore.ieee.org/document/9138931/)
