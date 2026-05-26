# Introduction:


# Directory Structure:
```
pump
	-unitcell
		-density_soc
		-wfnfolding
...
```
# Step 0: Check Structures


# Step 1: Folding
The first step is to unfold the k-grid of the SC onto the UC. In this step we will 
1. calculate the UC scf
2. map the SC k-grid to the UC
3. calculate the wfns for each set of UC k points mapped by a SC k point
4. select a range of wfns centered on the fermi band index to use later
5. plot where the selected wfns map to the UC BZ

Steps 2-5 are done using `map.py`.

The directory structure for this step is as follows:

```
unitcell
		density_soc
		wfnfolding
```

In `density_soc` we run the scf calculation (with SOC) for the UC. The contents are

```
scf.in
prefix.save
```

In `wfnfolding` we do the rest.

The contents are
```
prefix.save (copied from density_soc)
nscf.in
layer1.inp
map.inp
kpts_sc
```

## Input:

#### nscf.in:
This is the nscf QE calculation template input file that will be used by `map.py`. 

Ensure that:
1. verbosity = 'high'
2. The K_POINTS section is empty, no white space below, and is in tpiba format. `map.py` will fill out this section with the set of mapped UC k points for each SC k point.
3. nbnd = occupied + a few. Not many conduction bands are needed since a single UC conduction band folds into many SC conduction bands.
4. make sure to link to either pseudopotentials or include them in the directory

#### kpts_sc:
The list of SC k points in crystal coordinates.

Format:
```
 crystal
    [# k points]
  [kpt 1]
  [kpt 2]
  ... 

```


#### layer1.inp:
Contains structural and filename information for UC calculations.
`twist_angle` can be ignored.

Format:
```
# twist angle by which this layer should be rotated
twist_angle = 0.0

# Lattice parameters in Angstrom units
lattice_parameters = [3.15, 3.15,28.1694663881]

# Lattice vectors in units of the lattice
# parameters
lattice_vectors_block
1.0 0.0 0.0
-0.5 0.8660254038 0.0
0.0 0.0 1.0

QE_inp = "nscf.in"
QE_save = "WS2.save"

```


#### map.inp:
Information for running QE nscf calculations on each set of unfolded k points from the SC k grid. The first few lines can be ignored.

`nv_sc` and `nc_sc` are the number of pristine SC valence and conduction bands that will be used to represent the real SC states:
$$
\begin{align}
\ket{\psi_{nk}^{m}} = \sum_{i}^{}a_{i}\ket{\Phi_{ik}^{\text{pristine}}} 
\end{align}
$$

`nF_sc` defines the fermi level band index. The valence and conduction states chosen will be `nF_sc -(+) nv_sc(nc_sc)`

plot_map creates plots of the SC kpts and respective mapped UC kpts in the BZ.

Format:

```
# Number of layers
n_layers = 1
hex_lattice = False
# Specify either mn_value or i_value if hex_lattice = True


# Lattice parameters of the superlattice
lattice_parameters = [3.15, 3.15,28.1694663881]

# lattice vectors of supercell
superlattice_vectors_block
25.0 0.0 0.0
-12.5 21.6506350946 0.0
0.0 0.0 1.0

# Plot the map
plot_map = True

#
run_QE = True 
# Call to Quantum Espresso's pw.x
QE_command = "ibrun pw.x" 
# number of k-point pools for QE runs
nkpools = 9

nv_sc = 36
nc_sc = 36
nF_sc = 16250
```

Optionally, the band mapping can be restricted to specified patches in the UC BZ. In this mode any SC band that would map to a k point outside of these patches is ignored.

This is useful for instance if you are planning to do a patch sampling BSE calculation or want to bypass bands corresponding to dark transitions.

In order to use this mode, include the following in the input file
```
use_kpatch = True

kpatch_radius = 0.26
kpatch_centers_block 2
0.333333333 0.333333333 0.0
0.666666666 0.666666666 0.0
```


## Output:
`map.py` will create a new directory labelled `Layer1`. The contents are

```
Layer1
	QE_ksc_0
	QE_ksc_...
	...
	
```

