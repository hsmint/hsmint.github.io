---
title: "[TIL] 가우스 소거법과 rank, 그리고 det가 아니라 조건수로 판단해야 하는 이유"
date: 2026-09-01
description: "가우스 소거법은 [A|b]를 위삼각으로 만든 뒤 되짚어 올라가는 절차이고, 부분 피벗팅은 정렬이 아니라 나눗셈의 분모를 키우는 정확도 장치다. rank는 독립인 식의 개수다. det가 작다고 위험한 게 아니다. 수치적 위험성은 조건수가 판단한다"
image: "images/post/til-2026-09-01-banner.png"
categories: ["TIL"]
tags: ["mathematics", "python"]
type: "post"
nextp: ""
prevp: ""
---

## 가우스 소거법

연립방정식을 푸는 건 행을 더하고 빼서 위삼각 꼴로 만든 뒤, 맨 아래 식부터 되짚어 올라가는 일이다.

```text
2x₁ + 8x₂ + 2x₃ = 14
 x₁ + 6x₂ + 4x₃ = 15
2x₁ + 2x₂ + 3x₃ = 10
```

### 첨가행렬

계수행렬 `A` 오른쪽에 상수항 `b`를 붙여 `[A|b]`로 둔다. 이렇게 두면 행 연산 한 번이 방정식 하나에 통째로 적용돼서, 식을 따로 들고 다닐 필요가 없다.

```text
[ 2  8  2 | 14 ]
[ 1  6  4 | 15 ]
[ 2  2  3 | 10 ]
```

### 소거와 피벗팅

1열을 기준으로 아래 행에서 1행의 배수를 뺀다. 2행에서 1행의 `1/2`배를 빼면 `(0, 2, 3 | 8)`, 3행에서 1행을 빼면 `(0, −6, 1 | −4)`가 된다.

여기서 바로 2열로 넘어가지 않는다. 남은 행들 중 **2열의 절댓값이 가장 큰 행을 위로 올린다.** `|−6| > |2|`라 두 행을 맞바꾼다. 이게 부분 피벗팅(partial pivoting)이고, 최종 위삼각 행렬의 2행에 `2`가 아니라 `−6`이 앉아 있는 이유다.

```text
[ 2   8   2   | 14   ]
[ 0  −6   1   | −4   ]
[ 0   0  10/3 | 20/3 ]
```

### 후진 대입

| 단계 | 식 | 결과 |
|---|---|---|
| 3행 | `(10/3)x₃ = 20/3` | `x₃ = 2` |
| 2행 | `−6x₂ + x₃ = −4` | `x₂ = 1` |
| 1행 | `2x₁ + 8x₂ + 2x₃ = 14` | `x₁ = 1` |

```text
∴ x₁ = 1,  x₂ = 1,  x₃ = 2
```

### 피벗팅이 왜 정확도를 지키나

소거의 실제 계산은 `factor = U[r, col] / U[row, col]` 한 줄이다. 분모가 작으면 `factor`가 커지고, 큰 수를 빼는 과정에서 원래 값의 유효숫자가 날아간다.

절댓값이 가장 큰 행을 피벗으로 올려두면 `|factor| ≤ 1`이 보장된다. 피벗팅은 순서를 예쁘게 만드는 게 아니라 **나눗셈의 분모를 최대한 키워두는 장치**다.

```python
def gauss_eliminate(A, b, pivoting=True, verbose=False):
    A = np.array(A, dtype=np.float64)
    b = _as_vector(b)
    if A.shape[0] != b.shape[0]:
        raise ValueError("Row is not exact")
    U = np.hstack((A, b.reshape(-1, 1)))
    steps = [U.copy()]
    m, n = A.shape
    row = 0
    for col in range(n):
        if row >= m:
            break
        pivot = row
        if pivoting:
            pivot = row + np.argmax(np.abs(U[row:, col]))
        if abs(U[pivot, col]) < 1e-18:
            raise ValueError("0 on pivot")
        if pivot != row:
            U[[row, pivot]] = U[[pivot, row]]
            steps.append(U.copy())
        for r in range(row + 1, m):
            factor = U[r, col] / U[row, col]
            U[r, col:] -= U[row, col:] * factor
        steps.append(U.copy())
        row += 1

    root = []
    for row in range(n):
        foo = 0
        for col in range(row):
            foo += root[col] * U[n - row - 1, n - col - 1]
        root.append((U[-1 - row, -1] - foo) / U[-1 - row, -1 - row - 1])
    root.reverse()
    return root, steps
```

