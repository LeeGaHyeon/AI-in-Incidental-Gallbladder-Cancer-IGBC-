# 담낭암 vs. 급성 담낭염 CT 진단 AI 프로젝트

> **의료 영상(CT) 기반 담낭암(Gallbladder Cancer, GBC)과 급성 담낭염(Acute Cholecystitis, AC) 감별 진단을 위한 딥러닝 파이프라인**   

---

## 1. 프로젝트 개요
담낭벽 비후(thickening)는 염증(양성)과 암(악성) 모두에서 관찰되어 CT 상 감별이 어렵습니다. 본 프로젝트는 아래와 같은 **end-to-end AI 파이프라인**을 제안합니다.

1. **YOLOv5**: Axial CT 슬라이스에서 담낭 ROI 자동 탐지  
2. **ROI Crop (conf ≥ 0.8)**: 탐지 결과를 활용해 담낭 영역만 잘라 분류 입력으로 사용  
3. **Inception-ResNet-v2 기반 CNN**: 슬라이스 단위 GBC vs AC 이진 분류  
4. **Weighted Soft Voting**: 슬라이스 예측을 환자 단위로 집계

**최종 목표:** CT 영상에서 **GBC** 와 **AC** 를 자동 구별

---

## 2. 임상·해부학적 배경
- **IGBC(Incidental Gallbladder Cancer)**: 양성 담낭절제술 후 조직검사에서 우연히 발견된 담낭암  
- **발생 빈도**: 전체 양성 담낭절제술 환자 중 **0.3–2%**  
- **담낭 해부학**: 길이 5–10 cm, 폭 3–5 cm, 용적 ~50 mL, 정상 담낭벽 두께 2–3 mm  
  - 기저부(fundus) / 체부(body) / 경부(neck)  
  - 담낭관(cystic duct) ↔ 총담관(common bile duct)

- **담낭암 영상학적 유형**  
  1. 국소/미만성 담낭벽 비후형 (focal/diffuse wall thickening)  
  2. 담낭 내 돌출형 (polypoid mass)  
  3. 담낭 대체형 (mass replacing gallbladder)

본 연구는 특히 **1번 유형(벽 비후형)** 감별의 어려움을 AI로 보완하는 데 초점을 둡니다.

---

## 3. 관련 연구
- *The Value of Deep Learning in Gallbladder Lesion Characterization* (doi: 10.3390/diagnostics13040704)  
- *Radiomics-based machine learning and deep learning to predict serosal involvement in gallbladder cancer* (doi: 10.1007/s00261-023-04029-2)  
- *Analysis of enhancement pattern of flat gallbladder wall thickening on MDCT to differentiate gallbladder cancer from cholecystitis* (doi: 10.2214/AJR.07.3331)

---

## 4. 데이터 & 전처리 (Data & Preprocessing)

### 4.1 데이터 개요
| 구분 | 환자 수 | 
|-----|--------:| 
| GBC (담낭암) | 107명 |  
| AC  (담낭염) | 107명 |  
| 총합 | 214명 |  

### 4.2 라벨링 (Detection용)
- 각 클래스별 107명 중 **30명씩** 수동 라벨링(담낭 Bounding Box) → YOLOv5 학습  
- 라벨 포맷: YOLO (class x_center y_center width height) normalized coordinates

### 4.3 전처리 파이프라인
1. **DICOM → PNG 변환**
2. **리샘플링/정규화**:  
   - 해상도 통일 (예: 512×512)  
   - 픽셀값 정규화(0~1 또는 z-score)  
3. **탐지 모델 적용**: YOLOv5 추론 → conf ≥ 0.8 인 박스만 채택  
4. **ROI Crop**: 박스 영역을 margin(예: 10~20 px) 포함하여 crop, 필요 시 패딩 → `data/crops/` 저장  
5. **데이터 증강 (Classification)**:  
   - Random horizontal/vertical flip  
   - Random rotation, scale  
   - Intensity jitter (brightness/contrast)  

### 4.4 데이터 분할 (Split) 
- **기본 원칙: 환자 단위 분할(Patient-wise)**  
  - 한 환자의 모든 슬라이스는 **반드시 하나의 세트(train/val 중 하나)에만** 포함됩니다.  
  - 슬라이스 단위로 무작위 분할하면 데이터 누수(leakage)가 발생하므로 금지합니다.

---

## 5. 모델 학습(Training) 개요
- **Backbone**: Inception-ResNet-v2 (ImageNet 사전학습 가중치)  
- **출력층**: 2-class 분류용 FC 레이어로 교체  
- **Loss / Optimizer / Scheduler**  
  - Loss: Cross Entropy  
  - Optimizer: Adam
  - LR Scheduler: ReduceLROnPlateau (val loss 개선 없을 때 lr 감소)  
- **Early Stopping**: Validation loss 기준, patience 기반 중단  
---

## 6. 평가(Validation & Test)
- **지표(환자 단위)**: Accuracy, AUC 
- **집계 전략**: 슬라이스 확률의 가중 평균(Weighted Soft Voting)  
- **성능**  
Validation Acc: 0.8140 | Validation AUC: 0.8304

 
