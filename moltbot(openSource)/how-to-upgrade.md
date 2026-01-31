1. [[Role Refs]] 인식 모델의 다각화 

- https://huggingface.co/PaddlePaddle/PaddleOCR-VL-1.5 등의 초 경량 모델 + 내용을 경량화(컴포넌트 인식으로 제한)하여 성능을 끌어올리고 파라미터 수를 줄이는 방식으로 진행. 

1-2. 문제점 - 문맥 이해 부족 (Semantic Understanding)

문제: PaddleOCR은 글자는 잘 읽지만, 그게 "버튼"인지 "단순 텍스트"인지 구분을 못 할 수 있습니다. 예를 들어 "로그인"이라는 글자가 버튼인지 제목인지, OCR만으로는 알기 어렵습니다.
대안: 단순 OCR뿐만 아니라 UI 요소 탐지(Detection) 모델 (예: ScreenRecognition이나 Microsoft의 OmniParser)을 같이 사용해야 합니다.

1-3. Python 모델 통합 전략 (Integration Strategy)

Node.js 환경인 MoltBot에 Python 기반 OCR을 통합하는 방법은 크게 3가지가 있습니다. "설치 난이도"를 기준으로 나열합니다.

#### A. ONNX Runtime (Node.js) - **추천 (가장 깔끔함)**
*   **방식**: PaddleOCR 모델을 ONNX 포맷으로 변환한 뒤, Node.js의 `onnxruntime-node` 라이브러리로 직접 실행합니다.
*   **장점**: **Python 설치 불필요**. Node.js 모듈(`npm install`)만으로 설치가 끝납니다. 배포가 가장 쉽습니다.
*   **단점**: 모델 변환(Export) 과정이 필요하고, Post-processing(박스 좌표 계산) 로직을 JS로 포팅해야 합니다.
*   **적합성**: 배포 용이성이 중요할 때 최적.

- [[computer_control_analysis]]를 참조해서 설정을 연결한다.