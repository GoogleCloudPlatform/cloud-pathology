# Spec: Digital Pathology WSI Transformation Pipeline to DICOM

The Digital Pathology Whole Slide Imaging (WSI) Transformation Pipeline serves
as a robust and scalable solution, to convert a wide array of proprietary
digital pathology image formats into DICOM. This innovative approach paves the
way for vendor-neutral advantages through image standardization. This paper
outlines the pipeline's capabilities and configurability.

## Input

Inputs are dropped into a dedicated Google Cloud Storage (GCS) bucket. This can
be done using a managed service or other GCP methods. The transformation
pipeline detects new inputs in the buckets, GCS pub/sub notification, then
triggers and scales as needed to perform the image transformation.

### Primary Inputs Formats (Support Pixel Equivalent Ingestion)

_Leica SVS_ - Accepts Leica ScanScope Virtual Slide (SVS) format.
_Tiled TIFF_ - Supports Tiled Tagged Image File Format (TIFF) as an input.
_DICOM ZIP bundle_ - Allows inputting a bundle of DICOM images representing
a single slide in a ZIP archive.
_JPEG_ - Accepts standard JPEG images.
_PNG_ - Supports Portable Network Graphics (PNG) format as input.

### Secondary Inputs Formats (supported by openslide)

_Hamamatsu (.ndpi, .vms, .vmu), Leica (.scn), MIRAX (.mrxs), Philips
(.tiff), Sakura (.svslide), Trestle (.tif), Ventana (.bif, .tif)_ -
Compatible digital slide files from those scanning systems.

### Supported Metadata Inputs

_CSV metadata file_ - with custom metadata fields indexed by a unique ID
(Barcode Value).
_Unique ID_ - per slide either provided by the filename of the input or a
barcode in the slide label image.

## Output

### DICOM Store

The primary output of the transformation is a collection of DICOM WSI files.
These files are then stored in the Cloud Healthcare API DICOM store. The DICOM
store implements the DICOMWeb API standard for standards based access to these
DICOM assets.

_Study / Series_ - organization structure. The system organizes the
transformed DICOM WSI output using the DICOM standard  _Study Instance UID_
and _Series Instance UID_ hierarchical identification mechanisms. DICOM
Study Instance UID should define clusters of slide imaging for a single
patient, e.g., patients imaging from a single exam. Study Instance UID can
be defined within input metadata or  generated automatically by the
pipeline. The Series instance UID identifies unique acquisitions of the
slide. It is recommended that the ingestion pipeline is configured to
automatically assign Series Instance UID.

_Pixel equivalent_ - Pixel equivalent ingestion is supported for preferred
formats. For these formats, imaging is transformed into DICOM that
maintains, where possible, pixel equivalents between the input imaging and
generated DICOM. Specifically, when enabled pixel equivalent transformation
means no lossy imagery recompression will occur at highest magnification
image. When not used all generated images will undergo at most a single
decompression/re-compression operation.

_ICC Profile support_ - the original scanner or acquisition device has
likely generated a color profile which is important for digital pathology
because they help ensure that colors on the screen match those that were
seen by the acquisition device. The transformation pipeline picks up any
ICC profile embedded in the input and stores it according to the DICOM standard.

_Pyramid generation_ - DICOM represents WSI imaging as an image pyramid. The
pyramid levels are analogous to magnification levels and enable faster
visual navigation and facilitate multi-scale analysis. The original scale,
typically 40X produces gigapixel images which are difficult to handle as a
single image encoding. Having lower magnification is convenient for viewer
tools, machine learning and AI applications. The precise pyramid levels
(downsamples) generated by the pipeline are configurable to enable users to
optimize precisely which magnification levels are stored.

### Structure Logs

The transformation pipeline utilizes GCP structured logs to allow tracking and
monitoring of the process. Structured logs are also meant to be read by
machines, the machines that read them can perform searches on them faster,
produce cleaner output, and be used for dashboards. Each input is assigned an ID
that can be tracked throughout the process and flagged in case of error. All
DICOM images generated by the pipeline contain a DICOM tag that provides the ID
for the logs which generated the image.

### Success / Failure Destinations

Ingested files which triggered the pipeline are never deleted by the pipeline.
At the conclusion of ingestion input files are copied or moved, configurable, to
a destination in GCS based on their success or failure code. This makes it easy
to review failures and reprocess them as needed. All files written to the GCS
success/failure metadata buckets have metadata which identifies the structured
logs which resulted in the files placement in the bucket.

## Special Cases

Input Duplication - if the same input is provided after a transformation
already occurred the system will not transform again that input.
Rescan - if the same slide is scanned more than once, the transformation
pipeline will create a new series for each scan.

## Configurations

### Identification of Slide Metadata:

Ingestion requires that all slides have defined metadata. Metadata can be 
described in either a BigQuery Table (recommended) or one or more CSV files
placed in the metadata ingestion bucket. The metadata associated with a WSI
image is identified by identifying the metadata primary key of the ingested
image. There are multiple mechanisms through which a slide's metadata primary
key can be determined.

