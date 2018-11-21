# 實作-從YOLO到CoreML

****目標: 訓練自己的YOLO model並轉成iOS可以使用的coreML model。****



## Agenda



- 先認識基本的工具

​	- [darknet](#1)

​	- [darkflow](#2)

​	- [tf-coreml](#3)



\-  流程

​	- [1. 安裝darknet](#4)

​	- [2. 安裝darkflow](#5)

​	- [3. 準備自己的數據](#6)

​	- [4. 利用darkflow針對自己的數據進行客製化訓練](#7)

​	- [5. 利用darkflow將客製化的模型輸出成tensorflow](#8)

​	- [6. 利用tf-coreml將tensorflow轉成CoreML](#9)



<h2 id="1">darknet</h2>



\- [官方網頁](https://pjreddie.com/darknet/)

\- [github](https://github.com/pjreddie/darknet)



\- darknet主要功用

​	- 有很多已經訓練好現成可以用的模型。

​	- 這些模型都分別被拆成兩種檔案.cfg跟.weight。

​		- .cfg檔案: 定義了神經網路的架構。

​			- 例如，yolov3.cfg就是對應了yolov3的神經網路結構。

​		- .weight檔案: 訓練好的權重參數。

​			- 權重參數是先人辛苦訓練出來的精華，神經網路的架構越複雜，檔案就會越大。

​			- 例如，我決定使用YOLO_V3當作我的模型，你會發現在cfg的目錄有一個yolov3.cfg檔案，再去官網下載對應的yolov3.weight。



****到這裡，只要先了解.cfg跟.weight是做啥的就可以了。****	



<h2 id="2">darkflow</h2>	



\- [github](https://github.com/thtrieu/darkflow)



\- darkflow主要功用

​	- 可以直接載入darknet已經訓練好的模型。

​	- 可以重新進行訓練這些模型(ex. 客製化自己想要的模型)

​	- 可以輸出tensorflow graph的模型(.pb)



****darkflow在整個tutorial中扮演很重要的角色，因為客製化的模型就是在這裡執行。****





<h2 id="3">tf-coreml</h2>	



\- [github](https://github.com/tf-coreml/tf-coreml)



\- tf-coreml主要功用

​	- 將darkflow產生的Tensorflow graph的模型(.pb)變成coreML模型(.mlmodel)。



****目前，應該有大概猜到流程該怎麼做了，先從darknet選定要用的模型.cfg跟.weight，然後將這兩個東西丟到darkflow，並搭配我們自己的數據，進行客製化的模型訓練，最後再利用tf-coreml轉成iOS可以用的model。EASY !!****





<h2 id="4">安裝 darknet</h2>	



\```

git clone https://github.com/pjreddie/darknet.git



cd darknet



make

\```



如果你把darknet clone下來後，觀察darknet的目錄，會發現裡面一個cfg的目錄，裡面放了很多.cfg檔，這些都是已經定義好的神經網路架構。



![1](https://github.com/htaiwan/yolo_coreml/blob/master/Assets/1.png)





<h2 id="5">安裝darkflow</h2>	



\```

git https://github.com/thtrieu/darkflow.git

 

cd darkflow



pip install .

\```



安裝完成後，可以用這個指令來驗證是否有成功安裝。



\```

flow --h

\```



<h2 id="6">準備自己的數據</h2>	



\- 數據擺放位置，我在clone下來的darkflow目錄，****新增兩個目錄。****

​	- weight: 從.darknet下載下來的.weight

​	- train: 自己的數據，裡面分成兩個目錄。

​		- images: 自己的數據圖片(.jpg)。

​		- annotations: 數據圖片的對應資料(.xml)



其中images跟annotations的檔案名稱是一一對應的。



![2](https://github.com/htaiwan/yolo_coreml/blob/master/Assets/2.png)



\- XML裡面是什麼東西？

​	- 最主要就是框出照片中要辨識物件的座標跟類型。

​	

![3](https://github.com/htaiwan/yolo_coreml/blob/master/Assets/3.png)



\- 要如何標示數據呢 ? 要如何產生這樣XML ?

​	- goole搜尋: "yolo, label, tool"，會找到很多工具，可以幫忙我們從照片產生對應xml。

​		- [現成的label tool](https://github.com/tzutalin/labelImg)





<h2 id="7">利用darkflow針對自己的數據進行客製化訓練</h2>	

假設，我們現在要訓練自己的yolo model，可以區別3種不同物體(上衣，褲子，裙子)。



\- 先進入目錄 darkflow/cfg

\- 因為我想訓練自己的yolo model，複製目錄中的yolo.cfg，並命名成自己想要的名稱(yolo-ht.cfg)。

​	- 至於為何這麼做，在darkflow github上有解釋。

​	

​	> *Why should I leave the original tiny-yolo-voc.cfg file unchanged?*



​	> *When darkflow sees you are loading tiny-yolo-voc.weights it will look for tiny-yolo-voc.cfg in your cfg/ folder and compare that configuration file to the new one you have set with --model cfg/tiny-yolo-voc-3c.cfg. In this case, every layer will have the same exact number of weights except for the last two, so it will load the weights into all layers up to the last two because they now contain different number of weights.*

​	

![4](https://github.com/htaiwan/yolo_coreml/blob/master/Assets/4.png)



\- 修改yolo-ht.cfg的內容，因為我們要重新訓練成可以辨識不同的物體。

​	- 修改region中的class數目，這裡因為我們想區別3種，所以就改成3。

​	- 修改region上面convolutional中的filter數目，公式如下。

​		- filter數目 = num * (classes + 5)

​		- 40 = 5 * (3 + 5)



![6](https://github.com/htaiwan/yolo_coreml/blob/master/Assets/6.png)



\- 修改darkflow中的label.text內容。

​	- 就是想要區別的物體名稱 top, trousers, shirt。

​	- 這要跟xml中的object的tag是對應的。



![7](https://github.com/htaiwan/yolo_coreml/blob/master/Assets/7.png)



\- 從.darknet下載對應的yolo.weight到darkflow/weight



![5](https://github.com/htaiwan/yolo_coreml/blob/master/Assets/5.png)



\- 將之前辛苦準備好的數據放到darkflow/train/images跟darkflow/train/annotations



\- 接下來，就是最重要魔法指令，進行訓練。

​	- --model cfg/yolo-ht.cfg: 剛剛重新命名.cfg。

​	- --load weight/yolo.weights: 剛剛下載下來的weight檔案。

​	- --train --annotation train/Annotations --dataset train/Images: 開始根據annotations跟images目錄下的數據進行訓練。



\```

flow --model cfg/yolo-ht.cfg --load weight/yolo.weights --train --annotation train/Annotations --dataset train/Images

\```



\- 如果有看到下面的圖，那就恭喜了，表示客製化訓練正在進行中了。



![8](https://github.com/htaiwan/yolo_coreml/blob/master/Assets/8.png)



![9](https://github.com/htaiwan/yolo_coreml/blob/master/Assets/9.png)





<h2 id="8">利用darkflow將客製化的模型輸出成tensorflow</h2>



\- 透過這個指令會產生兩個檔案.pb和.meta。



\```

flow --model cfg/yolo-ht.cfg --load weight/yolo.weights --savepb

\```



<h2 id="9">利用tf-coreml將tensorflow轉成CoreML</h2>



\- [先安裝tf-coreml](https://github.com/tf-coreml/tf-coreml#installation)

\- [下載寫好的converter](https://github.com/syshen/YOLO-CoreML/blob/master/Converter/Convert_pb_coreml.ipynb)

\- 將.pb, .meta跟.ipynb放在同一個目錄下，然後執行.ipynb，就會看到感動的.mlmodel產生出來。



![10](https://github.com/htaiwan/yolo_coreml/blob/master/Assets/10.png)



\- 下載這個[專案](https://github.com/hollance/YOLO-CoreML-MPSNNGraph)，然後將剛剛訓練好的yolo.mlmodel放進去，看看執行的結果如何。

