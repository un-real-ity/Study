# 게임 루프와 리얼타임 시뮬레이션
- 게임 엔진이 다뤄야 할 시간?
  - 리얼타임
  - 게임 시간
  - 애니메이션의 로컬 타임라인
  - 함수 실행에 걸린 실제 CPU 주기
  - 등등.....

## 렌더링 루프
OS의 GUI는 대부분 정적이라 작은 부분들만이 모양을 바꿈.
- 그래서 스크린을 전부 다시 그리는게 아니라, 바뀐 일부분만을 다시 그림. (사각형 무효화 - Rectangle invalidation)
- 이건 2D에서도 쓰이는 기법.

하지만.... 3D에선 거의 매 순간 모든 스크린이 변하기 때문에 사용할 수 없음.
- 3D는 영화처럼 정지화면을 빠르게 스왑해서 움직임처럼 보이게 함.
- 그래서 빠르게 루프를 계속 돌려야 하는데, 이를 **렌더링 루프**라고 부름.

## 게임 루프
게임은 상호작용하는 다양한 하부 시스템으로 이뤄짐 (장치 I/O, 렌더링, 애니메이션, 물리 등)
- 대부분의 하부 시스템은 게임이 실행중일 때, 주기적으로 업데이트 해줘야함.
- 애니메이션은 통상 30Hz, 60Hz
- 동적 시뮬레이션은 더 자주(ex. 120Hz)
- AI 등의 하이레벨 시스템 - 초당 1,2번 등등

단순한 방법으로 하부 시스템을 업데이트 한다고 하면, 루프 하나에서 모든 업데이트를 처리하면 된다.
그래서 이 루프를 **게임 루프** 라고 부른다.

```cpp
// 예시 - Pong 게임
void main() {
	initGame();
	while (true) { // 게임 루프
		readHumanInterfaceDevices();
		if (quitButtonPressed()) break;

		movePaddles();
		moveBall();
		colideAndBounceBall();
	
		if (ballImpactedSide(LEFT_PLAYER)) {
			incrementScore(RIGHT_PLAYER);
			resetBall();
		} else if (ballImpactedSide(RIGHT_PLAYER)) {
			incrementScore(LEFT_PLAYER);
			resetBall();
		}

		renderPlayField();
	}
}
```
