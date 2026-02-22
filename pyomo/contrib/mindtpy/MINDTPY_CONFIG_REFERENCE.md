# MindtPy Config Reference

## Scope

This document explains the configuration system in `pyomo.contrib.mindtpy.config_options` and the runtime config adjustments performed by the solver classes.

It covers:

- config constructors for each registered solver
- grouped option declarations
- defaults and intent of each option
- important runtime overrides / interactions from `check_config()` methods

## Algorithm Mapping (`_supported_algorithms`)

Defined in `config_options.py`:

- `OA` -> `mindtpy.oa` (`Outer Approximation`)
- `ECP` -> `mindtpy.ecp` (`Extended Cutting Plane`)
- `GOA` -> `mindtpy.goa` (`Global Outer Approximation`)
- `FP` -> `mindtpy.fp` (`Feasibility Pump`)

This is used by `MindtPy.py` (the `mindtpy` dispatcher) to select the concrete solver.

## Config Constructors (Which Option Groups Each Solver Gets)

### `_get_MindtPy_config()` (dispatcher/general)

Includes groups:

- common
- subsolver
- tolerances
- feasibility pump
- bounds
- regularization (ROA)

### `_get_MindtPy_OA_config()`

Includes groups:

- common
- OA-specific
- ROA-specific
- feasibility pump
- OA-cuts
- subsolver
- tolerances
- bounds

### `_get_MindtPy_GOA_config()`

Includes groups:

- common
- GOA-specific
- OA-cuts (shared cut/slack options)
- subsolver
- tolerances
- bounds

### `_get_MindtPy_ECP_config()`

Includes groups:

- common
- ECP-specific
- OA-cuts (shared slack/linearization flags)
- subsolver
- tolerances
- bounds

### `_get_MindtPy_FP_config()`

Defines `init_strategy='FP'` first, then includes:

- common
- feasibility pump
- OA-cuts
- subsolver
- tolerances
- bounds

## Option Groups (Full Reference)

## Common Options (`_add_common_configs`)

These are the main behavior switches for most MindtPy workflows.

- `iteration_limit` (default `50`): Maximum main iterations.
- `stalling_limit` (default `15`): Stop if primal bound has not improved for this many iterations.
- `time_limit` (default `600`): Total wallclock budget (seconds).
- `strategy` (default `OA`): One of `OA`, `ECP`, `GOA`, `FP`.
- `add_regularization` (default `None`): Enables ROA/RLP-NLP regularization objective type.
  - Allowed values: `level_L1`, `level_L2`, `level_L_infinity`, `grad_lag`, `hess_lag`, `hess_only_lag`, `sqp_lag`
- `call_after_main_solve` (default `_DoNothing()`): Callback hook after each main MIP solve.
- `call_before_subproblem_solve` (default `_DoNothing()`): Callback hook before each NLP subproblem solve.
- `call_after_subproblem_solve` (default `_DoNothing()`): Callback hook after each NLP subproblem solve.
- `call_after_subproblem_feasible` (default `_DoNothing()`): Callback hook after a feasible fixed-NLP solve.
- `tee` (default `False`): MindtPy logging to terminal (not subsolver logs).
- `logger` (default `pyomo.contrib.mindtpy`): Logger instance/name.
- `logging_level` (default `logging.INFO`): Numeric Python logging level.
- `integer_to_binary` (default `False`): Apply integer-to-binary transformation (useful for no-good cuts on non-binary integers).
- `add_no_good_cuts` (default `False`): Add cuts that exclude previously seen binary assignments.
- `use_tabu_list` (default `False`): Use CPLEX incumbent callback to reject repeated integer solutions.
- `single_tree` (default `False`): Use lazy-callback single-tree LP/NLP workflow.
- `solution_pool` (default `False`): Use MIP solution pool (persistent CPLEX/Gurobi).
- `num_solution_iteration` (default `5`): How many pool solutions to process per main iteration.
- `cycling_check` (default `True`): Detect repeated integer assignments and terminate.
- `feasibility_norm` (default `L_infinity`): Feasibility-NLP slack objective norm (`L1`, `L2`, `L_infinity`).
- `differentiate_mode` (default `reverse_symbolic`): Jacobian differentiation mode (`reverse_symbolic`, `sympy`).
- `use_mcpp` (default `False`): Use MC++ bounds for epigraph slack variables / global cuts support.
- `calculate_dual_at_solution` (default `False`): Import/use duals from NLP solutions for OA/equality relaxation.
- `use_fbbt` (default `False`): Run FBBT before solving.
- `use_dual_bound` (default `True`): Add main-problem constraint enforcing best known dual bound.
- `partition_obj_nonlinear_terms` (default `True`): Partition nonlinear objective sums into multiple epigraph terms.
- `quadratic_strategy` (default `0`): How quadratic terms are treated in the main problem.
  - `0`: treat quadratics as nonlinear
  - `1`: allow quadratic objective terms only
  - `2`: allow quadratic objective and constraints
