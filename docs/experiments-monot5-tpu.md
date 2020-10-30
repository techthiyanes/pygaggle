# Neural Ranking Baselines on [MS MARCO Passage Retrieval](https://github.com/microsoft/MSMARCO-Passage-Ranking) - with TPU

This page contains instructions for running various monoT5 on the MS MARCO *passage* ranking task. We will run on the entire dev set. 

We will focus on using monoT5-3B to rerank, since it is difficult to run such a large model without a TPU.
- monoT5-3B: Document Ranking with a Pretrained Sequence-to-Sequence Model [(Nogueira et al., 2020)](https://arxiv.org/pdf/2003.06713.pdf)

Note that there are also separate documents to run MS MARCO ranking tasks on regular GPU. Please see [MS MARCO *document* ranking task](https://github.com/castorini/anserini/blob/master/docs/experiments-msmarco-doc.md), [MS MARCO *passage* ranking task - Subset](https://github.com/castorini/anserini/blob/master/docs/experiments-msmarco-passage-subset.md) and [MS MARCO *passage* ranking task - Entire](https://github.com/castorini/anserini/blob/master/docs/experiments-msmarco-passage-entrie.md).

Prior to running this, we suggest looking at our first-stage [BM25 ranking instructions](https://github.com/castorini/anserini/blob/master/docs/experiments-msmarco-passage.md).
We rerank the BM25 run files that contain ~1000 passages per query using monoT5.
monoT5 is a pointwise reranker. This means that each document is scored independently using T5.

## Data Prepare

Since we will use some scripts form Pygaggle to process data and evaluate results, you need to install Pygaggle.
```
git clone --recursive https://github.com/castorini/pygaggle.git
cd pygaggle
pip install .
```

We're first going to download the queries, qrels, run and corpus corresponding to the entire MS MARCO dev set considered. 

The run file is generated by following the BM25 ranking instructions by Anserini. Please see [here](https://github.com/castorini/anserini/blob/master/docs/experiments-msmarco-passage.md) for details.

We'll store all following files in the `data/msmarco_dev` directory.

- `queries.dev.small.tsv`: 6,980 queries from the MS MARCO dev set.
- `qrels.dev.small.tsv`: 7,437 pairs of query relevant passage ids from the MS MARCO dev set.
- `run.dev.small.tsv`: Approximately 6,980,000 pairs of dev set queries and retrieved passages using BM25.
- `collection.tar.gz`: All passages (8,841,823) in the MS MARCO passage corpus. In this tsv file, the first column is the passage id, and the second is the passage text.

For more description about the data, please see [here](https://github.com/castorini/duobert#data-and-trained-models)
```
cd data/msmarco_dev
wget https://storage.googleapis.com/duobert_git/run.bm25.dev.small.tsv
wget https://www.dropbox.com/s/hq6xjhswiz60siu/queries.dev.small.tsv
wget https://www.dropbox.com/s/5t6e2225rt6ikym/qrels.dev.small.tsv
wget https://www.dropbox.com/s/m1n2wf80l1lb9j1/collection.tar.gz
tar -xvf collection.tar.gz
mv run.bm25.dev.small.tsv run.dev.small.tsv
cd ../../
```
As a sanity check, we can evaluate the first-stage retrieved documents using the official MS MARCO evaluation script.

```
export DATA_DIR=data/msmarco_dev
python tools/eval/msmarco_eval.py ${DATA_DIR}/qrels.dev.small.tsv ${DATA_DIR}/run.dev.small.tsv
```

The output should be:

```
#####################
MRR @10: 0.18736452221767383
QueriesRanked: 6980
#####################
```

Then create query-doc pairs for monoT5 input format.
```
python -m pygaggle.data.create_msmarco_monot5_input --queries ${DATA_DIR}/queries.dev.small.tsv \
                                      --run ${DATA_DIR}/run.bm25.dev.small.tsv \
                                      --corpus ${DATA_DIR}/collection/collection.tsv \
                                      --t5_input ${DATA_DIR}/query_doc_pairs.dev.small.txt \
                                      --t5_input_ids ${DATA_DIR}/query_doc_pair_ids.dev.small.tsv
```
We will get two output files here:
- `query_doc_pairs.dev.small.txt`: The query-doc pairs for monoT5 input.
- `query_doc_pair_ids.dev.small.tsv`: The `query_id`s and `doc_id`s that mapping to the query-doc pairs. We will use this to map monoT5 output scores back to query-doc pairs.

Note that there will be a memory issue if the monoT5 input file is large. Thus, we will split the input file into multiple files.

```
split --suffix-length 3 --numeric-suffixes --lines 1000000 ${DATA_DIR}/query_doc_pairs.dev.small.txt ${DATA_DIR}/query_doc_pairs.dev.small.txt
```

For `query_doc_pairs.dev.small.txt`, we will get 7 files after split. i.e. (`query_doc_pairs.dev.small.txt000` to `query_doc_pairs.dev.small.txt006`)

Then copy these input files to Google Storage. TPU inference will read data directly from `gs`
```
export GS_FOLDER=<google storage folder to store input/output data>
gsutil cp ${DATA_DIR}/query_doc_pairs.dev.small.txt??? ${GS_FOLDER}
```

## Start a VM with TPU on Google Cloud

Define environment variables.
```
export PROJECT_NAME=<gcloud project name>
export PROJECT_ID=<gcloud project id>
export INSTANCE_NAME=<name of vm to create>
export TPU_NAME=<name of tpu to create>
```

Create the VM.
```
gcloud beta compute --project=${PROJECT_NAME} instances create ${INSTANCE_NAME} --zone=europe-west4-a --machine-type=n1-standard-4 --subnet=default --network-tier=PREMIUM --maintenance-policy=MIGRATE --service-account=${PROJECT_ID}-compute@developer.gserviceaccount.com  --scopes=https://www.googleapis.com/auth/cloud-platform --image=debian-9-stretch-v20191014 --image-project=debian-cloud --boot-disk-size=200GB --boot-disk-type=pd-standard --boot-disk-device-name=${INSTANCE_NAME} --reservation-affinity=any
```

After the VM created, we can `ssh` to the machine.  
Then create a TPU.

```
curl -O https://dl.google.com/cloud_tpu/ctpu/latest/linux/ctpu && chmod a+x ctpu

./ctpu up --name=${TPU_NAME} --project=${PROJECT_NAME} --zone=europe-west4-a --tpu-size=v3-8 --tpu-only --noconf
```

## Setup environment on VM
Install required tools.
```
sudo apt-get update
sudo apt-get install git gcc screen --yes
```

Install [Miniconda](https://docs.conda.io/en/latest/miniconda.html).
```
curl -O https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
bash ./Miniconda3-latest-Linux-x86_64.sh
```
When installation finished, do:
```
source ~/.bashrc
```

Then create a Python virtual environment for the experiments. 
```
conda init
conda create --y --name py36 python=3.6
conda activate py36
```
Install dependencies.
```
conda install -c conda-forge httptools jsonnet --yes
pip install tensorflow tensorflow-text t5[gcp]
git clone https://github.com/castorini/mesh.git
pip install --editable mesh
```

## Reranking with monoT5
On the TPU machine define model type and checkpoint.  
(Here we use our pretrained monoT5-3B as example)
```
export MODEL=3B
export CHECKPOINT=gs://neuralresearcher_data/doc2query/experiments/363
```

Then run following command to start the process in background and monitor the log
```
export EXPNO=001
for ITER in {000..006}; do
  echo "Running iter: $ITER" >> out.log_eval_$EXPNO
  nohup t5_mesh_transformer \
    --tpu="${TPU_NAME}" \
    --gcp_project=${PROJECT_NAME} \
    --tpu_zone="europe-west4-a" \
    --model_dir="${CHECKPOINT}" \
    --gin_file="gs://t5-data/pretrained_models/$MODEL/operative_config.gin" \
    --gin_file="infer.gin" \
    --gin_file="beam_search.gin" \
    --gin_param="utils.tpu_mesh_shape.tpu_topology = '2x2'" \
    --gin_param="infer_checkpoint_step = 1100000" \
    --gin_param="utils.run.sequence_length = {'inputs': 512, 'targets': 64}" \
    --gin_param="Bitransformer.decode.max_decode_length = 64" \
    --gin_param="input_filename = '${GS_FOLDER}/query_doc_pairs.dev.small.txt${ITER}'" \
    --gin_param="output_filename = '${GS_FOLDER}/query_doc_pair_scores.dev.small.txt${ITER}'" \
    --gin_param="tokens_per_batch = 65536" \
    --gin_param="Bitransformer.decode.beam_size = 1" \
    --gin_param="Bitransformer.decode.temperature = 0.0" \
    --gin_param="Unitransformer.sample_autoregressive.sampling_keep_top_k = -1" \
    >> out.log_eval_exp${EXPNO} 2>&1
done &

tail -100f out.log_eval_exp${EXPNO}
```

It takes about 35 hours to rerank on a TPU v3-8.

NOTE: We strongly encourage you to run above processes in `screen` to make sure the processes doesn't get interrupted.

## Evaluate Result
After rerank finished. Copy the results from GS to your work directory. And concate all score files back to one file.
```
gsutil cp ${GS_FOLDER}/query_doc_pair_scores.dev.small.txt???-1100000 ${DATA_DIR}/
cat ${DATA_DIR}/query_doc_pair_scores.dev.small.txt???-1100000 > ${DATA_DIR}/query_doc_pair_scores.dev.small.txt
```

Then convert the monoT5 output back to MSMARCO format.
```
python -m pygaggle.data.convert_t5_output_to_msmarco_run --t5_output ${DATA_DIR}/query_doc_pair_scores.dev.small.txt \
                                                --t5_output_ids ${DATA_DIR}/query_doc_pair_ids.dev.small.tsv \
                                                --msmarco_run ${DATA_DIR}/run.monot5_3b.dev.tsv
```

Now we can evaluate the rerank results using the official MS MARCO evaluation script.
```
python tools/eval/msmarco_eval.py ${DATA_DIR}/qrels.dev.small.tsv ${DATA_DIR}/run.monot5_3b.dev.tsv
```

The output should be:
```
#####################
MRR @10: 0.3983799517896949
QueriesRanked: 6980
#####################
```

You should see the same result.

If you were able to replicate these results, please submit a PR adding to the replication log! Please mention in your PR if you find any difference!

## Replication Log