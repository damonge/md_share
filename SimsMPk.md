# Marked Pk simulations

The simulations were run with a box of size L_box=700 Mpc/h, with 512^3 particles. Each sim takes only about 5h to run, so it's actually not that bad (although the cluster is only big enough to run 2 of these at a time, so it's not like we can run 100s of these willy-nilly).

The simulation snapshots are stored at:
`/mnt/extraspace/damonge/MarkedPk/<name>_512/Snap/snap_<name>_512_nside512_seed<seed>_<snapnum>`
where <name> is a unique identifier for each set of cosmological parameters (see below), <seed> is the seed used for the simulation (for now we only have one seed, 101), and <snapnum> is the snapshot number (see below).

## Snapshots

We have shapshots at 6 different redshifts

| Snapnum  | Redshift |
|----------|----------|
| 000      | 0.0      |
| 001      | 0.1      |
| 002      | 0.3      |
| 003      | 0.5      |
| 004      | 1.0      |
| 005      | 2.0      |

## Cosmological parameters

The cosmological parameters of the different sims are as follows

| Name      | Omega_m | sigma8 | h    | n_s  | Omega_b  |
|-----------|---------|--------|------|------|----------|
| fiducial  | 0.3     | 0.8    | 0.7  | 0.96 | 0.05     |
| Om_p      | 0.32    | ""     | ""   | ""   | ""       |
| Om_m      | 0.28    | ""     | ""   | ""   | ""       |
| s8_p      | ""      | 0.82   | ""   | ""   | ""       |
| s8_m      | ""      | 0.78   | ""   | ""   | ""       |
| h_p       | ""      | ""     | 0.75 | ""   | ""       |
| h_m       | ""      | ""     | 0.65 | ""   | ""       |
| n_s_p     | ""      | ""     | ""   | 0.99 | ""       |
| n_s_m     | ""      | ""     | "    | 0.93 | ""       |

## Simulations for covariances

A suite of 100 COLA simulations for the fiducial cosmology were also run to test covariances. The COLA sims use the same box size and number of particles as the other sims, but were only run for two redshifts (z=0, z=1). The snapshots can be found in

`/mnt/extraspace/damonge/MarkedPk/fiducial_512/COLA/seed<seed>/sn_fiducial_512_seed00<seed>_<snapnum>.<part>`

Where `<seed>` goes from 100 to 199, `<snapnum>` is 000 (z=1) or 001 (z=0), and `<part>` is an integer from 0 to 63 (the snapshots are divided into 64 parts - `nbodykit` should still be able to read them).
  
