# CharacterBERT-DR
The offcial repository for [CharacterBERT and Self-Teaching for Improving the Robustness of Dense Retrievers on Queries with Typos](https://arxiv.org/pdf/2204.00716.pdf), Shengyao Zhuang and Guido Zuccon, SIGIR2022
> Note: this repository is in refactoring.


## Installation
Our code is developed based on [Tevatron](https://github.com/texttron/tevatron) DR training toolkit (v0.0.1).

First clone this repository and then install with pip:
`pip install --editable .`

> Note: The current code base has been tested with, `torch==1.8.1`, `faiss-cpu==1.7.1`, `transformers==4.9.2`, `datasets==1.11.0`, `textattack=0.3.4`

## Preparing data and model

### Typo queries and dataset
All the queries and qrels used in our paper are in the `/data` folder.

### Download CharacterBERT
Download [CharacterBERT](https://github.com/helboukkouri/character-bert/tree/0c1f5c2622950988833a9d95e29bc26864298592#pre-trained-models) trained with general domain with this [link](https://docs.google.com/uc?id=11-kSfIwSWrPno6A4VuNFWuQVYD8Bg_aZ).

## Train

### CharacterBERT-DR + ST training
```
python -m tevatron.driver.train \
--model_name_or_path bert-base-uncased \
--character_bert_path ./general_character_bert \
--output_dir model_msmarco_characterbert_st \
--passage_field_separator [SEP] \
--save_steps 40000 \
--dataset_name Tevatron/msmarco-passage \
--fp16 \
--per_device_train_batch_size 16 \
--learning_rate 5e-6 \
--max_steps 150000 \
--dataloader_num_workers 10 \
--cache_dir ./cache \
--logging_steps 150 \
--character_query_encoder True \
--self_teaching True
```
If you want to do typo augmentation training introduced in our previous [paper](https://arxiv.org/pdf/2108.12139.pdf).
Replace `--self_teaching True` with `--typo_augmentation True`.

If you want to train a standard BERT DR intead of CharacterBERT DR, remove `--character_bert_path` and `--character_query_encoder` arguments.

If you do not want to train the model, we provide our trained model checkpoints for you to download:

| Model                  | Google drive     |
|------------------------|------------------|
| StandardBERT-DR        | available soon   |
| StandardBERT-DR + Aug  | available soon   |
| StandardBERT-DR + ST   | available soon   |
| CharacterBERT-DR       | available soon   |
| CharacterBERT-DR + Aug | available soon   |
| CharacterBERT-DR + ST  | available soon   |

## Inference

### Encode queries and corpus
After you have the trained model, you can run the following command to encode queries and corpus into dense vectors:

```
mkdir msmarco_charcterbert_st_embs
# encode query
python -m tevatron.driver.encode \
  --output_dir=temp \
  --model_name_or_path model_msmarco_characterbert_st/checkpoint-final \
  --fp16 \
  --per_device_eval_batch_size 128 \
  --encode_in_path data/dl-typo/query.typo.tsv \
  --encoded_save_path msmarco_charcterbert_st_embs/query_dltypo_typo_emb.pkl \
  --q_max_len 32 \
  --encode_is_qry \
  --character_query_encoder True


# encode corpus
for s in $(seq -f "%02g" 0 19)
do
python -m tevatron.driver.encode \
  --output_dir=temp \
  --model_name_or_path model_msmarco_characterbert_st/checkpoint-final \
  --fp16 \
  --per_device_eval_batch_size 128 \
  --p_max_len 128 \
  --dataset_name Tevatron/msmarco-passage-corpus \
  --encoded_save_path model_msmarco_characterbert_st/corpus_emb.${s}.pkl \
  --encode_num_shard 20 \
  --encode_shard_index ${s} \
  --cache_dir cache \
  --character_query_encoder True \
  --passage_field_separator [SEP]
done
```

If you running inference with standard BERT, remove `--character_query_encoder True` argument.

### Retrieval
Run the following commands to generate ranking file and convert it to TREC format:

```
python -m tevatron.faiss_retriever \
--query_reps model_msmarco_characterbert_st/query_dltypo_typo_emb.pkl \
--passage_reps model_msmarco_characterbert_st/'corpus_emb.*.pkl' \
--depth 1000 \
--batch_size -1 \
--save_text \
--save_ranking_to character_bert_st_dltypo_typo_rank.txt


python -m tevatron.utils.format.convert_result_to_trec \
              --input character_bert_st_dltypo_typo_rank.txt \
              --output character_bert_st_dltypo_typo_rank.txt.trec
```

### Evaluation
We use trec_eval to evaluate the results:

```
trec_eval -l 2 -m ndcg_cut.10 -m map -m recip_rank data/dl-typo/qrels.txt character_bert_st_dltypo_typo_rank.txt.trec
```

If you use our provided `CharacterBERT-DR + ST` checkpoint, you will get:

```
map                     all     0.3483
recip_rank              all     0.6154
ndcg_cut_10             all     0.4730
```

We note that if you train the model by yourself, you may get slightly different results due to the randomness of dataloader
and Tevatron self-contained msmarco-passage training dataset has been updated.