### 가우스-조던으로 역행렬

오른쪽에 `b` 대신 단위행렬을 붙이고, 아래쪽만이 아니라 위쪽까지 마저 소거해서 왼쪽을 단위행렬로 만든다. 그러면 오른쪽에 남는 게 `A⁻¹`이다.

```python
def inverse_gauss_jordan(A):
    A = np.array(A, dtype=np.float64)
    I = np.eye(A.shape[0])
    U = np.hstack((A, I))

    n, m = A.shape
    row = 0
    for col in range(n):
        if row >= m:
            break
        pivot = row + np.argmax(np.abs(U[row:, col]))
        if abs(U[pivot, col]) < 1e-18:
            raise ValueError("역행렬이 없습니다.")
        if pivot != row:
            U[[row, pivot]] = U[[pivot, row]]
        U[row] /= U[row, col]
        for r in range(m):
            if r == row:
                continue
            factor = U[r, col] / U[row, col]
            U[r, col:] -= U[row, col:] * factor
        row += 1
    return U[:, m:]
```

## rank

rank는 행이나 열의 개수가 아니라 **서로 독립인 식(정보)의 개수**다.

```text
 x +  y = 10
2x + 2y = 20
```

두 번째 식은 첫 번째 식의 2배일 뿐이라 새로운 정보가 하나도 없다. 식은 2개인데 rank는 1이다. 소거를 돌려보면 2행이 통째로 0이 되는 걸로 드러난다.

### 해의 판정 (미지수 `n`개)

| 조건 | 해 |
|---|---|
| `rank(A) = rank([A\|b]) = n` | 유일해 |
| `rank(A) = rank([A\|b]) < n` | 해가 무한히 많음 |
| `rank(A) ≠ rank([A\|b])` | 해가 없음 |

`A`만 보지 않고 `[A|b]`까지 같이 보는 게 핵심이다. 좌변끼리는 종속인데 우변이 그 관계를 안 따르면 식 자체가 모순이 된다.

```text
x + y = 2,   x + y = 3
```

좌변이 같은데 우변이 다르니 성립이 불가능하다. 이때 `rank(A) = 1`, `rank([A|b]) = 2`로 값이 갈린다.

## 행렬식

| 조건 | 의미 |
|---|---|
| `det(A) ≠ 0` | 역행렬 존재 (정칙, 유일해) |
| `det(A) = 0` | 역행렬 없음 |

### det의 크기로는 아무것도 판단할 수 없다

여기가 오늘 제일 크게 고쳐 잡은 부분이다. `det = 0`이 위험 신호니까 **det가 작으면 위험에 가깝다**고 읽고 있었는데, 그게 아니다.

`n × n` 행렬에 스칼라를 곱하면 det는 그 `n`제곱만큼 곱해진다.

```text
det(0.001A) = (0.001)³ det(A) = 1e-9 × det(A)
```

행렬의 성질은 하나도 안 변했는데 det만 `1e-9`배가 된다. 단위를 m에서 mm로 바꾸기만 해도 이런 일이 생긴다.

반례 두 개를 직접 찍어보면 확실해진다.

```python
C = 0.001 * np.eye(3)
np.linalg.det(C)     # 1.0e-09   det는 거의 0
np.linalg.cond(C)    # 1.0       완벽하게 안정적

B = np.array([[1., 1.], [1., 1.0001]])
np.linalg.det(B)     # 1.0e-04   det는 C보다 10만 배 크지만
np.linalg.cond(B)    # 40002.0   심하게 불안정
```

det가 더 작은 쪽이 오히려 멀쩡하다. **det는 0인지 아닌지만 말해주고, 얼마나 위험한지는 말해주지 않는다.**

## 조건수

### 눌림 현상

열벡터 `α`, `β`가 서로 가까워져 같은 방향으로 눌리면 두 벡터가 만드는 면적(부피)이 0이 되고, 그때 `det = 0`이 된다. `det = 0`은 차원이 무너지는 현상이다.

완전히 눌리지는 않았지만 거의 눌린 상태, 그게 ill-conditioned다.

### κ(A)

