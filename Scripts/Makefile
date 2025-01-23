#===============================================================================
SHELL := /bin/bash

# PATHS
DATA := data
SCRIPTS := /data/projects/emulsionSynth/scripts

# PROGRAMS
BBDUK := bbduk.sh
BBMERGE := bbmerge.sh
BBMAP := bbmap.sh

# DATA

# LIBs to MAP
LIBNAMES := 15-HiFi 16-HiFi

# VARS
THREADS ?= 20 # how many threads to run the BB* portion of pipelione


#===============================================================================

# RECIPIES
all: map
map: $(addprefix pipeline/, $(addsuffix .trans-map.txt, $(LIBNAMES)))

clean:
	rm -f pipeline/*

.PRECIOUS: $(addprefix pipeline/, %.map.csv %.merge.fastq %.perf.txt %.merge.txt)

#===============================================================================
# BARCODE MAPPING

# trim adapter sequences
pipeline/%.trim.fastq: adapters_mi3.fasta $(DATA)/%_R1.fastq $(DATA)/%_R2.fastq
	@echo "Trimming - $(word 2, $^) and $(word 3, $^)"
	@$(BBDUK) ref=$< \
	    in1=$(word 2, $^) \
	    in2=$(word 3, $^)\
	    out=$@ \
	    k=21 \
	    mink=8 \
	    hdist=2 \
	    hdist2=1 \
	    ktrim=r \
	    tpe=t \
	    tbo=f \
            tossjunk=t \
	    -Xmx1g \
	    overwrite=t \
	    stats=$(@:.fastq=.err) \
	    threads=$(THREADS) 2>> $(@:.fastq=.err)

# merge reads
pipeline/%.merge.fastq: pipeline/%.trim.fastq
	@echo "Merging - $<"
	@$(BBMERGE) in=$< \
	    outm=$@ \
	    interleaved=t \
	    threads=$(THREADS) \
	    2> $(@:.fastq=.err)

# concat reads together and map barcodes
pipeline/%.map.csv: pipeline/%.merge.fastq
	@echo "Barcode mapping - $<"
	@python $(SCRIPTS)/bc-map.py $< \
	    -v \
	    -j35 \
	    -s-20 \
	    --min-reads 1 \
	    -b $(@:.map.csv=.bad-bcs.txt) > $@ \
	    2> $(@:.csv=.err)

#-------------------------------------------------------------------------------
# calculate perfects by translating then mapping
# will output map in form: bc variant seq
pipeline/%.trans-map.txt: pipeline/%.map.csv %.trans.fasta
	@echo "Mapping protein sequences for - $<"
	@awk -F, '{print ">"$$1"\n"$$2}' $< | \
	    python $(SCRIPTS)/translate.py - \
	    -v \
	    -j35 \
	    -b $(@:.trans-map.txt=.bad-trans.csv) \
	    2> $(@:.txt=.err) | \
	    paste - - | \
	    sort -k2 > reads.tmp
	@cat $(word 2, $^) | \
	    paste - - | \
	    sort -k2 > ref.tmp
	@join -j2 -o 1.1,2.1,0 -e NA -a1 reads.tmp ref.tmp | \
	    sed 's/>//g' > $@
	@awk '{if($$2 == "NA") bad++; else good++ store[$$2]++} END \
	    {print "Mapped " good " BCs (" length(store) " unique variants)" \
	    "\nCould not map " bad " BCs"}' \
	    $@ >> $(@:.txt=.err)
	@rm -f ref.tmp reads.tmp

# trim the common sequence up until the stop codon
%.trans.fasta: %.fasta
	@echo "Translating reference sequence - $<"
	@python $(SCRIPTS)/translate.py $< > $@

#-------------------------------------------------------------------------------

# one-off recipie for s0/s0.5/s1 mapping
#pipeline/CatLib.map.csv: $(addprefix pipeline/, $(addsuffix .merge.fastq, $(CATREADS)))
#	@echo "Catting together - $^"
#	@echo "Barcode mapping - $(@:.map.csv=.merge.fastq)"
#	@cat $^ | python $(SCRIPTS)/bc-map.py - \
#	    -v \
#	    -j35 \
#	    -s-20 \
#	    --min-reads 1 \
#	    -b $(@:.map.csv=.bad-bcs.txt) > $@ 2> $(@:.csv=.err)


