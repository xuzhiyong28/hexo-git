---
title: fabric-study
date: 2022-08-22 11:02:26
tags:
---

### 本地搭建fabric

材料 :  https://www.aliyundrive.com/s/iwqsWupGKKr

由于科学上文的问题，根据`https://hyperledger-fabric.readthedocs.io/zh_CN/latest/install.html`教程无法下载，所以采用下面的方法去搭建。看`curl -sSL https://bit.ly/2ysbOFE | bash -s`的逻辑进行修改，跳过去外网下载和从git下载的步骤。

1. 下载材料里面的文件，新建目录`fabric-test`,
2. 再`fabric-test`目录下解压fabric-samples.zip
3. 进入fabric-samples目录，将`hyperledger-fabric-linux-amd64-2.4.6.tar`和`hyperledger-fabric-ca-linux-amd64-1.5.5.tar`放在此目录下。
4. 解压 `tar -zvxf hyperledger-fabric-linux-amd64-2.4.6.tar` 和 `tar -zvxf hyperledger-fabric-ca-linux-amd64-1.5.5.tar `
5. 将`bootstrap.sh`放到`fabric-test`目录下，执行`bash bootstrap.sh`。

