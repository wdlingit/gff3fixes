# gff3fixes

Collection of notes on *may-be-incorrectness* and *fixes* of GFF3 files.

Currently I have a number of tools for checking/fixing GFF3 files:
1. misc.GffTree (java): Parse a GFF3 file and report its feature hierarchy. Detects records with duplicated IDs and records with non-available parents.
2. misc.CanonicalGFF (java): Reads a GFF3 and report gene regions and exon regions (by merging all gene models). The gene region part would reflect gene records in the GFF3 file.
3. misc.ModelCGFF (java): Read a GFF3 and report (i) gene regions and exon regions (by merging all gene models) in a CGFF file, and (ii) model regions and exon regions in a model file. The gene region part would be concluded from corresponding exon regions.
4. GFF3checker.pl: multi-function script: (i) check redundancy (overlapping gene records) (ii) sort GFF3 (iii) remove/extract records fron the GFF3 object hierarchy (iv) merge records for duplicated gene annotations (v) check whether exons expand the gene region correctly
5. GFFExtractor.pl: extract specified records of specified features
6. cgff2GFF3.pl and model2GFF3.pl: transform a CGFF or model file into GFF3

Commonly seen fix operations would be listed in this page. Fixes of complex examples would be written in separate pages.

## 1. Different CDS or exon records with the same ID

Usually seen in GFF3 downloaded from Ensembl and NCBI.

An example fix for CDS records by removing their ID attributes.
```
$ cat sequence.gff3 | perl -ne 'if(/^#/){ print }else{ chomp; @t=split(/\t/); if($t[2] eq "CDS"){ $t[8]=~s/ID=.+?;// } print join("\t",@t)."\n"; }' > sequence.fix1.gff3
```

## 2. Non-coordinated feature hierarchy

In the following example, it would be better to have CDS corresponding to exons. A desirable hierarchy should be *gene-transcript-CDS* but not just *gene-CDS*. Usually seen in GFF3 downloaded from NCBI for small genomes.
```
$ cat sequence.fix1.gff3.features
GffRoot
  region*
  gene*
    CDS
    tRNA*
      exon*
    ncRNA*
      exon*
    rRNA*
      exon*
    hammerhead_ribozyme*
      exon*
    tmRNA*
      exon*
    RNase_P_RNA*
      exon*
    SRP_RNA*
      exon*
  pseudogene*
    CDS
  riboswitch*
  sequence_feature*
```

The fix is to insert dummy *transcript* records as children of *genes* and parents of *CDS*.
```
$ (cat sequence.fix1.gff3 ; echo "ROUND 1 END"; cat sequence.fix1.gff3) | perl -ne '
    chomp; 
    if(/^ROUND 1 END/){ 
        $flag=1; next 
    } 

    if(not $flag){ 
        chomp; 
        @t=split(/\t/); 
        $t[8].=";" if $t[8]!~/;$/; 
        if($t[2] eq "CDS"){ 
            $t[8]=~/Parent=(.+?);/; 
            $hash{$1}=1; 
        } 
    }else{ 
        chomp; 
        s/^\s+|\s+$//g; 
        next if length==0; 

        if(/^#/){
        }else{ 
            @t=split(/\t/); 
            $t[8].=";" if $t[8]!~/;$/; 
            if($t[8]=~/ID=(.+?);/ && exists $hash{$1}){ 
                print "$_\n"; 
                $t[2]="transcript"; 
                $id=$1; 
                $trans="$1-transcript"; 
                $t[8]="ID=$trans;Parent=$id"; 
            }elsif($t[8]=~/Parent=(.+?);/ && exists $hash{$1}){ 
                $id=$1; 
                $t[8]=~s/Parent=$id;/Parent=$id-transcript;/; 
            } 
            $_=join("\t",@t); 
        } 
        print "$_\n" 
    }
' > sequence.fix2.gff3
```

The fixed feature hierarchy.
```
$ cat sequence.fix2.gff3.features
GffRoot
  region*
  gene*
    transcript*
      CDS
    tRNA*
      exon*
    ncRNA*
      exon*
    rRNA*
      exon*
    hammerhead_ribozyme*
      exon*
    tmRNA*
      exon*
    RNase_P_RNA*
      exon*
    SRP_RNA*
      exon*
  pseudogene*
    transcript*
      CDS
  riboswitch*
  sequence_feature*
```
