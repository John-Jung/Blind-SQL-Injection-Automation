# 데이터베이스 테이블 및 컬럼 추출 스크립트

이 스크립트는 SQL 인젝션 공격을 통해 데이터베이스로부터 테이블 이름, 컬럼 이름 및 데이터를 추출하고, 결과를 Excel 파일로 저장하는 데 사용됩니다. 

## 사전 준비 사항

- Python 3.7 이상
- 다음 Python 라이브러리:
  - `requests`
  - `openpyxl`

필요한 패키지는 다음 명령어를 사용하여 설치할 수 있습니다:

```bash
pip install requests openpyxl
```


## 스크립트 개요
1. <b> 데이터베이스 정보에 대한 이진 탐색</b>: SQL 인젝션을 사용하여 테이블 및 컬럼 수를 효율적으로 결정하고, 테이블 및 컬럼 이름을 검색합니다.
2. <b>문자 인코딩 처리</b>: ASCII 및 한글 인코딩을 처리하여 비 ASCII 문자를 포함한 데이터베이스 내용을 정확하게 해석합니다.
3. <b>Excel 파일 생성</b>: 추출된 데이터를 openpyxl을 사용하여 Excel 파일로 저장합니다. 각 테이블은 tables 디렉토리에 별도의 Excel 파일로 저장됩니다.
4. <b>타임아웃에 대한 내구성</b>: 네트워크가 불안정한 경우에도 스크립트가 견고하게 작동하도록 타임아웃 발생 시 요청을 재시도하는 로직이 포함되어 있습니다.

## 설정 및 실행

### 설정
1. <b>쿠키 업데이트</b>: 스크립트는 인증을 위해 세션 쿠키를 사용합니다. cookies 딕셔너리에 유효한 세션 쿠키를 다음과 같이 추가

```python
cookies = {
    "JSESSIONID": "YOUR_VALID_SESSION_ID"
}
```
2. <b>대상 URL</b>: 스크립트는 SQL 인젝션이 시도될 특정 URL을 대상으로 합니다. url 변수가 올바른 엔드포인트를 가리키는지 확인

```python
url = 'URL and {}'
```
## 스크립트 실행
```bash
python answer.py
```
스크립트가 데이터를 추출하기 시작하고 각 테이블의 데이터를 tables 디렉토리 아래의 개별 Excel 파일에 저장합니다. 각 Excel 파일은 테이블 이름으로 명명된 시트를 포함하며 데이터베이스에서 추출된 행과 열을 포함합니다.

## 출력
- <b>Excel 파일</b>: 각 테이블은 tables 디렉토리 내의 개별 Excel 파일로 저장됩니다. 파일 이름은 테이블 이름과 동일합니다.
- <b>콘솔 로그</b>: 스크립트는 진행 상황을 콘솔에 기록하며, 테이블 수, 컬럼 이름 및 실행 중 발생한 타임아웃이나 권한 오류 등의 문제를 표시합니다.

## 오류 처리
- <b>타임아웃</b>: 타임아웃이 발생하면 스크립트는 자동으로 요청을 재시도합니다.
- <b>권한 오류</b>: 파일 저장 시 권한 오류가 발생하면 스크립트가 오류 메시지를 기록합니다.

## 한글 인코딩

특이사항으로는 글자를 추출할때 영어는 기본적으로 0-127까지 아스키코드를 사용하여 추출하면 되지만 한글 식별이 상당히 까다로웠다. 한글 입력값들이 기본적으로 euc-kr 혹은 utf-8로 인코딩 되어 있다. 그럼 그들의 차이가 뭘까?

EUC-KR: 16비트
UTF-8: 24비트

해당 DB는 UTF-8로 인코딩 되어 있었다.


### ASCII 범위 외 문자 처리

코드는 기본적으로 ASCII 문자(0~127 범위)를 처리하는 방식으로 설계되어있다. 그러나 한글은 이 범위를 벗어나는 문자로, UTF-8 인코딩에서 3바이트로 표현됩니다. ASCII 범위 외의 문자를 다루기 위해 코드에서는 한글 등 비ASCII 문자의 인코딩/디코딩 처리를 추가로 수행


### 한글 인코딩 처리 함수: insertPecsent

>
```python
def insertPecsent(s):
    return '%' + s[:2] + '%' + s[2:4] + '%' + s[4:6]
```

이 함수는 한글을 URL 인코딩 형식(%E3%84%A4)으로 변환하기 위해 사용된다. UTF-8로 인코딩된 한글은 16진수로 표현되며, 이 함수는 각 바이트를 %XX 형태로 변환한다.

### 한글 처리 로직: get_char_value 함수

>
```python
def get_char_value(query):
    ascii_val = BinarySearch(query, max_val=15572643)
    if ascii_val > 127:  # 한글 범위에서 탐색 필요
        hex_ascii_character = hex(ascii_val).replace("0x", "")
        encoded_hangeul = insertPecsent(str(hex_ascii_character))
        return parse.unquote(encoded_hangeul)
    return chr(ascii_val)
```

여기서 가장 중요한것은 **max_val=15572643** 이다. 이숫자는 UTF-8을 ord로 표현하여 한글이 표현되는 범위이다. 

자세한 설명은 맨 하단 참고 링크를 참조

이 함수는 SQL 인젝션을 통해 탐색된 ASCII 값을 바탕으로 문자를 구하는 함수

1. BinarySearch 함수를 통해 ASCII 값을 탐색한다.

2. 탐색된 값이 127보다 크면, 이 값은 한글이거나 다른 비ASCII 문자일 가능성이 큽니다. 이 경우 해당 값을 16진수로 변환하고 insertPecsent 함수를 통해 URL 인코딩 형식으로 변환한다.

3. 변환된 값은 parse.unquote 함수를 통해 실제 한글로 디코딩된다.

### URL 인코딩/디코딩

**'**parse.unquote**'** 함수는 URL 인코딩된 문자열을 원래 문자로 변환하는 데 사용됩니다. 예를 들어, '%E3%85%85'와 같은 인코딩된 문자열은 디코딩되어 한글 문자가 된다.

## 관련 게시물
https://velog.io/@wearetheone/%EB%AA%A8%EC%9D%98%ED%95%B4%ED%82%B9-%EC%88%98%EC%97%85-%EB%AA%A8%EB%93%88%ED%94%84%EB%A1%9C%EC%A0%9D%ED%8A%B8-2-%ED%95%9C%EA%B8%80-%EC%9D%B8%EC%BD%94%EB%94%A9

https://linarena.github.io/web_0x04
