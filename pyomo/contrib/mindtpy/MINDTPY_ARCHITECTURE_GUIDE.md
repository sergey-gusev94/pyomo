# MindtPy Architecture Guide

## Scope

This document explains `pyomo.contrib.mindtpy` as a package:

- what it is for
- which algorithms it implements
- how the solve flow works end-to-end
- how the major modules collaborate
- what capabilities and runtime modes are supported

Use this together with:

- `MINDTPY_CONFIG_REFERENCE.md` for all configuration options
- `MINDTPY_FILE_REFERENCE.md` for file-by-file details (including tests)

## What MindtPy Is

MindtPy (Mixed-Integer Nonlinear Decomposition Toolbox in Pyomo) is Pyomo's decomposition-based solver framework for MINLPs (Mixed-Integer Nonlinear Programs).

At a high level it solves MINLPs by alternating between:

- a relaxed/discrete master problem (usually MILP/MIQP) and
- nonlinear subproblems (NLPs) with integer variables fixed

It supports multiple decomposition strategies through a shared base engine and strategy-specific subclasses.

## Supported Algorithms (from code)

The package supports these strategy names in `config.strategy`:

- `OA` = Outer Approximation
- `ECP` = Extended Cutting Plane
- `GOA` = Global Outer Approximation
- `FP` = Feasibility Pump

Related variants are implemented as modes of the OA/GOA engines via configuration:

- ROA (Regularized OA) via `add_regularization`
- LP/NLP branch-and-bound via `single_tree=False` OA loop behavior
- RLP/NLP via OA + `add_regularization`
- GLP/NLP via GOA + single-tree / no-good-cut behavior

Solver registrations:

- `mindtpy` (dispatcher/meta solver)
- `mindtpy.oa`
- `mindtpy.ecp`
- `mindtpy.goa`
- `mindtpy.fp`

## Package Design (Layered)

### 1. Public entry and registration

- `MindtPy.py` registers `mindtpy` and dispatches to the strategy solver selected by config.
- `plugins.py` imports all strategy modules so Pyomo `SolverFactory` registrations are available.

### 2. Strategy subclasses

- `outer_approximation.py`
- `global_outer_approximation.py`
- `extended_cutting_plane.py`
- `feasibility_pump.py`

These subclasses mostly override:

- `CONFIG`
- `check_config()`
- `initialize_mip_problem()`
- `add_cuts()`
- sometimes `MindtPy_iteration_loop()` / termination logic

### 3. Shared engine

- `algorithm_base_class.py` contains `_MindtPyAlgorithm`

This file owns most runtime behavior:

- model cloning and reformulation
- solve orchestration
- initialization strategies
- main MIP and NLP subproblem solves
- bound updates and termination checks
- regularization workflow
- feasibility pump loop
- result loading and reporting

### 4. Cut and callback infrastructure

- `cut_generation.py`: OA, ECP, no-good, affine (MCPP) cuts
- `single_tree.py`: CPLEX/Gurobi lazy callbacks for single-tree LP/NLP and GOA
- `tabu_list.py`: CPLEX incumbent callback for tabu-list rejection of repeated integer solutions

### 5. Utility and config support

- `util.py`: Jacobians, norm objectives, Lagrangian regularization objectives, solver option adapters, epigraph reformulation, variable copying, result setup
- `config_options.py`: all config blocks and option declarations

## End-to-End Solve Flow (What Actually Happens)

### A. `SolverFactory('mindtpy').solve(...)` dispatches

`MindtPySolver.solve()` in `MindtPy.py`:

1. Builds the generic MindtPy config (`_get_MindtPy_config()`)
2. Reads `strategy`
3. Dispatches to the registered strategy solver name from `_supported_algorithms`

So `mindtpy` is a thin selector, not the algorithm implementation.

### B. Strategy solver class runs `_MindtPyAlgorithm.solve()`

Core steps in `algorithm_base_class.py`:

1. Build strategy-specific config (`self.CONFIG`) from kwargs/options
2. Normalize/adjust config (`check_config()` + subclass overrides)
3. Clone original model into `working_model`
4. Create ordered bookkeeping block `working_model.MindtPy_utils`
5. Initialize subsolvers (MIP/NLP, optional regularization MIP)
6. Validate model / handle LP-only or NLP-only shortcut cases
7. Set up result object and logging/reporting
8. Reformulate objective if nonlinear (epigraph reformulation)
9. Build `mip` clone and `fixed_nlp` clone
10. Run initialization strategy (`rNLP`, `max_binary`, `initial_binary`, `FP`)
11. Enter main algorithm loop (or FP loop for pure FP)
12. Load best solution back to original model
13. Compute primal/dual integrals and finalize results

## The `MindtPy_utils` Block (Important Internal Structure)

MindtPy adds an internal utility block named `MindtPy_utils` on the working/cloned models. It stores both metadata and generated modeling components.

Key contents created/populated by the base class:

- Ordered lists used for consistent value transfer across clones:
  - `variable_list`
  - `discrete_variable_list`
  - `continuous_variable_list`
  - `constraint_list`
  - `linear_constraint_list`
  - `nonlinear_constraint_list`
  - `objective_list`
  - `grey_box_list` (if external grey box support is available)
- `cuts` block:
  - `no_good_cuts`
  - plus strategy-specific lists like `oa_cuts`, `ecp_cuts`, `aff_cuts`
  - optional `slack_vars` for OA slack cuts
- `feas_opt` block for feasibility-NLP reformulation:
  - slack variables
  - feasibility constraints
  - `feas_obj`
- Objective reformulation artifacts (if objective is moved):
  - `objective_value` (epigraph slack vars)
  - `objective_constr`
  - replacement linearized/epigraph objective

This structure is the backbone of how MindtPy shares state across:

- working model
- MIP master clone
- fixed-NLP clone
- relaxed-NLP clone (`rnlp`)
- FP-NLP clones

## Objective Handling and Reformulation

MindtPy supports nonlinear objectives by moving them into constraints using epigraph reformulation.

Main behavior:

- If objective is nonlinear (or `move_objective=True`), the original objective is deactivated.
- Epigraph slack variables and epigraph constraints are added.
- A replacement objective minimizes or maximizes the slack sum.

Special logic included in code:

- Optional partitioning of nonlinear sum objectives (`partition_obj_nonlinear_terms`) to create multiple epigraph terms.
- MCPP/FBBT-based bounds for epigraph slack variables (`use_mcpp`, `compute_bounds_on_expr`).
- ROA/RLP-NLP special handling to avoid using a flat epigraph objective for branch-and-bound decisions.

## Main Algorithm Cycle (OA/GOA Default Pattern)

The default `MindtPy_iteration_loop()` (base class) does this repeatedly:

1. Solve main MIP/MIQP (`solve_main()`)
2. Handle MIP termination condition and update dual bound
3. Run post-main callback hook
4. Optionally solve regularization projection problem (ROA/RLP-NLP)
5. Check termination (gap, limits, cycling, stalling)
6. Solve fixed-NLP for current integer solution (unless single-tree callback mode handles it)
7. Handle NLP result:
   - optimal/feasible -> update primal bound, add cuts, store incumbent
   - infeasible -> solve feasibility NLP, add cuts
   - other -> limited fallback handling (e.g., max iterations)
8. Repeat

### Bound semantics

- `dual_bound`: comes from relaxed/main MIP or relaxed NLP results
- `primal_bound`: comes from feasible fixed-NLP solutions
- `abs_gap`, `rel_gap`: updated when bounds improve

MindtPy also records:

- bound progress over time
- primal/dual integrals
- primal-dual gap integral

## Initialization Strategies

Supported initialization modes (depending on strategy):

- `rNLP`: solve integrality-relaxed NLP, optionally generate initial OA cuts
- `max_binary`: solve a MILP maximizing active binaries to get a starting discrete point
- `initial_binary`: use user-provided initial discrete values and solve fixed NLP
- `FP`: run feasibility pump initialization / loop

Why initialization matters:

- It seeds cuts and/or a first incumbent.
- It can strongly affect convergence and robustness.

## Strategy-Specific Behavior

## OA (`outer_approximation.py`)

Primary subclass for:

- OA
- ROA
- LP/NLP
- RLP/NLP