조건수는 **입력의 상대오차가 출력에서 몇 배로 증폭되는지의 상한**이다. 특잇값의 최대/최소 비로 정의된다.

```text
κ(A) = σ_max / σ_min

‖δx‖/‖x‖  ≤  κ(A) · ‖δb‖/‖b‖
```

위의 `B`로 실제로 재봤다. `b`를 `0.0035%`만 흔들었는데 해는 `(1, 1)`에서 `(0, 2)`로 통째로 바뀐다.

```python
b, db = np.array([2., 2.0001]), np.array([0., 0.0001])
np.linalg.solve(B, b)        # [1., 1.]
np.linalg.solve(B, b + db)   # [0., 2.]

# 상대오차 증폭률
# 입력 3.5e-05  ->  출력 1.0  ->  약 28285배 (κ = 40002 이내)
```

증폭률 28285가 `κ = 40002` 아래에 얌전히 들어간다. 조건수는 최악의 경우를 재는 상한이다.

### det과 결정적으로 다른 점

조건수는 스케일에 영향을 받지 않는다. 비율이라 분자 분모가 같이 줄기 때문이다.

```python
np.linalg.cond(A)          # 9.114982570092485
np.linalg.cond(0.001 * A)  # 9.114982570092486
```

det는 `1e-9`배가 됐던 바로 그 변환이다. **수치적 위험성은 det의 절대적 크기가 아니라 조건수로 판단해야 한다.**

### 절댓값 임계값 가드는 이걸 못 잡는다

위 코드의 `abs(U[pivot, col]) < 1e-18` 가드를 믿고 있었는데, 조건수가 `1e16`인 행렬을 그대로 통과시킨다.

```python
N = np.array([[1., 1.], [1., 1.0000000000000002]])
np.linalg.cond(N)        # 1.6e+16

inverse_gauss_jordan(N)  # 예외 없이 통과
# [[ 4.5e+15, -4.5e+15],
#  [-4.5e+15,  4.5e+15]]   전부 쓰레기값
```

피벗이 `1`이라 절댓값 기준은 아무 문제도 못 느낀다. 그래서 임계값은 절댓값이 아니라 **행렬 전체 크기 대비 상대값**으로 잡아야 한다. 원본 코드에 주석으로 남겨둔 그 줄이 이걸 하는 것이다.

```python
tol = max(m, n) * np.finfo(float).eps * max(1.0, float(np.max(np.abs(U))))
```

## 정리

1. 가우스 소거법은 `[A|b]`를 위삼각으로 만든 뒤 후진 대입으로 되짚어 올라가는 절차다. 첨가행렬은 행 연산 하나를 방정식 하나에 통째로 적용하려고 쓴다.
2. **부분 피벗팅은 정렬이 아니라 정확도 장치다.** 절댓값이 가장 큰 행을 피벗으로 올리면 `|factor| ≤ 1`로 묶여서 유효숫자가 덜 날아간다.
3. 가우스-조던은 오른쪽에 `b` 대신 단위행렬을 붙이고 위쪽까지 소거해서 역행렬을 얻는 것이다.
4. **rank는 독립인 식의 개수**다. 식이 2개라도 하나가 다른 하나의 배수면 rank는 1이다.
5. 해의 판정은 `A`만이 아니라 `[A|b]`까지 봐야 한다. 둘의 rank가 갈리면 모순이라 해가 없고, 같으면서 `n`보다 작으면 해가 무한히 많다.
6. `det = 0`은 열벡터가 같은 방향으로 눌려 차원이 무너진 상태다.
7. **det가 작다고 위험한 게 아니다.** 스칼라 배에 `n`제곱으로 반응해서, `0.001I`는 det가 `1e-9`인데 조건수는 1이다. det는 0이냐 아니냐만 말한다.
8. 위험도는 조건수 `κ(A) = σ_max / σ_min`가 판단한다. 입력 상대오차가 출력에서 증폭되는 배율의 상한이고, **스케일에 영향받지 않는다.**
9. 피벗의 **절댓값** 임계값으로는 ill-conditioned를 못 잡는다. 조건수 `1e16`짜리도 피벗이 1이면 그냥 통과한다. 임계값은 행렬 크기 대비 상대값이어야 한다 — 어제 그람-슈미트에 걸어둔 가드와 같은 이유다.
