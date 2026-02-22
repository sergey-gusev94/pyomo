# MindtPy File-by-File Reference

## Scope

This is a complete file-level guide for `pyomo.contrib.mindtpy`.

It covers:

- every runtime module in the package
- the role of each class/function
- structural organization of the largest files
- every file in `tests/` (model fixtures and regression/unit tests)

Use with:

- `MINDTPY_ARCHITECTURE_GUIDE.md` (high-level behavior)
- `MINDTPY_CONFIG_REFERENCE.md` (full option surface)

## Package Inventory (All Python Files)

Core modules:

- `MindtPy.py` (133 lines): `MindtPySolver`
- `__init__.py` (13 lines): version tuple only
- `algorithm_base_class.py` (3210 lines): `_MindtPyAlgorithm` (core engine)
- `config_options.py` (873 lines): config builders and grouped option declarations
- `cut_generation.py` (499 lines): OA/ECP/no-good/affine cut builders
- `extended_cutting_plane.py` (174 lines): `MindtPy_ECP_Solver`
- `feasibility_pump.py` (81 lines): `MindtPy_FP_Solver`
- `global_outer_approximation.py` (113 lines): `MindtPy_GOA_Solver`
- `outer_approximation.py` (163 lines): `MindtPy_OA_Solver`
- `plugins.py` (19 lines): plugin loader imports/registration side effects
- `single_tree.py` (983 lines): lazy callback implementations (CPLEX + Gurobi)
- `tabu_list.py` (45 lines): CPLEX incumbent callback for tabu behavior
- `util.py` (1022 lines): utility functions and solver adapters

Test modules and models:

- `tests/MINLP2_simple.py` (118)
- `tests/MINLP3_simple.py` (85)
- `tests/MINLP4_simple.py` (66)
- `tests/MINLP5_simple.py` (61)
- `tests/MINLP_simple.py` (140)
- `tests/MINLP_simple_grey_box.py` (163)
- `tests/__init__.py` (11)
- `tests/constraint_qualification_example.py` (51)
- `tests/eight_process_problem.py` (263)
- `tests/feasibility_pump1.py` (62)
- `tests/feasibility_pump2.py` (59)
- `tests/from_proposal.py` (51)
- `tests/nonconvex1.py` (66)
- `tests/nonconvex2.py` (97)
- `tests/nonconvex3.py` (69)
- `tests/nonconvex4.py` (70)
- `tests/online_doc_example.py` (50)
- `tests/test_mindtpy.py` (547)
- `tests/test_mindtpy_ECP.py` (109)
- `tests/test_mindtpy_feas_pump.py` (129)
- `tests/test_mindtpy_global.py` (103)
- `tests/test_mindtpy_global_lp_nlp.py` (131)
- `tests/test_mindtpy_grey_box.py` (84)
- `tests/test_mindtpy_lp_nlp.py` (307)
- `tests/test_mindtpy_regularization.py` (274)
- `tests/test_mindtpy_solution_pool.py` (144)
- `tests/unit_test.py` (102)

## Core Runtime Modules (Detailed)

## `MindtPy.py`

### Purpose

Public dispatcher/meta-solver registered as `mindtpy`.

### Main class

- `MindtPySolver`

### What it does

- Declares `CONFIG = _get_MindtPy_config()` (generic config surface)
- Accepts user options/kwargs
- Reads `config.strategy`
- Dispatches solve to the concrete strategy solver (`mindtpy.oa`, `.ecp`, `.goa`, `.fp`)

### Key point

This file does not implement OA/GOA/ECP/FP logic. It only selects the implementation.

## `__init__.py`

### Purpose

Package version declaration.

### Contents

- `__version__ = (1, 0, 0)`

## `plugins.py`

### Purpose

Import-time registration helper.

### Function

- `load()`

### What `load()` does

Imports strategy modules so `SolverFactory.register(...)` decorators execute.

Imported modules:

- `MindtPy`
- `outer_approximation`
- `extended_cutting_plane`
- `global_outer_approximation`
- `feasibility_pump`

## `algorithm_base_class.py` (Core Engine)

### Purpose

