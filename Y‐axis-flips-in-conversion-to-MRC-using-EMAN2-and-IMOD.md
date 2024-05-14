## Introduction

This was prompted by discussion with David Waterman (https://github.com/dagewa) about comments in his recent [paper](https://doi.org/10.1016/j.str.2023.07.004). 

The RELION [source code](https://github.com/3dem/relion) contains an explanation of the problem with MRC files in [lines 227-240](https://github.com/3dem/relion/blob/e5c4835894ea7db4ad4f5b0f4861b33269dbcc77/src/rwTIFF.h#L227-L240) of `src/rwTIFF.h` 

`3dmod` (from [IMOD](https://bio3d.colorado.edu/imod/)) and `e2display.py` (from [EMAN2](https://blake.bcm.edu/emanwiki/EMAN2)) display MRC files with the 1st pixel in the image data as the lower left corner of the image. i.e. data in the file are flipped in Y before display. 

`dials.image_viewer` interprets MRC files following the FEI/ThermoFisher convention - Appendix C [here](https://www.ccpem.ac.uk/downloads/EPU_user_manual_AppendixCfordistribution.pdf) - where the 1st pixel value in the file is shown at the top left corner of the image. This is also the orientation of a TIFF image with the default orientation `TIFFTAG_ORIENTATION = 1`. 

Because of this confusion many cryoEM programs flip the slow axis when converting to/from MRC format. The image before and after conversion appear to have the same orientation but the slow axis has been flipped before writing the file.

## Examples

### TIFF display

I started with a (16 bit) TIFF made in GIMP. Displaying this file with `tifffile` and `matplotlib` shows that 0,0 is the top left corner. 

<img width="323" alt="Original TIFF" src="https://github.com/huwjenkins/em_image_conversions/assets/12660582/780d5e57-8133-489f-8262-33c991601ab1">


Opening this file in `e2display.py` from EMAN2 or `3dmod` from IMOD has 0,0 at the bottom left corner i.e. the image is flipped in Y. 

<img width="265" alt="e2display.py original TIFF" src="https://github.com/huwjenkins/em_image_conversions/assets/12660582/b00df2b0-a673-4fb1-96c5-1874791f835a">

<img width="265" alt="3dmod original TIFF" src="https://github.com/huwjenkins/em_image_conversions/assets/12660582/881a8e00-6baf-4270-a502-3f21e7382ad6">

### MRC display

I converted the TIFF to MRC using the following Python code

```Python
import tifffile
import mrcfile
data = tifffile.imread('arrow.tiff', key=0)

with mrcfile.new('arrow.mrc') as mrc:
  mrc.set_data(data)
  mrc.update_header_from_data()
  mrc.update_header_stats()
```

<img width="322" alt="converted to MRC" src="https://github.com/huwjenkins/em_image_conversions/assets/12660582/a429f86b-e4f1-42d0-88ce-b9c6aa9c6b0a">


As expected `e2display.py` (1st image) and `3dmod` (2nd image) orient the image differently to the order in which the data are written in the file.

<img width="265" alt="e2display.py converted MRC" src="https://github.com/huwjenkins/em_image_conversions/assets/12660582/e5154292-15aa-4f00-8de0-b64b43774b8b">
<img width="267" alt="3dmod converted MRC" src="https://github.com/huwjenkins/em_image_conversions/assets/12660582/899528d6-c2dd-4f91-b743-ddac62ded567">

### Converting TIFF to MRC with `e2proc2d.py` (EMAN2) or `tif2mrc` (IMOD)

I converted the TIFF to MRC using EMAN2:

```
e2proc2d.py arrow.tiff arrow_eman2.mrc
```

or `tif2mrc` from IMOD:

```
tif2mrc arrow.tiff arrow_imod.mrc
```

As can be seen by comparing `arrow.mrc` to either `arrow_eman2.mrc` or `arrow_imod.mrc` both EMAN2 and IMOD flip Y-axis during the conversion. 

<img width="728" alt="Converted MRC files" src="https://github.com/huwjenkins/em_image_conversions/assets/12660582/38dd3703-bd3a-46db-9990-782173af8369">


Note that both `e2display.py` (1st image) and `3dmod` (2nd image) orient the image the same as the original TIFF. 

<img width="265" alt="e2display.py EMAN2 converted MRC" src="https://github.com/huwjenkins/em_image_conversions/assets/12660582/262bcce0-63df-450a-b177-38e57b277eb6">
<img width="267" alt="3dmod IMOD converted MRC" src="https://github.com/huwjenkins/em_image_conversions/assets/12660582/b85a4f50-0d2b-40b9-86cc-aec408d36417">


### Converting MRC to TIFF with `e2proc2d.py` (EMAN2) or `mrc2tif` (IMOD)

I converted the unflipped MRC (`arrow.mrc`) to TIFF using EMAN2:

```
e2proc2d.py arrow.mrc arrow_eman2.tiff
```

or `mrc2tif` from IMOD:

```
mrc2tif arrow.mrc arrow_imod.tiff
```

As can be seen by comparing `arrow.mrc` to either `arrow_eman2.tiff` or `arrow_imod.tiff` again both EMAN2 and IMOD flip Y-axis during the conversion. 

<img width="732" alt="Converted TIFF files" src="https://github.com/huwjenkins/em_image_conversions/assets/12660582/16631522-2bf0-44c3-9f2f-6783905bde42">

## Conclusion

If you convert files in another image format to/from MRC with EMAN2 or IMOD then the MRC file will be written/read so the orientation of the image is preserved when the data are displayed in the order EMAN2 or IMOD interpret MRC files. This leads to a Y-axis flip.

This is by design and the authors of both EMAN2 and IMOD strongly believe that their interpretation of the MRC format is correct:

EMAN2 https://groups.google.com/g/eman2/c/Pk10hlyUiaY/m/F14iEENGAAAJ

IMOD https://forum.image.sc/t/mrc-files-are-read-in-flipped-in-y/2454

I suggest for 3D ED data no-one should use EMAN2 or IMOD to convert images to/from MRC format unless they know the resulting files will be interpreted correctly by downstream programs. 
