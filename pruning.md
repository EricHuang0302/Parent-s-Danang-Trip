# Data Pruning / Model Pruning 學習筆記

> 目標：先建立 pruning 的基本觀念，再理解它如何用在 traffic classification、encrypted traffic classification、malicious traffic detection 等任務的輕量化模型上。

---

## 0. 先釐清名詞

在文獻裡，`pruning` 可能指不同層次的「剪枝」：

| 名詞 | 剪掉什麼 | 主要目的 | 和本主題的關係 |
| --- | --- | --- | --- |
| Model pruning / Neural network pruning | weight、neuron、channel、filter、layer | 降低模型大小、FLOPs、latency、memory footprint | 主要閱讀主線 |
| Data pruning / Dataset pruning | 訓練資料中的冗餘、低品質、低資訊量樣本 | 用較少資料訓練，降低訓練成本或提升資料品質 | 可作為補充方向 |
| Feature pruning / Feature selection | flow feature、packet feature、statistical feature | 減少輸入維度，提高可解釋性或降低計算量 | 傳統 ML 或 flow-level 方法會用到 |

這份筆記的主線先以 **model pruning** 為主，因為它最直接對應到「把模型做小、做快、做得更容易部署」。如果之後題目真的要強調 data pruning，則需要另外補 dataset pruning / coreset selection 的文獻。

---

## 1. Pruning 要先懂的問題

讀 paper 前，先把 pruning 拆成幾個核心問題：

1. **剪什麼？**
   - Unstructured pruning：剪單一 weight。
   - Structured pruning：剪 neuron、channel、filter、block、layer。
   - 在實際部署上，structured pruning 通常比較重要，因為硬體和一般深度學習框架比較容易得到真實 speedup。

2. **什麼時候剪？**
   - Before training：訓練前決定要剪的結構。
   - During training：訓練過程中加入 sparsity 或 pruning policy。
   - After training：先訓練完整模型，再剪枝與 fine-tune。

3. **根據什麼標準剪？**
   - Magnitude-based：權重絕對值小就剪。
   - Gradient / Taylor-based：估計剪掉某個參數或結構對 loss 的影響。
   - BN scale-based：用 BatchNorm 的 scaling factor 判斷 channel 重要性。
   - Search-based：用 RL、evolutionary search、NAS 類方法搜尋每層 pruning ratio。

4. **剪完之後怎麼恢復效能？**
   - Fine-tuning。
   - Knowledge distillation。
   - Re-training from scratch。
   - Pruning + quantization 一起做。

5. **怎麼證明真的有用？**
   - 不只看 accuracy / F1。
   - 還要看 parameter count、model size、FLOPs、MACs、latency、throughput、memory usage、energy、是否真的能在目標平台加速。

---

## 2. 建議閱讀順序

### 第一輪：先建立主線

第一輪只需要讀 3 篇，目標是快速知道 pruning 是什麼、traffic 任務怎麼接、security 任務真正要解什麼問題。