Contains `_MindtPyAlgorithm`, the shared implementation used by all concrete MindtPy strategy solvers.

This is the most important file in the package.

### Class

- `_MindtPyAlgorithm`

### Major responsibilities

- solver state initialization
- logging setup and iteration logs
- model validation and ordered component extraction
- objective processing / epigraph reformulation
- initialization strategies (`rNLP`, `max_binary`, `initial_binary`, `FP`)
- MIP main solve and NLP subproblem solve orchestration
- feasibility subproblem solve for infeasible fixed-NLPs
- regularization (ROA/RLP-NLP) solve flow
- Feasibility Pump loop
- bound/gap tracking and termination logic
- result loading and final result object assembly
- callback setup hooks for single-tree / tabu list

### Internal state (important attributes initialized in `__init__`)

Examples of solver state tracked here:

- model clones: `working_model`, `mip`, `fixed_nlp`
- iteration counters: `mip_iter`, `nlp_iter`, `fp_iter`, `mip_subiter`
- bounds: `primal_bound`, `dual_bound`, `abs_gap`, `rel_gap`
- incumbent/best solution: `best_solution_found`, `best_solution_found_time`
- cycling/no-good bookkeeping:
  - `integer_list`
  - `integer_solution_to_cuts_index`
  - `stored_bound`
  - `num_no_good_cuts_added`
- timing + integral tracking arrays

### Structural map (method groups)

#### A. Context/metadata/logging

- `available`, `license_is_valid`, `version`
- `_log_solver_intro_message`
- `set_up_logger`
- `_log_header`

#### B. Model augmentation and validation

- `create_utility_block`
- `model_is_valid`
- `build_ordered_component_lists`
- `add_cuts_components`

#### C. Bound and gap accounting

- `get_dual_integral`, `get_primal_integral`, `get_integral_info`
- `update_gap`
- `update_dual_bound`, `update_suboptimal_dual_bound`
- `update_primal_bound`

#### D. Objective and solve-data setup

- `process_objective`
- `set_up_solve_data`
- `objective_reformulation` (base version; OA overrides it)

#### E. Initialization strategies

- `MindtPy_initialization`
- `init_rNLP`
- `init_max_binaries`

#### F. NLP subproblem flow

- `solve_subproblem`
- `handle_nlp_subproblem_tc`
- `handle_subproblem_optimal`
- `handle_subproblem_infeasible`
- `handle_subproblem_other_termination`

#### G. Feasibility-NLP flow (for infeasible fixed NLP)

- `solve_feasibility_subproblem`
- `handle_feasibility_subproblem_tc`

#### H. Main MIP / regularization MIP flow

- `solve_main`
- `solve_fp_main`
- `solve_regularization_main`
- `set_up_mip_solver`
- `handle_main_optimal`
- `handle_main_infeasible`
- `handle_main_max_timelimit`
- `handle_main_unbounded`
- `handle_regularization_main_tc`
- `setup_main`, `setup_fp_main`, `setup_regularization_main`
- `handle_main_mip_termination`

#### I. Subsolver initialization and backend integration

- `initialize_mip_problem`
- `initialize_subsolvers`
- `set_appsi_solver_update_config`
- `check_subsolver_validity`
- `check_config`

#### J. Feasibility Pump (full implementation lives here)

- `solve_fp_subproblem`
- `handle_fp_subproblem_optimal`
- `handle_fp_main_tc`
- `fp_loop`

#### K. Top-level solve / result handling

- `solve`
- `update_result`
- `load_solution`

#### L. Termination and progress checks

- `algorithm_should_terminate`
- `MindtPy_iteration_loop` (default OA/GOA-style loop)
- `bounds_converged`
- `reached_iteration_limit`
- `reached_time_limit`
- `reached_stalling_limit`
- `iteration_cycling`
- `fix_dual_bound`

#### M. Misc helpers

- `set_up_tabulist_callback`
- `set_up_lazy_OA_callback`
- `get_solution_name_obj`
- `add_regularization`

### Notes on structure

