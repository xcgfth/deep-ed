# Source code for "Deep Joint Entity Disambiguation with Local Neural Attention"

[O-E. Ganea and T. Hofmann, full paper @ EMNLP 2017](https://arxiv.org/abs/1704.04920)

## Pre-trained entity embeddings

Entity embeddings trained with our method using Word2Vec 300 dimensional pre-trained word vectors (GoogleNews-vectors-negative300.bin). They have norm 1 and are restricted only to entities appearing in the training, validation and test sets described in our paper. Available here: https://polybox.ethz.ch/index.php/s/sH2JSB2c1OSj7yv . Password can be requested from octavian.ganea at inf dot ethz dot ch .

## Full set of annotations made by our best global model

TODO

## How to run the system and reproduce our results

1) Install [Torch](http://torch.ch/)


2) Install torch libraries: cudnn, cutorch, [tds](https://github.com/torch/tds), gnuplot, xlua

```luarocks install lib_name```

Check that each of these libraries can be imported in a torch terminal.


3) Create a $DATA_PATH/ directoy. Create a directory $DATA_PATH/generated/ that will contain all generated files.


4) Download data files needed for training and testing from [this link](https://drive.google.com/drive/u/2/folders/0Bx8d3azIm_ZcRGVkQS1WYkJtcU0).
 Download basic_data.zip, unzip it and place the basic_data directory in $DATA_PATH/. All generated files will be build based on files in this basic_data directory.


5) Download pre-trained Word2Vec vectors GoogleNews-vectors-negative300.bin.gz from https://code.google.com/archive/p/word2vec/.
Unzip it and place the bin file in the folder $DATA_PATH/basic_data/wordEmbeddings/Word2Vec


Now we start creating additional data files needed in our pipeline:

6) Create wikipedia_p_e_m.txt: 

```th data_gen/gen_p_e_m/gen_p_e_m_from_wiki.lua -root_data_dir $DATA_PATH```

 
7) Merge wikipedia_p_e_m.txt and crosswikis_p_e_m.txt : 

```th data_gen/gen_p_e_m/merge_crosswikis_wiki.lua -root_data_dir $DATA_PATH```


8) Create yago_p_e_m.txt: 

```th data_gen/gen_p_e_m/gen_p_e_m_from_yago.lua -root_data_dir $DATA_PATH ```


9) Create a file ent_wiki_freq.txt with entity frequencies: 

```th entities/ent_name2id_freq/e_freq_gen.lua  -root_data_dir $DATA_PATH```


10) Generate all datasets for entity disambiguation: 

```
mkdir $DATA_PATH/generated/test_train_data/
th data_gen/gen_test_train_data/gen_all.lua -root_data_dir $DATA_PATH
```
 
Verify the statistics of these files as shown in the header comments of gen_ace_msnbc_aquaint_csv.lua and gen_aida_test.lua .


11) Create training data for learning entity embeddings:

  i) From Wiki canonical pages: 

```th data_gen/gen_wiki_data/gen_ent_wiki_w_repr.lua -root_data_dir  $DATA_PATH```

  ii) From context windows surrounding Wiki hyperlinks: 

```th data_gen/gen_wiki_data/gen_wiki_hyp_train_data.lua -root_data_dir $DATA_PATH```


12) Compute the unigram frequency of each word in the Wikipedia corpus: 

```th words/w_freq/w_freq_gen.lua -root_data_dir $DATA_PATH```


13) Compute the restricted training data for learning entity embeddings by using only candidate entities from the relatedness datasets and all ED sets:
  i) From Wiki canonical pages: 

```th entities/relatedness/filter_wiki_canonical_words_RLTD.lua  -root_data_dir  $DATA_PATH```

  ii) From context windows surrounding Wiki hyperlinks: 

```th entities/relatedness/filter_wiki_hyperlink_contexts_RLTD.lua -root_data_dir $DATA_PATH```

All files in the generated/ folder containing the substring "_RLTD" are restricted to this set of entities (should contain 276030 entities).

Your $DATA_PATH/generated/ folder should now contain the files : 

