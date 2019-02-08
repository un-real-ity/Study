# 3D 수학
## 벡터
- 방향 + 방향 = 방향
- 방향 - 방향 = 방향
- 점 + 방향 = 점
- 점 - 점 = 방향
- 점 + 점 = 성립 안 됨

- 제곱근은 연산량이 많으므로 걍 제곱써라


## 내적
- (a.b) = Ax*Bx + Ay*By + Az*Bz
- (a.b) = |a||b|cos(theta)
- a.b = b.a (교환법칙)
- a.(b+c) = a.b + a.c (분배법칙)
- |a|^2 = a.a (a와 a의 사이각은 0이라 cos0 = 1이기때문)
- 내적으로 두 벡터의 관계를 알아낼 수 있음
-- 평행, 같은 방향: (a.b) = |a||b|
-- 평행이고 정반대 방향: (a.b) = -ab
-- 수직: (a.b) = 0
-- 같은 방향: (a.b) > 0
-- 다른 방향: (a.b) < 0

## 외적
- |a x b| = (Ay*Bz - AzBy, Az*Bx - Ax*Bz, Ax*By - Ay*Bx)
- |a x b| = |a||b|sin(theta)
- 외적할 때, 오른손 좌표게면 오른손으로만 포개자. (반대도 마찬가지)
- a x b != b x a (교환법칙 X)
- a x (b + c) = (a x b) + (a x c) (덧셈에 대해선 분배법칙 성립)

## 행렬
- Affine Matrix
-- rotation, translation, scale, shear 합친 4x4 행렬
-- 무조건 역행렬 있음

- (ABC) 역행렬 = C역B역A역
- (ABC) 전치 = C전B전A전

- 동차좌표계(Homogeneous Coordinates): 3차원 벡터나 점을 4차원으로 확장한 것
-- 점은 w가 1임 (x,y,z,1)
-- 맨 우측 행은 항상 [0 0 0 1]이라서 4x4  말고 4x3으로 자주 씀.

## 좌표공간
- Pitch: 모델의 왼쪽(L) or 오른쪽(R) 기준으로 회전
- Yaw: 모델의 위(U) 기준으로 회전
- Roll: 모델의 앞(F) 기준으로 회전
