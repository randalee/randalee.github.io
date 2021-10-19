---
title: 파이썬 데코레이터를 이용한 재시도 로직
tags:
- python
- decorator
- loop
toc: true
toc_sticky: true
category: python
---

---
### 소개
개발을 하다보면 실패를 가정하고, 재시도를 해야하는 경우가 종종발생한다. 이러한 경우 다양한 방법이 있겠지만, 외부 API연동작업을 하다보니 생각보다 이런일이 많이 발생하였다. 

따라서 이를 간편하게 만들어보고자 데코레이터로 테스트하고, 구현하여 사용해보았다.


### 일반적인 방식
재시도는 똑같은 행동을 반복해야 하기 때문에 반복문을 사용하게 된다. python의 반복문은 `for`와 `while`이 존재하기에 이를 이용하여 처리하게 된다.

```python
# for를 이용한 경우
max_retry = 10
for i in range(max_retry):
    result = 메인코드 성공/실패
		
    if result:
        break # 메인코드 성공 시 루프 빠져나옴

# while을 이용한 경우
max_retry = 10
now_retry = 1
while now_retry <= max_retry:
    result = 메인코드 성공/실패
		
    if result:
        break # 메인코드 성공 시 루프 빠져나옴
    now_retry += 1  
```

사실 반복문을 사용한다해서 코드가 복잡하거나 그러지는 않는다. 다만 이를 자주 사용해야하는 경우, 그리고 최대 재시도를 몇번 줄것인가가 로직에 따라 다른 경우 비슷한 코드가 여러개 존재하게 되는데 이를 보드 깔끔하게 해결하기 위한 방법이라고 보면 된다.


### 데코레이터를 이용한 방식
데코레이터는 함수와 클레스로 만드는 2가지 방식이 존재한다. 여기서는 클래스를 이용하여 만들었다. 아래 코드를 보자
```python
import logging
import time


class RetryLoop():
    def __init__(self,
                 max_loop_count:int = 10,
                 loop_sleep_interval:int = 60
                 ):
        """
        :param max_loop_count: 최대 루프 카운드
        :param loop_sleep_interval: 다음루프를 진행하기까지 대기 시간(초)
        """
        self.max_loop_count = max_loop_count
        self.loop_sleep_interval = loop_sleep_interval

        self.now_loop = 0

    def __call__(self, func):
        def wrapper(*args, **kwargs):
            while self.now_loop < self.max_loop_count:
                self.now_loop += 1

                try:
                    result = func(*args, **kwargs)
                    self.now_loop = 0 # Reset loop pos
                    return result
                except Exception as e:
                    logging.warning(f'Loop failed. Reason for failure: {str(e)}')
                    logging.warning(f"After {self.loop_sleep_interval} seconds of sleep, enter the next loop.({self.now_loop}/{self.max_loop_count})")
                    time.sleep(self.loop_sleep_interval)

            self.now_loop = 0  # Reset loop pos
            return Exception(f'Maximum number of retries exceeded {self.max_loop_count}')
        return wrapper
```

1. class의 인자로 `max_loop_count`와 `loop_sleep_interval`을 받았다. 전자의 경우 최대 몇번을 실행 시킬 것인가를 설정하고 후자의 경우 재시도를 할 때 실패 후 몇초를 쉬고 재시도를 할 것인가를 정할 수 있도록 한 값이다. 별도로 명시하지 않은 경우 최대 10회 시도, 1분마다 다음 작업을 실행한다.  중간 로그 출력의 경우 logging 모듈을 이용했는데.. 이는 뭐 경우에 따라 `print`와 같은 출력으로 얼마든지 대체하면 된다.
2. 해당 데코레이터의 경우 데코레이터를 사용하는 함수를 `result` 객체에 저장하게 되는데, 만약 실패하는 경우 예외처리가 발생하면서 다음 루프로 진행하게 된다. 즉, 해당 데코레이터를 사용하는 함수는 문제발생시 반드시 예외가 발생하도록 구현하도록 강제하고 있다.  
3. 예외가 발생하지 않은 경우 결과를 리턴하면서 데코레이터는 종료되고, 함수의 결과를 리턴한다.
4. 최종적으로 최대 시도를 모두 소진한 경우 **Exception을 리턴**한다. (raise로 예외 발생시키는것이 아니다.)


위와 같이 만들어진 데코레이터는 아래와 같이 사용할 수 있다.
```python
import random

@RetryLoop(5,1) # 5회 재시도, 1초씩 슬립
def test():
    data=random.random()
    
    if data > 0.9: # 성공을 희박하게 하기 위해서. 0~1사이 랜덤
        return f'성공 ~~ {data}'
    raise Exception(f'실패 ~~ {data}')
```

test함수를 실행한다면 5회 이내에서 작업을 성공한다면 `f'성공 ~~ {data}'` 을 리턴값으로 받을 것이고, 최종적으로 실패한다면 데코레이터의 `Exception(f'Maximum number of retries exceeded {self.max_loop_count}')` 부분을 리턴 값으로 받게 된다.

따라서 해당 데코레이터를 사용할 경우 `isinstance`를 이용해서 리턴 값이 `Exception`인가 아닌가를 판단하여 이후 작업을 진행하는 구조로 만들면 된다. 아래는 결과에 대한 출력 샘플이다.
![Image](/assets/posts/202110/211019_python.png)
