# Phoenix Umbilical Support

Parametric rear umbilical / drag-chain support developed for the Sovol SV08 “Phoenix” project.

This redesign replaces the earlier dual-wall concept with a simpler and stronger single L-shaped structure, improves chain clearance, and keeps the umbilical path away from the rear gantry area.

## Included files

- `Phoenix_Umbilical_Support.FCStd`  
  Parametric FreeCAD source and master design file.

- `Phoenix_Umbilical_Support.step`  
  Neutral CAD exchange format for modification in other CAD systems.

- `Phoenix_Umbilical_Support.3mf`  
  Mesh export suitable for modern slicers.

- `Phoenix_Umbilical_Support.stl`  
  Universal printable mesh.

- `Phoenix_Umbilical_Support_Parameters.csv`  
  Human-readable export of the FreeCAD `Parameters` spreadsheet and the additional parameter properties used by the final model.

## Current status

Mechanical fit has been tested on the Phoenix printer using a PLA prototype.

The prototype confirmed:
- correct gantry mounting position;
- correct umbilical routing;
- correct drag-chain engagement after the final clearance cut;
- functional M4 gantry mounting holes;
- final L-corner 1.2 mm through-holes;
- improved structural transition between the L tower and chain-mount block.

PLA was used only as a rapid mechanical test material.

The final intended material for the installed part is a more temperature-resistant filament such as ASA.

## Notes

This component is designed specifically around the Phoenix hardware configuration and may require adaptation for other SV08 toolhead, cable-chain, enclosure, or umbilical layouts.

The FreeCAD file should be treated as the authoritative source for future modifications.
