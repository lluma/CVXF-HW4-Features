# CVFX HW4 Report  - Team 16

### 1. Take a sequence of moving-forward images in NTHU campus
這裡我們於人社院走廊取景，原因是景色具備重複性，之後呈現Infinite zooming effect時會較順利

![](https://i.imgur.com/XwE6wvr.gif)

### 2. Show feature extraction and matching results between two images
Ref: https://www.learnopencv.com/image-alignment-feature-based-using-opencv-c-python/

這裡我們使用了SURF extraction method，Threshold設成1800，去計算兩張連續image間的Feature point，再使用BFMatcher將其連接，並只取分數前15%的點呈現

![](https://i.imgur.com/nndwvIJ.gif)

### 3. Perform image alignment and generate infinite zooming effect
在做合成前，有兩件事要先進行，一是可以發現即使盡量保持平移拍攝，視角上仍有誤差，這時就要靠Alignment來處理，另一個則是Zooming時，由於要製造無限放大的感覺，中央最理想的是在遠端黑色的門，因此我們會事先對原始圖片做裁切
Alignment的方法一樣是參考2.之連結，透過兩連續圖的Keypoints算出Homography matrix，再用warpPerspective投影對齊後的結果

![](https://i.imgur.com/jFCjlh8.gif)

可以看到Alignment的圖片大部分會歪斜掉，原因猜測是出自取景的間隔過短，加上前幾張拍攝視角偏移的頗大，另外則是周圍會有黑邊，由於僅要製造無限放大的效果，我們最後只用"最後兩張"做合成GIF的來源，並取圖片中央 $\frac{3}{4}$ 處

而在生成無限放大效果的GIF Frame時，底部先舖上第一張圖，之後再慢慢疊上Alignment、原圖、Alignment...於上頭，由於重複的主體是兩側的柱子，縮放比例則抓剛好能重合前一張圖的柱子位置時的比例

下圖為Start Frame
![](https://i.imgur.com/3EWx52S.jpg)

之後的放大則是依照一格放大比率，依序放大為要覆蓋的圖後，再照上述步驟疊上，然後透過人工計算，在適當時機(約為疊上的第一張圖取代為最外圍者時)疊上新的圖片，維持無限放大的效果

結果影片(Youtube link，未經任何後製處理下)：

[![IMAGE ALT TEXT HERE](https://img.youtube.com/vi/obvUCl7Ey0U/0.jpg)](https://www.youtube.com/watch?v=obvUCl7Ey0U)

缺點: 連結的邊緣處仍然明顯，且周圍景色(如草叢)在連結上也有不協調之處

### 4. Comparison of different feature extrators - SURF vs ORB
ORB: FAST + BRIEF 的結合改良版，ORB在算法速度上有很大的提升，大約是SIFT的100倍，是SURF的10倍。該方法首先利用 FAST 來提取 features 並使用高斯金字塔來保持 multi-scale invariant 的 feature 特性，接著利用改良而具有 rotation invariant 的 steer BRIEF 和能保留 feature 匹配獨特性的 rBRIEF 來描述我們的 features 的方向和特性。

SURF: SIFT的改良版，解決SIFT的速度實在太慢的問題(一組配對需約幾十分鐘處理，且CPU處於100%，而適當Threshold下的SURF，一張圖不過十幾秒)，他先使用 Hessian matrix 再做 Non-maximum suppression， 並改變了 multi-scale pyramid space 的結構與使用多個 box filters 取代原來的 Guassian filter，以致計算 features 的 cost 大幅下降。而在 feature 方向決定上是在圓形區域內以某方向的扇形範圍內去預測主方向，也減少了原先在方形區域中的龐大計算量。

兩者上都比原 SIFT 作法快上許多而以 ORB 為尤，但在於主方向的預測上，ORB 做得比 SURF 好，因為 SURF 計算主方向時太過於依賴局部區域像素的梯度方向，導致預測不准確，而後面的 feature vector 提取以及匹配都嚴重依賴於主方向，即使偏差角度不大也可能造成在後面做特徵匹配時會將誤差放大，從而匹配不成功，結果變差。下方我們也做了一些小實驗來比較兩者在 rotation invariant 上的優異程度。

| | Time(s) | Keypoints | Keypoints<br>(180度翻轉後) | Keypoints<br>(45度翻轉後) | Matches rate<br>(180) |Matches rate<br>(45) |
|---|---|---|---|---|---|---|
| SURF | 2.57 | 16765 | 16768 | 13209 | 96% | 58% |
| ORB | 0.68 | 500 | 500 | 500 | 99% | 38% |

作業要求實作一個穩定、盡量不受偏移、縮放、旋轉來源圖片而改變的方法，因此我們這裡對圖片進行不同程度的旋轉，然後去比兩者得到的Keypoint相似程度，藉此比較各自的穩定程度
Notice: 表格的Keypoint僅代表數量，Match rate計算是取交集率

* 計算時間:
    
    由於ORB這裡對特徵的抓取採預設值(一律取前500)，在時間上比起我們實驗下能獲得最好結果的SURF Threshold，速度明顯快上很多，但SURF仍比彷彿跑不完的SIFT快上許多

* Keypoint match rate - 目標圖上下翻轉(180度):
    
    上下翻轉目標圖做Matching，兩者方法都有接近100%的結果，而ORB好上一些，我們推測在未改變視角的情況下，單純景物上下置換兩種演算法都能維持辨識效果

* Keypoint match rate - 目標圖旋轉45度，並配合大小縮放75%:
    
    這裡就有明顯的差異了，兩者表現都不如純粹上下翻轉，然而SURF尚能有6成附近的交集率，ORB卻跌到剩4成，這項操作顯示若目標圖與參考圖的視角差異過大，在這種多物件的風景圖上，OpenCV所能提供的幾種演算法都會大幅影響結果圖(即使ORB理論上能支援尺度不變及偵測特徵方向)，與在拍攝和製作Infinite zooming時需要注意的細節吻合，照的角度比起算法更重要；從方法來看，若不多做過多的前置處理，SURF能擁有較高的穩定度，同時速度也在可接受範圍內


### 5. Do additional image processing by Photoshop
[TODO]
    我們的影片是將所有圖片進行輸出後再合成的，所以我們有可以再對這些圖片進行微調，我們利用了Photoshop將這些圖片的背景進行模糊化，我們是利用很簡單的動態模糊來調整，這樣可以達到模擬相機往前的效果，也可以將一些銜接的部分變得比較不明顯。
![](ezgif-4-2a78730f9e44.gif)

＠新的GIF太大，要從github上來上傳。

