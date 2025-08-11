---
layout: post
title: 분산 시스템 환경을 위한 Exponential Backoff에 Jitter를 더한 Retry 전략
date: 2025-08-11 00:00:00 +0900
description: 분산 시스템 환경을 위한 Exponential Backoff에 Jitter를 더한 Retry 전략
img: 2025-08-11/full_jitter.png
tags: [분산 시스템, 메시징 서비스, AWS, SQS, SNS, backoff, retry, exponential backoff]
---

# 개요

분산 시스템에서는 서버, 네트워크, 소프트웨어, 운영자 실수 등 다양한 요인으로 인해 장애가 발생할 수 있으며, 완전 무결한 시스템 구축은 불가능합니다.

- **시간 제한**: 요청 대기 시간을 제한해 리소스 고갈을 방지하고, 적절한 시간 값을 설정하는 것이 핵심입니다. 지나치게 높거나 낮은 값은 각각 리소스 낭비와 불필요한 재시도 폭증을 초래할 수 있습니다.
- **재시도**: 부분적·일시적 장애에 효과적이지만, 무분별한 재시도는 부하를 악화시키고 복구를 지연시킬 수 있습니다. 멱등성을 보장하는 API 설계, 재시도 횟수 제한, 단일 레이어 재시도 전략 등이 필요합니다.
- **백오프**: 재시도 간 간격을 점진적으로 늘려(지수 백오프) 부하를 완화합니다. 최대 대기 시간 제한을 두고, 토큰 버킷 같은 스로틀링 기법을 활용해 재시도 폭주를 방지합니다.
- **지터**: 재시도 시점을 무작위로 분산시켜 대량의 요청이 한꺼번에 몰리는 현상을 방지합니다. 예약 작업이나 주기적인 워크로드에도 지터를 적용하면 서버 자원 사용을 고르게 분산시킬 수 있습니다.

결론적으로, 분산 시스템의 고가용성을 위해서는 **시간 제한으로 장애의 확산을 차단하고, 재시도/백오프/지터를 적절히 조합해 부하를 제어하는 전략**이 필요합니다.  
특히 재시도는 신중하게 설계·운영해야 하며, 잘못된 재시도는 문제를 개선하기보다 악화시킬 수 있다는 점을 인지해야 합니다.

## N명의 클라이언트 경쟁

첫 번째 시뮬레이션에서는 여러 클라이언트가 동시에 같은 자원을 요청하는 상황을 가정합니다. 경쟁이 일어나면 **한 번에 오직 한 명의 클라이언트만 요청에 성공**하고, 나머지는 실패합니다.
이 경우, 모든 클라이언트가 순서대로 성공하려면 라운드를 여러 번 반복해야 합니다.

예를 들어,

- 클라이언트가 5명이라면, 첫 라운드에 1명이 성공하고 4명은 대기합니다.
- 다음 라운드에 또 1명이 성공하고 3명이 남습니다.
- 이런 식으로 매 라운드마다 한 명씩만 성공하므로, 5명이 모두 성공하려면 총 5라운드가 필요합니다.

라운드 수가 곧 완료 시간과 비례하므로, 클라이언트 수(N)가 증가할수록 완료 시간도 N에 비례해 선형적으로 증가하게 됩니다.

## 단순 재시도 방법 적용(N번 요청, 대기 없음)

```typescript
async function retry(send: () => Promise<any>, maxRetries = 3) {
  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await send();
    } catch (err) {
      if (attempt === maxRetries) throw err;
    }
  }
}
```

![1]({{site.baseurl}}/assets/img/2025-08-11/1.png)

문제는 실패한 클라이언트들도 매 라운드마다 계속 요청을 시도한다는 점입니다.

- **첫 라운드**: N명이 요청 → 1명 성공, N-1명 실패
- **두 번째 라운드**: 남은 N-1명이 요청 → 1명 성공, N-2명 실패
- **세 번째 라운드**: 남은 N-2명이 요청 → 1명 성공, N-3명 실패

...이런 식으로 마지막 1명이 성공할 때까지 계속됩니다.

![2]({{site.baseurl}}/assets/img/2025-08-11/2.png)

결론은 위와 같이 **총 작업량이 N²에 비례하게 증가**합니다.

즉, 클라이언트 수가 늘어날수 록 시스템이 처리해야할 요청이 기하급수적으로 증가하게됩니다.  
이 때문에 경쟁이 심한 환경에서는 단순히 시간이 오래 걸리는 것뿐만 아니라, 시스템 리소스 소비량도 훨씬 더 가파르게 증가합니다.

## Exponential backoff 적용

```typescript
const sleep = (ms: number) => new Promise(r => setTimeout(r, ms));

function backoff(attempt: number, initialDelayMs = 1000, maxDelayMs = 10000) {
  return Math.min(initialDelayMs * 2 ** attempt, maxDelayMs);
}

async function retryBackoff(send: () => Promise<any>, maxRetries = 3) {
  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await send();
    } catch (err) {
      if (attempt === maxRetries) throw err;
      await sleep(backoff(attempt));
    }
  }
}
```

처음에는 N명의 클라이언트가 동시에 달려듭니다. 한 번 시도에서 한 명만 성공하니, 다음 라운드에는 N-1명이 다시 달려듭니다. 이런 식으로 계속 경쟁하면 매번 많은 요청이 한꺼번에 몰려서 리소스 낭비가 심해집니다.

