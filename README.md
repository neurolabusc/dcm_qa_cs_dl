## About

`dcm_qa_cs_dl` is a simple DICOM to NIfTI validator script and dataset. Some of these images are enhanced using [compressed sensing](https://en.wikipedia.org/wiki/Compressed_sensing) or [deep learning](https://en.wikipedia.org/wiki/Deep_learning). Each manufacturer uses their own algorithms, and these samples allow users to determine the vendor specific private DICOM tags.

## Siemens

These images demonstrate Compressed Sensing (CS) and the Deep Learning based [Deep Resolve Gain (DRG), Sharp (DRS) and Boost (DRB)](https://marketing.webassets.siemens-healthineers.com/43ac1c8627df5a23/9121a0dc7e9e/siemens-healthineers-mr-deep-resolve-family-infographic.pdf) filters. The images were acquired using a Siemens 1.5T Sola running XA51 and were provided by Paul S Morgan (University of Nottingham).

For Siemens, CS is currently only available for 3D turbo spin echo acquisitions. It is listed on the console as a iPAT (integrated Parallel Acquisition Techniques) method. So the iPAT options `CS`, `GRAPPA` or `mSENSE` are mutually exclusive.

Each series is saved as both enhanced and classic DICOM format.

 - `Si_2_t1_space_sag_cs4` : CSx4, TA 570.7s
 - `Si_3_t1_space_sag_NOcs` : no CS, TA 160.4s
 - `Si_4_t2_tse_tra_p2_DR_off` : GRAPPAx2 no DR, TA s
 - `Si_5_t2_tse_tra_p2_DR_on_BoostMidSharp` : GRAPPAx2, DRB, DRS, TA 98.7s
 - `Si_6_t2_tse_tra_p2_DR_on_BoostHighSharp` : GRAPPAx2, DRB, DRS, TA 98.7s
 - `Si_7_t2_tse_tra_p2_DR_on_Gain4_2_Sharp` : GRAPPAx2 no DRG, DRS, TA 98.7s
 - `Si_8_t2_tse_tra_p2_DR_on_Gain2_5_Sharp` : GRAPPAx2 no DRG, DRS, TA 98.7s

Detecting compressed sensing requires reading the private CSA Series Header Info (0021,1019). This will list a paradoxical combination where the total acceleration factor is greater than the product of in-plane and between-plane acceleration factors:

```
sPat.lAccelFactPE	 = 	1
sPat.lAccelFact3D	 = 	1
sPat.dTotalAccelFact	 = 	4.0
```

which dcm2niix will translate to the BIDS field:

```
"CompressedSensingFactor": 4,
```

There are several forms of deep learning, using the `Deep Resolve` branding. Deep Resolve is only available for 2D acquisitions. I have seen three of these reported by XA51 in private tag (0021,1175): Deep Resolve Gain (DRG), Deep Resolve Boost (DRB), Deep Resolve Sharp (DRS). I have yet to see examples of `Deep Resolve Swift Brain`, so dcm2niix will not detect this.

For example, an image with Gain of 4 and Sharp of 2 will report:

```
(0021,1175) CS [ORIGINAL\PRIMARY\M\DRG\NORM\DRS\DIS2D]
(0021,1176) LO [ChannelMixing:ND=true_CMM=1_CDM=1\ACCAlgo:9\IterativeDenoising:RelStrength=0.850_MeanRelRisk=1.489\NormalizeAlgo:PreScan\EdgeEnhancement_2]
```

which dcm2niix will translate to BIDS (though note that the `DR` flags will appear in `ImageType` for classic DICOMs and `ImageTypeText` for enhanced DICOMs):

```
"ImageTypeText": ["ORIGINAL", "PRIMARY", "M", "DRG", "NORM", "DRS", "DIS2D"],
"DeepLearning": true,
"DeepLearningDetails": "ChannelMixing:ND=true_CMM=1_CDM=1\\ACCAlgo:9\\IterativeDenoising:RelStrength=0.850_MeanRelRisk=1.424\\NormalizeAlgo:PreScan\\EdgeEnhancement_2",
```

## GE

These images demonstrate `HyperSense` Compressed Sensing (CS) as well as [AIR Recon DL](https://arxiv.org/pdf/2008.06559.pdf) Deep Learning reconstruction. These images simulate a GE 3T MR750 running MR30.1 and were provided by Jaemin Shin (GE HealthCare), [see here for more details](https://github.com/mr-jaemin/ge-mri/tree/main/data).

HyperSense (HS) must be used with 3D acquisitions (either gradient echo and spin echo) and [ARC (GRAPPA)](https://mriquestions.com/grappaarc.html) acceleration. It is not compatible with [ASSET (SENSE)](https://mriquestions.com/senseasset.html). ARC can be 1x1 but it’s still required. The acceleration factor is for only the randomly sampled k-space locations. This factor doesn’t account for any ARC acceleration across ky-kz, partial fourier across ky-kz, or elliptical corner cutting typically used in HyperSENSE. Because HyperSense samples a radial (rather than square) k-space an agressive HS factor of 1.5 will reduce acquisition time by more than 2 times (100%*.7/1.5=47%). Note that HS is both in-plane and out-of-plane. Therefore, HS is not a good match for the existing BIDS [ParallelReductionFactorInPlane](https://bids-specification.readthedocs.io/en/stable/glossary.html#objects.metadata.ParallelReductionFactorInPlane) tag.

 - `GE_10_MPRAGE_P2S1H1.24`: CSx1.24
 - `GE_13_Ax_T2_FLAIR_FS_DL_High`: DL high (0.75)
 - `GE_14_Ax_T2_FLAIR_FS_DL_Off`: DL off

[HyperSense will be reported in a DICOM private tag (0043,10b7)](https://github.com/mr-jaemin/ge-mri/tree/main/DICOM#acceleration). For example, a HS of 1.24 will yield:

```
(0043,10b7) LO [1.24\1\10\0]
```

which dcm2niix will translate to the BIDS field:

```
"CompressedSensingFactor": 1.24,
```

[AIR Recon DL will be reported in a DICOM private tag (0043,10CA)](https://github.com/mr-jaemin/ge-mri/tree/main/DICOM#air-recon-dl). DL has three levels (low 0.3, medium 0.5 and high 0.75) and is available for both 2D and 3D acquisitions. For example, a DL of High will yield:

```
(0043,10ca) LO [0.75\High]
```

which dcm2niix will translate to the BIDS field:

```
"DeepLearningDetails": "0.75\\High",
```

## Running

Assuming that the executable dcm2niix is in your path, you should be able to simply run the script `batch.sh` from the terminal.

