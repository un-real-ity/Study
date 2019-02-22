# 엔진 지원 시스템
* 엔진 시작과 종료
* 엔진과 게임을 설정
* 게임의 메모리를 관리
* 게임 리소스 처리 지원
* 디버깅 툴 제공

# 하부 시스템 시작, 종료
* 하부시스템: 엔진의 모듈, 구성요소
* 하부시스템들은 서로 의존적으로 구성될 수 있다.
* 의존적인 경우 초기화, 종료가 정해진 순서로 동작해야한다.

## C++ 정적 초기화 순서
* C++은 프로그램의 시작함수 (main)가 호출되기 전에 전역객체와 정적 객체를 생성한다.
* 전역, 정적 객체의 생성 순서는 임의적이다.
* 하부시스템은 전역에서 사용가능한 싱글턴 패턴을 사용한다.
* 싱글턴으로 구성된 하부 시스템들이 서로 의존적인 경우 C++ 의 기본 정적 전역 변수 초기화는 순서 문제로 사용하기 어렵다.

> 주문형 생성
* 함수 안에 선언된 정적 변수는 함수 호출시 최초 생성된다는 점을 사용한 초기화 트릭
```c++
// 주문형 생성 예시
class RenderManager
{
public:
  static RenderManager& get()
  {
    static RenderManager sSingleton;
    return sSingleton;
  }

  RenderManager()
  {
    // 의존적인 상위 하부시스템 생성
    VideoManager::get();
    TextureManager::get();
  }
};
```
* 이 경우 파괴 순서 제어가 어렵다는 단점이 있다.
* RenderManager::get() 호출 맥락, 시점을 알 수 없다.
* 최초 get에서 시간이 많이 필요할 수 있다.

> 추천 방법
* 각 싱글턴 매니저 클래스에 시작과 종료를 담당하는 함수를 명시적으로 정의하는 것
* 생성자와 파괴자는 아무것도 안함
* main 함수나 전체 엔진을 총괄하는 싱글턴 객체에서 모든 싱글턴을 정해진 순서로 시작, 종료 처리를 한다.
```c++
RenderManager gRenderManager;
PhysicsManager gPhysicsManager;
Animationmanager gAnimationManager;

int main()
{
  gRenderManager.startUp();
  gPhysicsManager.startUp();
  gAnimationManager.startUp();

  gSimulationManager.run();

  gaAnimationManager.shutDown();
  gPhysicsManager.shutDown();
  gRenderManager.shutDown();

  return 0;
}

```
* 추천하는 이유
    * 방식이 단순하고 구현하기 쉽다.
    * 명확하다. 코드만 보면 바로 시작순서를 알 수 있다.
    * 디버깅하고 유지하기 쉽다. 문제가 발생하면 순서만 바꿔주면 된다.

# 메모리 관리

* 메모리가 성능에 영향을 끼치는 형태
    * 동적 메모리 할당(new, malloc)의 지연시간
    * 메모리 단편화 현상으로 비효율적인 메모리 사용
    * 연속적으로 사용하는 변수가 연속된 메모리 블록에 들어가 있으면 처리속도가 훨씬 빠르다. (locality)

## 동적 메모리 할당 최적화
* 동적 메모리 할당이 느린이유
    * 동적 메모리 할당에 사용되는 힙할당자는 범용으로 다양한 크기의 할당을 처리할 수 있어야한다.
    * 다양한 크기의 메모리 조각을 관리하는 부가적인 비용이 필요하다.
    * 메모리를 할당하시 OS는 유저모드에서 커널모드로 컨텍스트 스위칭을 한다.
* 힙 할당은 최소화하고, 타이트 루프에선 힙할당 하지 말아라~
* 대부분의 게임엔진은 여러 종류의 할당자를 직접 구현해서 사용한다.
    * 미리 할당된 메모리 블록을 이용한다.
    * 운영체제의 컨텍스트 스위칭이 필요 없다.
    * 사용 패턴을 예측할 수 있어 범용에 비해 효율적으로 구현 가능하다.