- This file contains code for multiple conceptual subsystems, separated by comment banners (initialization, NLP solve, MIP solve, FP, iterate loop).
- The strategy subclasses keep this file reusable by overriding only a small number of hooks (`check_config`, `initialize_mip_problem`, `add_cuts`, loop/termination behavior).

## `outer_approximation.py`

### Purpose

Strategy subclass for the OA family and OA-derived variants.

### Class

- `MindtPy_OA_Solver` (registered as `mindtpy.oa`)

### What it covers

- OA
- ROA
- LP/NLP
- RLP/NLP

### Key overrides

- `CONFIG = _get_MindtPy_OA_config()`
- `check_config()`
  - handles regularization settings and single-tree restrictions
  - enables implied options (e.g., `calculate_dual_at_solution`)
- `initialize_mip_problem()`
  - precomputes Jacobians
  - adds `cuts.oa_cuts`
- `add_cuts(...)`
  - adds OA cuts and optional grey-box OA cuts
- `deactivate_no_good_cuts_when_fixing_bound()`
  - bound-fix support for OA no-good cut logic
- `objective_reformulation()`
  - custom handling for regularized variants / epigraph flatness issue

## `global_outer_approximation.py`

### Purpose

Strategy subclass for GOA / global LP-NLP variants.

### Class

- `MindtPy_GOA_Solver` (registered as `mindtpy.goa`)

### Distinguishing behavior

- Uses affine cuts (`add_affine_cuts`) rather than OA cuts
- Forces global-friendly defaults (`use_mcpp`, `use_fbbt`, no equality relaxation)
- Defaults to no-good cuts if neither tabu nor no-good is enabled

### Key overrides

- `check_config()`
- `initialize_mip_problem()` -> adds `cuts.aff_cuts`
- `update_primal_bound()` -> extra bookkeeping for no-good-cut count at incumbent
- `add_cuts()` -> affine cuts only
- `deactivate_no_good_cuts_when_fixing_bound()` -> deactivates only no-good cuts added after best incumbent

## `extended_cutting_plane.py`

### Purpose

Strategy subclass for ECP.

### Class

- `MindtPy_ECP_Solver` (registered as `mindtpy.ecp`)

### Distinguishing behavior

- ECP loop adds linear cuts based on nonlinear constraint violations without the usual fixed-NLP-per-iteration OA pattern.
- Terminates when nonlinear constraints are satisfied within `ecp_tolerance`.

### Key overrides

- `MindtPy_iteration_loop()` (ECP-specific loop)
- `check_config()` (defaults `ecp_tolerance` from absolute bound tolerance)
- `initialize_mip_problem()` (precompute Jacobians + add `cuts.ecp_cuts`)
- `init_rNLP()` (calls base `init_rNLP(add_oa_cuts=False)`)
- `algorithm_should_terminate()` (ECP-specific condition set)
- `all_nonlinear_constraint_satisfied()` (ECP stopping check and incumbent capture)

## `feasibility_pump.py`

### Purpose

Strategy subclass for Feasibility Pump-only mode.

### Class

- `MindtPy_FP_Solver` (registered as `mindtpy.fp`)

### Distinguishing behavior

- Most FP logic is implemented in the base class (`fp_loop` and related methods).
- This subclass mainly wires config and cut generation.

### Key overrides

- `check_config()`
  - forces `iteration_limit = 0`
  - forces `move_objective = True`
- `initialize_mip_problem()`
  - precomputes Jacobians and creates `cuts.oa_cuts`
- `add_cuts(...)`
  - uses OA cuts for FP cut transfer support
- `MindtPy_iteration_loop()` -> `pass` (intentional; FP runs via base initialization/loop path)

## `cut_generation.py`

### Purpose

Pure cut construction helpers used by strategy classes and callbacks.

### Functions

- `add_oa_cuts(...)`
- `add_oa_cuts_for_grey_box(...)`
- `add_ecp_cuts(...)`
- `add_no_good_cuts(...)`
- `add_affine_cuts(...)`

### Structure by function

#### `add_oa_cuts`

- Iterates all nonlinear constraints
- Uses precomputed Jacobians
- Handles equality relaxation separately (dual-sign-based cut orientation)
- Handles upper/lower linearizations of inequality constraints
- Optionally adds OA slack vars
- Supports Gurobi persistent lazy insertion via `cbLazy`

