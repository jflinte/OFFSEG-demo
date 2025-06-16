# OFFSEG-demo


# OFFSEG pipeline.py 

#### Header Information
```
# -*- coding: utf-8 -*-
"""
Created on Wed Feb 10 15:07:05 2021

@author: KASI VISWANATH && KARTIKEYA SINGH
"""
```
#### Imports 

`import cv2` opencv-python, module for computer vision and machine learning </br>
`import sys` system, module for allowing interaction with python interpreter</br>
`import numpy as np` NumPy, module for numerical computing</br>
`import tensorflow as tf` module for building and training ML </br>
`import pandas as pd` module for data manipulation and analysis</br>
`#from libKMCUDA import kmeans_cuda` from scikit-learn, needs to be studied more</br>
`from PIL import Image,ImageOps` Pillow, module for image  processing </br>
`import time` module with time operations</br>
`import logging` module for event tracking (debugging, monitoring, etc.)</br>
`import torch` PyTorch, module for deep learning and tesnor computation</br>
`sys.path.insert(0, '.')` check current directory before checking other locations</br>
`import argparse` module for parsing commandline arguments</br>
`torch.set_grad_enabled(False)` disable gradient computation (autograd stops tracking operations, preventing unnecessary calculations and imporoving memory efficiency and performance)</br>
`import os` module for interacting with the operating system</br>
`from sklearn.cluster import KMeans` class from sklearn.cluster, used for K-Means clustering algorithm</br>
`from fast_pytorch_kmeans import KMeans` PyTorch based implementation of K-Means clustering algorithm (significantly faster than sklearn.cluster,KMeans, as it has GPU acceleration support)</br>

`import lib.transform_cv2 as T` import custom module</br>
`from lib.models import model_factory` check models/__init__.py for more details (basically gives access to BiSeNetV1 and BiSeNetV2) </br>
`from configs import cfg_factory` check configs/__init__.py for more details (gets configeration details for BiSeNetV1 and BiSeNetV2, storing it in a dictionary)</br>

#### Inilitization
`np.random.seed(123)` generate a seed for a random number (this is legacy function and doesn't actually follow best practices, but its function here is so simple, that I don't think it really matters)
`pal= np.random.randint(0, 256, (256, 3), dtype=np.uint8)` get array (of size 256x3) of random 8-bit unsigned integers between 0 and 256


`#torch_kmeans = KMeans(n_clusters=4, mode='euclidean', verbose=1) # 4 classes` ignored, as it is commented out, will be returned to at later period 

```
#image path
img_path='/path/to/Directory'
#result path
final_path='/path/to/Directory'
#Path to Model save path.
dataset='/path/to/Directory/model_final.pth'
```
This sets paths to images, models, etc.

Note that *.pth files store model parameters, weights, or checkpoint states (or even pretrained models)

#### Functions
```
#Function for 4 calss image segmentation
def img_seg(im,net):
    im = to_tensor(dict(im=im, lb=None))['im'].unsqueeze(0).cuda()
    out = net(im)[0].argmax(dim=1).squeeze().detach().cpu().numpy()
    pred=pal[out]
    out=cv2.cvtColor(out.astype('uint8'),cv2.COLOR_GRAY2BGR)
    return pred , out
```
Note, there is a typo, as it should be 4 *class* image segmentation.

References
`image=cv2.imread(os.path.join(img_path,img_list[i]))` gets the image
`im=image.copy()[:, :, ::-1]` copies the image, I believe reversing the channel order (as OpenCV often uses BGR, this likely converts it back to RGB)
`pred,pool =img_seg(im,net)` 

