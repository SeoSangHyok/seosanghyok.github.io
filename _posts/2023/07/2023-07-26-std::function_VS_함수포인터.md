---
layout: post
title: std::function_VS_함수포인터
subtitle: 함수포인터와 std::function 의 비교및 사용법을 알아보자
categories: c++
tags: [c++, thread] 
---


## std::function_VS_함수포인터

c++11 부터 함수포인터를 대신할 수 있는 std::function 라이브러리가 추가됐다. 형변환이 명확해야 하는 함수포인터와 다르게 형식만 맞으면 유연하게 사용가능하고 std::bind 와 형행하면 인자처리도 유연하게 할 수 있다. 함수포인터도 많은곳에서 쓰이는 만큼 둘의 사용법을 모두 비교해본다.

### 기본 사용

#### 함수 포인터

함수포인터의 선언은 아래와 같다.
```
[반환형](*[변수명])([전달인자 형식들..])
```

만날 저렇게 선언하면 많이 블편한데 이경우 typedef를 사용하면 편하다.
```
typedef [반환형](*[typedef명])([전달인자 형식들..])
```

``` cpp
int TestFunc(double param)
{
    std::cout << param << std::endl;
    return param;
}

int TestFunc2(double param)
{
    std::cout << param*10 << std::endl;
    return param*10;
}

int main()
{
    // 함수포인터의 선언
    int(*fp)(double) = TestFunc;
    fp(100.0);

    // typedef 형식으로 함수포인터 형식 선언
    typedef int(*fp_type)(double);
    fp_type fp2 = TestFunc;
    fp2(200.0);

    fp2 = TestFunc2;
    fp2(300.0);
}
```

### std::function

위와같은 상황에서 function 클래스는 아래와 같이 선언할 수 있다.
```
std::function<[반환형]([인자형식 리스트])> [변수명] = [대상함수]
```

``` cpp
#include <functional>  // std::function을 사용하기 위해 필요

int main()
{
    std::function<int(double)> stdFunc = TestFunc;
    stdFunc(100.0);

    stdFunc = TestFunc2;
    stdFunc(200.0);
}
```

### 콜링 컨벤션 처리

c++의 콜링 컨벤션은 기본적으로 __cdecl 이지만 윈도우 기준 스레드 함수는 __stdcall 이다. 콜링 컨벤션을 처리하기 위해 함수포인터는 컨벤션을 명확하게 해야 하지만 std::function은 호환처리 가능하다.

``` cpp
int TestFunc(double param)
{
    std::cout << param << std::endl;
    return param;
}

int __stdcall TestFunc2(double param)
{
    std::cout << param*10 << std::endl;
    return param*10;
}

int main()
{
    // 함수포인터와 std::function 객체 생성
    typedef int(*cdclFpType)(double);
    std::function<unsigned(double)> stdFunc;
    
    cdclFpType cdclFp = TestFunc;  // 가능
    stdFunc =  TestFunc;        // 역시 가능

    cdclFp = TestFunc2;            // 에러!! TestFunc2의 콜링 컨벤션은 __stdcall이다

    // 아래처럼 __stdcall 콜링컨벤션을 사용하는 함수포인터 타입을 만들어야 대입가능
    typedef int(__stdcall*stdcallFpType)(double); 
    stdcallFpType stdcallFp = TestFunc2

    stdFunc = TestFunc2;        // 가능!! std::function은 콜링컨벤션 호환된다.
}
```

### 클래스 맴버함수 처리

클래스 맴버함수의 함수포인터의 경우 호출하고자 하는 대상 즉 **어떤 인스턴스의 함수를 호출했느냐??** 가 중요하다. 일반함수의 경우 전역 범위를 가지지만 클래스의 경우 **인스턴스**가 생성되고 각 인스턴스의 맴버함수가 호출되기 때문이다.

#### 함수포인터

함수포인터로 클래스 맴버함수를 표현하고 사용하는 방법은 아래와 같다.

``` cpp
class CTestClass
{
public:
    CTestClass(int initVal){m_val = initVal;}
    ~CTestClass(){;}
    int testFunc(){ return m_val; }
    int testFunc2(){ return m_val*10; }    

    int testFunc3() const 
    {
        return m_val*100;
    }
private:
    int m_val{ 0 };
}

int main()
{
    // 아래와 같이 클래스 맴버 함수 포인터 선언 가능
    int(CTestClass::*pCfp)() = &CTestClass::testFunc;

    // 타입정의도 가능
    typedef int(CTestClass::*ClassfpType)();
    ClassfpType pCfp2 = &CTestClass::testFunc2;

    // 여기까진 맴버 함수만 정의했을 뿐 대상 인스턴스가 없으므로 사용하 려면 인스턴스 생성먼저..
    CTestClass a(100);
    CTestClass b(200);

    // 사용하고자 하는 클래스 맴버 함수 포인터를 인스턴스와 함께 사용한다. 연산자 우선 순위가 있어 괄호로 묶어서 사용
    (a.*pCfp)();
    (a.*pCfp2)();
    (b.*pCfp)();
    (b.*pCfp2)();
}
```
위와 같이 클래스 맴버함수의 함수포인터를 호출할 경우 대상 인스턴스를 명확히 해야 한다.

#### std::function

std::function도 마찬가지로 사용하고자 하는 맴버를 명시해야 한다. 다만 함수포인터와 다르게 내부적으로 호출한 객체의 래퍼런스를 넘겨준다(this 포인터를 생각하면 되겠다)

``` cpp
// 클래스 맴버 함수 포인터 선언 및 할당
std::function<int(CTestClass&)> ClassFunc = &CTestClass::testFunc1;

// 클래스 인스턴스 생성
CTestClass a(100);
CTestClass b(200);

ClassFunc(a);   // a.testFunc1() 과 같음
ClassFunc(b);   // b.testFunc1() 과 같음

ClassFunc = &CTestClass::testFunc2;
ClassFunc(a);   // a.testFunc2() 와 같음
ClassFunc(b);   // b.testFunc2() 와 같음
```