#### `add_oa_cuts_for_grey_box`

- Reads Jacobian from external grey-box model output evaluations
- Builds OA cuts for grey-box outputs
- Supports OA cut storage in the same `oa_cuts` container

#### `add_ecp_cuts`

- Uses current slacks and `ecp_tolerance` to decide which cuts to add
- Builds cuts from Jacobian and slack value instead of dual-based OA linearizations

#### `add_no_good_cuts`

- Validates binary values near 0/1
- Builds a standard Hamming-distance-style exclusion cut
- Adds it to `cuts.no_good_cuts`
- Supports Gurobi lazy insertion in single-tree mode

#### `add_affine_cuts`

- Uses MC++ (`McCormick`) for convex/concave affine estimators
- Validates slopes/intercepts against `nan`/`inf`
- Adds concave lower and convex upper affine cuts to `cuts.aff_cuts`
- Used by GOA

## `single_tree.py`

### Purpose

Implements lazy callback logic for single-tree LP/NLP and GOA workflows.

### Why this file exists

In single-tree mode, the main branch-and-bound stays inside the MIP solver. MindtPy injects cuts lazily when incumbents appear, and solves fixed NLPs from callbacks.

### Contents

- `LazyOACallback_cplex` class (CPLEX lazy callback implementation)
- `LazyOACallback_gurobi(...)` function (Gurobi callback implementation)
- `handle_lazy_main_feasible_solution_gurobi(...)`

### `LazyOACallback_cplex` (major methods)

- `copy_lazy_var_list_values(...)`
  - solver-to-Pyomo value copy with integrality/bounds cleanup
- `add_lazy_oa_cuts(...)`
  - CPLEX-native lazy OA cut addition (`self.add(...)`)
- `add_lazy_affine_cuts(...)`
  - CPLEX-native affine cuts for GOA
- `add_lazy_no_good_cuts(...)`
  - CPLEX-native no-good cut addition
- `handle_lazy_main_feasible_solution(...)`
  - updates dual bound from callback objective info and prepares NLP warm start
- `handle_lazy_subproblem_optimal(...)`
  - updates primal bound, incumbent, adds cuts
- `handle_lazy_subproblem_infeasible(...)`
  - solves feasibility subproblem and adds cuts
- `handle_lazy_subproblem_other_termination(...)`
  - limited fallback behavior
- `__call__(...)`
  - main callback entrypoint; orchestrates all callback-time logic

### Gurobi callback path

`LazyOACallback_gurobi` mirrors the same broad flow but uses Gurobi callback APIs (`cbGet`, `cbLazy`, `terminate`) and stores cut-index ranges for previously seen OA integer assignments.

### Important details from code

- Handles repeated integer combinations (cycling) inside callback
- Can re-emit previously generated OA cuts for repeated Gurobi incumbents
- Supports regularization flow from callback when appropriate
- Supports incumbent-time cut generation (`add_cuts_at_incumbent`)
- Handles CPLEX MIP start edge cases (lazy cuts generated during mipstart may need to be re-added)

## `tabu_list.py`

### Purpose

CPLEX incumbent callback used for tabu-list style rejection of repeated discrete incumbents.

### Class

- `IncumbentCallback_cplex`

### Behavior

- Reads discrete variable incumbent values from CPLEX callback context
- Builds current integer tuple
- Rejects incumbent if tuple already appeared
- In `single_tree` mode, rejects incumbents directly (special callback interaction path)

## `util.py`

### Purpose

Central utility/helper module used by the core engine, callbacks, and cut-generation logic.

### Class

- `GurobiPersistent4MindtPy`
  - thin subclass of `GurobiPersistent`
  - adapts callback signature to pass MindtPy solver/config state

### Functions (grouped by role)

#### Derivatives / Jacobians

- `calc_jacobians(constraint_list, differentiate_mode)`
  - builds `ComponentMap` from constraint -> var -> symbolic derivative expression

#### Feasibility-subproblem setup and safety bounds

- `initialize_feas_subproblem(m, feasibility_norm)`
  - creates feasibility slack constraints/objective (`L1`, `L2`, or `L_infinity`)
