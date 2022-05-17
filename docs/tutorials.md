# Tutorials

## Basic Seismology

### Particle motion

#### Introduction

Sources generating eleastic waves generate different amplitudes as a function of takeoff angle and azimuth from the source. The convention in seismology is to envision a sphere surrounding the source and the to plot the signal amplitude on the sphere as the point where the ray from the source to the receiver intersects the sphere. Such a plot can be performed for any set of measurements, but typically one plots the P-wave, SV-wave, SH-wave amplitude or the S-wave polarization at this point.  Rather than plotting this on an actual sphere, the upper or lower half of this focal sphere is projected onto a piece of paper tangent to the pole.

The convention of the takeoff angle is that a value of 0 represents a ray going directly downward from the source into the earth, while a value of 180 represents a ray propagating upward. The azimuth is measured with respect to local north, with values of 0, 90, 180, and 270 representing north, east, south and west.

The lower hemisphere is defined by the combination of all azimuths and takeoff angles from 0 to 90 degrees, while the upper hemisphere has takeoff angles of 90 to 180 degrees.

The purpose of this section is to exercise the synthetics seismogram programs to ensure that the conventions used for the Green's functions and the focal mechanism are correct. We will accomplish this by making wholespace synthetics (because of speed) using the programs mkmod96, hprep96, hwhole96, hpulse96, fmech96 and f96tosac .  We then use gsac to convert the cylindrical coordinate traces to spherical radial, longitudinal and latitudinal and to create the plot.  We also use fmplot to plot the theoretical amplitudes on the focal spheres, the program CAL to annotate the plot, plotnps to convert from CALPLOT graphics to Encapsulated PostScript. We use the ImageMagick tool convert, available on LINUX,Cygwin and OSX, to convert the Encapsulated PostScript to a Portable Network Graphics (PNG) file. In addition we use the command gawk (locally called awk) as a calculator.

#### Trace Rotation

The computer programs in seismology package creates Z - vertical, R - radial and T - transverse component synthetics for a cylindrical coordinate system, with the convention that Z is positive up, R is positive away from the source, and T is positive in a direction of increasing azimuth (using the convention above).

On a sphere we want the rotation to create the far-field P, SV and SH waveforms, with P positive in a radial direction, SV positive as a point on the ray moves in the direction of increasing takeoff angle, and SH positive in the direction of increasing azimuth.  The necessary transformation is

      R = Uradial = -UZ*COS(IO) + UR*SIN(IO)
      T = Utheta  =  UZ*SIN(IO) + UR*COS(IO)
      P = Uphi    = UT

where IO is the takeoff angle in radians ( angle in radians = 3.1415926 * angle in degrees / 180)

#### Scripts