In the `Layer1` directory, run `plot_mapkv_kc.py` to plot the UC mapping of the chosen SC bands. 

`map_ibv(ibc).npy` are arrays of size SC_k_grid x num_selected_bands and contain the band indices of the selected SC bands in terms of the UC band indices. Typically many SC bands map to multiple k points in one or few UC bands. 

`map_Ev(Ec).npy` are the energies of these SC bands. 

`map_kv(kc).npy` are the UC k points of these bands

`kuc_map_crys(tpba).npy` give the mapped UC k points for every SC k point in crystal or tpiba form.

`E_map.npy` gives the calculated UC band energies from the nscf at every mapped UC k point for every SC k point

`Unique_k_crys(tpba).npy` ...........

`layer.pkl` .........

`fullkgrid_crys(tpba).dat` entire mapped SC to UC mapped k grid.

# Step 2: WFN_fullgrid

exit `wfn_folding` and within the `unitcell` dir create `BSE_FR`, inside that create `wfn_fullgrid`.

Here we do a QE nscf run for the full mapped UC k grid. We do this so that all wavefunctions can be generated with the same gauge.

copy the SCF save directory and `nscf.in` used in `wfn_folding` to this directory, then replace the k points in `nscf.in` with those from `fullkgrid_crys/tpba.dat` and run.

then, use `pw2bgw.x` in QE to convert to a BGW `WFN` file. 

Format:
```
&input_pw2bgw
   prefix = 'WS2'
   real_or_complex = 2
   wfng_flag = .true.
   wfng_file = 'WFN'
   wfng_kgrid = .true.
   wfng_nk1 = 75
   wfng_nk2 = 75
   wfng_nk3 = 1
   wfng_dk1 = 0.0
   wfng_dk2 = 0.0
   wfng_dk3 = 0.0
/
```

After this, the WFN then needs to be converted to h5 format. To do this, simply run `wfn2hd5.x BIN wfn_2v2c.cplx WFN_FullGrid.h5`

*make to to `module load phdf5/1.14.0` or whichever version you have compiled with BGW*

### Optional: wfn_modify
This WFN will be used for Kernel calculations, which only need the relevant bands for transitions. Therefore, one can optionally cut this WFN file using `wfn_modify.x` or `wfn_modify_spinor.x` in BGW. 

These scripts are not automatically available, and require recompiling the MeanField/Utilities dir in BGW. The instructions for this are elsewhere.

This program requires an input file,
Format:
```
wfn.cplx # name of wfn file
wfn_2v2c.cplx # name for output wfn file
1 # not sure
6084 # number of kpts in full grid
12 # min band index
15 # max band index
1 # range of bands within min/max that are valence, here it's 12 and 13 so 1-2
2 
```


# Step 3: Projection:
We want to take a dot-product projection between the constructed wfns from the pristince SC wfns and the actual SC wfns. 

There are multiple steps to this:
1. Obtain DM from SIESTA for SC
2. Obtain wfns from SIESTA for SC, selected wfns are those which are to be represented by pristine SC 
3. Convert wfns with `siesta2bgw.py` into numpy files
*pp.py here somwhere???*
4. Convert numpy files with `npy2wfn.py` 
5. Run `dotprod.py` 

For now I will assume it is known how to do steps 1 through 3 

`npy2wfn.py` should be run in the same directory as `siesta2bgw.py`
Format:
```
# NPY2WFN.PY INPUT FILE
prefix        = 'WSe2'
savedir       = '.'   # input/output file will be read/saved in {savedir}/Siesta2bgw.save/
noncolin      = True  # True for non-colinear calc.
mnband        = 8    # number of bands
nvband        = 4    # number of valance bands
ecutwfc       = 40.0  # in Ry
ecutrho       = ecutwfc*4  # Default
fftgrid       = [75, 75, 192]
fn_kpt        = "kpts.txt"  # k-points in kgrid.out format (= QE format)
wfng_dk       = [0.0, 0.0, 0.0]
wfng_nk       = [2, 1 , 1  ]
nosym         = True  # Currently only support uniform k-grid with no-symmetry
wfn_out       = "./WFN.h5"
```

In `Siesta/WFN_GK` create the new directory `DP`.

