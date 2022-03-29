# Python 개발 환경 구축하기

## 가상환경 구축하기

### Miniconda 설치

[다운로드 링크](https://docs.conda.io/en/latest/miniconda.html)에서 OS에 맞는 Miniconda 설치

환경변수 설정을 통해서 CMD로 접속 할 수 있지만, [Anaconda Prompt](https://katiekodes.com/setup-python-windows-miniconda/)를 통해서 접속 할 수도 있고, 공홈에서도 이 방법을 권장하고 있다.

### Miniconda를 이용한 가상환경 실행

1. 가상 환경 생성
   1. `conda create -n [가상 환경 이름] python=[python 버전]`
2. 가상 환경 확인
   1. `conda info --envs`
3. 가상 환경 삭제
   1. `conda remove --name [프로젝트 이름] --all`
4. 가상 환경 활성화
   1. `conda activate [프로젝트 이름]`
5. 가상 환경 비활성화
   1. `conda deactivate`

## 가상환경에 Jupyter 설치

`pip install jupyter`

## VSCODE에 extension 설치

[Microsoft 제공 Python Extension](https://marketplace.visualstudio.com/items?itemName=ms-python.python) 설치

## VSCODE에서 Jupyter로 Anaconda 가상환경 연결

[VSCODE 공식 홈페이지](https://code.visualstudio.com/docs/datascience/jupyter-notebooks) 참조하여 Jpuyter로 Miniconda로 생성한 환경에 접속