# DECIDER Cohort Visualization

This repository contains the visualization specification for the
[DECIDER](https://www.deciderproject.eu/) visualization showcased in
"Deciphering Cancer Genomes with GenomeSpy: A Grammar-Based Visualization
Toolkit" by Lavikka, et al. (2024). In addition, this repository contains a
small example data set to demonstrate the visualization.

> [!NOTE]
> This repository is a fork of the original repository published together with
> the GenomeSpy paper:
> https://github.com/HautaniemiLab/genomespy-paper-2024-spec
> It has been updated to use newer GenomeSpy features and the modern GenomeSpy
> visualization schema. The original specification still works.

An interactive version with the full data set is available for exploration at
https://csbi.ltdk.helsinki.fi/p/genomespy-paper-2024/. All visualization and
data processing occur in the web browser, which receives the data as static
files.

This README provides an overview of the visualization specification and how the
data are processed and visualized in GenomeSpy. Detailed description of the
visualization grammar and data processing can be found in the [GenomeSpy
documentation](https://genomespy.app/docs/).

## Running locally

You can alternatively access, explore, and modify the visualization locally:

1. Clone this repository or download it as a
   [zip](https://github.com/HautaniemiLab/genomespy-paper-2024-spec/archive/refs/heads/main.zip)
   archive.

2. Start a local web server in the repository directory. For instance, using
   [Python 3](https://docs.python.org/3/library/http.server.html):

   ```bash
   python3 -m http.server
   ```

3. Open the visualization in a web browser by navigating to
   [http://localhost:8000](http://localhost:8000).

## DECIDER Sample Data

The [`index.html`](index.html) file loads the GenomeSpy App JavaScript bundle
and initializes the visualization with the specification in the
[`spec.json`](spec.json) file. The specification has been split into multiple
files to make understanding, adapting, and reusing easier. These files are
[imported](https://genomespy.app/docs/grammar/import/#urlimport) into the main
specification. The track-based layout, familiar from genome browsers, is
implemented using [vertical
concatenation](https://genomespy.app/docs/grammar/composition/concat/#vertical),
one of the view composition methods provided by GenomeSpy.

In addition to several [annotation](#genome-annotations) tracks,
[`spec.json`](spec.json) displays the ovarian cancer samples as sub-tracks that
can be manipulated interactively through filtering, grouping, etc. The samples
are displayed using the [Sample
View](https://genomespy.app/docs/sample-collections/visualizing/#specifying-a-sample-view),
which applies the same specification to all samples.

### Metadata

The list of samples and their metadata are stored in
[`decider-data/samples.tsv`](decider-data/samples.tsv). Although the GenomeSpy
App provides default scales and color schemes for metadata attributes, some of
them have been customized for better visual presentation. For instance, the
timepoints have been given (subjectively) more meaningful colors:

```jsonc
  "sampleTime": {
    "type": "ordinal",
    "scale": {
      "domain": ["primary", "interval", "relapse"],
      "range": ["#facf5a", "#f95959", "#455d7a"]
    }
  },
```

### Segmented Copy-Number Data

The segmented copy-number data are stored in
[`decider-data/segments.tsv`](decider-data/segments.tsv), which contains one row
per segment per sample. The visualization specification is in the
[`cnv-sements.json`](cnv-segments.json) file, which provides two layers: logR
(Log2 copy number ratio) and LOH (loss of heterozygosity). In addition, both
layers are summarized into a _G score_ and _LOH score_, respectively.

Both layers use the same data but with partially different encodings. However,
the sample identifier and segments' start and end coordinates are common:

```jsonc
  "encoding": {
    "sample": {
      // Using the "sample" field as the sample identifier
      "field": "sample"
    },
    "x": {
      // Using both "chr" and "startpos" fields for the genomic coordinates.
      // GenomeSpy automatically linearizes the coordinates to a single axis.
      "chrom": "chr",
      "pos": "startpos",
      "type": "locus",
      ...
    },
    "x2": {
      "chrom": "chr",
      "pos": "endpos"
    }
  },
```

#### logR

LogR is visualized as a heatmap with a diverging color scale where gray
represents zero, which corresponds to the average ploidy of the sample.
Deletions and amplifications are shown as blue and red, respectively. The values
are read from the `purifiedLogR` field, which represents the logR value after
removing the normal contamination. Thus, it allows for easy comparison of
samples, while still capturing all subclonal copy-number variation.

```jsonc
  "encoding": {
    "color": {
      "type": "quantitative",
      "field": "purifiedLogR",
      "scale": {
        "domain": [-2.5, 0, 2.5],
        "range": ["#0050f8", "#f6f6f6", "#ff3000"],
        "clamp": true
      }
    },
    ...
  },
```

##### G score

G score summarizes the logR values separately for deletions and amplifications,
as descibed in [Beroukhim et al. ( 2007)](https://www.pnas.org/doi/10.1073/pnas.0710052104). It is the first step
in the GISTIC method, which Beroukhim et al. developed to find recurrent
copy-number alterations and driver regions in cancer genomes. However, it is
useful for summarizing the copy-number data for visualization purposes, as it
also facilitates the comparison of different groups of samples.

The G score track in the visualization is implemented using GenomeSpy's [data
transformation](https://genomespy.app/docs/grammar/transform/) pipeline. In
brief, it comprises the following steps:

1. Merge the segments from each sample group into a sorted (by chromosome and
   start position) stream of segments.
2. Using a [filter](https://genomespy.app/docs/grammar/transform/filter/) transform,
   retain only the segments having the absolute value of the logR above a certain
   threshold (e.g., 0.1).
3. Using a [formula](https://genomespy.app/docs/grammar/transform/formula/) transform,
   clamp the logR values to a certain range to avoid extreme values.
4. Split the data stream into two new streams that are filtered separately for
   deletions and amplifications.
5. Use the [coverage](https://genomespy.app/docs/grammar/transform/coverage/)
   transform to calculate a weighted coverage for the segments, using the logR
   value as the weight. This step produces a score that is equivalent to the G score.
6. Finally, divide the coverage values by the number of samples in the group to get
   a normalized value that allows for comparison between groups with different
   numbers of samples.

The logR threshold and logR limits have been implemented as
[parameters](https://genomespy.app/docs/grammar/parameters/) in the
specification to make it easier to adjust them interactively. Moreover, the
specification uses
[templates](https://genomespy.app/docs/grammar/import/#repeating-with-named-templates)
to avoid code duplication – deletions and amplifications are handled separately
and layered into the same view.

For instance, the following code snippet shows how the `threshold` parameter is
defined and used in the G score calculation:

```jsonc
  "params": [
    {
      "name": "threshold",
      "value": 0.5,
      "bind": {
        // Provides the user with a slider to adjust the threshold interactively
        "input": "range",
        "name": "LogR threshold",
        ...
      }
    },
    ...
  },
  ...
  "transform": [
    {
      "type": "filter",
      // Only segments with logR values above the threshold are retained
      "expr": "abs(datum.purifiedLogR) > threshold"
    },
    ...
  ]
```

#### Loss of Hetereozygosity (LOH)

Typically, LOH is visualized as B-allele frequency (BAF) or allelic fraction,
where 0.5 represents the heterozygous state. However, to emphasize the abnormal
state (the loss) and allow for easier comparison of samples, we define LOH as
follows: abs(BAF - 0.5) × 2. Thus, LOH gets values between zero (fully
heterozygous) and one (fully homozygous, _i.e._, all heterozygosity lost).

We encode LOH as the height of translucent gray bars
([`"rect"`](https://genomespy.app/docs/grammar/mark/rect/) marks) overlaid on
the logR heatmap:

```jsonc
  "encoding": {
    "y": {
      "field": "purifiedLoh",
      "type": "quantitative",
      "axis": null,
      "scale": {
        // 0 = fully heterozygous, 1 = fully homozygous
        "domain": [0, 1],
        // Clamp to ensure that the bars do not extend beyound the track height
        "clamp": true
      }
    },
    ...
  }
```

##### LOH score

LOH is summarized in a similar way as the G score. The LOH score is calculated
as a weighted coverage of the `purifiedLoh` values.

### Somatic Mutations

The [`short-variants.json`](short-variants.json) file specifies a visualization
for single and multi-nucleotide variants (SNVs and MNVs). They are visualized
using [`"point"`](https://genomespy.app/docs/grammar/mark/point/) marks, employing
multiple visual channels for different attributes:

1. **Color** encodes the functional category, e.g., missense, stopgain, etc.
2. **Size** encodes the variant allele frequency (VAF) of the mutation.
3. **Shape** encodes the filter status: circle = PASS, cross = all other.

The mutation data is stored as multiple files in the `data` directory. Each file
is named after the patient ID and contains the mutations for all samples of the
respective patient. The VAF of each sample is stored in a sample-specific
column. For instance, the [`decider-data/patient1.tsv`](decider-data/patient1.tsv)
file contains the following columns:

1. CHROM
2. POS
3. REF
4. ALT
5. ID
6. FILTER
7. CADD_phred
8. CLNSIG
9. Func
10. Gene.refGene
11. patient1_p1.AF
12. patient1_r1.AF
13. patient1_r2.AF

Columns 1-10 are common to all samples, while columns 11-13 contain the VAF for
each sample. This "wide" data is transformed into "long" format using the
[`"regexFold"`](https://genomespy.app/docs/grammar/transform/regex-fold/)
transform:

```jsonc
  "transform": [
    {
      "type": "regexFold",
      // Match all columns that end with ".AF"
      // The part before the ".AF" is used as the sample identifier
      "columnRegex": "^(.*)\\.AF$",
      // The value from the matched column is stored in a new "VAF" field
      "asValue": "VAF",
      // The extracted sample identifier is stored in a new "sample" field
      "asKey": "sample"
    },
    ...
```

The data could be alternatively stored in a single file with a row for each
mutation and sample. However, the "wide" format is more compact, avoiding
redundant information, such as genomic coordinates, functional annotations, and
other attributes, which are the same for all samples.

#### Semantic zooming

[Score-based semantic
zooming](https://genomespy.app/docs/grammar/mark/point/#semantic-zoom)
facilitates exploration by displaying only the most important mutations at each
zoom level. The interactive filtering is coupled to the zoom level and is
performed using GPU acceleration to ensure smooth interaction even with large
datasets.

The visualization uses the `"CADD_phred"` field as the semantic score. The
`semanticZoomFraction` property controls the fraction of mutations displayed at
each zoom level. In this case, the fraction can be adjusted using an
interactive slider, which makes the setting available as a parameter called
`semanticZoomFraction`.

```jsonc
  "params": [
    {
      "name": "semanticZoomSlider",
      "value": 0.015,
      "bind": {
        "input": "range",
        ...
      },
    ...
  },
  "mark": {
    "type": "point",
    "semanticZoomFraction": { "expr": "semanticZoomSlider" },
    ...
  },
  "encoding": {
    "semanticScore": {
      "field": "CADD_phred"
    },
    ...
  }
```

## Genome Annotations

Several annotation tracks are included in the visualization to provide context
for the copy-number aberrations and mutations.

### Chromosome Ideogram

The chromosome ideogram (e.g., cytobands) is visualized using the
[`cytobands.json`](cytobands.json) specification. A comprehensively commented
specification is available in the [Annotation
tracks](https://observablehq.com/@tuner/annotation-tracks?collection=@tuner/genomespy)
ObservableHQ notebook.

### Blacklist Regions

The blacklist tracks show regions that were excluded from copy-number
segmentation due to repetitive sequences, mapping artifacts, or other reasons.
Two blacklist are provided.

The two blacklist tracks are bundled together using vertical concatenation. The
additional level of nesting allows the user to toggle the visibility of both
tracks simultaneously.

#### ENCODE Blacklist

ENCODE blacklist contains regions provided in [Ameniya et al.
2019](https://doi.org/10.1038/s41598-019-45839-z). The spec and data are stored
in the following files: [`hg38-blacklist.v2.json`](hg38-blacklist.v2.json) and
[`external-data/hg38-blacklist.v2.tsv`](external-data/hg38-blacklist.v2.tsv).

#### DECIDER Blacklist

DECIDER blacklist covers regions that contain aberrations in at least three
normal samples in the DECIDER cohort. The aberrations may be due to mapping
artifacts, common germline copy-number variations, or other reasons. The spec
and data are stored in the following files:
[`decider-cnv-blacklist.json`](decider-cnv-blacklist.json) and
[`decider-data/decider-cnv-blacklist-v1.tsv`](decider-data/decider-cnv-blacklist-v1.tsv).

### RefSeq Gene Annotation

The RefSeq gene visualization is explained comprehensively in the [Annotation
tracks](https://observablehq.com/@tuner/annotation-tracks?collection=@tuner/genomespy)
ObservableHQ notebook.

### Platinum Resistance Genes

This track visualizes "A highly annotated database of genes associated with
platinum resistance in cancer", introduced in [Huang et al.
2021](https://doi.org/10.1038/s41388-021-02055-2).

The data are stored in the [`platinum_resistance.tsv`](platinum_resistance.tsv)

## License

The visualization specification is public domain under the [CC0 1.0 Universal
(CC0 1.0) Public Domain
Dedication](https://creativecommons.org/publicdomain/zero/1.0/). You are fee to
adapt it for your own data sets.