```
#Function for Kmeans clustering.     
def palette_lst(masked_img,n_classes=4):
    height=masked_img.shape[0]
    width=masked_img.shape[1]
    h,w=int(height),int(width)
    masked_img=cv2.resize(masked_img,(w,h))
    data = pd.DataFrame(masked_img.reshape(-1, 3),columns=['R', 'G', 'B'],dtype=np.float32)
    
    ## Uncomment below code for GPU based Kmeans clustering.
    #palette, data['Cluster']= kmeans_cuda(data, n_classes, verbosity=1, seed=1)

    ## Uncomment below code for CPU based Kmeans clustering.
     
#     outp=KMeans(n_clusters=4,random_state=0).fit(data)
#     print(outp)
#     palette=outp.cluster_centers_
#     data['Cluster']=outp.labels_

    ## Uncomment below code for Pytorch based Kmeans clustering.
    torch_kmeans = KMeans(n_clusters=n_classes, mode='euclidean', verbose=1)
    input = torch.tensor(data)
    data['Cluster'] = torch_kmeans.fit_predict(input).cpu().numpy()
    palette = torch_kmeans.centroids.cpu().numpy() 

    palette_list = list()
    for color in palette:
        palette_list.append([[tuple(color)]])
    data['R_cluster'] = data['Cluster'].apply(lambda x: palette_list[x][0][0][0])
    data['G_cluster'] = data['Cluster'].apply(lambda x: palette_list[x][0][0][1])
    data['B_cluster'] = data['Cluster'].apply(lambda x: palette_list[x][0][0][2])
    img_c = data[['R_cluster', 'G_cluster', 'B_cluster']].values
    img_c = img_c.reshape(h,w, 3)
    img_lst=[]
    for i in range(1,n_classes):
        j=img_c.copy()
        j[j!=palette_list[i]]=0
        img_lst.append(j)
    #returns list of masked images
    return img_lst

#Function for extracting traversable section from RGB image.
def trav_cut(img,lpool):
    lpool[lpool!=1]=255
    lpool=cv2.resize(lpool,(img.shape[1],img.shape[0]))
    dst= cv2.addWeighted(lpool,1,img,1,0)
    h, w, c = img.shape
    #dst=cp.asarray(dst)
# append Alpha channel -- required for BGRA (Blue, Green, Red, Alpha)
    image_bgra = np.concatenate([dst, np.full((h, w, 1), 255, dtype=np.uint8)], axis=-1)
# create a mask where white pixels ([255, 255, 255]) are True
    #dst=np.asarray(dst)
    white = np.all(dst == [0,0,0], axis=-1)
# change the values of Alpha to 0 for all the white pixels
    image_bgra[white, -1] = 0
    
# save the image
    masked_img = cv2.cvtColor(image_bgra, cv2.COLOR_BGRA2BGR)
    return masked_img

#Function for Mask Prediction.
def mask_pred(img_lst,model):
    mask_class=[]
    for i in img_lst:
        im=cv2.resize(i,(224,224))
        #im=cv2.cvtColor(im.astype(np.float32),cv2.COLOR_BGR2RGB)
        im=np.reshape(im,(1,224,224,3))
        im=np.asarray(im)
        nim=(im/ 127.0) - 1
        prediction = model.predict(nim)
#class in prediction:0=grass;1=puddle;2=dirt
        # if np.argmax(prediction)==1:
        #     mask_class.append(np.argmax(prediction)+5)
        # else:
        #     mask_class.append(np.argmax(prediction)+4)
        mask_class.append(np.argmax(prediction)+4)
    return mask_class
#function for combining the Mask.
def mask_comb(newpool,img_lst,mask_class):
    newpool=cv2.cvtColor(newpool.astype('float32'),cv2.COLOR_BGR2GRAY)
    for i in range(len(mask_class)):
        img=img_lst[i]
        ima=cv2.resize(img,(newpool.shape[1],newpool.shape[0]))
        #ima=cv2.erode(ima,cv2.getStructuringElement(cv2.MORPH_ELLIPSE,(3,3)))
        #im=cv2.erode(im,kernel,iterations=1)
        #im=cv2.GaussianBlur(im,(3,3),1)        
        #im=cv2.dilate(im,kernel,iterations=1)
        ima=cv2.cvtColor(ima.astype('float32'),cv2.COLOR_BGR2GRAY)
        ima[ima>0]=mask_class[i]
        bm=(mask_class[i]-ima)/mask_class[i]
        fp=np.multiply(newpool,bm)
        newpool=np.add(fp,ima)
        newpool=np.asarray(newpool,dtype=np.uint8)
    return newpool

def col_seg(image,pool,model):
    travcut=trav_cut(image,pool)
    msk_img=palette_lst(travcut)
    predicts=mask_pred(msk_img,model)
    return msk_img,predicts

```

#### Main
##### Commandline arguments
```
parse = argparse.ArgumentParser()
parse.add_argument('--model', dest='model', type=str, default='bisenetv2',)
parse.add_argument('--weight-path', type=str, default=dataset,)
args = parse.parse_args()
cfg = cfg_factory[args.model]

```
##### Load models
```
#Load the classification Model
model = tf.keras.models.load_model('../classification/model/keras_model.h5')
#Loading Segmentation Model.
net = model_factory[cfg.model_type](4)
net.load_state_dict(torch.load(args.weight_path, map_location='cpu'))
net.eval()
net.cuda()
```
##### Tensor
```
to_tensor = T.ToTensor(
    mean=(0.3257, 0.3690, 0.3223), # city, rgb
    std=(0.2112, 0.2148, 0.2115),
)
```

##### sort images
```
img_list=sorted(os.listdir(img_path))
#label_list=sorted(os.listdir(save_path))
```

##### Do semantic segmentation on images (Not actually sure what's going on here, that's just my initial guess)
```
for i in range(len(img_list)):
    image=cv2.imread(os.path.join(img_path,img_list[i]))
    #pool=cv2.imread(os.path.join(save_path,label_list[i]))
    im=image.copy()[:, :, ::-1]
    #image=cv2.resize(image,(1024,640))
    pred,pool =img_seg(im,net)
    cv2.imwrite(os.path.join(final_path,img_list[i][:len(img_list[i])]),pred)
    pool1=pool.copy()
    msk_img,predicts= col_seg(image,pool,model)
    final_pool=mask_comb(pool1,msk_img,predicts)
    cv2.imwrite(os.path.join(final_path,img_list[i]),pal[final_pool])
```