The scripts used are as follow
Lower hemisphere script
```
#!/bin/sh

#####
#	shell script to make synthetics for different 
#	takeoff angles for a lower hemisphere
#
#	this is a two step process 
#	a) compute the Green functions
#	b) apply the focal mechanism
#
#	For speed only the wholespace solution is considereed
#####

#####
#	define the basic parameters for the synthetics
#
#####
DT=1.0
NPTS=512
GCARC=30
DIST=`echo ${GCARC} | awk '{print $1*111.195}' `
STK=0
DIP=45
RAKE=45
MW=5.0
T0=`echo ${DIST} | awk '{print $1/10.0 -50.0}' `

#####
#	define the current TOP directory 
#	for safety in case the loops break
#####
TOP=`pwd`

#####
#	create a work directory
#####
if [ ! -d WORK ]
then
	mkdir WORK
fi

cat > simple.mod << EOF
MODEL.01
Simple velocity model
ISOTROPIC
KGS
FLAT EARTH
1-D
CONSTANT VELOCITY
LINE08
LINE09
LINE10
LINE11
 H(KM)  VP(KM/S)  VS(KM/S) RHO(GM/CC)  QP     QS   ETAP ETAS FREFP FREFS
1000.00  10.0000  5.0000  3.3000      0.00   0.00  0.00 0.00  1.00  1.00  
EOF

#####

#	begin IO loop
#####
for IO in 10 30 50 70
do
#####
#	create a result directory for this takeoff angle
#	Note for uniformity of naming, we will make IO two digits, e.g.,
#	00 instead of 0
#####
if [ ! -d SYN${IO} ]
then
	mkdir SYN${IO}
fi
#####
#	for this takeoff angle, define the values of
#	the receiver depth and radius in cylindrical coordinates
#	also define the SIN and COS of the IO angle for use in the
#	transformation to RADIAL and LATITUDINAL
#####
Z=`echo $IO $DIST | awk '{ printf "%10.2f", $2* cos($1 * 3.1415927/180.0) }' ` 
R=`echo $IO $DIST | awk '{ printf "%10.2f", $2* sin($1 * 3.1415927/180.0) }' `
C=`echo $IO  | awk '{ printf "%12.7f",  cos($1 * 3.1415927/180.0) }' ` 
S=`echo $IO  | awk '{ printf "%12.7f",  sin($1 * 3.1415927/180.0) }' ` 

cat > dfile << EOF
${R} ${DT} ${NPTS} ${T0} 0.0
EOF

#####
#	perform the hspec96 run for this value of IO
#####
hprep96 -M simple.mod -d dfile -HR ${Z} -HS 0.0 -EQEX
hwhole96
hpulse96 -p -l 4 -V > file96

#####
#	go to WORK directory to make three component traces
#	
#####
cd WORK

#####
#	begin AZ loop
#####
for AZ in \
	000 030 060 \
	090 120 150 \
	180 210 240 \
	270 300 330
do
rm -f *.sac
cat ../file96 | fmech96 -S ${STK} -D ${DIP} -R ${RAKE} -A ${AZ} -ROT -MW ${MW} | f96tosac -B

#####
#	rename
#####
mv B00101Z00.sac Z0.sac
mv B00102R00.sac R0.sac
mv B00103T00.sac T0.sac

#####
#	now we do the transformation from cylindrical to spherical
#	where IO is the takeoff angle measured from the downward vertical
#
#	Uradial = -UZ*COS(IO) + UR*SIN(IO)
#	Utheta  =  UZ*SIN(IO) + UR*COS(IO)
#	Uphi    = UT
#####
gsac << EOF
r Z0.sac
mul ${S}
w ZS.sac

r Z0.sac
mul ${C}
w ZC.sac
#####
#	for some strange reason I cannot multiply by a -1 in this script
#####

r R0.sac
mul ${C}
w RC.sac

r R0.sac
mul ${S}
w RS.sac
#####
#	now combine
#####
r RS.sac ZC.sac 
subf
mul -1
w  RSmRS.sac Uradial
r ZS.sac RC.sac
addf
w 2ZZ.sac Utheta
cp T0.sac Uphi
r Uradial Utheta Uphi
int
cd ../SYN${IO}
w ${IO}_${AZ}.R ${IO}_${AZ}.T ${IO}_${AZ}.P
quit
EOF



done
#####
#	end AZ loop
#####
cd ${TOP}

#####
#	go to the synthetic directory to make the plots
#####
cd SYN${IO}

gsac << EOF
bg plt
fileid name
pctl xlen 2 ylen 10
ylim all
r *.R
p
r *.T
p
r *.P
p
quit
EOF
rm -f junk
#####
#	use CALPLOT reframe to merge the pictures
#####
reframe -N1 -O -XL1200 -XH4000 -YL1000 -X0+0000 < P001.PLT >> junk
reframe -N1 -O -XL1200 -XH4000 -YL1000 -X0+2200 < P002.PLT >> junk
reframe -N1 -O -XL1200 -XH4000 -YL1000 -X0+4400 < P003.PLT >> junk
#####
#	make the focal mechanism plots and overlay
#####
fmplot -S $STK -D $DIP -R $RAKE -FMPLMN -P  -X0 9.0 -RAD 1.0 -Y0 9.0 -ANN
cat FMPLOT.PLT >> junk
fmplot -S $STK -D $DIP -R $RAKE -FMPLMN -SV -X0 9.0 -RAD 1.0 -Y0 6.0 -ANN
cat FMPLOT.PLT >> junk
fmplot -S $STK -D $DIP -R $RAKE -FMPLMN -SH -X0 9.0 -RAD 1.0 -Y0 3.0 -ANN
cat FMPLOT.PLT >> junk
calplt << EOF
NEWPEN
2
CENTER
9.0 10.5 0.2 "Io=${IO}" 0.0
PEND
EOF
cat CALPLT.PLT  >> junk
cat junk | reframe -N1 -O -X0-1000 |plotnps -F7 -W10 -EPS -K -S0.9  > LH${IO}.eps
convert -trim LH${IO}.eps LH${IO}.png

cd ${TOP}
done
#####
#	end  IO loop
#####

#####
#       clean up
#####
rm -fr WORK
rm -f SYN*/junk SYN*/*PLT SYN*/*.cmd
rm -f hspec96.*
rm -fr dfile file96
```

