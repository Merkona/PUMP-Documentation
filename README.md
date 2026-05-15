# PUMP-Documentation

# Directory Structure:
```
pump
	-unitcell
		-density_soc
		-wfnfolding
...
```

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
	
	
```

