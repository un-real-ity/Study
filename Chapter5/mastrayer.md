# 엔진 지원 시스템
- 엔진을 시작하고 끝내는 일
- 엔진과 게임을 설정하는 일
- 게임의 메모리 사용을 관리하는 일
- 파일 시스템 접근을 처리하는 일
- 온갖 종류의 게임 자원(메시, 텍스쳐, 애니메이션, 오디오 등)을 처리하게 지원하는 일
- 디버깅 툴을 제공하는 일


## C++ 정적 초기화 순서
- 는 main이 실행되기 전에 전역 객체와 정적 객체를 생성.
- 하지만 그 순서는 완전히 임의적.
- 그래서 의존성이 있는 여러 객체들간의 순서를 보장할 수 없어서 트릭을 이용.
- 함수 안에서 선언된 정적 변수는 해당 함수가 처음 호출될 때 생성됨.
```cpp
class RenderManager {
public: 
	static RenderManager& get() {
		static RenderManager sSingleton;
		return sSingleton;
	}
	RenderManager() {
		// 의존성이 있는 다른 매니저들의 get을 호출해 먼저 시작하게 함
		VideoManager::get();
		TextureManager::get();
		// RenderManager 시작
		...
	}
	~RenderManager() {
		...
	}
}
```
- 그러나... 이렇게 쓰면 어떤 부분에서 처음 초기화되는지, 그리고 병목이 생길지 예측이 불가능해서 좀 더 직접적인 접근이 필요.

```cpp
class RenderManager {
public: 
	RenderManager() { /* 아무것도 안함 */	}
	~RenderManager() { /* 아무것도 안함 */ }
	void startUp() {
		// 매니저 시작
	}
	void shutDown() {
		// 매니저 종료
	}
}

RenderManager gRenderManager;
PhysicsManager gPhysicsManager;
int main() {
	// 의존성을 고려해서 필요한 순서대로 매니저 시작
	gPhysicsManager.startUp();
	gRenderManager.startUp();
	...
	
	// 역순으료 종료
	gRenderManager.shutDown();
	gPhysicsManager.shutDown();
	
	...
}
```

- Ogre엔진은 Root라는 싱글톤 객체 하나가 있고, Root가 각종 매니저들을 포인터로 가지고 있으며 한 번에 관리함.

## 메모리 관리
1. 동적 메모리 할당은 굉장히 느림. 동적할당을 피하거나, 할당 비용을 줄일 수 있는 메모리 할당자(allocator)를 직접 만들어야 함.
2. 성능은 메모리 접근 패턴에 좌우되는 경우가 많다. 같은 데이터라도 작고 연속적인 메모리 블록이 유리함.

### 동적 메모리 할당 최적화
- malloc, free, new, delete 등 동적 할당은 엄청나게 느림.
- 힙 할당자는 범용적이라 1바이트부터 기가를 넘는 단위까지 처리할 수 있어야해서 관리하는데 부가적인 비용이 들기 때문에 느림.
- 그리고 malloc, free 등을 호출할 때,  커널과 유저모드간의 컨텍스트 스위칭이 일어나서 느림.
> 힙 할당은 최소화하고 타이트루프 안에서는 절대 힙 할당을 하지 말 것
- 커스텀한 할당자는 왜 더 빠르냐면..
	1. 미리 할당된 메모리 블록(처음에 동적할당하거나 전역변수로 선언)을 이용할 수 있다. (커널과의 컨텍스트 스위칭 X)
	2. 예측 가능한 사용 패턴 덕분에 훨씬 효율적으로 동작 가능
	3. http://www.swedishcoding.com/2008/08/31/are-we-out-of-memory 가 도움이된다네.

#### 스택 기반 할당자
- 새로운 레벨을 불러올 때마다, 메모리를 할당.
- 레벨 로딩이 끝나면 동적 메모리 할당은 거의 없거나 아주 적은 수만 발생.
- 레벨이 끝나면 메모리를 해제.
- 구현하기 쉽다. 크고 연속적인 메모리 블록을 할당하기만 하면 됨.
- **임의의 순서로 메모리를 해제할 수 없음**