1. [A Survey on Deep Neural Network Pruning: Taxonomy, Comparison, Analysis, and Recommendations](https://arxiv.org/abs/2308.06767)
2. [Bufferless network traffic classification using reinforcement learning-based joint pruning-quantization](https://link.springer.com/article/10.1007/s12065-025-01122-x)
3. [Detecting Unknown Encrypted Malicious Traffic in Real Time via Flow Interaction Graph Analysis](https://www.ndss-symposium.org/ndss-paper/detecting-unknown-encrypted-malicious-traffic-in-real-time-via-flow-interaction-graph-analysis/)

### 第二輪：補 traffic pruning 與基礎方法

第二輪再補 pruning 在 traffic classification 的例子，以及經典 pruning 方法。

4. [Pruned-F1DCN: A lightweight network model for traffic classification](https://doi.org/10.1145/3584714.3584719)
5. [Learning both Weights and Connections for Efficient Neural Networks](https://arxiv.org/abs/1506.02626)
6. [Learning Efficient Convolutional Networks through Network Slimming](https://arxiv.org/abs/1708.06519)

### 如果題目真的要走 data pruning

如果之後想把重點從「模型剪枝」轉成「資料剪枝」，再讀：

- [Beyond neural scaling laws: beating power law scaling via data pruning](https://arxiv.org/abs/2206.14486)

這篇比較偏 dataset pruning / data selection，不是模型壓縮，但可以幫助理解「少資料是否也能訓練好模型」這條路線。

---

## 3. 核心必讀文獻整理

### 3.1 A Survey on Deep Neural Network Pruning

**定位：pruning 入門地圖。**

這篇不是要直接拿來實作，而是用來建立 pruning 的完整分類。讀這篇時，不需要一開始就看懂所有方法細節，重點是先知道不同 pruning 方法各自在解什麼問題。

**要抓的重點：**

- Structured pruning vs unstructured pruning。
- Global pruning vs layer-wise pruning。
- Training-aware pruning vs post-training pruning。
- Pruning 和 quantization、knowledge distillation、NAS 的關係。
- Pruning 指標不能只看 parameter reduction，還要看實際 latency 和 hardware friendliness。

**讀完要能回答：**

- 為什麼 unstructured pruning 參數變少，不一定代表 inference 真的變快？
- 為什麼 channel pruning / filter pruning 通常比較適合部署？
- 什麼情況下 pruning 後需要 fine-tuning？
- pruning ratio 越高一定越好嗎？

**對後續研究的用途：**

這篇可以當作 related work 的骨架。之後在寫背景時，可以先說明 pruning 的分類，再說明自己選擇哪一類方法，以及為什麼這一類方法適合 traffic/security 任務。

---

### 3.2 Bufferless network traffic classification using reinforcement learning-based joint pruning-quantization

**定位：最接近「輕量化 traffic classification」的主線 paper。**

這篇把 pruning 和 quantization 一起放進 network traffic classification。它的重點不是單純追求最高 accuracy，而是把模型部署效率一起放進設計目標。

**要抓的重點：**

- 為什麼 traffic classification 模型會遇到 edge deployment 的限制。
- 為什麼只剪 fully connected layer 不夠，convolution layer 的計算也要處理。
- 為什麼 layer-wise pruning 比 global pruning 更有彈性。
- 為什麼 pruning 和 quantization 可以互補。
- RL 在這裡負責搜尋每一層的 pruning / quantization 設定。

**讀 paper 時優先看：**

1. Introduction：看問題定義和 deployment motivation。
2. Related Work：整理 traffic classification 的輕量化方法。
3. Methodology：看 baseline CNN、RL search space、reward 設計。
4. Evaluation：看它怎麼同時比較 accuracy、complexity、storage、latency。

**可以學的寫法：**

這篇的寫法適合用來學「怎麼把準確率和部署效率一起講」。未來如果做類似題目，不應該只報 F1-score，而是要同時報：

- Accuracy / Precision / Recall / F1。
- Model size。
- Parameter count。
- FLOPs / MACs。
- Inference latency。
- Throughput。
- Memory footprint。
- Accuracy-efficiency trade-off。

---

### 3.3 Pruned-F1DCN: A lightweight network model for traffic classification

**定位：traffic classification 中 pruning-based lightweight model 的直接例子。**

這篇可以當作「pruning 已經被用在 traffic classification」的領域證據。它比 survey 更貼近應用，比 joint pruning-quantization 那篇更像單純 pruning-based traffic model。

**要抓的重點：**

- 原始模型架構是什麼。
- 剪枝前後模型大小、參數量、計算量如何變化。
- 剪枝後 accuracy / F1 是否維持。
- 它剪的是 weight、channel、filter，還是整個結構。
- 它的 baseline 和 dataset 是什麼。

**讀完要能回答：**

- 這篇的 pruning 是為了解決 storage 還是 computation？
- 它有沒有測實際 inference time？
- 它的模型是否容易放到 resource-constrained device？
- 它的實驗是否只在 accuracy 上有效，還是真的有部署價值？

**對後續研究的用途：**

這篇可以放在 related work 的 traffic pruning 區塊，用來說明 pruning 並不是只存在於 image classification，也可以接到 traffic classification。

---

### 3.4 Detecting Unknown Encrypted Malicious Traffic in Real Time via Flow Interaction Graph Analysis

**定位：幫助理解 security 任務本身，不是 pruning paper。**

這篇的重點是 unknown encrypted malicious traffic detection。它不靠已知攻擊 label，而是用 flow interaction graph 的結構特徵來偵測異常互動模式。

**要抓的重點：**

- encrypted traffic 為什麼讓 payload-based detection 變困難。
- unknown attack 和 closed-set traffic classification 有什麼差異。
- flow interaction graph 怎麼表示 traffic behavior。
- unsupervised detection 的優點與限制。
- real-time detection 的 throughput / latency 評估方式。

**讀完要能回答：**

- malicious traffic detection 和一般 traffic classification 差在哪裡？
- unknown / encrypted 的困難點是什麼？
- 如果模型做輕量化，會不會犧牲偵測未知攻擊的能力？
- 評估安全任務時，除了 F1，還需要看哪些指標？

**對後續研究的用途：**

這篇可以幫助把題目從「我想做 pruning」拉回到「我想解安全偵測問題」。也就是說，pruning 是手段，真正的研究問題應該是讓惡意流量偵測模型更適合即時或資源受限環境。

---

### 3.5 Learning both Weights and Connections for Efficient Neural Networks

**定位：經典 unstructured pruning paper。**

這篇提出常見的三步驟流程：

1. Train 原始模型。
2. Prune 不重要的 connection。
3. Retrain / fine-tune 剩下的 connection。

**要抓的重點：**

- magnitude-based pruning 的基本想法。
- 為什麼神經網路裡有大量 redundancy。
- pruning 後 fine-tuning 的必要性。
- unstructured pruning 的優點是壓縮率高，缺點是未必能直接帶來硬體加速。

**讀完要能回答：**

- 剪掉小權重的直覺是什麼？
- 為什麼參數變少不等於 latency 一定變低？
- sparse model 需要什麼樣的 library 或 hardware 才能真的加速？

**對後續研究的用途：**

這篇適合當作 pruning 的歷史起點。即使未來不採用 unstructured pruning，也應該知道這條路線的基本假設和限制。

---

### 3.6 Learning Efficient Convolutional Networks through Network Slimming

**定位：經典 structured / channel pruning paper。**

這篇用 BatchNorm 的 scaling factor 來判斷 channel 重要性，透過 sparsity regularization 讓不重要的 channel 變得容易被剪掉。

**要抓的重點：**

- channel pruning 為什麼比單純剪 weight 更容易產生實際 speedup。
- BN scaling factor 如何代表 channel 重要性。
- 訓練時加入 sparsity constraint，剪枝後再 fine-tune。
- structured pruning 對 CNN 類模型特別重要。

**讀完要能回答：**

- channel pruning 和 weight pruning 的部署差異是什麼？
- BN-based channel pruning 的假設是什麼？
- 如果 traffic model 是 1D-CNN，這種方法是否可以改用？

**對後續研究的用途：**

如果未來要做 packet-level 或 byte-level 1D-CNN 的輕量化，這篇很值得參考。因為 1D-CNN 也有 channel / filter，可以思考是否能把 channel pruning 接到 traffic model 上。

---

## 4. 建議的筆記模板

每讀一篇 paper，可以用下面格式整理。

```markdown
## Paper Title

### 基本資料

- Year:
- Venue:
- Task:
- Dataset:
- Model:
- Code:

### 研究問題

這篇想解決什麼問題？

### 方法

它怎麼做 pruning？

- 剪什麼：
- 何時剪：
- 重要性指標：
- 是否 fine-tune：
- 是否搭配 quantization / distillation：

### 實驗設計

- Baseline:
- Metrics:
- Accuracy / F1:
- Model size:
- Parameters:
- FLOPs / MACs:
- Latency:
- Throughput:
- Hardware / environment:

### 我覺得可以借用的地方

- TBD

### 我覺得有疑問的地方

- TBD

### 和未來題目的關係

- TBD
```

---

## 5. 之後可以發展的題目方向

### 方向 A：Pruning for lightweight traffic classification

核心問題：

> 如何在維持 traffic classification 效能的情況下，降低模型大小、計算量與延遲？

可能做法：

- 以 1D-CNN / CNN-based traffic classifier 當 baseline。
- 做 channel pruning 或 layer-wise pruning。
- 和 quantization 比較或結合。
- 評估 accuracy-efficiency trade-off。

適合的重點：

- 模型壓縮。
- Edge deployment。
- Packet-level classification。
- Encrypted traffic classification。

---

### 方向 B：Lightweight malicious traffic detection

核心問題：

> 惡意流量偵測模型能不能在資源受限或即時環境下維持偵測能力？

可能做法：

- 先建立 malicious / benign classifier。
- 再對模型做 structured pruning。
- 比較剪枝前後對 rare class、unknown attack、encrypted attack 的影響。
- 不只看 overall accuracy，也看 recall、false negative rate、AUC、per-class F1。

適合的重點：

- Security relevance 比單純壓模型更強。
- 可以討論 pruning 是否會傷害對少數類別或未知攻擊的偵測能力。
- 可以把效率指標和安全指標一起分析。

---

### 方向 C：Data pruning for traffic datasets

核心問題：

> 是否能移除冗餘或低品質 traffic samples，讓模型用更少資料訓練但維持效果？

可能做法：

- 對 flow / packet samples 做 ranking。
- 移除高度重複、低資訊量、疑似 mislabeled 的資料。
- 比較 full dataset vs pruned dataset 的訓練時間和效能。
- 檢查 pruning 是否造成 class imbalance 或攻擊類別被過度移除。

適合的重點：

- Dataset quality。
- Training efficiency。
- Label noise。
- Imbalanced traffic dataset。

注意：

這條路線和 model pruning 不一樣。它主要降低訓練資料量，不一定會讓 inference model 變小。

---

## 6. 第一個月閱讀計畫

### Week 1：建立 pruning 概念

- [ ] 讀 survey 的 abstract、introduction、taxonomy。
- [ ] 整理 structured / unstructured 的差異。
- [ ] 整理 pruning timing：before / during / after training。
- [ ] 整理 pruning 評估指標。

### Week 2：看 traffic pruning

- [ ] 讀 Bufferless joint pruning-quantization。
- [ ] 整理它的 baseline model。
- [ ] 整理它的 pruning / quantization search space。
- [ ] 整理它如何報告 latency、storage、complexity。

### Week 3：看 security 任務

- [ ] 讀 HyperVision。
- [ ] 整理 unknown encrypted malicious traffic 的問題定義。
- [ ] 整理它的 graph representation。
- [ ] 整理它的 throughput / latency 評估。

### Week 4：補一篇 traffic pruning 和一篇經典 pruning

- [ ] 讀 Pruned-F1DCN。
- [ ] 讀 Network Slimming 或 Han et al.。
- [ ] 整理哪些 pruning 方法比較適合 1D-CNN traffic model。
- [ ] 寫出一頁題目想法。

---

## 7. 讀完後應該形成的研究定位

比較好的定位不是：

> 我想研究 pruning 演算法。

而是：

> 我想研究如何讓 traffic / malicious traffic detection 模型在即時或資源受限環境下仍然可用，而 pruning 是其中一種降低模型成本的方法。

這樣定位會比較穩，因為研究主軸是 security / traffic detection，pruning 是幫助部署的技術手段。

---

## 8. 目前最優先要讀的兩篇

如果時間很少，先讀：

1. **A Survey on Deep Neural Network Pruning**
   - 用來建立 pruning 地圖。

2. **Bufferless network traffic classification using reinforcement learning-based joint pruning-quantization**
   - 用來看 pruning 如何真的接到 traffic classification。

讀完這兩篇後，再回頭決定要補 traffic pruning、malicious traffic detection，還是 dataset pruning。
