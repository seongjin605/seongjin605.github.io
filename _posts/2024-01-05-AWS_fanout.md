---
layout: post
title: LocalStack으로 구현하는 AWS SNS 메시지 필터링 시스템
date: 2024-01-05 00:00:00 +0900
description: LocalStack과 AWS SNS MessageAttributes를 활용해 색상별, 조건별로 메시지를 필터링하는 Fan-out 패턴 구현하기
img: 2024-01-05/fanout.png
tags: [AWS, SNS, SQS, LocalStack, MessageFiltering, Microservices]
---

# LocalStack으로 구현하는 AWS SNS 메시지 필터링 시스템

마이크로서비스 아키텍처에서 서비스 간 메시지 전달은 핵심적인 요소입니다. 특히 하나의 이벤트를 여러 서비스에 전달하되, 각 서비스가 필요한 메시지만 받도록 하는 것은 효율성과 성능 측면에서 매우 중요합니다.

오늘은 AWS SNS의 메시지 필터링 기능을 활용해 색상별, 조건별로 메시지를 라우팅하는 시스템을 LocalStack으로 구현해보겠습니다.

## 프로젝트 목표

다음과 같은 요구사항을 가진 메시지 필터링 시스템을 구현합니다:

1. **ColorQueue**: 모든 색상(blue, red, yellow, green) 메시지 수신
2. **BlueYellowQueue**: blue와 yellow 메시지만 수신
3. **RedQueue**: red 메시지만 수신
4. **GreenHighQueue**: green 메시지 중 quantity 값이 100 이상인 메시지만 수신
5. **GreenLowQueue**: green 메시지 중 quantity 값이 100 미만인 메시지만 수신

## 기술 스택

- **AWS SNS**: 메시지 발행 및 필터링
- **AWS SQS**: 메시지 큐잉
- **LocalStack**: 로컬 AWS 환경 시뮬레이션
- **Node.js**: 메시지 발송 및 수신 로직
- **AWS SDK v3**: JavaScript용 AWS 클라이언트

## 아키텍처 설계

```
SNS Topic (sample-topic)
├── ColorQueue (모든 색상)
├── BlueYellowQueue (blue, yellow)
├── RedQueue (red only)
├── GreenHighQueue (green + quantity 값≥100)
└── GreenLowQueue (green + quantity 값<100)
```

## LocalStack 환경 설정

먼저 LocalStack 환경을 구성하고 필요한 큐와 구독을 설정합니다.

### localstack.sh