- `move_objective` (default `False`): Force objective epigraph reformulation even if objective is linear.
- `add_cuts_at_incumbent` (default `False`): In single-tree mode, add lazy cuts at incumbent callback point.

## OA-Specific Options (`_add_oa_configs`)

- `heuristic_nonconvex` (default `False`): Enables nonconvex OA heuristics by combining equality relaxation + slack usage.
- `init_strategy` (default `rNLP`): OA initialization strategy.
  - Allowed: `rNLP`, `initial_binary`, `max_binary`, `FP`

## OA/ECP Shared Cut Options (`_add_oa_cuts_configs`)

Used by OA, GOA, ECP, FP configs.

- `add_slack` (default `False`): Add slack variables to OA/ECP cuts (nonconvex handling support).
- `max_slack` (default `1000.0`): Upper bound for OA cut slack variables.
- `OA_penalty_factor` (default `1000.0`): Penalty multiplier for OA cut slacks in main objective.
- `equality_relaxation` (default `False`): Use dual-based OA cuts for equality constraints.
- `linearize_inactive` (default `False`): Also linearize inactive nonlinear constraints.

## GOA-Specific Options (`_add_goa_configs`)

- `init_strategy` (default `rNLP`): GOA initialization strategy.
  - Allowed: `rNLP`, `initial_binary`, `max_binary`

Note: GOA runtime logic forces additional settings (see Runtime Overrides section).

## ECP-Specific Options (`_add_ecp_configs`)

- `ecp_tolerance` (default `None`): Nonlinear constraint satisfaction tolerance used for ECP stopping.
  - If `None`, runtime defaults to `absolute_bound_tolerance`.
- `init_strategy` (default `max_binary`): ECP initialization strategy.
  - Allowed: `rNLP`, `max_binary`, `FP`

## Subsolver Options (`_add_subsolver_configs`)

### NLP solver selection

- `nlp_solver` (default `ipopt`)
  - Allowed: `ipopt`, `appsi_ipopt`, `gams`, `baron`, `cyipopt`
- `nlp_solver_args` (default empty implicit `ConfigBlock`): Extra options passed to NLP solver.

### MIP solver selection

- `mip_solver` (default `glpk`)
  - Allowed: `gurobi`, `cplex`, `cbc`, `glpk`, `gams`, `gurobi_persistent`, `cplex_persistent`, `appsi_cplex`, `appsi_gurobi`, `appsi_highs`
- `mip_solver_args` (default empty implicit `ConfigBlock`): Extra options passed to main MIP solver.
- `mip_solver_mipgap` (default `1e-4`): Relative MIP gap for main MIP solves.

### Threads / logging

- `threads` (default `0`): Solver thread count (0 = backend default).
- `regularization_mip_threads` (default `0`): Thread count for regularization MIP solver.
- `solver_tee` (default `False`): Convenience switch to enable both MIP and NLP solver output.
- `mip_solver_tee` (default `False`): MIP solver log streaming.
- `nlp_solver_tee` (default `False`): NLP solver log streaming.

### Regularization MIP solver (ROA / RLP-NLP)