_Filename:_
    _Filename to slide ID rule_ - a regular expression that identifies the
    slide id from the file name of an input. This gives the flexibility in case
    the file names are externally generated and may contain more information
    than just the slide id.

_Barcode:_
    _ID from Barcode_ - To identify the slide ID, use a barcode decoder. The
    slide label may contain a QR code or barcode that contains the slide ID. If
    the feature is enabled, the barcode will be used as a secondary ID method
    after the file name-based extraction.  Supported Barcode formats: UPC-A,
    UPC-E, EAN-8, EAN-13, Code 39, Code 93, Code 128, ITF, Codabar,  RSS-14
    (all variants), QR Code, Data Matrix, Aztec, PDF 417, and MaxiCode.

_DICOM Barcode Value Tag:_
    _Ingested DICOM can define image BarcodeValue in a root level BarcodeValue
    tag._

### DICOM Slide Metadata

DICOM images (instances) describe pixel level metadata and high level (e.g.,
patient, diagnostic level) metadata. Pixel level metadata is generated by the
transformation pipeline. Higher level metadata is provided by writing metadata
to a Big Query table (Recommended) or placing one or more CSV files in the
metadata GCS bucket.

Column header names in the metadata table identify  row values. All slide
metadata values are required to be formatted in conformance with the
requirements defined by DICOM Standard for the tag the value is being mapped to
(see metadata mapping schema). Blank or missing metadata values should be
described in Big Query or CSV tables as empty field values.

Required metadata fields (all slides):

-  Metadata Primary Key: The transformation pipeline uses the value defined
    in the environmental variable METADATA_PRIMARY_KEY_COLUMN_NAME, default
    value ‘Bar Code Value', to identify the metadata primary key values used to
    join metadata with imaging. Metadata primary key column is required to be
    defined for every slide.

-  Study Instance UID:  All DICOM imaging is required to define a valid
    Study Instance UID. The Study Instance UID defines logical collections of
    slides for a single patient, e.g., all slides from the same case, exam, or
    experiment. The transformation pipeline has three ways to define the Study
    Instance UID for imaging 1) definition in ingestion metadata, 2) pipeline
    initialization via definition of secondary study level metadata, and 3)
    where applicable the re-use of values in ingested DICOM imaging.

Option 1: Definition of the Study Instance UID in metadata enables users
of the system to directly control the definition of the Study Instance UID
for each ingested asset. Metadata, schema and value tables are  required to
define a valid DICOM UID for slides ingested using this system.

Option 2: Transformation pipeline definition, the transformation pipeline
can be configured to auto-generate Study Instance UID for ingested slides
using the slide Accession Number metadata. This mechanism can be used in
conjunction with option 1 and requires  DICOM metadata define a value for
the Accession Number for all ingested slides that do not define the Study
Instance UID. If ingested imaging lacks a Study Instance UID the
transformation pipeline will pace the ingested imaging within the study
with matching accession number or, if one is not present, allocate a new
Study Instance UID. Accession numbers are required to have a 1-to-1 mapping
with Study Instance UID.

Option 3: When DICOM imaging is being transformed, the pipeline can be
configured to either re-define the Study Instance UID in the ingested
imaging using values defined using options 1 and/or 2, or to re-use the
value stored within the ingested DICOM.

Optional metadata:

-  Zero or more additional columns can be added to the metadata value table
    (Big Query or CSV) to describe additional metadata field values.

Metadata Mapping Schema: Metadata mapping schema's are *.json files which
describe where within a DICOM instance a metadata value should be placed.

### WSI Image Generation

The way pixel data is stored in the DICOM store can be configured for
magnification, tiles sizes, and image compression.

_Downsampling_ - Magnification pyramid layers are generated based on rules
provided in a pixel spacing config. The pipeline always generates and
stores the highest magnification image and thumbnail version of the image.
The generation of other downsampled representations are configurable
through a config which maps an input image's pixel spacing to one or more
downsampled representations.

_WSI tiling_ - The pipeline can be configured to change the size of tiles
or frames in a WSI image from the scanner acquired dimensions to a
preferred size. Image re-tiling is not compatible with pixel equivalent
transformation and is automatically disabled when pixel equivalent
transformations are used.

_Image compression_ - The compression format (lossy jpeg or lossless
jpeg2000) used to save generated images can be configured as can the
quality settings used to save imaging. When pixel equivalent
transformations are used this parameter has no effect on pyramid layers
which are generated with pixel equivalence; the compression used for these
frames will match what was used in the source imaging.

### Generation of Non-Tiled Microscope Images

The pipeline supports transforming untitled images (e.g. PNG, JPG, or
non-tiled TIFF) into non-tiled DICOM images. The Barcode Value for these
images is defined using the filename mechanisms only.

## Limitations

iSyntax is currently not supported.

Input DICOM must be well formed.

As of June 2023, the maximum file size limit on the highest magnification
images is 20 GB. Increasing the maximum file size may be requested.