[localstack.sh](https://github.com/seongjin605/aws-fanout-pattern/blob/main/localstack.sh)

## 핵심 개념: SNS FilterPolicy

AWS SNS의 필터 정책은 구독자가 받고 싶은 메시지를 선택적으로 수신할 수 있게 해주는 기능입니다.

### 1. 기본 문자열 필터링

```json
{ "color": ["blue", "yellow"] }
```

### 2. 숫자 조건 필터링

> 약속한 quantity라는 값을 지정하여 필터링을 설정합니다.

```json
{ "color": ["green"], "quantity": [{ "numeric": [">=", 100] }] }
```

이 필터는 메시지 속성의 `quantity` 값이 100 이상인 green 메시지만 허용합니다.

### 주요 숫자 연산자

- `=`: 같음
- `>`: 초과
- `>=`: 이상
- `<`: 미만
- `<=`: 이하

## 메시지 발송 및 수신 로직

### 핵심 기능

#### 1. MessageAttributes를 포함한 SNS 메시지 발송

SNS 메시지를 발송할 때는 `MessageAttributes`를 통해 필터링에 사용할 속성들을 함께 전송합니다. 이 속성들은 메시지 본문과는 별개로 관리되어 필터링 성능을 향상시킵니다.

핵심은 `color`와 `quantity` 속성을 MessageAttributes에 포함시키는 것입니다:

```javascript
const messageAttributes = {
  color: {
    DataType: 'String',
    StringValue: color
  }
};

// quantity가 제공되면 MessageAttributes에 추가
if (quantity) {
  messageAttributes.quantity = {
    DataType: 'Number',
    StringValue: quantity.toString()
  };
}
```

이렇게 설정된 속성들은 SNS 필터 정책에서 조건으로 사용됩니다.

#### 2. 향상된 SQS 메시지 수신

SQS에서 메시지를 수신할 때는 `MessageAttributeNames: ['All']` 옵션을 사용하여 모든 메시지 속성을 함께 가져옵니다. 이를 통해 어떤 속성으로 필터링되었는지 확인할 수 있습니다.([AWS Request Parameters 참조](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/APIReference/API_ReceiveMessage.html#API_ReceiveMessage_RequestParameters))

수신된 메시지는 SNS를 통해 전달된 것이므로 JSON 형태로 파싱하여 실제 메시지 내용과 속성들을 확인합니다:

```javascript
const snsMessage = JSON.parse(message.Body);
console.log(`실제 메시지: ${snsMessage.Message}`);

// MessageAttributes 출력
if (snsMessage.MessageAttributes) {
  Object.entries(snsMessage.MessageAttributes).forEach(([key, value]) => {
    console.log(`  - ${key}: ${value.Value} (타입: ${value.Type})`);
  });
}
```

이를 통해 각 큐가 올바른 필터 조건으로 메시지를 수신했는지 검증할 수 있습니다.

## 테스트 시나리오

필터링 시스템의 정확성을 검증하기 위해 다양한 조건의 메시지를 전송합니다. 총 8개의 테스트 메시지를 사용하여 각 큐의 필터링 로직을 검증합니다:

- **기본 색상 메시지**: blue, red, yellow (각 1개씩) ➡️ 메시지 **3개** 전송
- **Green 메시지 (quantity 값별)**: quantity=50(적은 값), quantity=150(큰 값), quantity=100(경계값) ➡️ 메시지 **3개** 전송
- **Blue/Yellow & quantity 조합 메시지**: blue(quantity=75), yellow(quantity=200) ➡️ 메시지 **2개** 전송

이러한 다양한 조합을 통해 각 큐의 필터 조건이 정확히 작동하는지 확인할 수 있습니다.

## 테스트 결과

실제 테스트를 실행한 결과, 필터링이 정확하게 작동함을 확인했습니다:
![color_fanout]({{site.baseurl}}/assets/img/2024-01-05/color_fanout.png)

| 큐 이름             | 예상 메시지 수 | 수신할 메시지           | 필터 조건                                                 |
| ------------------- | -------------- | ----------------------- | --------------------------------------------------------- |
| **ColorQueue**      | 8개            | 모든 색상 메시지        | `{"color":["blue","red","yellow","green"]}`               |
| **BlueYellowQueue** | 4개            | blue, yellow 메시지만   | `{"color":["blue","yellow"]}`                             |
| **RedQueue**        | 1개            | red 메시지만            | `{"color":["red"]}`                                       |
| **GreenHighQueue**  | 2개            | green + quantity 값≥100 | `{"color":["green"],"quantity":[{"numeric":[">=",100]}]}` |
| **GreenLowQueue**   | 1개            | green + quantity 값<100 | `{"color":["green"],"quantity":[{"numeric":["<",100]}]}`  |

### 상세 결과 분석

1. **ColorQueue**: 8개 메시지 모두 수신
2. **BlueYellowQueue**: Blue(2개) + Yellow(2개) = 4개 수신
3. **RedQueue**: Red 메시지 1개만 수신
4. **GreenHighQueue**: Green(quantity=150), Green(quantity=100) = 2개 수신
5. **GreenLowQueue**: Green(quantity=50) = 1개 수신

## 실행 방법

프로젝트를 실행하는 방법은 다음과 같습니다:

1. **환경 설정**: `./localstack.sh` 명령어로 LocalStack 환경을 구성합니다.
2. **전체 테스트**: `node fanout.js`로 모든 큐를 정리한 후 전체 테스트를 실행합니다.
3. **특정 큐 확인**: `node fanout.js ColorQueue`와 같이 특정 큐만 확인할 수 있습니다.
4. **큐 정리 후 확인**: `node fanout.js ColorQueue clear`로 특정 큐를 정리한 후 확인합니다.
5. **전체 정리**: `node fanout.js clear`로 모든 큐를 정리합니다.

이러한 다양한 실행 옵션을 통해 개발 과정에서 필요한 부분만 선택적으로 테스트할 수 있습니다.

## 주요 학습 포인트

### 1. MessageAttributes의 중요성

SNS의 MessageAttributes는 필터링의 핵심입니다. 메시지 본문이 아닌 속성으로 필터링하므로 성능이 우수합니다.

### 2. 숫자 조건 필터링

`{"numeric":[">=",100]}` 형태로 메시지 속성의 숫자 값에 대한 복잡한 조건도 설정 가능합니다.

### 3. Fan-out 패턴의 효율성

하나의 메시지를 여러 구독자에게 전달하되, 각자 필요한 것만 받을 수 있어 효율적입니다.

### 4. LocalStack의 활용

실제 AWS 비용 없이 로컬에서 완전한 테스트 환경을 구축할 수 있습니다.

## 실제 운영 시 고려사항

1. **에러 핸들링**: DLQ(Dead Letter Queue) 설정
2. **모니터링**: CloudWatch 메트릭 활용
3. **비용 최적화**: 필터링으로 불필요한 처리 감소
4. **확장성**: 새로운 조건 추가 시 기존 구독자에 영향 없음

## 마무리

AWS SNS의 메시지 필터링 기능을 활용하면 복잡한 메시지 라우팅 로직을 간단하고 효율적으로 구현할 수 있습니다. 특히 마이크로서비스 환경에서 서비스 간 느슨한 결합을 유지하면서도 정확한 메시지 전달이 가능합니다.

이번 실습을 통해 LocalStack을 활용한 로컬 개발 환경 구축부터 실제 필터링 테스트까지 전 과정을 경험해볼 수 있었습니다. 실제 프로덕션 환경에서도 동일한 원리로 적용할 수 있으니, 여러분의 프로젝트에도 활용해보시기 바랍니다!

---

_전체 소스코드는 [GitHub 저장소](https://github.com/seongjin605/aws-fanout-pattern/tree/main)에서 확인하실 수 있습니다._