> 스택기반 할당자
* 스택형테로 메모리를 할당한다.
    * 새로운 레벨에 사용되는 메모리를 스택 형태로 쌓는다.
    * 레벨이 종료되면 레벨에서 사용된 메모리를 해제한다.
* 할당
    * 스택에 사용될 연속적인 메모리 블록을 할당한다.
    * 포인터 하나를 둬서 스택의 꼭대기 위치를 기억한다.
    * 이 포인터보다 작은 메모리 주소는 사용중이고 아닌것은 사용가능한 공간이다.
    * 할당 요청시마다 현재 포인터 위치의 메모리 주소를 넘기고 스택의 포인터를 할당된 공간만큼 올려준다.    
* 해제
    * 가장 근래의 블록을 해제할때 꼭대기 포인터를 블록의 크기만큼 내리면 된다.
    * 해제는 할당 역순으로만 가능하다.
    * 개별 블록 해제는 제공하지 않고 스택의 특정 집합을 롤백하는 방법으로 해제하는것이 좋다.
    * 롤백의 효율적인 동작을 위해 스택할당자는 현재 스택의 꼭대기를 리턴하는 마커를 리턴하는 함수를 제공한다.
    * 메모리 할당시 미리 마커를 받아서 유지하다가 롤백시에 마커를 인자로 넘겨서 해제한다.
* 더블 포인터 스택
    * 포인터를 두개두고 하나는 아래서부터, 하나는 위에서부터 사용한다.
    * 할당 해제 단위를 두개로 나눠서 더 유연하게 사용할 수 있다.
    * 오래동안 사용되지 않는 메모리 영역을 줄일 수 있다. (내생각)

> 풀 할당자
* 작은 메모리 블록을 여러개 할당하는 경우 유용
* 행렬, 반복자, 노드, 메시 인스턴스등
* 구현
    * 개별 원소들의 크기에 배수가 되는 메모리 블록을 할당
    * 풀의 원소는 사용가능 리스트에서 뽑아 쓴다.
    * 해제할때 다시 리스트에 넣는다.
    * 할당 해제 모두 포인터 처리 두어개로 O(1) 안에 처리된다.
    * 각 원소의 크기가 포인터 크기보다 큰 경우 원소에 다음 원소를 나타내는 포인터를 저장하면 된다.
    * 더 작은 경우는 풀의 원소를 나타내는 인덱스로 리스트를 구현한다.

> 정렬된 할당자
* 변수와 데이터 객체들은 메모리 정렬 조건을 갖는다.
* 메모리 할당자는 정렬된 메모리를 리턴할 수 있어야 한다.
* 구현
    * 실제 요청된 것보다 조금 큰 메모리를 할당한다.
    * 블록의 주소를 조정해서 정렬을 맞춘다
    * 조정값은 주소에 정렬값 - 1을 마스킹해서 구한다.
    * 주소 + 정렬값 - 조정값이 조정된 주소이다.
    * 해제를 위해 조정된 주소 바로앞 한바이트에 조정값을 저장한다.
    * 조정된 주소를 리턴한다.

```c++
void* allocateAligned(U32 size_bytes, U32 alignment) 
{
    // 할당할 메모리 크기를 결정한다.
    U32 expandedSize_bytes = size_bytes + alignment;
    // 정렬되지 않은채로 메모리 블록을 할당한다.
    U32 rawAddress = (U32)allocateUnaligned(expendedSize_bytes);
    // 주소의 낮은 비트들을 마스킹해서 조정값을 결정한다.
    U32 mask = (alignment - 1);
    U32 misalignment = (rawAddress & mask);
    U32 adjustment = alignment - misalignment;
    // 조정한 주소값을 구한다.
    U32 alignedAddress = rawAddress + adjustment;
    // 1바이트 앞에 조정값을 저장한다.
    U8* pAdjustment = (U8*) (alignedAddress - 1);
    *pAdjustment = adjustment;

    return (void*)alignedAddress;
}

// 정렬된 메모리 해제
void freeAligned(void* p)
{
    U32 alignedAddress = (U32)p;
    U8* pAdjustment = (U8*) (alignedAddress - 1);
    U32 adjustment = (U32)*pAdjustment;
    U32 rawAddress = alignedAddress - adjustment;
    freeUnaligned((void*)rawAddress);
}
```

