
# <font color = 'brown'>Politics Project (공정배 민유진 유영재 장은경)<font>

## <font color = 'blue'>주제 : 네이버 상위권 뉴스와 정부 지지율 관계성 파악<font>
> - 정치 / 경제 / 사회 3가지 카테고리를 이용
- 기사의 헤드라인의 키워드 빈도수를 이용
- 기사 랭크 top5 와 top30 나눠서 분석
- 문재인 정부의 1년 간의 지지율 데이터 추출 및 관계도 파악 (2017년 11월 1일 ~ 2018년 11월 15일)

## <font color = 'red'>기사 크롤링 함수<font>


```python
from bs4 import BeautifulSoup 
from urllib.request import urlopen

import numpy as np
import pandas as pd


# 크롤링 하는 함수
def crawling_news(startDay, endDay, numOfNews):
    Day = []
    polTitle = []
    ecoTitle = []
    socTitle = []
    totalTitle = []
    
    # 시작과 끝 날짜 입력
    startDay, endDay = str(startDay), str(endDay)
    
    # 날짜목록 만들기
    dayList = pd.date_range(start = startDay, end = endDay)
    dayList = dayList.strftime('%Y%m%d').tolist()
    
    # 카테고리
    categorys = ['100', '101', '102']  # 100 : 정치 / 101 : 경제 / 102 : 사회
    
    # 크롤링
    url = 'https://news.naver.com/main/ranking/popularDay.nhn?rankingType=popular_day&sectionId={}&date='
    
    for category in categorys:
        urlBasic = url.format(category)
    
        for day in dayList:
            # html 수집
            urlDay = urlBasic + day
            html = urlopen(urlDay)
            soup = BeautifulSoup(html, "lxml")
        
            # 기사 헤드라인 수집
            newsTitle = soup.find_all('div', 'ranking_headline')
        
            for n in range(numOfNews):
                # 정치 기사만 수집
                if category == '100':
                    Day.append(day)
                    polTitle.append(newsTitle[n].get_text().strip())
                
                # 경제 기사만 수집
                if category == '101':
                    ecoTitle.append(newsTitle[n].get_text().strip())
                
                # 사회 기사만 수집
                if category == '102':
                    socTitle.append(newsTitle[n].get_text().strip())
    
    # 정치/경제/사회 헤드라인 합치기 과정
    for n in range(len(polTitle)):
        tmpTitle =  polTitle[n] + ' ' + ecoTitle[n] + ' ' + socTitle[n]
        totalTitle.append(tmpTitle)
    
    # 데이터 프레임 생성
    data = {'날짜' : Day, '정치' : polTitle, '경제' : ecoTitle, '사회' : socTitle, '종합' : totalTitle}
    news = pd.DataFrame(data)
    
    return news
```

## <font color = 'red'>대통령 지지율 크롤링 함수<font>
> - 취임부터 2018년 11월까지 데이터 추출
- 출처: 위키피디아, 한국갤럽
- https://ko.wikipedia.org/wiki/%EB%8C%80%ED%95%9C%EB%AF%BC%EA%B5%AD%EC%9D%98_%EB%8C%80%ED%86%B5%EB%A0%B9_%EC%A7%80%EC%A7%80%EC%9C%A8


```python
import re

# 대통령 지지율 크롤링 함수
def support():
    # 위키피디아 html 가져오기
    url  = 'https://ko.wikipedia.org/wiki/%EB%8C%80%ED%95%9C%EB%AF%BC%EA%B5%AD%EC%9D%98_%EB%8C%80%ED%86%B5%EB%A0%B9_%EC%A7%80%EC%A7%80%EC%9C%A8#cite_note-%EC%B6%94%EC%84%9D-8'
    html = urlopen(url)
    soup = BeautifulSoup(html, "lxml")
    div_title = soup.find_all('tbody')[6].find_all('td')
    
    # 이중 리스트 구조를 이용해 각 주의 지지율 리스트 만들기
    weekSupportList = []
    weekSupport = []
    for n in range(len(div_title)):
        element = div_title[n].get_text().strip()
        if '미조사' in element:
            weekSupport.append('NA')
            continue
        if '월' in element:
            weekSupport = []
            weekSupportList.append(weekSupport)
        weekSupport.append(element)
    
    # 주 정보에서 년도 월 추출
    YEAR=[]
    MONTH=[]
    for element in weekSupportList:
        pattern = re.compile('(\d{4}\w)')
        year = int(pattern.search(element[0]).group()[:-1])
        pattern = re.compile('(\s\d{1,2}\w)')
        month = str(int(pattern.search(element[0]).group()[:-1])).zfill(2)
    
        YEAR.append(year)
        MONTH.append(month)
    
    # 년도와 월 합치기
    DATE=[]
    for i in range(len(MONTH)):
        date= '{year}{month}'.format(year=YEAR[i], month=MONTH[i])
        DATE.append(int(date))
        
    uniqueDate = set(DATE)
    uniqueDate = list(uniqueDate)
    
    # 지지율 추출
    PERCENT=[]
    for i in range(len(weekSupportList)):
        percent = weekSupportList[i][1]
        PERCENT.append(percent)
    
    print(len(DATE))
    print(len(PERCENT))
    
    # 초기 데이터 프레임 생성
    data = pd.DataFrame({"날짜": DATE,"지지율": PERCENT})
    data = data.set_index("날짜")
    
    # 월 지지율을 마지막 주의 지지율로 대체
    date = []
    support = []
    for i in range(len(uniqueDate)):
        if data.loc[uniqueDate[i]].iloc[-1][0] == 'NA':
            support.append(data.loc[uniqueDate[i]].iloc[-2][0])
        else:
            support.append(data.loc[uniqueDate[i]].iloc[-1][0])
        date.append(uniqueDate[i])
        
    # 최종 데이터 프레임 생성
    result = pd.DataFrame({'날짜' : date, '지지율' : support})
    result.sort_values(by = '날짜', inplace=True)
    result = result.iloc[5:]  # 1년간의 데이터만 추출
    result.set_index("날짜", inplace=True)
    
    return result
```

