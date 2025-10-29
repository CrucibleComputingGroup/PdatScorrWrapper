# PDAT Scorr Wrapper

Integrated toolchain for RISC-V processor design, verification, and optimization using DSL-based instruction constraints.

## Overview

This repository integrates three components of the PDAT (Processor Design and Test) ecosystem:

- **PdatDsl** - Domain-Specific Language for specifying RISC-V ISA subsets
- **ScorrPdat** - RTL synthesis optimization and formal equivalence checking
- **CoreSim** - Constrained-random simulation framework with VCD-based analysis

Together, these tools enable:
- Specification of processor ISA constraints
- Automated synthesis optimization
- Simulation-based signal correspondence analysis
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
- **SKY130 PDK** - For gate-level synthesis

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
./synth_ibex_with_constraints.sh ../PdatDsl/examples/example_spec.dsl --gates
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
│   ├── scripts/            # Analysis tools
│   ├── rtl_scorr/          # SMT-based verification
│   ├── synth_ibex_with_constraints.sh
│   ├── batch_synth.sh
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

## License

This project is licensed under the Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International License (CC BY-NC-SA 4.0).

- **Non-commercial use**: Free to use, modify, and share under the ShareAlike terms
- **Commercial use**: Contact Nathan Bleier (nbleier@umich.edu) for commercial licensing options

See individual submodule LICENSE files for details.

## Related Projects

- **RtlScorr** - Yosys plugin for formal signal correspondence verification
- **Ibex Core** - lowRISC's 2-stage/3-stage RV32IMC processor

## Contributing

Contributions are welcome! Please see individual submodule repositories for contribution guidelines.

## Contact

For questions or commercial licensing:
- Nathan Bleier - nbleier@umich.edu

## References

- [RISC-V ISA Specification](https://riscv.org/technical/specifications/)
- [Yosys Open SYnthesis Suite](https://yosyshq.net/yosys/)
- [lowRISC Ibex Core](https://github.com/lowRISC/ibex)
