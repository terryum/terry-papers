# ONTOLOGY.md — Paper Knowledge Graph 어휘

이 문서는 `posts/papers/<slug>/meta.json`의 `relations[]`, `papers/<slug>.json`의 `relations[]`, `knowledge-index.json`의 `knowledge_graph.edges`에서 사용하는 노드/엣지 어휘를 정의한다. **기존 코드의 임의 어휘를 학술 표준(CiTO/SPAR + Dublin Core)에 정렬**하는 것이 목적이며, 미래에 RDF/JSON-LD로 마이그레이션하더라도 손실 없도록 설계됐다.

## 네임스페이스

| Prefix | URI | 용도 |
|---|---|---|
| `cito` | `http://purl.org/spar/cito/` | CiTO — 인용/관계의 의미적 타입 (학술 표준) |
| `dcterms` | `http://purl.org/dc/terms/` | Dublin Core — 일반 메타 |
| `schema` | `https://schema.org/` | SEO/JSON-LD 호환 |
| `kg` | `https://terrypapers.local/kg#` | **로컬 확장** — CiTO에 없는 관계 타입 (이 KG 한정) |

JSON 파일에서는 prefix를 생략하고 **predicate 이름만** 사용한다 (e.g. `extends`, `usesMethodIn`). prefix는 이 문서에서 결정 근거로만 명시한다.

## 노드 타입

현재는 **단일 타입**:

| `node_type` | 의미 | 식별자 |
|---|---|---|
| `paper` | 학술 논문/기술 블로그/리포트 단위 | `slug` (e.g. `2505-dexumi`) |

향후 확장 여지(현재 도입하지 않음): `concept`, `author`, `method`, `dataset`. 도입 시 `meta.json`의 `node_type` 필드로 구분.

## 엣지 어휘 (Predicate)

### 표준 7종 + 로컬 2종

엣지에는 다음 9개 predicate만 사용한다. 그 외 값은 `validate-post.mjs`가 거부.

| predicate | 출처 | 의미 | 방향 | inverse | strength |
|---|---|---|---|---|---|
| `cites` | cito | A가 B를 명시적 인용 (참고문헌) | 방향 | `isCitedBy` | foundational |
| `extends` | cito | A가 B의 방법/모델을 직접 확장·개선 | 방향 | `isExtendedBy` | foundational |
| `usesMethodIn` | cito | A가 B에서 제안된 방법을 차용 | 방향 | `providesMethodFor` | foundational |
| `reviews` | cito | A가 B를 (벤치마크/실증 비교의 형태로) 비교 검토 | 무방향 (대칭) | `reviews` | direct |
| `critiques` | cito | A가 B의 한계·실패를 직접 비판하거나 보완 | 방향 | `isCritiquedBy` | direct |
| `sharesGoalWith` | **kg** | A와 B가 동일한 task/문제를 다룸 (방법은 다를 수 있음) | 무방향 | `sharesGoalWith` | direct |
| `sharesTopicWith` | **kg** | A와 B가 주제/concept만 공유 (가장 약한 관계) | 무방향 | `sharesTopicWith` | tangential |

### 로컬 확장 두 가지의 정당성

CiTO 41개 sub-property를 검토했으나 **사용자의 실제 사용 패턴**에 정확히 맞는 표준 predicate가 없는 두 경우를 로컬 확장으로 추가:

- `kg:sharesGoalWith` — 동일 task를 다른 방법으로 푸는 두 논문 관계. CiTO `cito:agreesWith`는 결론 일치를 의미하므로 부적절. `dcterms:subject`는 노드가 아닌 리소스 속성이라 엣지로 부적절.
- `kg:sharesTopicWith` — 주제 겹침 정도의 약한 연관. 가장 자주 쓰는 관계지만 CiTO에는 표준화된 어휘 없음. `dcterms:relation`은 너무 일반적.

### `compares_with` 분기 처리

기존 `compares_with`는 두 의미가 섞여 있었다:
1. 같은 벤치마크에서 비교 → `reviews`
2. 다른 방법으로 같은 task를 풀어 비교 → `sharesGoalWith`

마이그레이션 시 사용자가 원본 의도를 모르는 경우 기본값은 `reviews` (더 강한 관계 보존). 이후 수동으로 `sharesGoalWith`로 다운그레이드 가능.