- `add_var_bound(model, config)`
  - adds fallback bounds to otherwise unbounded vars appearing in nonlinear constraints (mainly for single-tree robustness)

#### Distance and regularization objectives

- `generate_norm2sq_objective_function`
- `generate_norm1_objective_function`
- `generate_norm_inf_objective_function`
- `generate_lag_objective_function`
- `generate_norm1_norm_constraint`

`generate_lag_objective_function` is the most complex utility in this file; it:

- clones a model at the incumbent/setpoint
- builds a `PyomoNLP`
- computes objective gradient, Jacobian, and (optionally) Hessian of the Lagrangian
- constructs a first-order or second-order regularization objective for ROA/RLP-NLP

#### Solver option adapters

- `update_solver_timelimit`
- `set_solver_mipgap`
- `set_solver_constraint_violation_tolerance`

These are backend-specific compatibility helpers so the main algorithm can remain solver-agnostic.

#### Integer solution extraction and value transfer

- `get_integer_solution`
- `copy_var_list_values_from_solution_pool`
- `copy_var_list_values`
- `set_var_valid_value`

`set_var_valid_value` is critical for robustness because it:

- rounds integers within tolerance
- clips to variable bounds when needed
- zeroes near-zero values if domain allows
- raises a `ValueError` on irreconcilable assignments

#### Objective reformulation and result initialization

- `epigraph_reformulation`
- `setup_results_object`

#### Feasibility Pump helpers

- `fp_converged`
- `add_orthogonality_cuts`
- `generate_norm_constraint`

## `config_options.py`

### Purpose

Defines all MindtPy `ConfigBlock` constructors and grouped option declarations.

### Main functions

Config constructors:

- `_get_MindtPy_config`
- `_get_MindtPy_OA_config`
- `_get_MindtPy_GOA_config`
- `_get_MindtPy_ECP_config`
- `_get_MindtPy_FP_config`

Group builders:

- `_add_oa_configs`
- `_add_oa_cuts_configs`
- `_add_goa_configs`
- `_add_ecp_configs`
- `_add_common_configs`
- `_add_subsolver_configs`
- `_add_tolerance_configs`
- `_add_bound_configs`
- `_add_fp_configs`
- `_add_roa_configs`

### What to look for in this file

- `_supported_algorithms` dispatch map used by `MindtPy.py`
- default values and allowed domains for each option
- which strategy-specific config constructor includes which groups

See `MINDTPY_CONFIG_REFERENCE.md` for detailed option descriptions.

## Tests Package Overview

The `tests/` folder serves two roles:

- model fixture library (small benchmark MINLPs, nonconvex examples, grey-box example)
- regression/integration/unit test suite for MindtPy features and solver combinations

A useful pattern in these files:

- model fixtures usually subclass `ConcreteModel` and populate:
  - variables/constraints/objective
  - `optimal_value`
  - `optimal_solution`
- regression tests usually call `SolverFactory('mindtpy')` with explicit `strategy`, `mip_solver`, and `nlp_solver` options

## Test Model Fixture Files (Per File)

## `tests/MINLP_simple.py`

### Purpose

Canonical convex MINLP fixture used broadly across tests.

### Class

- `SimpleMINLP(ConcreteModel)`

### Structure

- defines binary vars `Y`, continuous vars `X`
- defines 7 constraints and nonlinear convex objective
- supports `grey_box=True` mode by replacing objective with an `ExternalGreyBoxBlock` output (`z`)
- stores `optimal_value` and `optimal_solution`

### Why it matters

This is the primary sanity/regression model for OA behavior and many feature tests.

## `tests/MINLP_simple_grey_box.py`

### Purpose

Grey-box external model implementation used by `MINLP_simple.py` when `grey_box=True`.

### Main contents

- conditional import of external grey-box support (`egb`)
- `GreyBoxModel(egb.ExternalGreyBoxModel)` when available
- `build_model_external(m)` helper that attaches `ExternalGreyBoxBlock`

### Structure and capability demonstrated

Implements:

- named inputs (`X1`, `X2`, `Y1`, `Y2`, `Y3`)
- single output `z`
- output evaluation and exact output Jacobian
- bound/initialization handling in `finalize_block_construction`

This file proves MindtPy can generate OA cuts for supported grey-box outputs.

## `tests/MINLP2_simple.py`

### Purpose

Reimplementation of a classic OA/ECP benchmark example (Duran/Grossmann family).

### Class

- `SimpleMINLP(ConcreteModel)`

### Role in suite

Used in OA/ECP and solution-pool tests.

## `tests/MINLP3_simple.py`

### Purpose

Reimplementation of a Quesada and Grossmann benchmark example.

### Class

- `SimpleMINLP(ConcreteModel)`

### Role

Used in OA / LP-NLP behavior tests and callback-related paths.

## `tests/MINLP4_simple.py`

### Purpose

Example from regularization / second-order OA paper.

### Class

- `SimpleMINLP4(ConcreteModel)`

### Role

Supports regularization feature testing.

## `tests/MINLP5_simple.py`

### Purpose

Another regularization-paper-inspired example.

### Class

- `SimpleMINLP5(ConcreteModel)`

### Role

Used by unit tests (e.g., `add_var_bound` behavior).

## `tests/eight_process_problem.py`

### Purpose

Reimplementation of the eight-process flowsheet benchmark.

### Class

- `EightProcessFlowsheet(ConcreteModel)`

### Role

Larger and more realistic benchmark used across OA/GOA/FP/regularization tests.

## `tests/constraint_qualification_example.py`

### Purpose

Small example illustrating constraint qualification issues.

### Class

- `ConstraintQualificationExample(ConcreteModel)`

### Role

Used in tests related to cycling/regularization robustness.

## `tests/from_proposal.py`

### Purpose

Example model taken from David Bernal's PhD proposal.

### Class

- `ProposalModel(ConcreteModel)`

### Role

Additional OA/ECP/FP coverage model.

## `tests/online_doc_example.py`

### Purpose

Model matching the online MindtPy documentation example.

### Class

- `OnlineDocExample(ConcreteModel)`

### Role

Ensures documentation example stays executable and correct.

## `tests/feasibility_pump1.py`

### Purpose

Feasibility Pump example 1 from the FP MINLP paper.

### Class

- `FeasPump1(ConcreteModel)`

### Role

FP algorithm regression coverage.

## `tests/feasibility_pump2.py`

### Purpose

Feasibility Pump example 2 from the FP MINLP paper.

### Class

- `FeasPump2(ConcreteModel)`

### Role

FP algorithm regression coverage.

## `tests/nonconvex1.py`

### Purpose

Nonconvex benchmark problem A from a separable nonconvex MINLP OA paper.

### Class

- `Nonconvex1(ConcreteModel)`

### Role

GOA/global tests.

## `tests/nonconvex2.py`

### Purpose

Nonconvex benchmark problem B.

### Class

- `Nonconvex2(ConcreteModel)`

### Role

GOA/global tests.

## `tests/nonconvex3.py`

### Purpose

Nonconvex benchmark problem C.

### Class

- `Nonconvex3(ConcreteModel)`

### Role

GOA/global tests.

## `tests/nonconvex4.py`

### Purpose

Nonconvex benchmark problem D.

### Class

- `Nonconvex4(ConcreteModel)`

### Role

GOA/global tests.

## `tests/__init__.py`

### Purpose

Test package marker; no runtime MindtPy logic.

## Solver Regression and Feature Test Files (Per File)

## `tests/test_mindtpy.py`

### Purpose

Primary OA-focused integration regression suite for `SolverFactory('mindtpy')`.

### Class

- `TestMindtPy(unittest.TestCase)`

### What it covers (from test method names)

- OA with `rNLP` initialization
- OA callback path
- OA on an extreme model
- OA with different feasibility norms (`L2`, `L_infinity`)
- OA with `max_binary` and `initial_binary`
- no-good cuts
- quadratic strategy options
- APPSI MIP/NLP solver paths
- `cyipopt` path
- integer-to-binary transform
- objective nonlinear-term partitioning
- OA slack for nonconvex handling
- nonconvex OA heuristic cases
- iteration/time limit termination
- maximization objective
- infeasible model handling