Notable overrides:

- `check_config()` sets regularization-related flags and single-tree constraints
- `initialize_mip_problem()` precomputes Jacobians and creates `oa_cuts`
- `add_cuts()` uses `add_oa_cuts()` and grey-box OA cuts if needed
- `objective_reformulation()` customizes epigraph handling for regularized variants

## GOA (`global_outer_approximation.py`)

Global variant using affine McCormick-style cuts (via MCPP).

Notable behaviors:

- forces/adjusts config for global behavior (`use_mcpp=True`, `use_fbbt=True`, no equality relaxation)
- enables no-good cuts by default if tabu/no-good not already enabled
- adds `aff_cuts` instead of OA cuts
- custom primal-bound bookkeeping for bound-fixing after no-good cuts

## ECP (`extended_cutting_plane.py`)

ECP differs structurally from OA:

- typically does not solve fixed-NLP every iteration
- solves main problem and directly adds ECP cuts from current nonlinear violations
- terminates when nonlinear constraints are satisfied within `ecp_tolerance`

It overrides:

- iteration loop
- termination logic (`all_nonlinear_constraint_satisfied()`)
- initialization to avoid OA cuts in `rNLP`

## FP (`feasibility_pump.py` + base-class FP methods)

The FP subclass is intentionally thin because most FP logic lives in base class methods:

- `fp_loop()`
- `solve_fp_main()`
- `solve_fp_subproblem()`
- `handle_fp_*`

The subclass mainly:

- sets FP config defaults (`iteration_limit=0`, `move_objective=True`)
- prepares OA-cut infrastructure/Jacobians for cut transfer
- leaves main iteration loop empty (`pass`) because FP runs through initialization/base flow

## Cut Families and Where They Are Added

## OA cuts (`cut_generation.add_oa_cuts`)

Linearizations of nonlinear constraints using Jacobians and current point values.

Supports:

- inequality cuts (lb/ub sides)
- equality relaxation using dual sign logic
- optional OA slack variables (`add_slack`)
- lazy callback insertion for Gurobi persistent single-tree
- epigraph/objective-constraint linearization

## Grey-box OA cuts (`add_oa_cuts_for_grey_box`)

Uses external grey-box Jacobian output evaluations to construct OA cuts for `ExternalGreyBoxBlock` models.

## ECP cuts (`add_ecp_cuts`)

Uses Jacobians plus constraint slacks to cut violated or active nonlinear constraints under ECP logic.

## No-good cuts (`add_no_good_cuts`)

Prevents revisiting the same binary assignment.

Notes from code:

- Requires binary variables (integer-to-binary transform may be used)
- Validates binary values are near 0/1
- Can also be lazily added in single-tree callbacks

## Affine cuts (`add_affine_cuts`)

GOA/global support via MCPP convex/concave relaxations.

- computes convex/concave slopes and intercepts
- filters invalid `nan`/`inf` cuts
- adds lower/upper affine under/over-estimators

## Single-Tree Mode (Lazy Callback LP/NLP)

Single-tree mode is implemented in `single_tree.py` and activated when:

- `single_tree=True`
- MIP solver is `cplex_persistent` or `gurobi_persistent`

Behavior:

- Main MIP branch-and-bound stays active
- When an incumbent integer solution is found, a lazy callback:
  - copies incumbent into fixed-NLP
  - solves fixed-NLP/feasibility NLP
  - adds lazy OA/affine/no-good cuts
  - updates primal/dual bounds
  - may terminate on convergence/time limit

There are separate implementations for:

- CPLEX callback class (`LazyOACallback_cplex`)
- Gurobi callback function (`LazyOACallback_gurobi`)

Important practical code constraints:

- single-tree forces `threads=1` (callbacks conflict with multithread mode)
- only persistent CPLEX/Gurobi solvers are supported

## Tabu List Mode

`tabu_list.py` provides a CPLEX incumbent callback that rejects repeated discrete incumbents.

Use case:

- avoid cycling / revisiting same integer assignments without explicit no-good cuts in some workflows

Code behavior:

- callback reads current discrete incumbent values
- rejects incumbent if integer tuple already seen
- in single-tree mode, callback rejects incumbents directly

