This notebook was made originally to plot Color magnitude diagrams for type II supernovas by reading the .h5 output file of the hst123 pipeline (C. D. Kilpatrick, hst123: HST download, alignment, drizzle, and DOLPHOT photometry pipeline, https://github.com/charliekilpatrick/hst123). 
I also added functions to fit isochrones, evolutionary track, estimate progenitor mass, and communicate with DS9 to perform other image checks.

### **Section 1: Setup & Data Input**
* **Purpose**: Load target specific parameters (SN1964H) that everything downstream depends on.
* One "DATA INPUT" cell holds filepath (dolphot .h5 file), distance_mpc and distance_mpc_err (from NED), and PHYSICAL_RADIUS_PC = 300 (the "local environment" aperture, justified by citing Singleton, Maund & Sun 2025, who used the same method on another CCSN).
h5py reads the dolphot photometry file. The certifi/SSL fix cell just patches a macOS cert issue for Gaia/IRSA queries

### **Section 2: CMD Plotting (Color Magnitude Diagram)**
* **Purpose**: Visualize the stars near the SN to pick out the population it likely came from.
* Pulls F555W/F814W magnitudes from the .h5 file, cuts DOLPHOT's "no measurement" 99.99 flags, then progressively adds a radius cut (converts the 300 pc physical aperture to pixels using distance plus ACS/WFC plate scale) and quality cuts (SNR, sharpness, crowding, object type) to clean the sample.
distance modulus is calculated as 5 times log base 10 of distance in parsecs, minus 5. The radial cut is a manual workaround because hst123's built in radius cut isn't working (noted directly in the comments). 
* Reddening E(B-V) is auto queried from IRSA dust maps and applied via ACS/WFC extinction coefficients.

### **Section 3: Isochrone Fitting**
* **Purpose**: Determine the age of the local stellar population.
* Loads a MIST isochrone grid matching host metallicity, shifts it by distance modulus plus extinction, and overlays it on the cleaned CMD. 
Then scans a grid of ages, computes an error weighted chi squared statistic, converts it to a likelihood/PDF, and takes the mode as the best fit age with a 68% credible interval.
* Has a MIST isochrone file reader function and a best age fitting function that gets reused later by the Monte Carlo step.

### **Section 4: Track Interpolation to Progenitor Mass**
* **Purpose**: This is the actual goal, converting the fitted age into a progenitor mass.
* Unzips a MIST evolutionary track bundle (5 to 35 solar masses), reads each track's total lifetime (last row equals core collapse), builds a mass to lifetime curve, and interpolates the mass whose lifetime equals the fitted age.
* Has a MIST track file reader function, and a progenitor mass lookup function that gets reused by the Monte Carlo.
Uncertainty on mass is now done via Monte Carlo sampling from the age PDF, not analytic error propagation. This was a deliberate fix because the analytic method spiked unrealistically where the mass to lifetime curve went flat.
* There's also a coverage check that warns if part of the age PDF falls outside the mass grid's range.

### **Section 5: Distance Monte Carlo**
* **Purpose**: Propagate distance uncertainty into the age and mass results, since earlier steps only reflected photometric scatter.
* Draws 500 trial distances from a Gaussian, refits age and mass for each using the reusable functions above, and combines that spread in quadrature with the photometric only credible interval.
* This means you get two separate errors, photometric versus photometric plus distance.

### **Section 6: DS9 and Gaia Cross Check**
* **Purpose**: Visually verify star selection and aperture placement, and flag foreground Milky Way stars contaminating the "local environment" sample.
* Sends regions live to DS9 via pyds9 (or writes a region file as fallback) showing selected stars, the SN, and multiple candidate aperture radii. Cross matches local stars against Gaia and flags any with a greater than 3 sigma significant parallax, which would rule out host galaxy membership at this distance.
* This is a real physical filter, not just a diagnostic. If stars get flagged as foreground, they arguably should be excluded from the CMD and age fit upstream (but this hasn’t been a problem thus far, more of a precaution).

**Section 7: Diagnostics**
* **Purpose**: Sanity check cells only (chi squared contribution per star, star count funnels for different cuts, peak significance check). Not part of the main result, useful only if you need to defend or debug a specific number in the meeting.
