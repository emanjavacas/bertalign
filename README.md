# Bertalign
Word Embedding-Based Bilingual Sentence Aligner

Bertalign is designed to facilitate the construction of bilingual parallel corpora, which have a wide range of applications in translation-related research such as corpus-based translation studies, contrastive linguistics, computer-assisted translation, translator education and machine translation.

Bertalign uses [cross-lingua embedding models](https://github.com/UKPLab/sentence-transformers) to represent source and target sentences so that semantically similar sentences in different languages can be mapped onto similar vector spaces. According to our explements, Bertalign achieves more accurate results than the traditional length-, dictionary-, or MT-based alignment methods such as [Galechurch](https://aclanthology.org/J93-1004/), [Hunalign](http://mokk.bme.hu/en/resources/hunalign/) and [Bleualign](https://github.com/rsennrich/Bleualign). It also performs better than [Vecalign](https://github.com/thompsonb/vecalign) on MAC, a manually aligned parallel corpus of Chinese-English literary texts.

## Installation

Bertalign is developed using Python and tested on Windows 10 and Linux systems. You need to install the following packages before running Bertalign:

```
pip install numba
pip install faiss-cpu
pip install faiss-gpu
pip install sentence-transformers
```

Please note that embedding sentences on GPU-enabled machines is much faster than those with CPU only. The following experiments are conducted using [Google Colab](https://colab.research.google.com/) which provides free GPU service.

## Evaluation Corpora
Bertalign is language-agnostic thanks to the cross-language embedding models  [sentence-transformers](https://github.com/UKPLab/sentence-transformers).

For now, we only use the following two Chinese-English corpora to evaluate the performance of Bertalign. Dataset with other language pairs are to be added later.

### MAC-Dev

[MAC-Dev](./data/mac/dev) is the development set selected from the MAC corpus, a manually aligned corpus of Chinese-English literary texts. The sampling schemes for MAC-Dev can be found at [meta_data.tsv](./data/mac/dev/meta_data.tsv).

There are 4 subdirectories in MAC-Dev:

The [zh](./data/mac/dev/zh) and [en](./data/mac/dev/en) directories contain the sentence-split and tokenized source texts, target texts and the machine translations of source texts. Hunalign requires tokenized source and target sentences for dictionary search of similar words. Bleualign uses MT translations of source texts to compute the Bleu similarity score between source and target sentences.

We use [Moses sentence splitter](https://github.com/moses-smt/mosesdecoder/blob/master/scripts/ems/support/split-sentences.perl) and [Stanford CoreNLP](https://stanfordnlp.github.io/CoreNLP/usage.html) to split and tokenize English sentences, while [pyltp](https://github.com/HIT-SCIR/pyltp) and [jieba](https://github.com/fxsjy/jieba) are used to split and tokenize Chinese sentences. The MT of source texts are generated by [Google Translate](https://translate.google.cn/).

The [auto](./data/mac/dev/auto) and [gold](./data/mac/dev/gold) directories are for automatic and gold alignments. All the gold alignments are created manually using [Intertext](https://wanthalf.saga.cz/intertext).

### Bible
[The Bible corpus](./data/bible), consisting of 5,000 source and 6,301 target sentences, is selected from the public [multilingual Bible corpus](https://github.com/christos-c/bible-corpus/tree/master/bibles). This corpus is mainly used to evaluate the speed of Bertalign.

The directory makeup is similar to MAC-Dev, except that the gold alignments for the Bible corpus are generated automatically from the original verse-aligned Bible corpus.

In order to compare the sentence-based alignments returned by various aligners with the verse-based gold alignments, we put the verse ID for each sentence in the files [en.verse](./data/bible/en.verse) and [zh.verse](./data/bible/zh.verse), which are used to merge consecutive sentences in the automatic alignments if they belong to the same verse.

## System Comparisons
All the experiments are conducted on [Google Colab](https://colab.research.google.com/).

### Sentence Embeddings
We use the Python script [embed_sents.py](./bin/embed_sents.py) to create the sentence embedddings for the MAC-Dev and the Bible corpus. This script is based on [Vecalign developed by Brian Thompson](https://github.com/thompsonb).
```
# Embedding MAC-Dev Chinese
!python bin/embed_sents.py \
  -i data/mac/dev/zh \
  -o data/mac/dev/zh/overlap data/mac/dev/zh/overlap.emb \
  -m data/mac/dev/meta_data.tsv \
  -n 8
  
# Embedding MAC-Dev English
!python bin/embed_sents.py \
  -i data/mac/dev/en \
  -o data/mac/dev/en/overlap data/mac/dev/en/overlap.emb \
  -m data/mac/dev/meta_data.tsv \
  -n 8

# Embedding Bible English
!python bin/embed_sents.py \
  -i data/bible/en \
  -o data/bible/en/overlap data/bible/en/overlap.emb \
  -m data/bible/meta_data.tsv \
  -n 5

# Embedding Bible Chinese
!python bin/embed_sents.py \
  -i data/bible/zh \
  -o data/bible/zh/overlap data/bible/zh/overlap.emb \
  -m data/bible/meta_data.tsv \
  -n 5
```
The parameter *-n* indicates the maximum number of overlapping sentences allowed on the source and target side, which is similar to word *n*-grams applied to sentences. After running the script, the overlapping sentences *overlap* in the source and target texts and their embeddings *overlap.emb* are saved in the directory [mac/dev/zh](./data/mac/dev/zh), [mac/dev/en](./data/mac/dev/en),  [bible/zh](./data/bible/zh), and [bible/en](./data/bible/en).

### Evaluation on MAC-Dev

#### Bertalign

```
# Run Bertalign on MAC-Dev
!python bin/bert_align.py \
  -s data/mac/dev/zh \
  -t data/mac/dev/en \
  -o data/mac/dev/auto \
  -m data/mac/dev/meta_data.tsv \
  --src_embed data/mac/dev/zh/overlap data/mac/dev/zh/overlap.emb \
  --tgt_embed data/mac/dev/en/overlap data/mac/dev/en/overlap.emb \
  --max_align 8 --margin

# Evaluate Bertalign on MAC-Dev
!python bin/eval.py \
  -t data/mac/dev/auto \
  -g data/mac/dev/gold
```

```
 ---------------------------------
|             |  Strict |    Lax  |
| Precision   |   0.909 |   0.996 |
| Recall      |   0.914 |   1.000 |
| F1          |   0.911 |   0.998 |
 ---------------------------------
```

Please refer to [Sennrich & Volk (2010)](https://aclanthology.org/people/r/rico-sennrich/) for the difference between Strict and Lax evaluation method. We can see that the F1 score is 0.91 when aligning MAC-Dev using Bertalign.

Please note that aligning literary texts is not an easy task, since they contain more interpretive and free translations than non-literary works. You can refer to [Xu et al. (2015)](https://aclanthology.org/2015.lilt-12.6/) for more details about sentence alignment of literary texts. Let's see how the other systems perform on MAC-Dev:

#### Baseline Approaches

##### Gale-Church

```
# Run Gale-Church on MAC-Dev
!python bin/gale_align.py \
  -m data/mac/dev/meta_data.tsv \
  -s data/mac/dev/zh \
  -t data/mac/dev/en \
  -o data/mac/dev/auto
  
# Evaluate Gale-Church on MAC-Dev
!python bin/eval.py \
  -t data/mac/dev/auto \
  -g data/mac/dev/gold
```
```
---------------------------------
|             |  Strict |    Lax  |
| Precision   |   0.472 |   0.727 |
| Recall      |   0.497 |   0.746 |
| F1          |   0.485 |   0.737 |
 ---------------------------------
```

##### Hunalign

```
# Run Hunalign on MAC-Dev
!python ext-lib/hunalign/hunalign.py \
  -m data/mac/dev/meta_data.tsv \
  -s data/mac/dev/zh \
  -t data/mac/dev/en \
  -o data/mac/dev/auto \
  -d ec.dic
  
# Evaluate Hunalign on MAC-Dev
!python bin/eval.py \
  -t data/mac/dev/auto \
  -g data/mac/dev/gold
```
```
---------------------------------
|             |  Strict |    Lax  |
| Precision   |   0.573 |   0.811 |
| Recall      |   0.653 |   0.897 |
| F1          |   0.610 |   0.852 |
 ---------------------------------
```

The parameter *-d* designates the location of the bilingual dictionary for Hunalign to calculate bitext similarity. For fair comparison with other systems, we use [a large Chinese-English dictionary](https://web.archive.org/web/20130107081319/http://projects.ldc.upenn.edu/Chinese/LDC_ch.htm) with 224,000 entries for Hunalign to achieve higher accuracy.

##### Bleualign

```
# Run Bleualign on MAC-Dev
!python ext-lib/bleualign/bleualign.py \
  -m data/mac/dev/meta_data.tsv \
  -s data/mac/dev/zh \
  -t data/mac/dev/en \
  -o data/mac/dev/auto
  
# Evaluate Bleualign on MAC-Dev
!python bin/eval.py \
  -t data/mac/dev/auto \
  -g data/mac/dev/gold
```
```
---------------------------------
|             |  Strict |    Lax  |
| Precision   |   0.676 |   0.929 |
| Recall      |   0.609 |   0.823 |
| F1          |   0.641 |   0.873 |
 ---------------------------------
```

##### Vecalign

```
# Run Vecalign on MAC-Dev
!python ext-lib/vecalign/vecalign.py \
  -s data/mac/dev/zh \
  -t data/mac/dev/en \
  -o data/mac/dev/auto \
  -m data/mac/dev/meta_data.tsv \
  --src_embed data/mac/dev/zh/overlap data/mac/dev/zh/overlap.emb \
  --tgt_embed data/mac/dev/en/overlap data/mac/dev/en/overlap.emb \
  -a 8 -v
  
# Evaluate Vecalign on MAC-Dev
!python bin/eval.py \
  -t data/mac/dev/auto \
  -g data/mac/dev/gold
```
```
---------------------------------
|             |  Strict |    Lax  |
| Precision   |   0.848 |   0.995 |
| Recall      |   0.878 |   0.999 |
| F1          |   0.863 |   0.997 |
 ---------------------------------
```

### Evaluation on Bible

#### Bertalign

```
# Run Bertalign on Bible
!python bin/bert_align.py \
  -s data/bible/en \
  -t data/bible/zh \
  -o data/bible/auto \
  -m data/bible/meta_data.tsv \
  --src_embed data/bible/en/overlap data/bible/en/overlap.emb \
  --tgt_embed data/bible/zh/overlap data/bible/zh/overlap.emb \
  --max_align 5 --margin

# Evaluate Bertalign on Bible
!python bin/eval_bible.py \
  -t data/bible/auto/001.align \
  -g data/bible/gold/001.align \
  --src_verse data/bible/en.verse \
  --tgt_verse data/bible/zh.verse
```

```
It takes 1.695 seconds to align all the sentences.
---------------------------------
|             |  Strict |    Lax  |
| Precision   |   0.981 |   1.000 |
| Recall      |   0.976 |   1.000 |
| F1          |   0.979 |   1.000 |
 ---------------------------------
```

It only takes Bertalign 1.695 seconds to align 5,000 English sentences and 6,301 Chinese sentences. The verse-based evaluation shows that Bertalign can achieve 0.98 F1 score on the Bible corpus.

Now let's see how fast other systems can run on the Bible corpus:

#### Baseline Approaches

##### Gale-Church

```
# Run Gale-Church on Bible
!python bin/gale_align.py \
  -m data/bible/meta_data.tsv \
  -s data/bible/en \
  -t data/bible/zh \
  -o data/bible/auto

# Evaluate Gale-Church on Bible
!python bin/eval_bible.py \
  -t data/bible/auto/001.align \
  -g data/bible/gold/001.align \
  --src_verse data/bible/en.verse \
  --tgt_verse data/bible/zh.verse
```
```
It takes 3.742 seconds to align all the sentences.
 ---------------------------------
|             |  Strict |    Lax  |
| Precision   |   0.817 |   0.928 |
| Recall      |   0.824 |   0.934 |
| F1          |   0.821 |   0.931 |
 ---------------------------------
```

##### Hunalign

```
# Run Hunalign on Bible
!python ext-lib/hunalign/hunalign.py \
  -m data/bible/meta_data.tsv \
  -s data/bible/en \
  -t data/bible/zh \
  -o data/bible/auto \
  -d ce.dic
  
# Evaluate Hunalign on Bible
!python bin/eval_bible.py \
  -t data/bible/auto/001.align \
  -g data/bible/gold/001.align \
  --src_verse data/bible/en.verse \
  --tgt_verse data/bible/zh.verse
```
```
It takes 2.394 seconds to align all the sentences.
 ---------------------------------
|             |  Strict |    Lax  |
| Precision   |   0.887 |   0.964 |
| Recall      |   0.904 |   0.980 |
| F1          |   0.895 |   0.972 |
 ---------------------------------
```

##### Bleualign

```
# Run Bleualign on Bible
!python ext-lib/bleualign/bleualign.py \
  -m data/bible/meta_data.tsv \
  -s data/bible/en \
  -t data/bible/zh \
  -o data/bible/auto \
  --tok
  
# Evaluate Bleualign on Bible
!python bin/eval_bible.py \
  -t data/bible/auto/001.align \
  -g data/bible/gold/001.align \
  --src_verse data/bible/en.verse \
  --tgt_verse data/bible/zh.verse
```
```
It takes 78.898 seconds to align all the sentences.
 ---------------------------------
|             |  Strict |    Lax  |
| Precision   |   0.827 |   0.961 |
| Recall      |   0.753 |   0.880 |
| F1          |   0.788 |   0.919 |
 ---------------------------------
```

##### Vecalign

```
# Run Vecalign on Bible
!python ext-lib/vecalign/vecalign.py \
  -s data/bible/en \
  -t data/bible/zh \
  -o data/bible/auto \
  -m data/bible/meta_data.tsv \
  --src_embed data/bible/en/overlap data/bible/en/overlap.emb \
  --tgt_embed data/bible/zh/overlap data/bible/zh/overlap.emb \
  -a 5 -v
  
# Evaluate Vecalign on Bible
!python bin/eval_bible.py \
  -t data/bible/auto/001.align \
  -g data/bible/gold/001.align \
  --src_verse data/bible/en.verse \
  --tgt_verse data/bible/zh.verse
```
```
It takes 4.676 seconds to align all the sentences.
---------------------------------
|             |  Strict |    Lax  |
| Precision   |   0.977 |   1.000 |
| Recall      |   0.978 |   1.000 |
| F1          |   0.978 |   1.000 |
 ---------------------------------
```

## TODO List

Evaluate Bertalign on datasets containing language pairs other than Chinese and English.