## <font color = 'red'>키워드 추출하는 함수<font>
> - 빈도수가 높은 순으로 정렬
- 원하는 카테고리를 정해서 추출


```python
from konlpy.tag import Okt
from collections import Counter
import re

# 상위 키워드 뽑아내는 함수
def extract_keyword(yearMonth, numOfWord, category, data,filterWord=[]):
    
    noFilterNoun = []       # 정제되지 않은 단어를 담을 공간
    filterNoun = []         # 정제된 단어를 담을 공간
    wordList = []           # 정제하고 빈도까지 체크한 단어를 담을 공간
    wordFreq = []           # 정제한 단어의 빈도 수를 담을 공간
    
    # 년도와 월 필터
    day = re.compile('{}\d+'.format(str(yearMonth)))
    
    # 지정한 년, 월에 해당하는 단어만 추출하는 과정
    okt = Okt()
    for n in range(len(data['종합'])):
        if day.search(str(data['날짜'][n])):
            noFilterNoun.extend(okt.nouns(data['{}'.format(category)][n]))
    
    # 단어 정제 과정
    for word in noFilterNoun:
        if len(word) == 1: continue
        elif word in filterWord: continue
        filterNoun.append(word)
    
    # 빈도수가 높은 단어 추출
    cnt = Counter(filterNoun)
    for word, freq in cnt.most_common(numOfWord):
        wordList.append(word)
        wordFreq.append(freq)
    
    # 데이터 프레임 형성
    colName = [(yearMonth, '단어'), (yearMonth, '빈도')]
    
    df = pd.DataFrame({'단어' :wordList, '빈도':wordFreq})
    df.rename(index=str, columns={'단어' : colName[0], '빈도' : colName[1]}, inplace=True)
    df.columns = pd.MultiIndex.from_tuples(df.columns, names=['날짜', '종류'])
    
    return df
```

## <font color = 'red'>단어 필터<font>
> 버전을 2가지로 형성 (많은 키워드 수집을 위한 약한 필터 / 임팩트 있는 키워드 수집을 위한 강한 필터)
- 정치
- 경제
- 사회
- 종합 (정치 경제 사회 모두를 한꺼번에 고려한 것)

### 정치 필터


```python
# 약한 필터
polMeanlessWordsWeak = []
polRepetiveWordsWeak = []
polFilterWordsWeak = polMeanlessWordsWeak + polRepetiveWordsWeak

# 강한 필터
polMeanlessWordsStr = []
polRepetiveWordsStr = []
polFilterWordsStr = polMeanlessWordsStr + polRepetiveWordsStr
```

### 경제 필터


```python
# 약한 필터
ecoMeanlessWordsWeak = []
ecoRepetiveWordsWeak = []
ecoFilterWordsWeak = ecoMeanlessWordsWeak + ecoRepetiveWordsWeak

# 강한 필터
ecoMeanlessWordsStr = ['종합','단독','10','11','내년','한국','경제','주택자','이유','회장','개월','국민','13',
                    '종합2보']
ecoRepetiveWordsStr = ['파리','게뜨','군제','코스','패딩','가상','화폐','비트코','비트','상화','상화폐','거래'
                    '동연','군산','공장','금호','조현','대한','양호','바이오','항공','압수','수색',
                    '부세']
ecoFilterWordsStr = ecoMeanlessWordsStr + ecoRepetiveWordsStr
```

### 사회 필터


