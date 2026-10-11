# hardware

Schematics and layout land here, drawn by hand in EasyEDA (deliberately not
generated), following the family convention.

- [`BOM.md`](BOM.md) — costed bill of materials, both digit sizes
- Board set per the partitioning in [display-driving.md](../docs/display-driving.md) §7:
  control board, digit boards, 6-wire chain between them

Nothing has been laid out. The first task is not a schematic — it is measuring
the segment forward voltage of real displays, because the sources disagree by
almost 2× and the rail voltage depends on the answer.