### Why it matters

This file is the broadest feature smoke/integration suite for the general MindtPy interface.

## `tests/test_mindtpy_ECP.py`

### Purpose

ECP strategy regression tests.

### Class

- `TestMindtPy`

### Coverage

- baseline ECP solve
- ECP with `add_slack`

## `tests/test_mindtpy_feas_pump.py`

### Purpose

Feasibility Pump strategy tests.

### Class

- `TestMindtPy`

### Coverage

- FP solves on standard model set
- FP with L1 norm settings
- FP + OA interactions on eight-process problem
- helper `get_config()` for consistent FP settings

## `tests/test_mindtpy_global.py`

### Purpose

GOA/global tests on nonconvex benchmark set.

### Class

- `TestMindtPy`

### Coverage

- GOA solves
- GOA with tabu list behavior

## `tests/test_mindtpy_global_lp_nlp.py`

### Purpose

Global LP/NLP tests for MindtPy (GOA-related branch-and-bound mode).

### Class

- `TestMindtPy`

### Coverage

- GOA / global LP-NLP paths
- tabu list support
- Gurobi-specific path
- MCPP availability interaction

## `tests/test_mindtpy_grey_box.py`

### Purpose

Grey-box OA integration test.

### Class

- `TestMindtPy`

### Coverage

- OA + `cyipopt` + MIP solver on `SimpleMINLP(grey_box=True)`
- solver availability / differentiation / numpy-scipy guards

## `tests/test_mindtpy_lp_nlp.py`

### Purpose

LP/NLP and regularized LP/NLP (single-tree and non-single-tree depending solver path) regression tests.

### Top-level helper

- `known_solver_failure(mip_solver, model)`

### Class

- `TestMindtPy`

### Coverage (from test names)

- LP/NLP with CPLEX and Gurobi
- RLP/NLP regularizers:
  - `L1`, `L2`, `Linf`
  - `grad_lag`
  - `hess_lag`
  - `hess_only_lag`
  - `sqp_lag`

## `tests/test_mindtpy_regularization.py`

### Purpose

ROA regularization regression tests (OA + regularization projection problem).

### Class

- `TestMindtPy`

### Coverage

- ROA with level norms (`L1`, `L2`, `Linf`)
- ROA with lagrangian-based regularizers (`grad_lag`, `hess_lag`, `hess_only_lag`, `sqp_lag`)
- equality relaxation interaction
- no-good cuts interaction
- `level_coef` tuning path

## `tests/test_mindtpy_solution_pool.py`

### Purpose

Solution-pool feature tests for persistent CPLEX/Gurobi MIP solvers.

### Class

- `TestMindtPy`

### Coverage

- OA + solution pool with `cplex_persistent`
- OA + solution pool with `gurobi_persistent`
- additional coverage/behavior checks around pool handling

## `tests/unit_test.py`

### Purpose

Small unit tests for low-level utility functions / helpers.

### Class

- `UnitTestMindtPy`

### Coverage

- `set_var_valid_value`
- `add_var_bound`

## Common Test Suite Patterns (Useful for Understanding Real Usage)

Across the tests, you will repeatedly see:

- `with SolverFactory('mindtpy') as opt:`
- explicit `strategy='OA'|'ECP'|'GOA'|'FP'`
- explicit `mip_solver=...`, `nlp_solver=...`
- optional `calculate_dual_at_solution=True`
- assertions on `TerminationCondition`
- assertions comparing final objective and `optimal_solution` maps in model fixtures

These tests are the best practical examples of how the package is intended to be used.

## Suggested Code Reading Path (File-Oriented)

If you want to understand the whole library by source file:

1. `MindtPy.py`
2. `outer_approximation.py`
3. `algorithm_base_class.py`
4. `cut_generation.py`
5. `util.py`
6. `single_tree.py`
7. `global_outer_approximation.py`
8. `extended_cutting_plane.py`
9. `feasibility_pump.py`
10. `config_options.py`
11. `tests/test_mindtpy.py` and strategy-specific test files
12. model fixtures in `tests/`

