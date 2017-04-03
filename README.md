
## Kaggle - The Nature Conservancy

### Steps to execute the process
1) Set up environment on a aws p2 instance with ~ 200GB space
     - Install Ubuntu environment as laid out here : http://course.fast.ai/lessons/aws.html
     - Install R, and package data.table
2) Place the train-all folder under /data/fish/
3) Place the test-stg2 files under /data/fish/test and also place them under /data/fish/test/test
4) Create directory data/fish/results
5) Check that the annotations (from kaggle forum here: http://bit.ly/2nN05Ld) are in ../data/fish/annos/{}_labels.json
6) Check that the labels from Liu Weijie in this post http://bit.ly/2nBceSA are under ../data/fish/annos1/{}.json
7) Place the relabelling file from this post (http://bit.ly/2o8W9Vw) under the directory ../data/fish/relabel/relabels.csv
8) Change R blending script called *** to point at home directory in setwd() step.
9) Create directory data/fish/pseudo and data/fish/pseudoresnet
10) Pretrained weights were sourced from http://www.platform.ai/models/ and https://keras.io/applications/ and https://pjreddie.com/darknet/yolo/ 
All weights should be downloadable on executuion of the code. I have created and saved my own weights for the resnet models; however executing the code below should create the same weights. 


###########################################################

### Yolo set up part #1

```
chmod +v yolo_setup1.sh
./yolo_setup1.sh
```

Manually do the next step :
Place the labels from Liu Weijie in this post http://bit.ly/2nBceSA under fish/darknet/FISH/annos

Yolo set up part #2 - it may be better to do this manually, as some steps are not automated. Execute from main directory. 
```
python voc_label_FISH1.py
cp train-all.txt darknet/train.txt 
cp test.txt darknet/test.txt 
```

#### Train yolo on 414x414 images
Manually, at train time, in the darknet/Makefile, change the line "CUDNN=0", to "CUDNN=1"; if needed, and then execute ./darknet/make to remake
The next step runs for about a day, run this from the darknet directory
```
./darknet detector train cfg/voc.FISH1.data cfg/yolo-voc-FISH.cfg darknet19_448.conv.23 
```
Manually, at predict time, in the darknet/Makefile, change the line "CUDNN=1", to "CUDNN=0" and then execute ./darknet/make to remake
Run this from the darknet directory
```
./darknet detector valid cfg/voc.FISH1.data cfg/yolo-voc-FISH.cfg backupFISH/yolo-voc-FISH_11000.weights
```
Run the below from the main directory
```
cp darknet/results/comp4_det_test_FISH.txt yolo_coords/comp4_det_test_FISH.txt
cp darknet/results/comp4_det_test_FISH.txt yolo_coords/comp4_det_test_FISH540.txt
```

#### Train yolo on 544x544 images

Manually, at train time, in the darknet/Makefile, change the line "CUDNN=0", to "CUDNN=1"; if needed, and then execute ./darknet/make to remake
The next step runs for about a day, run this from the darknet directory
```
nohup ./darknet detector train cfg/voc.FISH1.data cfg/yolo-voc-FISH544.cfg darknet19_448.conv.23 > nohup544.out 2>&1&
```

Manually, at predict time, in the darknet/Makefile, change the line "CUDNN=1", to "CUDNN=0" and then execute ./darknet/make to remake
Run this from the darknet directory

```
./darknet detector valid cfg/voc.FISH1.data cfg/yolo-voc-FISH544.cfg backupFISH/yolo-voc-FISH544_11000.weights
```
Run the below from the main directory
```
cp darknet/results/comp4_det_test_FISH.txt yolo_coords/comp4_det_test_FISH544.txt
```

### Run Python Scripts
Create the below directories
```
mkdir final/checkpoints/checkpoint03A
mkdir final/checkpoints/checkpoint03B
mkdir final/checkpoints/checkpoint03C
mkdir final/checkpoints/checkpoint04A
mkdir final/checkpoints/checkpoint04B
mkdir final/checkpoints/checkpoint04C
mkdir final/checkpoints/checkpoint05A
mkdir final/checkpoints/checkpoint05B
mkdir final/checkpoints/checkpoint05C
mkdir final/checkpoints/checkpoint08A
mkdir final/checkpoints/checkpoint08B
mkdir final/checkpoints/checkpoint08C
```

Run the following from the `final/` directory. Note, the models marked `*A_*`, `*B_*` and `*C_*` are the exact same except the wieghts are svaed to different locations to allow predictions to be bagged or averaged. 
```
cd final
# Script 1 - VGG training on boundary boxes and classes simulatneously
nohup python 1_conv_all_anno.py &> 1_conv_all_anno.out&

# Script 2 - VGG training on boundary boxes and classes simulatneously; using relabel classes from the forums
nohup python 2_conv_all_relabel.py &> 2_conv_all_relabel.out&

# Script 3 - 3 x Bagged resnet with data augmentation on Yolo 414 crops
nohup python 3A_resnet_crop_partial.py &> 3A_resnet_crop_partial.out&
nohup python 3B_resnet_crop_partial.py &> 3B_resnet_crop_partial.out&
nohup python 3C_resnet_crop_partial.py &> 3C_resnet_crop_partial.out&

# Script 4 - 3 x Bagged resnet with data augmentation on Yolo 414 crops, resizing the crops to be a square cut out of original image
nohup python 4A_resnet_cropsq_partial.py &> 4A_resnet_cropsq_partial.out&
nohup python 4B_resnet_cropsq_partial.py &> 4B_resnet_cropsq_partial.out&
nohup python 4C_resnet_cropsq_partial.py &> 4C_resnet_cropsq_partial.out&

# Script 6 - Make initial prediction with the above models to be used for pseudo labelling
python distances_test.py
Rscript avg_subs_final_round1.R # Run this rscript
nohup python 6_conv_all_pseudo.py &> 6_conv_all_pseudo.out&

# Script 7 - 3 x Bagged resnet with data augmentation on Yolo 544 crops, resizing the crops to be a square cut out of original image
nohup python 7A_resnet_544predonly_partial.py &> 7A_resnet_544predonly_partial.out&
nohup python 7B_resnet_544predonly_partial.py &> 7B_resnet_544predonly_partial.out&
nohup python 7C_resnet_544predonly_partial.py &> 7C_resnet_544predonly_partial.out&

# Script 8 - VGG same as script 1, just augmenting training data with high confidence pseudo labels 
nohup python 8A_resnet_pseudo_partial.py  &> 8A_resnet_pseudo_partial.out&
nohup python 8B_resnet_pseudo_partial.py  &> 8B_resnet_pseudo_partial.out&
nohup python 8C_resnet_pseudo_partial.py  &> 8C_resnet_pseudo_partial.out&

# Blend all the scripts - weighted average of all scripts with some thresholding of minority classes.
Rscript avg_subs_final_vggpseudo.R
Rscript avg_subs_final_resnetpseudo.R
```
