<details>
  <summary>Intro</summary>
![plot](https://i.ibb.co/3TC1WmG/specter-logo-cropped.png)

# SPECTER: Document-level Representation Learning using Citation-informed Transformers

[**SPECTER**](#specter-document-level-representation-learning-using-citation-informed-transformers) | [**Pretrained models**](#How-to-use-the-pretrained-model) | [**Training your own model**](#advanced-training-your-own-model) | 
[**SciDocs**](https://github.com/allenai/scidocs) | [**Public API**](#Public-api) | 
[**Paper**](https://arxiv.org/pdf/2004.07180.pdf) | [**Citing**](#Citation) 


This repository contains code, link to pretrained models, instructions to use [SPECTER](https://arxiv.org/pdf/2004.07180.pdf) and link to the [SciDocs](https://github.com/allenai/scidocs) evaluation framework.



***** New Jan 2021: HuggingFace models *****

Specter is now accessible through HuggingFace's transformers library.  

*Thanks to [@zhipenghoustat](https://github.com/zhipenghoustat) for providing the Huggingface training scripts and the checkpoint.*

See below:
</details>

# How to use the pretrained model
<details>
  <summary>Alternative: through Huggingface Transformers Library</summary>
  

Requirement: `pip install --upgrade transformers==4.2`

```python
from transformers import AutoTokenizer, AutoModel

# load model and tokenizer
tokenizer = AutoTokenizer.from_pretrained('allenai/specter')
model = AutoModel.from_pretrained('allenai/specter')

papers = [{'title': 'BERT', 'abstract': 'We introduce a new language representation model called BERT'},
          {'title': 'Attention is all you need', 'abstract': ' The dominant sequence transduction models are based on complex recurrent or convolutional neural networks'}]

# concatenate title and abstract
title_abs = [d['title'] + tokenizer.sep_token + (d.get('abstract') or '') for d in papers]
# preprocess the input
inputs = tokenizer(title_abs, padding=True, truncation=True, return_tensors="pt", max_length=512)
result = model(**inputs)
# take the first token in the batch as the embedding
embeddings = result.last_hidden_state[:, 0, :]
```

A sample script to run the model in batch mode on a dataset of papers is provided under `scripts/embed_papers_hf.py`

How to use:
```
CUDA_VISIBLE_DEVICES=0 python scripts/embed_papers_hf.py \
--data-path path/to/paper-metadata.json \
--output path/to/write/output.json \
--batch-size 8
```

** Note that huggingface model yields slightly higher average results than those reported in the paper.
To reproduce our exact numbers use our original implementation see [**reproducing results**](#How-to-reproduce-our-results).

*Expected SciDocs results from the huggingface model:*

| mag-f1 	| mesh-f1 	| co-view-map 	| co-view-ndcg 	| co-read-map 	| co-read-ndcg 	| cite-map 	| cite-ndcg 	| cocite-map 	| cocite-ndcg 	| recomm-ndcg 	| recomm-P@1 	| Avg  	|
|--------	|---------	|-------------	|--------------	|-------------	|--------------	|----------	|-----------	|------------	|-------------	|-------------	|------------	|------	|
| 79.4   	| 87.7    	| 83.4        	| 91.4         	| 85.1        	| 92.7         	| 92.0     	| 96.6      	| 88.0       	| 94.7        	| 54.6        	| 20.9       	| 80.5 	|
</details>
          
<details>
          <summary>Errata for paper</summary>
         In the paper we mentioned that we take the representation corresponding to the `[CLS]` token as the aggregate representation of the sequence. However, in the AllenNLP v0.9 implementation of BERT embedder, each token representation is a [scalar mix](https://github.com/allenai/allennlp/blob/542ce5d9137840e8197ef5781cd12f02f1c86f79/allennlp/modules/scalar_mix.py#L10) of all layer representations. To get aggregate representation of the input in a single vector, average pooling is used. Therefore, the original SPECTER model uses scalar mixing of layers and average pooling to embed a given document as opposed to taking the final layer represenation of the `[CLS]` token.
The Huggingface model above uses final layer represnation of `[CLS]`. In paractice this doesn't impact the results and both models perform comparably.

</details>

### Clone the repo and pretrained model

<details>
          <summary>Download</summary>

Download the tar file at: [**download**](https://ai2-s2-research-public.s3-us-west-2.amazonaws.com/specter/archive.tar.gz) [833 MiB]  
The compressed archive includes a `model.tar.gz` file which is the pretrained model as well as supporting files that are inside a `data/` directory. 

</details>

```python
git clone https://github.com/lecfab/specter.git

cd specter

wget https://ai2-s2-research-public.s3-us-west-2.amazonaws.com/specter/archive.tar.gz

tar -xzvf archive.tar.gz 
```


### Install the environment

```python
conda create --name specter python=3.7 setuptools  

conda activate specter  

# if you don't have gpus, remove cudatoolkit argument
conda install pytorch cudatoolkit=10.1 -c pytorch   

pip install -r requirements.txt  

python setup.py install

# for TypeError: ArrayField.empty_field: None is not a <class 'allennlp.data.fields.field.Field'> 
pip install --force-reinstall overrides==3.1.0
```
(https://github.com/allenai/specter/issues/27)

### Embed papers or documents
Specter requires two main files as input to embed the document. A text file with ids of the documents you want to embed and a json metadata file consisting of the title and abstract information. Sample files are provided in the `data/` directory to get you started. Input data format is according to:

```python
metadata.json format:
{
    'doc_id': {'title': 'representation learning of scientific documents',
               'abstract': 'we propose a new model for representing abstracts'},
}
```

To use SPECTER to embed your data use the following command:

```python
python scripts/embed.py \
--ids data/sample.ids --metadata data/sample-metadata.json \
--model ./model.tar.gz \
--output-file output.jsonl \
--vocab-dir data/vocab/ \
--batch-size 16 \
--cuda-device -1
```

Change `--cuda-device` to `0` or your specified GPU if you want faster inference.  
The model will run inference on the provided input and writes the output to `--output-file` directory (in the above example `output.jsonl` ).  
This is a jsonlines file where each line is a key, value pair consisting the id of the embedded document and its specter representation.


# Training your own model

### Organise your data

You will need the following files:
* `data.json` containing the document ids and their relationship.  
* `metadata.json` containing mapping of document ids to textual fiels (e.g., `title`, `abstract`)
* `train.txt`,`val.txt`, `test.txt` containing document ids corresponding to train/val/test sets (one doc id per line).

The `data.json` file should have the following structure (a nested dict):  
```python
{"docid1" : {  "docid11": {"count": 1}, "docid12": {"count": 5}, "docid13": {"count": 1}, .... },
"docid2":   {  "docid21": {"count": 1}, ....} ....}
```

Where `docids` are ids of documents in your data and `count` is a measure of importance of the relationship between two documents. In our dataset we used citations as indicator of relationship where `count=5` means direct citation while `count=1` refers to a citation of a citation.  
  

### Create pickled training instances
The `create_training_files.py` script processes this structure with a triplet sampler that selects both easy and hard negatives (as described in the paper) according the `count` value in the above structure. For example papers with `count=5` are considered positive candidates, papers with `count=1` considered hard negatives and other papers that are not cited are easy negatives. You can control the number of hard negatives by setting `--ratio_hard_negatives` argument in the script.  

```python
python specter/data_utils/create_training_files.py \
--data-dir data/training \
--metadata data/training/metadata.json \
--outdir data/preprocessed/
```

After preprocessing the data you will have three pickled files containing training instannces as well as a `metrics.json` showing number of examples in each set.

### Run the training model
```python
./scripts/run-exp-simple.sh -c experiment_configs/simple.jsonnet \
-s model-output/ --num-epochs 2 --batch-size 4 \
--train-path data/preprocessed/data-train.p --dev-path data/preprocessed/data-val.p \
--num-train-instances 55 --cuda-device -1
```

Note that you need to set the correct `--num-train-instances` for your dataset. This number is stored in `metrics.json` file output from the preprocessing step.

You can stop the process at any time with keyboard interrupt (`ctrl+C`) and resume the training later with `--recover`.

In this example, the model's checkpoint and logs will be stored in `model-output/ `.  
You can monitor the training progress using `tensorboard`: `tensorboard --logdir model-output/  --bind_all`

---


* Public API: [**allenai/paper-embedding-public-apis**](https://github.com/allenai/paper-embedding-public-apis) 

* Reproduce our results: [SciDocs](https://github.com/allenai/scidocs) provides the embeddings for the evaluation tasks and instructions on how to run the benchmark to get the results.

* SciDocs benchmark: [https://github.com/allenai/scidocs](https://github.com/allenai/scidocs) consists of a suite of evaluation tasks designed for document-level tasks. 


* Citation: [SPECTER paper](https://arxiv.org/pdf/2004.07180.pdf)