## Utility Module Responsibilities (`util.py`)

`util.py` is a broad support module. Main categories:

### Derivatives and Jacobians

- `calc_jacobians()` precomputes symbolic Jacobians for nonlinear constraints

### Feasibility subproblem construction

- `initialize_feas_subproblem()` creates feasibility-slack constraints/objective (L1/L2/Linf)

### Distance / regularization objectives

- `generate_norm1_objective_function()`
- `generate_norm2sq_objective_function()`
- `generate_norm_inf_objective_function()`
- `generate_lag_objective_function()` (gradient/Hessian/SQP-Lagrangian regularization)

### Solver interface adaptation

- `update_solver_timelimit()` maps remaining time to solver-specific option names
- `set_solver_mipgap()` maps MIP gap options by solver backend
- `set_solver_constraint_violation_tolerance()` maps NLP feasibility tolerance/warm-start settings

### Value transfer / solver pool helpers

- `copy_var_list_values()`
- `copy_var_list_values_from_solution_pool()`
- `set_var_valid_value()`
- `get_integer_solution()`

### Objective reformulation and reporting

- `epigraph_reformulation()`
- `setup_results_object()`

### Feasibility Pump support

- `fp_converged()`
- `add_orthogonality_cuts()`
- `generate_norm_constraint()`

### Gurobi callback integration

- `GurobiPersistent4MindtPy` wraps `GurobiPersistent` to route callback signature/data expected by MindtPy

## Configuration System (High-Level)

The config surface is defined in `config_options.py` with grouped helper builders.

Major categories:

- common algorithm behavior (`strategy`, limits, callbacks, cycling, objective handling)
- OA/GOA/ECP-specific behavior
- cut/slack behavior
- subsolver selection and solver args
- tolerances
- bounds and fallback bounds
- feasibility pump options
- regularization (ROA/RLP-NLP) options

See `MINDTPY_CONFIG_REFERENCE.md` for the complete option-by-option reference.

## Capabilities Present in This Codebase (from implementation + tests)

The package implements and tests support for:

- OA, ECP, GOA, FP strategies
- ROA / regularized LP-NLP variants with multiple regularizers:
  - `level_L1`, `level_L2`, `level_L_infinity`
  - `grad_lag`, `hess_lag`, `hess_only_lag`, `sqp_lag`
- LP/NLP and single-tree LP/NLP (lazy callbacks)
- GOA affine cuts via MCPP
- no-good cuts and tabu-list cycling control
- solution pool usage (CPLEX/Gurobi persistent)
- objective epigraph reformulation and nonlinear objective partitioning
- quadratic strategy modes (`quadratic_strategy` 0/1/2)
- multiple subsolver backends (including persistent and APPSI variants)
- grey-box external models (tested via OA + cyipopt + MIP solver)
- feasibility pump cut transfer and orthogonality cuts
- callback hooks before/after main/NLP solves

## Important Behavioral Notes / Caveats (from code)

- MindtPy clones the model and transfers values by ordered component lists; preserving deterministic ordering is a core design constraint.
- If there are no discrete variables, MindtPy short-circuits and uses the configured NLP or LP solver directly.
- Multiple active objectives are rejected.
- Some features force config changes:
  - `use_tabu_list` forces `cplex_persistent`
  - `solution_pool` forces a persistent solver (prefers CPLEX persistent if unsupported)
  - `single_tree` forces `threads=1`
- Bound fixing may run an extra solve when no-good cuts / tabu list distort the final dual bound.
- FP subclass relies on base-class initialization/FP loop; its own `MindtPy_iteration_loop()` is intentionally empty.

## Reading Order Recommendation

If your goal is to fully understand the codebase quickly, read in this order:

1. `MindtPy.py` (dispatcher)
2. `outer_approximation.py` and `global_outer_approximation.py` (strategy flavor)
3. `algorithm_base_class.py` (core engine)
4. `cut_generation.py` (how cuts are formed)
5. `single_tree.py` (callback mode)
6. `util.py` (helper mechanics)
7. `config_options.py` (option surface)
8. `tests/` (practical coverage and usage examples)

