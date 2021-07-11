# 1. Competition Summary
![image](https://user-images.githubusercontent.com/75927764/125186300-8e7b8a00-e264-11eb-8939-76a5ed29a7be.png)
- 이미지에서 쓰레기나 물건 등을 픽셀 단위로 분류하는 task
- train[image:2617, annotation:21116],   test[image: 837개]
- background, general trash, paper, plastic bag 등 총 12개 class
- class간 불균형이 매우 심하다. (개체 수 기준 7000개~50개)
- 학습용 데이터는 매우 깔끔하게 전처리되어 제공되었으며, 512x512 사이즈
</br></br>
# 2. Work Flow
### Day01                           (0.5754 : mIoU)

- baseline 코드 작성
- backbone 실험
- 각 아키텍쳐별로 러닝타임 계산

### Day02                           (0.5795)

- train데이터에 overfitting 실험 (loss값 기준으로 확인)
- input size를 256으로 줄여 실험 시간이 4배 이상 빨라짐
- loss함수 및 scheduler 탐색

### Day03                           (0.6020)

- validation metric ver.2 추가. (mIoU 계산 방식 변경)
- wandb의 AutoML로 배치사이즈 및 시드 탐색
- backbone(encoder) 탐색

### Day04                           (0.6275)

- classmix 구현 및 실험
- augmentation 탐색
- backbone(decoder) 탐색
- validation metric ver.3 추가 (mIoU 계산 방식 변경)

### Day05                                    (0.6521)

- classmix : 불균형한 클래스에 weight를 크게 주어 실험
- Flip을 이용한 TTA
- scheduler 추가 탐색

### Day06                                    (0.6773)

- pseudo labeling
- Scale TTA
- model Ensemble

### Day07                                    (0.6798)

- KFold
- TTA 추가 (rotate90, counterRotate90)

### Day08                                    (0.6913)

- 2차 pseudo labeling (로직 변경)
- EfficientDet 구현 (segmentation head 연결)

### Day09~10                              (0.6991)

- Ensemble list에 EfficientNet+FPN 모델 추가
- team Ensemble (4가지 모델 사용)
</br></br></br>

# 3. Experiments
### iou loss + CrossEntropy

- giou라고 하는 detection용 loss함수에서 1-iou값을 사용하겠다는 포인트를 착안하였습니다.
- iou loss 자체만으로는 성능 하락이 있었지만, CrossEntropy와 적절히 섞어 사용했더니 조금의 성능 향상을 보였습니다.
- 여러 비율을 실험해본 결과, (1-iou)*0.4+CE*0.6 비율이 가장 좋았습니다.
- [논문 출처](https://arxiv.org/pdf/1902.09630.pdf), [블로그 설명글](https://gaussian37.github.io/vision-detection-giou/)

### TTA (Test Time Augmentation)

- 학습 때 rotate와 flip을 사용했으니 inference 때도 분명 효과가 있을 것이라 판단하여 flip을 적용한 결과와 soft ensemble 하였습니다.
- 학습 때는 RadomRotate를 사용했는데, test time에서는 inference 후에 다시 회전하기 전으로 돌려줘야 하기 때문에 직접 회전시켜 주었습니다. 시계 방향 90도, 반시계 방향 90도를 추가하였습니다.
- 총 3가지의 TTA를 추가하였는데, 꽤 괜찮은 성능 향상이 있었습니다.

### Pseudo labeling

- P stage 1,2 때도 사용해봤었지만 많은 성능 향상을 보진 못했습니다. 이번 대회에서는 pseudo labeling 덕분에 많은 성능 향상을 보았습니다.
- 픽셀 단위로 학습에 사용할지 말지를 정해야하니 고민이 컸습니다. 처음에는 픽셀 단위로 max prob 값이 0.9 이상인 값만 사용하고 나머지는 backgroung(0)로 채웠습니다. 깨끗하지 못한 데이터가 너무 많이 생겼고, 손으로 일일이 걸러줘야 했습니다.
- 두번째 방법에서 픽셀 단위보다는 이미지 단위로 생각해 봤습니다. 한 이미지에서 백그라운드가 아닌 개체로 분류된 픽셀들의 개수를 구하고, 그 중에서 max prob 값이 0.8 이하인 픽셀들의 수를 구했습니다. 0.8 이하인 픽셀들의 수가 10%보다 많으면 해당 이미지는 제외했습니다.
(팀원 분들이 도움을 주신 아이디어입니다.)

### Ensemble

- 어떤 대회에서보다 ensemble 방법에 가장 집중했던 것 같습니다.
- 이미지 scale(resolution)별로, 여러가지 augmentation별로, 같은 모델이라도 epoch별로, 여러가지 아키텍쳐별로 앙상블을 진행하였습니다.
- scale앙상블 ≥ KFold 앙상블 ≥여러가지 아키텍쳐 앙상블 > augmentation 앙상블(TTA) > epoch 앙상블 순으로 효과적이었습니다.
- 이번 실험을 통해 receptive field의 중요성을 다시금 깨닫게 되었습니다. 넓고 다양할수록 성능이 올라갔습니다.
- 데이터에 대한 중요성도 다시 한번 느꼈습니다. 단순히 학습 데이터를 일부만 바꿔가며 만든 모델들을 앙상블(5-Fold Ensemble) 했더니 많은 성능 향상이 있었습니다.

### validation metric 추가

- 성능 향상에 직접적인 도움은 아니었지만, 개선 덕분에 validation 스코어의 신뢰도가 높아졌습니다. 제출해보지 않고도 진행한 실험의 결과를 기록할 수 있었습니다.
- 기본으로 제공된 miou metric 함수는 LB스코어와 갭이 0.2가까이로 너무 컸습니다.
- 기존 metric 함수는 배치 단위로 miou를 계산하는 문제점이 있었고, 모든 이미지에 대해 한번에 miou를 계산하는 방식으로 수정하였습니다. (ver.2)
- ver.2가 LB스코어와 어느정도 근접하게 되었지만, 그래도 갭은 존재했기에 아예 대회 스코어 계산 방식과 동일하게 metric을 수정해 주었습니다. (ver.3)
각각 이미지에 대해 miou 계산 후 누적하여 전체를 평균하는 방식입니다.

### ClassMix Augmentation

![image](https://user-images.githubusercontent.com/75927764/125187384-59723600-e26a-11eb-9dea-055a131de28a.png)

- augmentation 방법 중 하나로, 다른 이미지에 있는 개체를 픽셀 단위로 가져와 붙여넣는 방식입니다. torch.where함수를 이용해 간단하게 구현에 성공했지만 성능은 하락했습니다.
- 부족한 클래스의 개체 위주로 붙여넣는 방식으로도 사용해봤지만, 성능 향상은 없어서 매우 아쉬웠습니다.
- [논문 출처](https://arxiv.org/pdf/2007.07936.pdf)

### EfficientDet Customizing

![image](https://user-images.githubusercontent.com/75927764/125187377-5119fb00-e26a-11eb-8066-04579138fb6a.png)

- EfficientNet + BiFPN 형태로, 구글링을 통해 detection head가 연결된 아키텍쳐 구현 코드를 발견했습니다.
- detection head를 제거하고 segmentation head를 연결하는데 성공하였지만, 성능이 좋지 않았습니다.