```
$DATA_PATH/generated $ ls -lah ./
total 147G
9.5M all_candidate_ents_ed_rltd_datasets_RLTD.t7
775M crosswikis_wikipedia_p_e_m.txt
5.0M empty_page_ents.txt
520M ent_name_id_map.t7
95M ent_wiki_freq.txt
7.3M relatedness_test.t7
8.9M relatedness_validate.t7
220 test_train_data
1.5G wiki_canonical_words_RLTD.txt
8.4G wiki_canonical_words.txt
88G wiki_hyperlink_contexts.csv
48G wiki_hyperlink_contexts_RLTD.csv
329M wikipedia_p_e_m.txt
11M word_wiki_freq.txt
749M yago_p_e_m.txt

$DATA_PATH//generated/test_train_data $ ls -lah ./
total 124M
14M aida_testA.csv
13M aida_testB.csv
50M aida_train.csv
723K wned-ace2004.csv
1.6M wned-aquaint.csv
31M wned-clueweb.csv
1.6M wned-msnbc.csv
15M wned-wikipedia.csv
```

14) Now we train entity embeddings for the restricted set of entities (written in all_candidate_ents_ed_rltd_datasets_RLTD.t7). This is the step described in Section 3 of our paper.

To check the full list of parameters run: 

```th entities/learn_e2v/learn_a.lua -help```

Optimal parameters (see entities/learn_e2v/learn_a.lua):

```
optimization = 'ADAGRAD'
lr = 0.3
batch_size = 500
word_vecs = 'w2v'
num_words_per_ent = 20
num_neg_words = 5
unig_power = 0.6
entities = 'RLTD'
loss = 'maxm'
data = 'wiki-canonical-hyperlinks'
num_passes_wiki_words = 400
hyp_ctxt_len = 10
```

To run the embedding training on one GPU: 

```
mkdir $DATA_PATH/generated/ent_vecs
CUDA_VISIBLE_DEVICES=0 th entities/learn_e2v/learn_a.lua -root_data_dir $DATA_PATH |& tee log_train_entity_vecs
```

Warning: This code is not sufficiently optimized to run at maximum speed and GPU usage. Sorry for the inconvenience. It only uses the main thread to load data and perform word embedding lookup. It can be made to run much faster.

During training, you will see (in the log file log_train_entity_vecs) the validation score on the entity relatedness dataset of the current set of entity embeddings. After around 60 hours (this code is not optimized!), you will see no improvement and can thus stop the training script. Pick the set of saved entity vectors from the folder generated/ent_vecs/ corresponding to the best validation score on the entity relatedness dataset which is the sum of all validation metrics (the TOTAL VALIDATION column). In our paper, we reported in Table 1 the results on the test set corresponding to this best validation score. 

You should get some numbers similar to the following (may vary a little bit due to random initialization):

```
Entity Relatedness quality measure:	
measure    =	NDCG1	NDCG5	NDCG10	MAP	TOTAL VALIDATION	
our (vald) =	0.683	0.641	0.674	0.620	2.617	
our (test) =	0.638	0.610	0.641	0.577	
Yamada'16  =	0.59	0.56	0.59	0.52	
WikiMW     =	0.54	0.52	0.55	0.48
==> saving model to $DATA_PATH/generated/ent_vecs/ent_vecs__ep_228.t7	
```

We will call the name of this file with entity embeddings as $ENTITY_VECS. In our case, it is 'ent_vecs__ep_228.t7'


15) Run the training for the global/local ED neural network. Arguments file: ed/args.lua . To list all arguments: 

```th ed/ed.lua -help```

Command to run the training: 

```
mkdir $DATA_PATH/generated/ed_models/
mkdir $DATA_PATH/generated/ed_models/training_plots/
CUDA_VISIBLE_DEVICES=0 th ed/ed.lua -root_data_dir $DATA_PATH -ent_vecs_filename $ENTITY_VECS -model 'global' |& tee log_train_ed
```

Let it train for XX (TODO: fill) hours, until the validation accuracy does not improve any more or starts dropping. As we wrote in the paper, we stop learning if
the validation F1 does not increase after 500 full epochs of the AIDA train dataset. Validation F1 can be following using the command:

```cat log_train_ed | grep -A20 'Micro F1' | grep -A20 'aida-A'```

The best ED models will be saved in the folder generated/ed_models. This will only happen after the model gets > 90% F1 score on validation set (see test.lua).

Statistics, weights and scors will be written in the log_train_ed file. Plots of micro F1 scores on all the validation and test set will be written in the folder $DATA_PATH/generated/ed_models/training_plots/ .


16) After training is terminated, one can re-load and test the best ED model using the command:

```CUDA_VISIBLE_DEVICES=0 th ed/test/test_one_loaded_model.lua -root_data_dir $DATA_PATH -model global -ent_vecs_filename $ENTITY_VECS  -test_one_model_file $ED_MODEL_FILENAME```

where $ED_MODEL_FILENAME is a file in $DATA_PATH/generated/ed_models/ .


17) Enjoy!
