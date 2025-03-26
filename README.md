# Chromatin Interaction Neural Network (ChINN) - Modified Version
## Adapted for Comparative Analysis with [HiCDiffusionLooping](https://github.com/SFGLab/HiCDiffusionLooping)

### This version of ChINN has been modified for research purposes, specifically to compare with the HiCDiffusionLooping model. Please note that it deviates from the [original implementation of ChINN](https://github.com/mjflab/chinn).

Chromatin Interaction Neural Network (ChINN) only uses DNA sequences of the interacting open chromatin regions. ChINN is able to predict CTCF-, RNA polymerase II- and HiC- associated chromatin interactions between open chromatin regions. 

ChINN was able to identify convergent CTCF motifs, AP-1 transcription family member motifs such as FOS, and other transcription factors such as MYC as being important in predicting chromatin interactions.

ChINN also shows good across-sample performances and captures various sequence features that are predictive of chromatin interactions. 

### Environment

```shell
git clone https://github.com/SFGLab/chinn

# enter the directory. Use the repository's root directory as working directory.
cd chinn

# setup python env
conda create --name chinn python=3.8.20
pip install -r requirements.txt torch==1.9.1+cu111 -f https://download.pytorch.org/whl/torch_stable.html
conda activate chinn
```

download singularity images for data preprocessing
```shell
singularity pull tools.sif library://m10an/genomics/tools
singularity pull samtools.sif library://millironx/default/samtools
```

### Dataset

```shell
# https://ftp.1000genomes.ebi.ac.uk/vol1/ftp/technical/reference/GRCh38_reference_genome/GRCh38_full_analysis_set_plus_decoy_hla.fa

singularity exec samtools.sif samtools faidx GRCh38_full_analysis_set_plus_decoy_hla.fa
singularity exec samtools.sif samtools faidx GRCh38_full_analysis_set_plus_decoy_hla.fa $(seq -f 'chr%g' 1 22) chrX chrY > out_dir/hg38.fa
singularity exec samtools.sif samtools faidx out_dir/hg38.fa
```

```shell
data
├── gm12878_ctcf/hg38_lifted  # uplifted version of original data
│   ├── pairs.bedpe
│   ├── peaks.bed
│   └── dnase.bed
└── gm12878_ctcf/hg38         # data used for training HiCDiffusionLooping
    ├── 4DNFI9SL1WSF.bedpe    # https://data.4dnucleome.org/files-processed/4DNFI9SL1WSF/
    ├── 4DNFIV1N7TLK.bed      # https://data.4dnucleome.org/files-processed/4DNFIV1N7TLK/
    └── ENCFF759OLD.bed       # https://www.encodeproject.org/files/ENCFF759OLD/
```

__Note__: How up lifting of original hg19 data was made described in [data/gm12878_ctcf/hg38_lifted directory](https://github.com/SFGLab/chinn/tree/hg38/data/gm12878_ctcf/hg38_lifted)

__Note__: to run the scripts, use the root directory of this repository as the working directory.

### Pipeline
```shell
# prepare output directory
mkdir out_dir

singularity exec samtools.sif samtools faidx out_dir/hg38.fa
export PYTHONPATH=$PWD

bash preprocess/pipe.sh data/gm12878_ctcf/hg38/4DNFI9SL1WSF.bedpe \
                        data/gm12878_ctcf/hg38/ENCFF759OLD.bed \
                        data/gm12878_ctcf/hg38/4DNFIV1N7TLK.bed \
                        gm12878_ctcf \
                        out_dir

python data_preparation.py -m 1000 -e 500 \
                      --pos_files out_dir/gm12878_ctcf.clustered_interactions.both_dnase.bedpe \
                      --neg_files out_dir/gm12878_ctcf.neg_pairs_5x.from_singleton_inter_tf_random.bedpe \
                      -g out_dir/hg38.fa \
                      -n gm12878_ctcf_distance_matched -o out_dir

python train_distance_matched/train_distance_matched.py \
                       out_dir/gm12878_ctcf_distance_matched_singleton_tf_with_random_neg_seq_data_length_filtered \
                       gm12878_ctcf_model \
                       out_dir

python data_preparation.py -m 1000 -e 500 \
    --pos_files out_dir/gm12878_ctcf.clustered_interactions.both_dnase.bedpe \
    --neg_files out_dir/gm12878_ctcf.neg_pairs_5x.from_singleton_inter_tf_random.bedpe \
                out_dir/gm12878_ctcf.extended_negs_with_intra.bedpe \
    -g out_dir/hg38.fa \
    -n gm12878_ctcf_extended -o out_dir

for i in train valid test; 
do 
  python generate_factor_output.py \
                  out_dir/gm12878_ctcf_model.model.pt \
                  out_dir/gm12878_ctcf_distance_matched_singleton_tf_with_random_neg_seq_data_length_filtered_${i}.hdf5 \
                  gm12878_ctcf_${i} \
                  out_dir;
done

python train_extended/train_extended.py out_dir gm12878_ctcf out_dir/

python predict.py -m out_dir/gm12878_ctcf_model.model.pt -c out_dir/gm12878_ctcf_depth6.gbt.pkl --data_file out_dir/gm12878_ctcf_extended_singleton_tf_with_random_neg_seq_data_length_filtered_test.hdf5 --output_pre out_dir/gm12878_ctcf_extended_test -d

python predict_bedpe.py -m out_dir/gm12878_ctcf_model.model.pt \
                -c out_dir/gm12878_ctcf_depth6.gbt.pkl \
                --pos_files data/positives.bedpe \
                --neg_files data/negatives.bedpe \
                -g out_dir/hg38.fa \
                --min_size 1000 -e 500 -d \
                --output_pre out_dir/hg38

```