```python
# 약한 필터
socMeanlessWordsWeak = []
socRepetiveWordsWeak = []
socFilterWordsWeak = socMeanlessWordsWeak + socRepetiveWordsWeak

# 강한 필터
socMeanlessWordsStr = []
socRepetiveWordsStr = []
socFilterWordsStr = socMeanlessWordsStr + socRepetiveWordsStr
```

### 종합 필터


```python
# 약한 필터
totalMeanlessWordsWeak = []
totalRepetiveWordsWeak = []
totalFilterWordsWeak = totalMeanlessWordsWeak + totalRepetiveWordsWeak

# 강한 필터
totalMeanlessWordsStr = []
totalRepetiveWordsStr = []
totalFilterWordsStr = totalMeanlessWordsStr + totalRepetiveWordsStr
```

## <font color = 'red'>main 함수<font>
> - extract_keyword 내부에서 호출 함수 이용
- 키워드를 얻기 위한 자잘한 과정들을 내부적으로 모듈화


```python
def main(startDay, endDay, numOfWord, category, data, filterWord=[]):
    
    # 날짜 목록 만들기
    yearMonthList = pd.period_range(start=startDay, end=endDay, freq='M')
    yearMonthList = yearMonthList.strftime('%Y%m').tolist()
    
    # 정제한 키워드 단어를 담을 데이터 프레임
    resultWords = pd.DataFrame()
    
    # 키워드 뽑아내기 (기존에 만든 extract_keyword 함수 호출)
    for yearMonth in yearMonthList:
        tmpDf = extract_keyword(yearMonth, numOfWord, category, data, filterWord)
        resultWords = pd.concat([resultWords, tmpDf], axis=1)
    
    return resultWords
```

## <font color = 'red'>전체적인 동작<font>
> - 기사 크롤링 함수 동작
- 정부 지지율 크롤링 함수 동작
- main 함수로 키워드 추출
- 데이터 시각화

### 기사 크롤링 동작


```python
newsTop30 = crawling_news(20171101, 20181115, 30)
newsTop30
```


```python
# 기사 TOP 30 파일 저장

newsTop30.to_csv('data/naver_news_top30.csv', sep=',', encoding='UTF-8')
newsTop30.to_excel('data/naver_news_top30.xlsx')
```


```python
# 저장한 뉴스 헤드라인 파일 읽어오기

newsTop30 = pd.read_csv('data/naver_news_top30.csv', encoding='UTF-8')
newsTop30.drop('Unnamed: 0', axis=1, inplace=True)
newsTop30.head()
```

### 정부 지지율 크롤링 동작


```python
governmentSupport = support()
governmentSupport.head()
```

    79
    79
    




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>지지율</th>
    </tr>
    <tr>
      <th>날짜</th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>201710</th>
      <td>73.0%</td>
    </tr>
    <tr>
      <th>201711</th>
      <td>75.0%</td>
    </tr>
    <tr>
      <th>201712</th>
      <td>68.5%</td>
    </tr>
    <tr>
      <th>201801</th>
      <td>64.0%</td>
    </tr>
    <tr>
      <th>201802</th>
      <td>64.0%</td>
    </tr>
  </tbody>
</table>
</div>




```python
# 정부 지지율 파일 저장

governmentSupport.to_csv('data/governmentSupport.csv')
governmentSupport.to_excel('data/governmentSupport.xlsx')
```


```python
# 정부 지지율 파일 읽어오기

governmentSupport = pd.read_csv('data/governmentSupport.csv', encoding='UTF-8')
governmentSupport.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>날짜</th>
      <th>지지율</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>201710</td>
      <td>73.0%</td>
    </tr>
    <tr>
      <th>1</th>
      <td>201711</td>
      <td>75.0%</td>
    </tr>
    <tr>
      <th>2</th>
      <td>201712</td>
      <td>68.5%</td>
    </tr>
    <tr>
      <th>3</th>
      <td>201801</td>
      <td>64.0%</td>
    </tr>
    <tr>
      <th>4</th>
      <td>201802</td>
      <td>64.0%</td>
    </tr>
  </tbody>
</table>
</div>



### main 함수 동작 > 키워드 추출


```python
totalWords30 = main(20171101, 20181101, 50, '종합', news, tmpFilter)
totalWords30
```


```python
# 키워드 파일 저장

totalWords30.to_csv('data/keyword_total_top30.csv', sep=',', encoding='UTF-8')
totalWords30.to_excel('data/keyword_total_top30.xlsx', encoding='UTF-8')
```


```python
# 키워드 파일 읽어오기

newsTop30 = pd.read_csv('data/keyword_total_top30.csv', encoding='UTF-8')
newsTop30.drop('Unnamed: 0', axis=1, inplace=True)
newsTop30.head()
```


```python
import matplotlib.pyplot as plt
from matplotlib import font_manager, rc
font_name = font_manager.FontProperties(fname="C:/Windows/Fonts/malgun.ttf").get_name()
rc('font', family=font_name)

%matplotlib inline
```
