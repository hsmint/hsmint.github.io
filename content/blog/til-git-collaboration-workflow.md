---
title: "[TIL] Git과 협업 워크플로우"
date: 2026-08-05T21:00:00+09:00
description: "KANT Level 1 · 4강 - Git 구조, merge와 rebase, 충돌 해결, PR 협업 사이클 핵심만 정리"
image: "images/post/post-2.jpg"
categories: ["TIL"]
tags: ["git", "github"]
type: "post"
nextp: ""
prevp: ""
---

## Git의 구조
- 커밋 = 스냅샷 하나 (diff가 아님), 부모 커밋을 가리킴
- 브랜치 = 특정 커밋을 가리키는 포인터.
- 분산형이라 로컬에 전체 이력이 있음. 네트워크는 push·pull·clone에만 필요

## 기본 흐름
```
작업 디렉터리 -(add)-> 스테이징 -(commit)-> 로컬 저장소 -(push)-> 원격
```
- `git status`, `git diff`로 먼저 확인하는 습관이 중요
- 커밋 메시지는 "무엇을, 왜"를 담고, 한 커밋 = 한 가지 일

## merge vs rebase
- **merge**: 병합 커밋을 만들어 이력을 그대로 보존
- **rebase**: 커밋을 새로 써서 이력을 한 줄로 정리
- **이미 push한(공유된) 커밋은 rebase 하지 않는다**

## 충돌 해결
1. 두 변경의 의도를 읽고 올바른 코드로 직접 고쳐 쓴다
2. `git add`로 해결 표시
3. `git commit` 또는 `git rebase --continue`로 마무리
- 해결 후엔 반드시 빌드·테스트 확인

## PR 협업 사이클
```
브랜치 생성 → 커밋 → 푸시 → PR → 코드 리뷰 → CI 통과 → main 병합
```
- 리뷰는 코드를 지적하는 것이지 사람을 지적하는 게 아님
- PR은 작게 쪼개야 제대로 리뷰됨
- CI가 push·PR마다 자동으로 빌드/테스트를 돌려 main을 보호

## 되돌리기
| 상황 | 명령 |
|---|---|
| 작업 디렉터리 수정 취소 | `git restore` |
| 스테이징 취소 | `git restore --staged` |
| 공유 커밋 되돌리기 | `git revert` |
| 로컬 커밋 되돌리기 | `git reset` |
| 커밋 복구 | `git reflog` |

> 브랜치는 짧게, 리뷰는 꼼꼼하게 — 충돌과 사고를 줄이는 가장 확실한 방법.
