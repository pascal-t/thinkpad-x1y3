# FreeCAD + PrusaSlicer

https://discourse.flathub.org/t/launching-flatpak-inside-flatpak/5708

## Allow FreeCAD to launch other Flatpaks

`sudo flatpak override org.freecad.FreeCAD --talk-name=org.freedesktop.Flatpak`

## Add Macro `Export2Slicer`

In FreeCAD:

1. Macro > Macros ... > Download: Download and install `Export2Slicer`
2. Tools > Edit parameters ... 
  1. Navigate to: `BaseApp/Preferences/Macros/Export2Slicer`
  2. Set `SlicerCommand` to `flatpak-spawn --host /usr/bin/flatpak run com.prusa3d.PrusaSlicer "{file}"`