> 단일 프레임과 이중버퍼 메모리 할당자
* 임시 데이터 할당
    * 루프가 끝나면 버려지거나 다음 프레임에 쓰고 버리는 데이터를 위한 임시 할당
* 단일 프레임 할당자
```c++
StackAllocator g_singleFrameAllocator;

// 게임루프
while(true)
{
    // 매 프레임마다 프레임버퍼 초기화
    g_singleFrameAllocator.clear();
    // ...
    // 단일 프레임 버퍼에서 할당하고 해제 신경 ㄴㄴ
    // 프레임 끝나면 지워지니깐 조심해
    void* p = g_singleFrameAllocator.alloc(nBytes);
    // ...
}
```
* 이중버퍼 할당자
```c++
class DoubleBufferedAllocator
{
private:
    U32             m_curStack;
    StackAllocator  m_stacks[2];
public:
    void swapBuffers()
    {
        m_curStack = (U32) !m_curStack;
    }

    void clearCurrentBuffer()
    {
        m_stacks[m_curStack].clear();
    }

    void* alloc(U32 nBytes)
    {
        return m_stacks[m_curStack].alloc(nBytes);
    }
}

DoubleBufferedAllocator g_doubleBufAllocator;

// 게임루프
while(true)
{
    // 이중버퍼 할당자 버퍼 스왑
    g_doubleBufAllocator.swapBuffers();
    // 새로 활성화된 버퍼 초기화 
    // 이전프레임 버퍼는 유지
    g_doubleBufAllocator.clearCurrentBuffer();
    // ...
    // 2 프레임간 사용가능한 메모리 할당
    void* p = g_doubleBufAllocator.alloc(nBytes);
    // ...
}
```
* 프레임 의존적인 비동기 작업 처리할때 유용
    * i번째 프레임에서 비동기작업 결과 저장할 주소로 이중버퍼 할당자에서 할당한 메모리 주소를 넘김
    * i + 1 번째 프레임 시작하면 저장된 결과물 사용가능

## 메모리 단편화
* 동적 힙 할당자 막 쓰면 빈 구멍 많아진다.
* 빈 공간이 충분히 있어도 메모리 할당에 실패할 수 있다.
* 가상메모리 지원하는 OS는 메모리 단편화 별로 걱정안해도된다.
    * 물리 메모리를 페이지라는 가상 주소 공간으로 매핑해서 제공
    * 오래 안쓴 페이지는 물리메모리 부족하면 하드디스크에 옮겼다가 필요하면 다시 불러옴
    * 하드디스크 내려갔다 올라가면 느려서 성능 문제 있긴함

> 스택 할당자, 풀 할당자로 단편화 예방
* 스택 할당자는 언제나 연속적으로 할당하고, 연속적으로 해제하니깐 메모리 단편화 없음
* 풀 할당자는 풀들 자체는 단편화 될 수 있으나 모든 블록 크기 같아서 연속된 공간 부족 문제는 없다.

> 조각모음과 재배치
* 크기가 제각각, 정해진 순서없이 할당 해제하는 경우 스택, 풀 할당자 못씀
* 힙을 주기적으로 조각모음해야한다.
* 조각모음
    * 힙에있는 구멍을 하나로 모으는 과정
    * 할당된 블록을 최대한 낮은 메모리 주소로 이동한다.
    * 맨 아래 구멍을 찾은뒤 바로 뒤 블록을 구멍의 첫 위치로 옮김, 그리고 그다음 구멍 채우고 ...
* 재배치 
    * 이미 포인터로 사용중인 데이터가 이동하면 포인터가 이상해져버렷
    * 조각모음후 기존 포인터를 새로 이동한 주소로 재배치필요
    * 첨부터 재배치에 적합한 스마트 포인터나 핸들 쓰면 좋다.
