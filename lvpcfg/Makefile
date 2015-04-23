################################################################################
##                                                                            ##
##  This file is part of ModelBlocks. Copyright 2009, ModelBlocks developers. ##
##                                                                            ##
##  ModelBlocks is free software: you can redistribute it and/or modify       ##
##  it under the terms of the GNU General Public License as published by      ##
##  the Free Software Foundation, either version 3 of the License, or         ##
##  (at your option) any later version.                                       ##
##                                                                            ##
##  ModelBlocks is distributed in the hope that it will be useful,            ##
##  but WITHOUT ANY WARRANTY; without even the implied warranty of            ##
##  MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the             ##
##  GNU General Public License for more details.                              ##
##                                                                            ##
##  You should have received a copy of the GNU General Public License         ##
##  along with ModelBlocks.  If not, see <http://www.gnu.org/licenses/>.      ##
##                                                                            ##
################################################################################

################################################################################
#
#  i. Macros & variables
#
################################################################################

.SECONDARY:
.SUFFIXES:
.SECONDEXPANSION:

THISDIR := $(dir $(abspath $(word $(words $(MAKEFILE_LIST)),$(MAKEFILE_LIST))))

EDU     := $(THISDIR)edu
FIG     := $(THISDIR)fig
SCRIPTS := $(THISDIR)scripts

BIN      := bin
GENMODEL := genmodel

################################################################################
#
#  ii. User-specific parameter files (not shared; created by default with default values)
#
#  These parameter files differ from user to user, and should not be checked in.
#  This script just establishes 'official' default values for these parameters.
#
################################################################################

#### java compile flags, for berkeley trainer
user-javaflags.txt:
	echo '-Xmx8g' > $@
	@echo ''
	@echo 'ATTENTION: I had to create "$@" for you, which may not be to your liking'
	@echo 'edit it to give java as much memory as you want to, and re-run make to continue!'
	@echo ''

################################################################################
#
#  iii. Code compilation items
#
################################################################################

#### java executable objects
.PRECIOUS: %.class
%.class:  %.java  #$$(addsuffix .class,$$(subst .,/,$$(subst import ,,$$(shell grep -o 'import edu[.a-zA-Z0-9]*' $$(subst .class,.java,$$@)))))
	javac -cp $(dir $(EDU)) $<

#### berkeley parser shortcut "executable"
bin/parser-fullberk:  user-javaflags.txt  $(EDU)/berkeley/nlp/PCFGLA/BerkeleyParser.class  |  bin  #$$(abspath $$(dir $$@))
	echo "#!/bin/sh" > $@
	echo "java $(shell cat $(word 1,$^)) -cp $(dir $(EDU)) edu/berkeley/nlp/PCFGLA/BerkeleyParser -substates -gr \$$1" >> $@
	chmod u+x $@
bin/parser-vitberk:  user-javaflags.txt  $(EDU)/berkeley/nlp/PCFGLA/BerkeleyParser.class  |  bin  #$$(abspath $$(dir $$@))
	echo "#!/bin/sh" > $@
	echo "java $(shell cat $(word 1,$^)) -cp $(dir $(EDU)) edu/berkeley/nlp/PCFGLA/BerkeleyParser -viterbi -substates -gr \$$1" >> $@
	chmod u+x $@

################################################################################
#
#  1. Berkelely LV PCFG trainer
#
#  to construct the following file types:
#    <x>.<n>sm.gr    : serialized java data structures
#
################################################################################

#### add extra paren and change '-' ops to '+' for berkeley trainer
%.extrparplusops.linetrees:  %.linetrees
	cat $<  |  sed 's/^\((.*\)$$/(\1)/g'  |  perl -pe 's/\-(?=[a-z][^\)]* )/+/g'  >  $@

#### sm grammar using berkeley trainer
.PRECIOUS: %sm.gr
%sm.gr:  $(EDU)/berkeley/nlp/PCFGLA/GrammarTrainer.class  $$(basename $$(basename %)).extrparplusops.linetrees  $$(basename %)last.extrparplusops.linetrees  user-javaflags.txt
	java  $(shell cat $(word 4,$^))  -cp $(dir $(EDU))  edu.berkeley.nlp.PCFGLA.GrammarTrainer  -SMcycles $(subst .,,$(suffix $*))  -path $(word 2,$^)  -validation $(word 3,$^)  -treebank SINGLEFILE  -out $@
#	java  -cp $(EDU)  $(shell cat $(word 4,$^))  $(subst /,.,$(basename $<))  -SMcycles $(subst .,,$(suffix $*))  -path $(word 2,$^)  -validation $(word 3,$^)  -treebank SINGLEFILE  -out $@

#### silly shortcut for berkeley parser
%.fullberk.model:  %.gr
	ln -sf $(notdir $<) $@
%.vitberk.model:  %.gr
	ln -sf $(notdir $<) $@

#### obtain text grammar from javadump
.PRECIOUS: %.splits %.grammar %.lexicon
%.splits %.grammar %.lexicon:  $(EDU)/berkeley/nlp/PCFGLA/WriteGrammarToTextFile.class  %.gr  user-javaflags.txt
	java  $(shell cat $(word 3,$^))  -cp $(dir $(EDU))  edu.berkeley.nlp.PCFGLA.WriteGrammarToTextFile  $(word 2,$^)  $(basename $@)

################################################################################
#
#  2. Berkeley parser
#
#  to construct the following file types:
#    <x>.sents                        : sentences, one per line, consisting of only tokenized words, delimited by spaces
#    <w>.<x>_<y>_<z>_parsed.linetrees : .linetrees file resulting from applying parser-<z>, using parameter <y>, and model <x>.<z>.model, to <w>.sents file
#    <x>.syneval : evaluation report for parsing
#
################################################################################

##### obtain model-specific parser output by running sentences through parser given flags and model, from including makefile's genmodel dir:
##### <testset>.<trainset>.<model>.(<params>_)parsed  ---->  genmodel/<testset>.sents  bin/parser-<model>  genmodel/<trainset>.<model>.model
# $$(basename $$(basename $$(basename %))).$$(findstring hy,$$*)sents
.PRECIOUS: %parsed.output
%parsed.output:  $$(basename $$(basename $$(basename %))).sents \
		$(BIN)/parser-$$(subst .,,$$(suffix $$(basename $$*))) \
		$(GENMODEL)/$$(subst -,.,$$(subst .,,$$(suffix $$(basename $$(basename $$*)))))$$(suffix $$(basename $$*)).model
	@echo "WARNING: long build for '$@'!  Press CTRL-C to abort!"
	@sleep 5
	cat $<  |  $(word 2,$^)  $(subst _, ,$(subst .,,$(suffix $*)))  $(word 3,$^)  >  $@

#### obtain linetrees by removing extra parens and converting '+'s in categories to '-'s
.PRECIOUS: %berk.parsed.linetrees
#%berk.parsed.linetrees:  $$(word 1,$$(subst ., ,%))$$(suffix $$(basename $$*))$$(suffix $$*)berk.parsed.output
%berk.parsed.linetrees:  %berk.parsed.output
	cat $<  |  perl -pe 's/\+(?=[^\)]* )/\-/g'  |  perl -pe 's/^ *\( *//;s/ *\) *$$//;s/-\d+ / /g'  >  $@
