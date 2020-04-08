# Extending CCL

Our main goal is to generalise the CCL `Cosmology` object so it can be created from a set of distances and spectra. As we said this should be done in four instalments. I'd suggest that we do each of them as individual PRs (rather than having a single humongous one).

- **Instalment 1:** get CCL to store splines containing distances/H(z)/growth(z) from a set of input arrays.
- **Instalment 2:** the same for the linear power spectrum.
- **Instalment 3:** the same for the non-linear matter power spectrum.
- **Instalment 4:** extend CCL so it can store a set of power spectra between the most relevant perturbations (\phi, \delta etc.).

## Instalment 1
I would do this in the following steps:
1. Modify the `Cosmology` constructor [here](https://github.com/LSSTDESC/CCL/blob/b1218ee9a4358c5e692c1407171cb86b13fda586/pyccl/core.py#L152) so it can take in a set of arrays defining \chi(z), H(z), D(z) and maybe f(z). Get the python-level object to store these arrays (i.e. do not pass them to C yet), and create a new boolean attribute (e.g. `background_on_input` or something like that) that defines whether the distances/growth were passed as input.
2. Change `compute_distances` (see [here](https://github.com/LSSTDESC/CCL/blob/b1218ee9a4358c5e692c1407171cb86b13fda586/pyccl/core.py#L633)) so that it calls the standard distance calculation (`lib.cosmology_compute_distances`, in line 638) if `background_on_input == False`, but calls a new distance calculation in `ccllib`  (which you haven't written yet) if you passed distance arrays when creating the cosmology (i.e. if `background_on_input == True`).
3. Create a C-level function in `ccl_background.c` ([here](https://github.com/LSSTDESC/CCL/blob/master/src/ccl_background.c)) that does the same thing that `ccl_cosmology_compute_distances` does ([here](https://github.com/LSSTDESC/CCL/blob/b1218ee9a4358c5e692c1407171cb86b13fda586/src/ccl_background.c#L444)), i.e. create a bunch of splines, but it creates them from a set of input arrays instead of calculating them.
4. Create a wrapper function in `ccl_background.i` ([here](https://github.com/LSSTDESC/CCL/blob/master/pyccl/ccl_background.i)) that connects the C-level function you just created with with a set of input numpy arrays. This is the function that you'd call in point 2 above.

I'd do this first and then basically repeat the exercise for growth. The python-level function is [here](https://github.com/LSSTDESC/CCL/blob/b1218ee9a4358c5e692c1407171cb86b13fda586/pyccl/core.py#L641), the C-level function whose functionality you need to reproduce is [here](https://github.com/LSSTDESC/CCL/blob/b1218ee9a4358c5e692c1407171cb86b13fda586/src/ccl_background.c#L631), and the wrapping function that helps pass the numpy arrays should be created [here](https://github.com/LSSTDESC/CCL/blob/master/pyccl/ccl_background.i).

## Instalment 2
Here's a proposal:
1. Create an attribute of the `Cosmology` object called `linear_power_on_input` that defaults to `False`.
2. Create a method `Cosmology.set_linear_power_from_arrays` that takes three arrays: `a_arr`, `k_arr` and `pk_arr`, with shapes `(n_a)`, `(n_k)` and `(n_a, n_k)` respectively. `pk_arr[i_a, i_k]` should contain the linear matter power spectrum at scale k=`k_arr[i_k]` and at scale factor a=`a_arr[i_a]`. These arrays should be stored as attributes of the `Cosmology` object, and the attribute `linear_power_on_input` should be set to `True` after this method is called. Also, at the beginning of the method, you should check for `self.has_linear_power` (the same way you're checking for `self.has_distances` in `set_background_from_arrays`).
3. Change the name of the `Cosmology.compute_linear_power` method [here](https://github.com/chrgeorgiou/CCL/blob/7214e78498e67d3816a6b63d98db078fec89993e/pyccl/core.py#L714) into `Cosmology._compute_linear_power_internal`.
4. Create a method `Cosmology._compute_linear_power_from_arrays()`. This function should create a `Pk2D` object from the arrays stored after calling `set_linear_power_from_arrays`, and then pass them to C by calling `lib.cosmology_compute_linear_power` as done [here](https://github.com/chrgeorgiou/CCL/blob/7214e78498e67d3816a6b63d98db078fec89993e/pyccl/core.py#L756) (note that, in this line, `psp` is the corresponding `Pk2D` object).
5. Recreate the method `Cosmology.compute_linear_power` that you deleted in step 3. Now, if `linear_power_on_input` is True, then this should call `_compute_linear_power_from_arrays`. Otherwise, it should cal `_compute_linear_power_internal`.

Additional steps:

6. Create a new `transfer_function_t` element [here](https://github.com/LSSTDESC/CCL/blob/f6a2fd7feeb382c5f44ab7fa9b8d2b7991c13719/include/ccl_config.h#L22) called `ccl_external_pklin` or something like that.
7. Make a copy of the function `ccl_compute_nonlin_power_from_f2d` [(this one)](https://github.com/LSSTDESC/CCL/blob/f6a2fd7feeb382c5f44ab7fa9b8d2b7991c13719/src/ccl_power.c#L475) and call it `ccl_compute_linear_power_from_f2d`. Within it, change `p_nl` to `p_lin`.
8. Add one more case to `ccl_cosmology_compute_linear_power` after the `ccl_boltzmann_camb` case [(here)](https://github.com/LSSTDESC/CCL/blob/f6a2fd7feeb382c5f44ab7fa9b8d2b7991c13719/src/ccl_power.c#L456) for the `ccl_external_pklin` case you created in step 6. Within it, call the `ccl_compute_linear_power_from_f2d` function you created in step 7.
9. Declare the `ccl_external_pklin` type at the python level [here](https://github.com/LSSTDESC/CCL/blob/f6a2fd7feeb382c5f44ab7fa9b8d2b7991c13719/pyccl/core.py#L21).
10. When you pass external P(k) arrays [(here)](https://github.com/chrgeorgiou/CCL/blob/001dd5e0fa1d7f6bf9c136f02e536620206f9180/pyccl/core.py#L1011), check that `self._config_init_kwargs['transfer_function'] == 'external_pklin'` (or whatever you've decided to call it), and raise an exception otherwise. This means that users will need to set the `transfer_function` argument to `'external_pklin'` whenever they create a cosmology object if they're gonna provide the power spectrum externally and otherwise things will fail when they do so.

## Instalment 3

## Instalment 4
