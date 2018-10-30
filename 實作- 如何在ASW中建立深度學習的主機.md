# 實作- 如何在ASW中建立深度學習的主機

## Agenda

- [介紹](#1)
- [註冊帳號](#2)
- [填寫需求申請](#3)
- [建立密鑰](#4)
- [啟動實例](#5)
  - [選擇AMI](#6)
  - [選擇實例類型](#7)
  - [配置安全組](#8)
  - [配置密鑰對](#9)
- [連接實例](#10)
- [Jupyter使用](#11)

## Note

<h4 id="1">介紹</h4>

- [Amazon EC2 P2 介紹](https://aws.amazon.com/cn/ec2/instance-types/p2/)

  - P2 实例最多可提供 16 个 NVIDIA K80 GPU、64 个 vCPU 和 732GiB 主机内存，以及共计 192GB 的 GPU 内存组合、40000 个并行处理核心、70 万亿次单精度浮点运算处理能力，以及超过 23 万亿次双精度浮点运算处理能力。P2 实例还能为多达 16 个 GPU 交付 GPUDirect™ (对等 GPU 通信) 功能，这样，多个 GPU 就能在一个主机内相互合作。

- [2018 Amazon EC2 P3 介紹 ](https://aws.amazon.com/cn/ec2/instance-types/p3/)

  - Amazon EC2 P3 实例最多配备 8 块最新一代的 NVIDIA Tesla V100 GPU，每个实例可以实现最高 1 petaflop 的混合精度性能，显著加快机器学习和高性能计算应用程序的速度。事实证明，Amazon EC2 P3 实例可以将机器学习训练时间从几天缩短为几分钟，并且缩短了高性能计算获得结果的时间。

- P3比P2效能更強。

- [EC2 Dashboard](https://ap-southeast-1.console.aws.amazon.com/ec2/v2/home?region=ap-southeast-1#Home:)


<h4 id="2">註冊帳號</h4>

- 註冊amazon帳號
- 需要填寫信用卡
- 註冊完成後，需要填寫p2或p3的需求申請單



<h4 id="3">填寫需求申請</h4>

- [需求申請單](https://console.aws.amazon.com/support/v1#/case/create?issueType=service-limit-increase&limitType=service-code-ec2-instances)

- Regarding - **Service Limit Increase**

- Limit Type - **EC2 Instances**

- Region - **隨意**

- Primary Instance Type - **Instance Limit**

- New limit value - **1**

- Use Case Description - **to train my neural network**

- 當送出需求申請單，就要等候回復了，通常都需要2~3天，但還是要申請region而定，之前申請p2有被打槍過一次，他們回信跟我說，現在p2需求量大，暫時不在提供需求申請，後來隔一陣子，再申請一次，就過了。


<h4 id="4">建立密鑰</h4>

- 找到Dashboard的側邊欄中->網路與安全->密鑰對

- 創建密鑰對
- 創建後公鑰是放在AWS server中，私鑰放在本機端
- 下載下來的的私鑰檔案格式為pem
- 之後建立SSH連線所需要這把密鑰



<h4 id="5">啟動實例</h4>

- 找到Dashboard的側邊欄中->實例->啟動實例
- **步骤 1: 选择一个 Amazon 系统映像(AMI)**
  - 選擇Deep Learning AMI，因为已經幫我們裝好NVIDIA驅動, Python, TensorFlow, Keras 等環境，省去安裝的麻煩。
- **步骤 2: 选择一个实例类型**
  - 根據需求: 在篩選條件進行刪選 -> 我是選擇GPU計算
  - 選擇p2.xlarge
    - 看之前需求申請單，你提出的需求是什麼就選什麼，記得千萬一定要通過需求，不然一定會啟動失敗。
- **步骤 3: 配置实例详细信息**
  - 請求競價型實例
    -  什麼是競價型實例？ 
      - 竞价的原理就是把**闲置**的资源，拿出来拍卖，不卖也浪费了，所以便宜。既然是闲置，那么人家不闲的时候就不给你用了，如果资源紧张，提前几分钟通知你，然后人家关机销毁你的机器了，所以开机就狂用，用完就遗弃。基本上能节省70%的费用，但缺陷也很明显，每次环境都要重新建立
- **步骤 4: 添加存储**
  - PASS，使用預設
- **步骤 5: 标签竞价请求**
  - PASS，使用預設
- **步骤 6: 配置安全组**
  - 选择 “创建**新的**安全组"
  - 将”安全组名称”设为"Jupyter"
  - 将 “说明”设为"Jupyter"
  - 点击“添加规则”
  - 设置”自定义 TCP 规则”
    - 将“端口范围”设为"8888"
    - 将“来源”设为“任何地方”
- **步骤 7: 查看竞价型实例请求**
  - 啟動
    - 選擇現有密鑰對，選擇剛剛創建密鑰，並確認私鑰已經有存在本機端

<h4 id="10">連接實例</h4>

```
// 修改key的權限
centuryweed-lm:~ htaiwan$ chmod 700 /Users/htaiwan/Desktop/amazon_htaiwan.pem 

// 進行ssh連線
centuryweed-lm:~ htaiwan$ ssh -i /Users/htaiwan/Desktop/amazon_htaiwan.pem ubuntu@ec2-54-255-147-199.ap-southeast-1.compute.amazonaws.com
```



<h4 id="11">Jupyter使用</h4>

- Jupyter 設置

  ```
  // 進入根目錄
  cd ~
  // 創立config
  jupyter notebook --generate-config
  // 建立密碼
  jupyter notebook password
  ```

- 設置可以允許從外部連線(從本機端進行操作)

  ```
  vim ~/.jupyter/jupyter_notebook_config.py
  // 按I按钮进入 Insert 模式
  
  // 修改config下列幾行
  c.NotebookApp.ip='*'
  c.NotebookApp.open_browser = False
  c.NotebookApp.port =8888 
  
  // 依次按 Esc -> : -> w -> q ->enter
  ```

- 啟動Jupyter notebook

  ```
  jupyter notebook
  ```

- 測試Jupyter notebook

  - 將server ip:8888 複製到瀏覽器上，確認是否可以順利使用
  - 輸入剛剛建立的密碼



<h4 id="12">參考</h4>

- [利用AWS学习深度学习 For Udacity P5](https://zhuanlan.zhihu.com/p/33176260)
- [在AWS上配置深度学习主机](https://zhuanlan.zhihu.com/p/25066187)
- [AWS GPU 实例](https://classroom.udacity.com/nanodegrees/nd009-cn-advanced-career/parts/c5f59999-fb1a-49a3-bccc-ff9063011ba1/modules/0ab8dde3-b4e5-4f2d-adae-8a391c32758b/lessons/52fc79a7-13ff-4065-b3c6-8203ec9ef60c/concepts/ced1fa22-5723-4212-b73f-08f7f6613cae)