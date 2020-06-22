Zoner's Half-Life Compilation Tools 2.5.3, CUSTOM BUILD 1.6.1
(readme.txt Revision 13/07/02 Anthony Moore)

Be sure to consult instructions.html for proper usage instructions
Previous versions are available at http://moore.lorikeet.id.au/nulltex/

Change Log:

Verison 1.6.1
 - Fixed issue of all null textured brush mysteriously dissapering (merkaba@onthebog.net)

Version 1.6
 - Included Zipster's max vis distance into hlvis
 - Included fix by hullu for wad.cfg on *nix systems (kijuhe00@students.oamk.fi)
 - All tools now support info_compile_parameters entity
 - Included hlrad support for info_texlights entity
 - Included various hlrad features from Adam Foster (afoster@compsoc.man.ac.uk)

Version 1.5.2
 - Fixed issue of last wadpath ignorance when using mapfile wadpaths (AmericanRPG)
 - Fixed issue of ÿ interfering with wad.cfg
 - wad.cfg should now work from the directory the tools are installed to

Version 1.5.1
 - Fixed Mixed Face Contents error bug when using NULL with water textures (alex.taylor3@ntlworld.com)
 - Attempted fix of compile log output not having linebreaks

Version 1.5
 - Re-wrote support for wad.cfg file
 - Fixed issue of hlcsg.exe crashing under Win2k with a custom wad configuration
 - Wrote in support for automatic wad detection
 - Included Adam Foster's Sky Diffuse hack for hlrad (afoster@compsoc.man.ac.uk)

Version 1.4
 - Wrote in support for the multiple wad configuration file (wad.cfg)
   Consult instructions.html for a detailed usage guide to this feature

Beta 1.3
 - Wrote in support for clipnode economy mode, can be turned off with -noclipeconomy 
   The following will not generate clipnodes:
   + func_illusionarires
   + func_train or func_tracktrain with flag 4 set (not solid) 
   + func_conveyor with flag 2 set (not solid)
   + func_rot_button with flag 1 set (not solid)
   + func_rotating with flag 7 set (not solid)   

Beta 1.2
 - Added in -nonulltex command line switch to turn the feature off
 - Removed clip face stripping
 - Fixed spam debug message in HLCSG

Beta 1.1
 - Removed Skip texture hack 
 - Wrote in functionality for NULL texture
 - Clip texture now works in the same fashion as the NULL texture for backwards compadibility

(wish list, i have to stick this somewhere, so heres good a place as any)

- -texonly compile mode
- one way vis portals
- transparent vis blockers (ie. sky texture without the skyness)
- plane stripping for culled surfaces (? trivial)
- increase floating point capacity