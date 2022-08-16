# HTTF MOOSE-RELAP-7 Model Description

*Contact: Thomas Freyman - Thomas.Freyman@inl.gov  Lise Charlot - Lise.Charlot@inl.gov*

The MOOSE MultiApp system was used to model the PG-26 transient utilizing 3-D heat conduction within the core and 1-D system-level 
fluid flow. Due to radial variations in the HTTF core, the 3-D model allowed for high fidelity analysis of ceramic core block 
and helium coolant temperatures during the transient which a 2-D radial averaged model might not capture. This also allowed 
for direct comparison between data recorded by thermocouples from the experiment and temperatures recorded at the same locations 
within the 3-D model. Using 1-D flow within RELAP-7 allows for the potential coupling with a larger 1-D model of the entire HTTF 
primary and secondary systems. This would allow system-level transient analysis of the entire HTTF while also obtaining 
high-fidelity temperature profiles within the HTTF core. For the sake of this analysis the primary system was not coupled as the 
main purpose was to provide an example of the 3-D heat conduction and 1-D fluid flow coupling methodology.

In this model the MOOSE [heat conduction module](modules/heat_conduction/index.md) operated as the main app while the RELAP-7 
input files were solved as the sub apps. Below is a description of the methodology used to solve the 3-D heat conduction, 1-D 
fluid flow, and the coupling of the two phenomena with the MultiApp system.

# Heat Conduction

The MOOSE [heat conduction module](modules/heat_conduction/index.md) was used to solve heat conduction throughout the HTTF core, 
reflector, core barrel, RPV, and RCCS in 3-D. Surfaces between the core and reflector were merged within the mesh to allow for 
heat conduction across the boundary and differing materials. To facilitate heat transfer from the reflector to the core barrel, 
`GapHeatTransfer` was used. While the reflctor and core barrel blocks both contain side sets in the same location, in reality 
there is a slight gap between the objects at the HTTF. The `GapHeatTransfer` block accurately models this situation by calculating 
radiative heat transfer bewteen the surfaces as well as conduction through the small amount of helium gas present in betweeen the 
surfaces. Similarly, `GapHeatTransfer` is used to solve heat transfer between the outer surface of the core barrel, and the inner 
surface of the RPV, as well as the outer surface of the RPV and inner surface of the RCS. 

Boundary conditions are placed on heat conduction using various coolant temperatures solved in RELAP-7. Multiple 
`CoupledConvectiveHeatFluxBC` were used to place Robin boundary conditions on surfaces across the 3-D mesh allowing for a coupling 
of the convection-diffusion boundaries.

Individual electric heaters were not modeled in this analysis, instead the experimentally recorded power supplied by each 
heater bank was input into the model and converted to a heat flux. Total heater bank output was divided by the number of heating 
elements in each bank, and then divided by the surface area of the core block each element was exposed to. This resulted in a heat 
flux supplied to the HTTF by each individual heater during the transient. A `ParsedFunction` was used to specifiy the axial length 
along the core heater channel the heat flux was present. To apply the heat flux `FunctionNeumannBC` was used as a boundary 
condition on the side sets representing each heater bank. In reality, some heat loss between the heating elements and the core 
occurred, however due to the compact nature of the system this was assumed to be negligible and was ignored. 

# Fluid Flow

## Components

RELAP-7 was used to calculate 1-D fluid flow of the helium coolant in the HTTF core and water in the RCCS. A total of 7 input files 
were created accounting for each type of coolant channel, bypass channel, the upcomer, and the RCCS. In the HTTF there are 3 sizes of 
coolant channels, denoted in this model as small, medium, and large, and 2 sizes of bypass channels denoted as small and large. The 
input files `small_coolant.i`, `medium_coolant.i`, `large_coolant.i`, `small_bypass.i`, and `large_bypass.i` function exactly the 
same with the only difference being the coolant channel diameter. Each input file models a single coolant or bypass channel using 
`FlowChannel1Phase` within the `Components` structure. To initialize the flow channel an inlet was specified using 
`InletMassFlowRateTemperature1Phase`. As the name suggests this required an inlet temperature and mass flow rate. Temperature was 
supplied by a dummy value which was overwritten by the control logic (discussed later) while mass flow rate was supplied by a 
variable calculated at the beginning of the input file. Similarly, to terminate the flow channel, the component `Outlet1Phase` was 
used which required an outlet pressure the channel flows into. Like the inlet temperature, a dummy value was supplied for pressure 
which was overwritten by the control logic system (discussed later). `HeatTransferFromExternalAppTemperature1Phase` coupled the individual flow channels to the HTTF core and activated heat transfer between the two.

## Control Logic

Control Logic within RELAP-7 allows for changing component parameters and setting conditions for given transients. Using `TimeFunctionComponentControl` recorded primary pressure from the PG-26 transient was applied to the pressure variable in `Outlet1Phase`. Similarly, the outlet temperature from the upcomer was applied to the inlet temperature of the channels to 