- `mip_regularization_solver` (default `None`)
  - Same solver choices as `mip_solver`
  - If `None` and regularization is enabled, OA runtime may set it to `mip_solver`

## Tolerance Options (`_add_tolerance_configs`)

- `absolute_bound_tolerance` (default `1e-4`): Absolute primal-dual gap tolerance.
- `relative_bound_tolerance` (default `1e-3`): Relative primal-dual gap tolerance.
- `small_dual_tolerance` (default `1e-8`): Ignore tiny duals during cut generation.
- `integer_tolerance` (default `1e-5`): Tolerance for integrality checks/rounding.
- `constraint_tolerance` (default `1e-6`): Tolerance for constraint satisfaction / trivial-constraint deactivation.
- `variable_tolerance` (default `1e-8`): Tolerance for variable bounds.
- `zero_tolerance` (default `1e-8`): General near-zero tolerance.

## Bound / Fallback Bound Options (`_add_bound_configs`)

- `obj_bound` (default `1e15`): Artificial bound used if the main MIP becomes unbounded after objective reformulation.
- `continuous_var_bound` (default `1e10`): Added bound for unbounded continuous vars in nonlinear constraints (single-tree support).
- `integer_var_bound` (default `1e9`): Added bound for unbounded integer vars in nonlinear constraints (single-tree support).
- `initial_bound_coef` (default `0.1`): Used in primal/dual integral calculations to synthesize initial bound values.

## Feasibility Pump Options (`_add_fp_configs`)

- `fp_cutoffdecr` (default `0.1`): Relative improvement margin for FP objective cutoff cuts.
- `fp_iteration_limit` (default `20`): Max FP iterations.
- `fp_projcuts` (default `True`): Add orthogonality/projection cuts.
- `fp_transfercuts` (default `True`): Transfer OA/no-good cuts learned during FP back into main OA flow.
- `fp_projzerotol` (default `1e-4`): Threshold for treating projection distance as converged.
- `fp_mipgap` (default `1e-2`): MIP gap for FP regularization MIP solves.
- `fp_discrete_only` (default `True`): Compute FP distances only over discrete vars.
- `fp_main_norm` (default `L1`): Norm for FP-MIP regularization objective (`L1`, `L2`, `L_infinity`).
- `fp_norm_constraint` (default `True`): Add monotonicity norm constraint in FP-NLP.
- `fp_norm_constraint_coef` (default `1`): Coefficient (beta-like parameter) for FP norm constraint.

## Regularization / ROA Options (`_add_roa_configs`)

- `level_coef` (default `0.5`): Weight between primal and dual bounds in level-set regularization estimate.
- `solution_limit` (default `10`): Early stop limit for regularization MIP solutions.
- `sqp_lag_scaling_coef` (default `fixed`): Scaling mode for `sqp_lag` quadratic term.
  - Allowed: `fixed`, `variable_dependent`

## Runtime Config Overrides and Interactions (Important)

The declared defaults are not the whole story. `_MindtPyAlgorithm.check_config()` and subclass `check_config()` methods modify options at runtime.

## Base-class runtime adjustments (`_MindtPyAlgorithm.check_config`)

- If `init_strategy == 'FP'`:
  - forces `add_no_good_cuts = True`
  - forces `use_tabu_list = False`
- If `solver_tee=True`:
  - forces `mip_solver_tee = True`
  - forces `nlp_solver_tee = True`
- If `add_no_good_cuts=True`:
  - forces `integer_to_binary = True`
- If `use_tabu_list=True`:
  - forces `mip_solver = 'cplex_persistent'`
  - forces `threads = 1` (if >1)
- If `solution_pool=True` and solver is unsupported:
  - may force `mip_solver = 'cplex_persistent'`
- APPSI solver handling:
  - disables standard `load_solutions` path for certain APPSI solvers and uses direct value update behavior

## OA subclass runtime adjustments (`MindtPy_OA_Solver.check_config`)

When `add_regularization` is set:

- may force `calculate_dual_at_solution = True` for Lagrangian-based regularizers
- may default `regularization_mip_threads` from `threads`
- may force `add_cuts_at_incumbent=True` in single-tree regularized mode
- may default `mip_regularization_solver = mip_solver`
- computes internal `regularization_mip_type` = `MILP` or `MIQP` based on regularizer type

Single-tree OA restrictions:

- sets `iteration_limit = 1`
- forces `add_slack = False`
- requires `cplex_persistent` or `gurobi_persistent`
- forces `threads = 1`

Nonconvex heuristic shortcut:

- `heuristic_nonconvex=True` implies:
  - `equality_relaxation=True`
  - `add_slack=True`

Other OA interactions:

- `equality_relaxation=True` implies `calculate_dual_at_solution=True`
- `init_strategy='FP'` or any regularization implies `move_objective=True`

## GOA subclass runtime adjustments (`MindtPy_GOA_Solver.check_config`)

GOA enforces global-cut-friendly defaults:

- `add_slack = False`
- `use_mcpp = True`
- `equality_relaxation = False`
- `use_fbbt = True`
- if neither tabu nor no-good is enabled, GOA enables `add_no_good_cuts=True`

Single-tree GOA has the same persistent-solver + thread restrictions as OA.

## ECP subclass runtime adjustments (`MindtPy_ECP_Solver.check_config`)

- If `ecp_tolerance is None`, it is set to `absolute_bound_tolerance`.

## FP subclass runtime adjustments (`MindtPy_FP_Solver.check_config`)

- forces `iteration_limit = 0` (FP runs via FP loop, not base main loop)
- forces `move_objective = True`

## Solver-Backend Option Mapping (implemented in `util.py`)

MindtPy adapts generic options to backend-specific option names.

### Time limits (`update_solver_timelimit`)

Mapped examples:

- CPLEX/Gurobi (+ persistent/appsi variants): `timelimit`
- `appsi_highs`: `opt.config.time_limit`
- `ipopt`: `max_cpu_time`
- `cyipopt`: `opt.config.options['max_cpu_time']`
- `glpk`: `tmlim`
- `baron`: `MaxTime`
- `gams`: appends `Reslim` command to `add_options`

### MIP gap (`set_solver_mipgap`)

Mapped examples:

- many solvers: `mipgap`
- `appsi_cplex`: `mip_tolerances_mipgap`
- `appsi_highs`: `opt.config.mip_gap`
- `gams`: `optcr`

### Constraint violation tolerance (`set_solver_constraint_violation_tolerance`)

Mapped to backend-specific options, including GAMS-generated option files and warm-start settings for IPOPT-based solvers.

## Practical Config Recipes (Common Patterns)

### Standard OA

- `strategy='OA'`
- `init_strategy='rNLP'`
- `mip_solver='cplex'|'gurobi'|'appsi_highs'`
- `nlp_solver='ipopt'|'cyipopt'|'appsi_ipopt'`
- `calculate_dual_at_solution=True` (often useful for OA quality)

### Single-tree LP/NLP (OA or GOA)

- `single_tree=True`
- `mip_solver='cplex_persistent'` or `gurobi_persistent`
- `threads=1` (enforced if larger)

### ROA / RLP-NLP

- `strategy='OA'`
- `add_regularization='level_L2'` (or other supported regularizer)
- optional `mip_regularization_solver=...`
- often `calculate_dual_at_solution=True` (auto-enabled for lag-based modes)

### GOA (global nonconvex-oriented)

- `strategy='GOA'`
- expect `use_mcpp`, `use_fbbt`, and no-good cuts to be enabled/forced by runtime logic

### Feasibility Pump only

- `strategy='FP'`
- tune `fp_iteration_limit`, `fp_main_norm`, `fp_transfercuts`, `fp_projcuts`

## Notes

- `config_options.py` imports `deprecation_warning`, but this file currently does not define active deprecation translations for MindtPy options.
- Some declared options are only meaningful for specific strategies/modes; MindtPy generally tolerates extra options but only uses the relevant subset.