* 스마트 포인터
    * 포인터를 래핑한 클래스
    * 재배치 처리하게 코드를 짜자
    * 스마트 포인터가 스스로 전역 리스트에 추가되게 만들어본다.
    * 메모리 블록이 재베치되면 포인터 리스트에서 해당 포인터를 찾아서 새로운 주소로 업데이트
* 핸들
    * 포인터가 담긴 테이블 내의 번호
    * 테이블의 메모리는 재배치 하지말자
    * 블록 이동하면 핸들 테이블을 검색해 해당 포인터들을 업뎃
* 사용 라이브러리에서 스마트포인터나 핸들을 지원하지 않는 경우
    * 재배치 불가능한 메모리 영역에서 특수한 버퍼를 만들고 라이브러리가 그 버퍼를 사용하게 만들기
* 조각모음 최적화
    * 여러 프레임에 걸쳐 나눠서 조각모음 수행
    * 큰 블록은 몇개의 작은 블록으로 쪼개서 재배치
    * 가능하면 작은 객체만 재배치하는게 좋다.

## 캐시 일관성

* 메인RAM읽는것 느려서 고성능 메모리 캐시를 사용한다
* 캐시는 CPU가 읽고 쓰는 더 빠른 메모리
* 데이터 읽을때 캐시에 있으면 캐시만 읽음 없으면 메모리 읽음
* 데이터 쓸때 우선적으로 캐시에 쓰고 최대한 나중에 메모리에 씀
* 캐시 미스를 최소화 하면 성능이 올라간다.

> L1 캐시와 L2 캐시
* 더 빠른 캐시와 느리지만 좀더 용량이 큰 캐시
* 로드 힛 스토어 문제
    * CPU가 메모리 주소에 데이터를 씀
    * 데이터가 L1까지 가기전에 데이터를 다시 읽을때

> 명령어 캐시와 데이터 캐시
* 데이터와 코드가 모두 캐시가 적용된다
* 명령어(I) 캐시는 실행 파일의 명령어 코드가 실행되기 전에 미리 불러오는것
* 데이터(D) 캐시는 데이터를 메인 RAM에 읽고 쓰는 속도 올릴때 사용하는 것

> 캐시 미스 피하기
* D 캐시 미스 피하기
    * 데이터를 가능한 잘게 쪼개서 연속적인 블록으로 배치하고 순서대로 접근하는 것
    * 데이터가 연속적이면 캐시 미스 발생시 한번에 최대한 연관된 데이터를 많이 읽어 들일 수 있다.
    * 데이터가 작으면 캐시라인 하나에 전부 들어갈 확률이 커진다.
    * 데이터를 순서대로 접근하면 RAM의 한 영역에 대한 캐시라인을 여러번 불러올 필요가 없어진다.
* I 캐시 미스 피하기 원론편
    * D 캐시 미스랑 비슷한데 코드가 메모리에 어떻게 올라가는지는 컴파일러랑 링커에 달려있다.
    * C++ 링커의 규칙을 알면 유리하게 이용할 수 있다.
    * 함수 하나를 나타내는 기계어 코드는 거의 항상 메모리에서 연속적이다.
    * 번역 단위의 소스코드(cpp)에 나오는 순서대로 함수들이 메모리에 배치된다.
    * 따라서 같은 번역 단위에 있는 함수들은 메모리에 연속적으로 놓인다.
* I 캐시 미스 피하기 실전편
    * 성능이 중요한 코드는 명령어 수가 적게 짜자
    * 성능에 영향을 끼치는 코드는 함수호출을 줄이자
    * 함수를 반드시 호출해야된다면 최대한 호출 함수와 가까운 곳에 위치하자. 다른 번역단위는 피하자
    * 인라인 함수를 너무 많이 쓰면 코드의 크기가 커진다. 
    * 루프 안에서 코드가 적게 들어가게 구현하자.