Link  `WFN_FullGrid.h5` from `wfn_fullgrid` to `WFN_bot.h5`
Link  `kpts_sc` from `wfn_folding` to the same

Run `dotprod.py`

Format:
```
nk_moire, nb_v_moire, nb_c_moire
2 4 4

Super-cell vectors wrt bottom layer
3 0
0 3

nb_v_pris_bot, nb_c_pris_bot
4 4

nb_v_pris_top, nb_c_pris_top
0 0

path_to_collect_folder_bot
/global/homes/m/mitn/scratch/Moire/PUMP_tutorial/MonoWSe2_3x3/WFNFolding_new/Gamma/Layer1

project_only_bot
True

path_to_moire_wfn
/global/homes/m/mitn/scratch/Moire/PUMP_tutorial/MonoWSe2_3x3/Siesta/WFN_GK/Siesta2bgw.save
```

*Make sure nv_v_moire + nv_c_moire = the amount of bands given in npy2wfn.inp*

### Can also do noeh absorption in WFN_GK/Dipole


# Step 4: GetQ:

For a $Q=0$ transition in the SC, unfolding to the UC turns many of these into finite $Q$ transitions. Therefore we must identify the UC finite $Q$ transitions which correspond to the $Q=0$ transition in the SC. Furthermore, conditions require that we only use $Q$ transitions which are all parallel to each other. To do this we use `get_uniqueQ.py`

Go back to the `wfn_folding` directory and create the `GetQ` dir. From `Layer1` copy `layer.pkl$` as `layer_top.pkl`

Then run `get_uniqueQ.py 2`. This is a parallel code, however it is not heavy so 1 node is plenty.

## Output:

(I'm not sure I can explain these outputs well right now)


In the `BSE_FR/wfn_fullgrid` directory, make a new directory called `Gen_shiftQWFN`.

In this directory copy `wfn_fullgrid.h5` from `wfn_fullgrid`, `Q_crys_umk_unique.txt` from `wfn_folding/GetQ`, and `Unique_k_crys.txt` from `wfn_folding/Layer1`.

Then, run the following programs:
`extract_wfn2patch_old.py` and `extract_wfn2patch_Q0_old.py`

These will create ...


# Step 5: Epsilon:
We need to calculate epsilon for the upcoming Kernel calculations. In order to have a well converged epsilon on a sufficiently fine grid, we use a trick to reduce computational time.

The polarizability $\chi$ includes a term $E_{k\text{, }n}$ where $n$ runs over many bands, and $E_{k+q\text{, }n'}$ where $n'$ runs over just the occupied bands. This allows us to instead generate two WFN files for epsilon. The first has all of the necessary conduction bands and is on a coarse k-grid, and the second is on a fine k-grid but only contains occupied states.

In this case, the fine grid is the same grid used in `wfn_fullgrid`. The coarse grid should be a subset of this grid in order for the trick to work. This is because we need $k+q$ to lie on the fine grid for $E_{k+q}$. For instance, my fullgrid is gamma centered and uniform 75x75 so I can use a 15x15 gamma centered uniform grid because 75 is divisible by 15. For moire systems this does not work and you must take a subset directly from the fullgrid.

Three QE calculations are done, one to get WFN_co, one for WFN_fi, and one for WFNq_fi. This third wavefunction is calculated on the entire fullgrid with a small shift (like a 0.01 shift in the x-coord for instance) to handle the $q\to{0}$ case in epsilon. 

Two epsilon calculations are done. One to get epsmat.h5 using WFN_co and WFN_fi, and one to get eps0mat.h5 using WFN_co and WFNq_fi. 

For the first epsilon calculation, the entire q-grid from fullgrid (which should be the same) *check that* is calculated. The gamma point is removed, and all points have the tag `-1` so that they use WFN_fi. 
*it's the same for a uniform gamma-centered grid like mine, if it's a moire system then one grid is twisted so it may not generally be the case*


For the second epsilon calculation, only the shifted gamma point is calculated, with the tag `1` as usual to indicate it's a $q\to 0$ point.

The actual names of the WFN files for both cases should be `WFN, WFNq` 