그래서 재시도 속도를 늦추는 방법이 필요합니다.
가장 흔한 방법이 제한 지수 백오프인데,

- 실패할 때마다 대기 시간을 2배, 4배, 8배, ... 이렇게 늘려가고
- 너무 길어지지 않도록 최대 대기 시간(maxDelayMs)을 설정합니다.

결과적으로, 실패 직후에는 빨리 재시도하고, 여러 번 실패하면 점점 느리게 재시도해서 서버 부담을 줄입니다.

![3]({{site.baseurl}}/assets/img/2025-08-11/3.png)

백오프를 적용하였지만 약간 도움이 되지만 문제를 해결하지는 못하는 것으로 나타났습니다. 클라이언트 작업량은 약간만 줄었습니다.

문제를 파악하는 가장 좋은 방법은 기하급수적으로 줄어드는 호출이 발생하는 시간을 살펴보는 것입니다.

![4]({{site.baseurl}}/assets/img/2025-08-11/4.png)

Exponential backoff가 작동하고 있다는 것은 분명합니다. 호출 빈도가 점점 줄어들고 있습니다. **문제는 여전히 호출이 뭉쳐 있다는 것**입니다.  
매 라운드마다 경쟁하는 클라이언트 수를 줄이는 대신, **"클라이언트가 경쟁하지 않는 시간대를 만들었습니다."**  
네트워크 지연의 자연스러운 변동성으로 인해 약간의 분산이 발생했지만, 경합은 크게 줄어들지 않았습니다.

## 백오프 + 지터 적용

```typescript
const sleep = (ms: number) => new Promise(r => setTimeout(r, ms));

function fullJitter(attempt: number, initialDelayMs = 1000, maxDelayMs = 10000) {
  const maxDelay = Math.min(initialDelayMs * 2 ** attempt, maxDelayMs);
  return Math.floor(Math.random() * maxDelay);
}

async function retryBackoffJitter(send: () => Promise<any>, maxRetries = 3) {
  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await send();
    } catch (err) {
      if (attempt === maxRetries) throw err;
      await sleep(fullJitter(attempt));
    }
  }
}
```

해결책은 백오프를 제거하는 것이 아니라 지터를 추가하는 것입니다.  
하지만, 지터가 반직관적인 생각처럼 보일 수 있습니다만 무작위성을 추가하여 시스템 성능을 향상시키려는 것입니다.

위에 **"클라이언트가 경쟁하지 않는 시간대를 만들었습니다."**에 보면 변동성(jitter)을 주어 그 간격을 채워 보완하는 개념이라고 생각하면 좋을 것 같습니다.(스파이크를 거의 일정한 속도로 분산)  
아래의 시계열은 지터의 장점을 잘 보여줍니다. 지터를 추가하는 것은 sleep 함수를 약간 변경하는 것입니다.

![5]({{site.baseurl}}/assets/img/2025-08-11/5.png)

시계열이 훨씬 좋아 보입니다. 갭이 사라졌고, 초기 급증 이후 거의 일정한 통화율을 보이고 있습니다. 총 통화량에도 큰 영향을 미쳤습니다.

![6]({{site.baseurl}}/assets/img/2025-08-11/6.png)

경쟁 클라이언트가 100명인 경우, 호출 횟수를 절반 이상 줄였습니다. 또한 지터링 없는 지수 백오프 방식과 비교했을 때 완료 시간도 크게 단축되었습니다.

![7]({{site.baseurl}}/assets/img/2025-08-11/7.png)

위 그래프에서 Exponential 방식의 Completion Time이 시간이 지날수록(클라이언트 수가 많아질수록) 급격히 길어지는 이유는 백오프 간격이 기하급수적으로 커지기 때문입니다.

풀어서 설명하면:

- Exponential Backoff는 실패할 때마다 대기 시간을 2배, 4배, 8배, ...로 늘립니다.
- 클라이언트가 많을수록 경쟁이 치열해지고, 동시에 실패하는 요청 수도 많아집니다.
- 각 클라이언트는 여러 번 실패를 거듭하면서 대기 시간이 급격히 길어집니다.
- 이렇게 되면 모든 요청이 완료되기까지의 **총 소요 시간(Completion Time)**이 크게 늘어납니다.

반면 Full Jitter(Backoff + Full Jitter)는 매번 대기 시간을 0~최대값 사이에서 랜덤으로 뽑기 때문에,

- 일부 클라이언트는 짧은 대기 후 재시도하게 되고
- 요청들이 더 균등하게 분산되며
- 전체 Completion Time이 짧게 유지됩니다.

None은 대기 없이 즉시 재시도하기 때문에 Completion Time 자체는 짧게 나올 수 있지만,  
그만큼 서버와 네트워크 부하는 심하고(부하로 인한 잦은 클라이언트 에러를 마주할 가능성이 높음), 실제 운영 환경에서는 오히려 장애를 더 악화시킬 수 있습니다.

## 참고

- [https://aws.amazon.com/ko/builders-library/timeouts-retries-and-backoff-with-jitter](https://aws.amazon.com/ko/builders-library/timeouts-retries-and-backoff-with-jitter)
- [https://aws.amazon.com/ko/blogs/architecture/exponential-backoff-and-jitter](https://aws.amazon.com/ko/blogs/architecture/exponential-backoff-and-jitter)
