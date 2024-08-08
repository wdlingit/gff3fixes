# gff3fixes

Collection of notes on *may-be-incorrectness* and *fixes* of GFF3 files.

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
