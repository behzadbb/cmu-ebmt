INSTALLING AND RUNNING CMU-EBMT
===============================

INSTALLATION
------------

If you are using 64-bit Linux, all you should need to do is invoke
    ./install-all

If you are using 32-bit Linux, or want to generate 32-bit executables
on a 64-bit system, try
    ./install-all -32bit

There is also experimental support for multithreaded operation of the
translator, but it is known to have one race condition in the EBMT
engine (thus results will vary somewhat from run to run) and what
appears to be cache contention in the decoder, resulting in more CPU
being used without speeding up the decoding.  These will be addressed
in a future release, but if you with to try multithreading, add
    -multithread
to the end of either command above.

For other operating systems, you will need to edit the makefiles.  The
main components should compile under Windows (though this has not been
tested recently), the utilities have not been compiled under Windows
and will probably require entirely new makefiles.

CMU-EBMT has been designed to be a complete, standalone system.
However, you will need to obtain an evaluation program if you want to
use the parameter tuning utility.  By default, the tuning code looks
for the NIST mteval-v11b.pl script, which is available from
http://www.itl.nist.gov/iad/mig/tests/mt/2009/ (you may also use a
newer version, just change the "mteval-v11b" in eval-panlite.sh to the
appropriate value).


TRAINING A TRANSLATOR
---------------------

A "toy" Haitian Creole system is included in this distribution, and is
installed by the installation script.  The English-to-Haitian
direction is pre-built, to build the Haitian-to-English direction,
follow these steps:

1. Acquire parallel text and format it into an interleaved set of
sentence pairs, i.e.
	 H1
	 E1
	 H2
	 E2
	 ...
The file ./data/med-train.ebmt contains the training data for the
sample Haitian system.

2. Acquire monolingual text in the target language.  This may be
either one sentence per line (in which case context markers can be
added automatically), or freeform text with <s> and </s> context
markers at sentence boundaries.  You can use the included 'sentbrk'
utility to insert these context markers into English and most European
languages.

For the sample system, we will use the English half of the training
data, i.e. ./data/med-train.en.

3. Create a configuration file to control the system's operation and
tell it where to store its files.  We'll start with the included
./cfg/eng2hai.cfg and create a new ./cfg/hai2eng.cfg.  First,
    cp ./cfg/eng2hai.cfg ./cfg/hai2eng.cfg
Then edit hai2eng.cfg and make the following changes:
  <Source-Language: "English"
  <Target-Language: "Haitian Creole"
  ---
  >Source-Language: "Haitian Creole"
  >Target-Language: "English"
The above is used to automatically swap directions in some of the
subsidiary data files.
Change all occurrences of "eng2hai" to "hai2eng" to avoid conflicting
with the English-to-Haitian system.
  <Target-Roots:	  "+ebmt/roots/haitian.synonyms"
  ---
  >Target-Roots:	  "+ebmt/roots/english.synonyms"
Under Models:, change "Haitian-*" to "English-*".  Following
[LM-Default], change all the section headings to English from Haitian,
e.g.  "[LM-Haitian-Medical]" becomes "[LM-English-Medical]", and swap
"hc-" and "en-" in filenames.

4. Run the build scripts.  First, generate a file containing a listing
of the bilingual training files, one per line:
   echo ${cwd}/data/med-train.ebmt >filelist

Add the directory with the compiled executables to your path:
   PATH=${cwd}/bin:$PATH; export PATH   	[bash]
   setenv PATH ${cwd}/bin:$PATH; rehash		[c-shell]

Next, run the build-ebmt script.  Because our training file is actually
English-to-Haitian, we add the -r flag to have the system reverse the
halves of the training data.  We also apply truecasing to the first
word of each sentence to avoid having uppercase words in the middle
of a translation if the match used was as the start of the sentence:
   ./scripts/build-ebmt.sh -r -D -U utf8 cfg/hai2eng.cfg 5 \
	${cwd}/filelist ./bin/build-dict-thresh1.dat \
	./bin/build-dict-thresh2.dat

Finally, generate a language model with the make-lm script.  Because
the sample training data is only a few thousand words, we'll only build
a four-gram model, rather than the typical 5-gram or 6-gram, and we'll
generate a generalization class <rare> for all singletons:
    ./scripts/make-lm -rank 4 -rare 1 -line -enc UTF8 \
	-out models/en-medical ./data/med-train.en
Ignore the warnings about missing configuration section [LMWeights].

At this point, you should have a working translator, although it will
probably perform much worse than it could (we'll take care of that in
the next section, by tuning the system's parameters).  Try it now, by
feeding it some sentences from the held-out data:
    head refs/refmed-tune_1.txt | ./bin/panlite -s./cfg/hai2eng.cfg


TUNING PARAMETERS
-----------------

Since it is unlikely that the parameters and features weights in your
configuration file are optimal for the data you are using for training
and testing the system, you should tune the parameters.  The actual
parameter tuner is ./bin/tune, but it takes an extremely long and
complex command line (listing all of the parameters to tune and their
values), so the ./scripts/tune-panlite.sh convenience script supplies
all the necessary arguments for you.  If you have not yet obtained
the mteval script from NIST (see above), do so now, and place it in
the ./scripts directory.

First, we need to set up the reference translation(s) where the
eval-panlite.sh script can find them.  The distribution comes with
English test files and Haitian Creole reference translations for those
files, so let's just swap them for the Haitian-to-English translator:
      cp refs/refmed-tune_1.txt data/engmed-tune.ht
      cp refs/refmed-test_1.txt data/engmed-test.ht
      cp data/med-tune.en refs/refengmed-tune_1.txt
      cp data/med-test.en refs/refengmed-test_1.txt
(as you can see, the references are expected to be in the subdirectory
./refs, and are named refFOO_1.txt, refFOO_2.txt, etc. for input files
called */FOO.*)

Now we can call the tuning script:
    ./scripts/tune-panlite.sh -C 2 -c cfg/hai2eng-tuned.cfg -ng -opt \
	-r -r -ebmt -u ./scripts/eval-panlite.sh \
	cfg/hai2eng.cfg data/engmed-tune.ht
This will run up to two evaluations concurrently (use -C 4 for
quad-core or higher machines), will create a new configuration file
with updated parameter values, will tune some language model
parameters which are not tuned by default, will run the new internal
optimizer for the log-linear feature weights instead of performing
coordinate ascent on them, and will start by randomly exploring values
for the other parameters which are not directly part of the log-linear
score.  It will also skip tuning parameters for the separate
dictionary engine, which we are not using.

Compare the output with tuned parameters against the untuned parameters:
    head data/engmed-test.ht | ./bin/panlite -s./cfg/hai2eng.cfg
    head data/engmed-test.ht | ./bin/panlite -s./cfg/hai2eng-tuned.cfg

==========================