Upper hemisphere script:

```
#!/bin/sh

#####
#	shell script to make synthetics for different 
#	takeoff angles for a lower hemisphere
#
#	this is a two step process 
#	a) compute the Green functions
#	b) apply the focal mechanism
#
#	For speed only the wholespace solution is considereed
#####

#####
#	define the basic parameters for the synthetics
#
#####
DT=1.0
NPTS=512
GCARC=30
DIST=`echo ${GCARC} | awk '{print $1*111.195}' `
STK=0
DIP=45
RAKE=45
MW=5.0
T0=`echo ${DIST} | awk '{print $1/10.0 -50.0}' `

#####
#	define the current TOP directory 
#	for safety in case the loops break
#####
TOP=`pwd`

#####
#	create a work directory
#####
if [ ! -d WORK ]
then
	mkdir WORK
fi

cat > simple.mod << EOF
MODEL.01
Simple velocity model
ISOTROPIC
KGS
FLAT EARTH
1-D
CONSTANT VELOCITY
LINE08
LINE09
LINE10
LINE11
 H(KM)  VP(KM/S)  VS(KM/S) RHO(GM/CC)  QP     QS   ETAP ETAS FREFP FREFS
1000.00  10.0000  5.0000  3.3000      0.00   0.00  0.00 0.00  1.00  1.00  
EOF

#####

#	begin IO loop
#####
for IO in 10 30 50 70
do
#####
#	create a result directory for this takeoff angle
#	Note for uniformity of naming, we will make IO two digits, e.g.,
#	00 instead of 0
#####
if [ ! -d SYN${IO} ]
then
	mkdir SYN${IO}
fi
#####
#	for this takeoff angle, define the values of
#	the receiver depth and radius in cylindrical coordinates
#	also define the SIN and COS of the IO angle for use in the
#	transformation to RADIAL and LATITUDINAL
#####
Z=`echo $IO $DIST | awk '{ printf "%10.2f", $2* cos($1 * 3.1415927/180.0) }' ` 
R=`echo $IO $DIST | awk '{ printf "%10.2f", $2* sin($1 * 3.1415927/180.0) }' `
C=`echo $IO  | awk '{ printf "%12.7f",  cos($1 * 3.1415927/180.0) }' ` 
S=`echo $IO  | awk '{ printf "%12.7f",  sin($1 * 3.1415927/180.0) }' ` 

cat > dfile << EOF
${R} ${DT} ${NPTS} ${T0} 0.0
EOF

#####
#	perform the hspec96 run for this value of IO
#####
hprep96 -M simple.mod -d dfile -HR ${Z} -HS 0.0 -EQEX
hwhole96
hpulse96 -p -l 4 -V > file96

#####
#	go to WORK directory to make three component traces
#	
#####
cd WORK

#####
#	begin AZ loop
#####
for AZ in \
	000 030 060 \
	090 120 150 \
	180 210 240 \
	270 300 330
do
rm -f *.sac
cat ../file96 | fmech96 -S ${STK} -D ${DIP} -R ${RAKE} -A ${AZ} -ROT -MW ${MW} | f96tosac -B

#####
#	rename
#####
mv B00101Z00.sac Z0.sac
mv B00102R00.sac R0.sac
mv B00103T00.sac T0.sac

#####
#	now we do the transformation from cylindrical to spherical
#	where IO is the takeoff angle measured from the downward vertical
#
#	Uradial = -UZ*COS(IO) + UR*SIN(IO)
#	Utheta  =  UZ*SIN(IO) + UR*COS(IO)
#	Uphi    = UT
#####
gsac << EOF
r Z0.sac
mul ${S}
w ZS.sac

r Z0.sac
mul ${C}
w ZC.sac
#####
#	for some strange reason I cannot multiply by a -1 in this script
#####

r R0.sac
mul ${C}
w RC.sac

r R0.sac
mul ${S}
w RS.sac
#####
#	now combine
#####
r RS.sac ZC.sac 
subf
mul -1
w  RSmRS.sac Uradial
r ZS.sac RC.sac
addf
w 2ZZ.sac Utheta
cp T0.sac Uphi
r Uradial Utheta Uphi
int
cd ../SYN${IO}
w ${IO}_${AZ}.R ${IO}_${AZ}.T ${IO}_${AZ}.P
quit
EOF



done
#####
#	end AZ loop
#####
cd ${TOP}

#####
#	go to the synthetic directory to make the plots
#####
cd SYN${IO}

gsac << EOF
bg plt
fileid name
pctl xlen 2 ylen 10
ylim all
r *.R
p
r *.T
p
r *.P
p
quit
EOF
rm -f junk
#####
#	use CALPLOT reframe to merge the pictures
#####
reframe -N1 -O -XL1200 -XH4000 -YL1000 -X0+0000 < P001.PLT >> junk
reframe -N1 -O -XL1200 -XH4000 -YL1000 -X0+2200 < P002.PLT >> junk
reframe -N1 -O -XL1200 -XH4000 -YL1000 -X0+4400 < P003.PLT >> junk
#####
#	make the focal mechanism plots and overlay
#####
fmplot -S $STK -D $DIP -R $RAKE -FMPLMN -P  -X0 9.0 -RAD 1.0 -Y0 9.0 -ANN
cat FMPLOT.PLT >> junk
fmplot -S $STK -D $DIP -R $RAKE -FMPLMN -SV -X0 9.0 -RAD 1.0 -Y0 6.0 -ANN
cat FMPLOT.PLT >> junk
fmplot -S $STK -D $DIP -R $RAKE -FMPLMN -SH -X0 9.0 -RAD 1.0 -Y0 3.0 -ANN
cat FMPLOT.PLT >> junk
calplt << EOF
NEWPEN
2
CENTER
9.0 10.5 0.2 "Io=${IO}" 0.0
PEND
EOF
cat CALPLT.PLT  >> junk
cat junk | reframe -N1 -O -X0-1000 |plotnps -F7 -W10 -EPS -K -S0.9  > LH${IO}.eps
convert -trim LH${IO}.eps LH${IO}.png

cd ${TOP}
done
#####
#	end  IO loop
#####

#####
#       clean up
#####
rm -fr WORK
rm -f SYN*/junk SYN*/*PLT SYN*/*.cmd
rm -f hspec96.*
rm -fr dfile file96
```





### Geometrical Spreading


## Synthetics


## Surface Waves


## Receiver Functions


## Earth Structure


## Sources


## Seismic Data and Instrumentation