## 엣지 양방향성 정책

`knowledge-index.json`의 `knowledge_graph.edges`에는 forward 1개 + inverse 1개를 모두 저장한다. **단, 기존의 무의미한 `reverse_<type>`을 폐기하고 진짜 inverse predicate**(`isExtendedBy` 등)를 사용한다.

대칭 관계(`reviews`, `sharesGoalWith`, `sharesTopicWith`)는 한 번만 저장하고 `directional: false`로 표시한다.

엣지 객체 형식:

```json
{
  "from": "2505-dexumi",
  "to": "2402-umi-universal-manipulation-interface",
  "predicate": "extends",
  "directional": true,
  "inverse_of": null,
  "strength": "foundational"
}
```

inverse 엣지의 경우:

```json
{
  "from": "2402-umi-universal-manipulation-interface",
  "to": "2505-dexumi",
  "predicate": "isExtendedBy",
  "directional": true,
  "inverse_of": "extends",
  "strength": "foundational"
}
```

대칭 엣지:

```json
{
  "from": "2505-dexumi",
  "to": "2407-stretchable-glove-hand-pose",
  "predicate": "reviews",
  "directional": false,
  "strength": "direct"
}
```

## meta.json의 `relations[]` 형식

소스 오브 트루스(`posts/papers/<slug>/meta.json`)에서는 forward edge만 적는다 — inverse는 `export-knowledge.mjs`가 자동 생성.

```json
{
  "relations": [
    { "target": "2402-umi-universal-manipulation-interface", "type": "extends" },
    { "target": "2407-stretchable-glove-hand-pose", "type": "reviews" },
    { "target": "2501-dexforce-force-informed-actions", "type": "sharesGoalWith" }
  ]
}
```

`type` 값은 위 9종 중 하나여야 한다. inverse predicate(`isExtendedBy` 등)는 meta.json에 직접 쓰지 않는다 — forward로 뒤집어 표현하거나, 양방향 관계라면 대칭 predicate 사용.

## strength → traversal 가중치

`/paper-search`의 graph traversal에서 사용:

| strength | edge weight | 의미 |
|---|---|---|
| foundational | 1.0 | 학문적 직접 계보 (cites/extends/usesMethodIn) |
| direct | 0.6 | 명시적 비교·비판·동일 task (reviews/critiques/sharesGoalWith) |
| tangential | 0.3 | 약한 주제 공유 (sharesTopicWith) |

경로 길이 d에 대해 `weight^d`로 누적. 이 가중치는 `scripts/export-knowledge.mjs`의 `assignRelationStrength()`가 결정한다.

## 마이그레이션 매핑 (현재 → 새 어휘)

`scripts/migrate-relation-vocab.mjs`가 사용:

| 현재 `type` | 새 `type` | 비고 |
|---|---|---|
| `related` | `sharesTopicWith` | 가장 흔함 (61건). 약한 주제 공유. |
| `compares_with` | `reviews` | 21건. 기본값. 일부 수동 재분류 가능. |
| `builds_on` | `extends` | 9건. CiTO 표준명. |
| `addresses_task` | `sharesGoalWith` | 4건. 동일 task, 다른 접근. |
| `uses_method` | `usesMethodIn` | 3건. CiTO camelCase로 정렬. |
| `extends` | `extends` | 코드에만 정의, 실사용 0. 그대로. |
| `fills_gap_of` | `critiques` | 코드에만 정의, 실사용 0. 한계 보완 의미 보존. |
| `cites` | `cites` | 신규 도입. 마이그레이션 시 추가는 안 함. |

## JSON Schema와의 관계

`schemas/paper-meta.schema.json`의 `relations.items.properties.type.enum`이 위 9개 forward predicate를 강제한다. inverse predicate는 자동 생성물이므로 schema 검증 대상이 아니다.

## SEO/JSON-LD 호환 (미도입, 참고)

향후 각 페이지에 `<script type="application/ld+json">`으로 `schema:ScholarlyArticle` + `cito:cites` 인용을 inline 삽입할 수 있다. 현재 어휘는 그대로 schema.org/CiTO URI로 매핑 가능 — 마이그레이션 비용 0.
