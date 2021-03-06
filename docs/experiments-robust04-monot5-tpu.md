# Neural Pointwise Ranking Baselines on Robust04 - with TPU

This page contains instructions for running monoT5 on the Robust 04 collection using Google TPUs.

To learn more about monoT5, please read "Document Ranking with a Pretrained Sequence-to-Sequence Model" [(Nogueira et al., 2020)](https://www.aclweb.org/anthology/2020.findings-emnlp.63.pdf)


**Note**: Robust04 uses [TREC Disks 4 & 5](https://trec.nist.gov/data/cd45/index.html) corpora, which are only available after filling and signing a release form from NIST. Therefore, only proceed with this documentation if you already have the corpus.

We will focus on using monoT5-3B since it is difficult to run such a large model without a TPU.
We also mention the changes required to run monoT5-base for those with a more constrained compute budget.

Prior to running this, we suggest looking at our first-stage [BM25 ranking instructions](https://github.com/castorini/anserini/blob/master/docs/regressions-robust04.md).
We rerank the BM25 run files that contain ~1000 documents per query using monoT5.
monoT5 is a pointwise reranker. This means that each document is scored independently using T5.

Note that we do not train monoT5 on Robust04. Hence, the results are **zero-shot**.

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
gcloud beta compute --project=${PROJECT_NAME} instances create ${INSTANCE_NAME} --zone=europe-west4-a --machine-type=e2-standard-8 --subnet=default --network-tier=PREMIUM --maintenance-policy=MIGRATE --service-account=${PROJECT_ID}-compute@developer.gserviceaccount.com  --scopes=https://www.googleapis.com/auth/cloud-platform --image=ubuntu-1804-bionic-v20210129 --image-project=ubuntu-os-cloud --boot-disk-size=50GB --boot-disk-type=pd-standard --boot-disk-device-name=${INSTANCE_NAME} --reservation-affinity=any
```

After the VM is created, we can `ssh` to the machine.  
Make sure to initialize `PROJECT_NAME` and `TPU_NAME` from within the machine too.
Then create a TPU.

```
curl -O https://dl.google.com/cloud_tpu/ctpu/latest/linux/ctpu && chmod a+x ctpu
./ctpu up --name=${TPU_NAME} --project=${PROJECT_NAME} --zone=europe-west4-a --tpu-size=v3-8 --tpu-only --noconf  --tf-version=2.3
```

## Setup environment on VM

Install the required tools, including [Miniconda](https://docs.conda.io/en/latest/miniconda.html).
```
sudo apt-get update
sudo apt-get install git gcc screen --yes
curl -O https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
bash ./Miniconda3-latest-Linux-x86_64.sh
source ~/.bashrc
```

Then create a Python virtual environment for the experiments and install dependencies.
```
conda init
conda create --y --name py36 python=3.6
conda activate py36
pip install cloud-tpu-client tensorflow==2.3.1 tensorflow-text t5==0.7.1
git clone https://github.com/castorini/mesh.git
pip install --editable mesh
git clone --recursive https://github.com/castorini/pygaggle.git
cd pygaggle
pip install .
cd tools/eval && tar xvfz trec_eval.9.0.4.tar.gz && cd trec_eval.9.0.4 && make && cd ../../..
```

## Data Prep
We store all the files in the `data/robust04` directory.
```
export DATA_DIR=data/robust04
mkdir ${DATA_DIR}
```

We download the query, qrels, run and corpus files. 

The run file is generated by following the Anserini's [BM25 ranking instructions](https://github.com/castorini/anserini/blob/master/docs/regressions-robust04.md).

In short, the files are:
- `topics.robust04.txt`: 250 queries (also called "topics") from Robust04.
- `qrels.robust04.txt`: 311,410 pairs of query and relevant document ids.
- `run.bm25.txt`: 242,339 pairs of queries and retrieved documents using Anserini's BM25.
- `trec_disks_4_and_5_concat.txt`: TREC disks 4 & 5 documents (528,164) concatenated as a single text file.

Let's start.
```
cd ${DATA_DIR}
wget https://raw.githubusercontent.com/castorini/anserini/master/src/main/resources/topics-and-qrels/topics.robust04.txt
wget https://raw.githubusercontent.com/castorini/anserini/master/src/main/resources/topics-and-qrels/qrels.robust04.txt
wget https://storage.googleapis.com/castorini/robust04/run.robust04.bm25.txt
wget https://storage.googleapis.com/castorini/robust04/trec_disks_4_and_5_concat.txt
cd ../../
```

As a sanity check, we can evaluate the first-stage (BM25) retrieved documents using the `trec_eval` tool:
```
tools/eval/trec_eval.9.0.4/trec_eval -m map -m ndcg_cut.20 ${DATA_DIR}/qrels.robust04.txt ${DATA_DIR}/run.robust04.bm25.txt
```

The output should be:
```
map                     all     0.2531
ndcg_cut_20             all     0.4240
```

**Note:** You may skip the remaining of this section by downloading the already preprocessed files from our [bucket](https://console.cloud.google.com/storage/browser/castorini/monot5/data/robust04).

Then, we prepare the query-doc pairs in the monoT5 input format. 

```
python ./pygaggle/data/create_robust04_monot5_input.py \
      --queries=${DATA_DIR}/topics.robust04.txt \
      --run=${DATA_DIR}/run.robust04.bm25.txt \
      --corpus=${DATA_DIR}/trec_disks_4_and_5_concat.txt \
      --output_segment_texts=${DATA_DIR}/segment_texts.txt \
      --output_segment_query_doc_ids=${DATA_DIR}/segment_query_doc_ids.tsv

```
This conversion should take approximately 40 minutes and it creates two files:
- `segment_texts.txt`: 2,018,386 query-doc pairs for monoT5 input.
- `segment_query_doc_ids.tsv`: 2,018,386 query and doc ids that are aligned with segment_texts.txt. We will use this file to map query-doc pairs to their corresponding monoT5 output scores.

Since there might be a memory error if the input file to monoT5 is too large, we split it into multiple files:
```
split --suffix-length 3 --numeric-suffixes --lines 500000 ${DATA_DIR}/segment_texts.txt ${DATA_DIR}/segment_texts.txt
```

We get 5 files after split (i.e., `segment_texts.txt000` to `segment_texts.txt004`).

We then copy these input files to Google Storage. TPU inference will read data directly from there.
```
export GS_FOLDER=<google storage folder to store input/output data>
gsutil cp ${DATA_DIR}/segment_texts.txt??? ${GS_FOLDER}
```

## Rerank with monoT5

Let's first define the model type and checkpoint.

```
export MODEL_NAME=<base or 3B>
export MODEL_DIR=gs://castorini/monot5/experiments/${MODEL_NAME}
export CHECKPOINT_STEP=1100000
```

Then run the following command to start the process in background and monitor the log:
```
for ITER in {000..004}; do
  echo "Running iter: $ITER" >> logs/out.log
  nohup t5_mesh_transformer \
    --tpu="${TPU_NAME}" \
    --gcp_project="${PROJECT_NAME}" \
    --tpu_zone="europe-west4-a" \
    --model_dir="${MODEL_DIR}" \
    --gin_file="gs://t5-data/pretrained_models/${MODEL_NAME}/operative_config.gin" \
    --gin_file="infer.gin" \
    --gin_file="beam_search.gin" \
    --gin_param="utils.tpu_mesh_shape.tpu_topology = '2x2'" \
    --gin_param="infer_checkpoint_step = ${CHECKPOINT_STEP}" \
    --gin_param="utils.run.sequence_length = {'inputs': 512, 'targets': 2}" \
    --gin_param="Bitransformer.decode.max_decode_length = 2" \
    --gin_param="input_filename = 'gs://castorini/monot5/data/robust04/segment_texts.txt${ITER}'" \
    --gin_param="output_filename = '${DATA_DIR}/monot5-${MODEL_NAME}_scores.txt${ITER}'" \
    --gin_param="utils.run.batch_size=('tokens_per_batch', 65536)" \
    --gin_param="Bitransformer.decode.beam_size = 1" \
    --gin_param="Bitransformer.decode.temperature = 0.0" \
    --gin_param="Unitransformer.sample_autoregressive.sampling_keep_top_k = -1" \
    >> logs/out.log 2>&1
done &

tail -100f logs/out.log
```

Using a TPU v3-8, it takes approximately 1.5 and 3.5 hours to rerank with monoT5-base and monoT5-3B, respectively.

You might want to run this process using `screen` to make sure it does not get killed.

## Evaluate reranked results
After reranking is done, we concatenate all the score files into one file.
```
cat ${DATA_DIR}/monot5-${MODEL_NAME}_scores.txt???-1100000 > ${DATA_DIR}/monot5-${MODEL_NAME}_scores.txt
```

We then convert the monoT5 output to the required `trec_eval` format.
```
python pygaggle/data/convert_run_from_t5_to_trec_format.py \
    --predictions=${DATA_DIR}/monot5-${MODEL_NAME}_scores.txt \
    --query_run_ids=${DATA_DIR}/segment_query_doc_ids.tsv \
    --output=${DATA_DIR}/run.monot5-${MODEL_NAME}.txt
```

Now we can evaluate the reranked results using the `trec_eval` tool:
```
tools/eval/trec_eval.9.0.4/trec_eval -m map -m ndcg_cut.20 ${DATA_DIR}/qrels.robust04.txt ${DATA_DIR}/run.monot5-${MODEL_NAME}.txt
```

For monoT5-base, the output should be:

```
map                     all     0.3291
ndcg_cut_20             all     0.5302
```

For monoT5-3B, the output should be:

```
map                     all     0.3874
ndcg_cut_20             all     0.6088
```

If you were able to replicate these results, please submit a PR adding to the replication log, along with the model(s) you replicated. 
Please mention in your PR if you note any differences.


## Replication Log
