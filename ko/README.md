# PHP의 예제 별 단위 테스트 팁

<a name="introduction"></a>
## 소개

이 글은 https://github.com/sarven/unit-testing-tips 를 한국어로 번역한 글입니다.<br><br>이 시대에 단위 테스트 작성의 이점은 엄청납니다. 최근에 시작된 대부분의 프로젝트에는 단위 테스트가 포함되어 있다고 생각합니다. 비즈니스 로직이 많은 엔터프라이즈 애플리케이션에서 단위 테스트는 가장 중요한 테스트입니다. 구현체를 빠르고 정확하게 즉시 확인할 수 있기 때문입니다. 그러나 가끔 프로젝트에서 테스트가 문제를 일으키기도 합니다. 따라서 이러한 테스트의 이점은 좋은 단위 테스트가 있을 때만 크게 효과를 볼 수 있습니다. 이 예제에서는 좋은 단위 테스트를 작성하기 위해 해야 할 일에 대한 몇 가지 팁을 공유하려고합니다.

## 목차

1. [소개](#introduction)
2. [테스트 더블](#test-doubles)
3. [이름 짓기](#naming)
4. [AAA 패턴](#aaa-pattern)
5. [부모 객체](#object-mother)
6. [매개 변수화 된 테스트](#parameterized-test)
7. [두 학교의 단위 테스트](#two-schools-of-unit-testing)
    - [Classical](#classical)
    - [Mockist](#mockist)
    - [의존성](#dependencies)
8. [Mock vs Stub](#mock-vs-stub)
9. [세 가지 스타일의 단위 테스트](#three-styles-of-unit-testing)
    - [Output](#output)
    - [State](#state)
    - [Communication](#communication)
10. [기능적 아키텍처 및 테스트](#functional-architecture-and-tests)
11. [관찰 가능한 동작과 구현 세부 정보](#observable-behavior-vs-implementation-details)
12. [행동 단위](#unit-of-behavior)
13. [겸손한 패턴](#humble-pattern)
14. [사소한 테스트](#trivial-test)
15. [깨지기 쉬운 테스트](#fragile-test)
16. [테스트 픽스처](#test-fixtures)
17. [일반적인 테스트의 안티 패턴](#general-testing-anti-patterns)
    - [비공개 상태 노출](#exposing-private-state)
    - [도메인 정보 유출](#leaking-domain-details)
    - [구체적인 클래스 모킹-mocking](#mocking-concrete-classes)
    - [비공개 메서드 테스트](#testing-private-methods)
    - [휘발성 의존성으로서의 시간](#time-as-a-volatile-dependency)
18. [100 % 테스트 커버리지가 목표가되어서는 안됩니다.](#100-test-coverage-shouldnt-be-the-goal)
19. [추천 도서](#recommended-books)

<a name="test-doubles"></a>
## 테스트 더블

테스트 더블은 테스트에 사용되는 가짜 의존성입니다.

### 스텁

#### 더미

더미는 아무것도하지 않는 단순한 구현입니다.

```php
final class Mailer implements MailerInterface
{
    public function send(Message $message): void
    {
    }
}
```

#### Fake

Fake는 원래 동작을 시뮬레이션하기위한 단순화 된 구현입니다.

```php
final class InMemoryCustomerRepository implements CustomerRepositoryInterface
{
    /**
     * @var Customer[]
     */
    private array $customers;

    public function __construct()
    {
        $this->customers = [];
    }

    public function store(Customer $customer): void
    {
        $this->customers[(string) $customer->id()->id()] = $customer;
    }

    public function get(CustomerId $id): Customer
    {
        if (!isset($this->customers[(string) $id->id()])) {
            throw new CustomerNotFoundException();
        }

        return $this->customers[(string) $id->id()];
    }

    public function findByEmail(Email $email): Customer
    {
        foreach ($this->customers as $customer) {
            if ($customer->getEmail()->isEqual($email)) {
                return $customer;
            }
        }

        throw new CustomerNotFoundException();
    }
}
```

#### 스텁-stub

스텁은 하드 코딩 된 동작이 있는 가장 간단한 구현입니다.

```php
final class UniqueEmailSpecificationStub implements UniqueEmailSpecificationInterface
{
    public function isUnique(Email $email): bool
    {
        return true;
    }
}
```

```php
$specificationStub = $this->createStub(UniqueEmailSpecificationInterface::class);
$specificationStub->method('isUnique')->willReturn(true);
```

### 모크

#### 스파이

스파이는 특정 동작을 확인하기위한 구현입니다.

```php
final class Mailer implements MailerInterface
{
    /**
     * @var Message[]
     */
    private array $messages;
    
    public function __construct()
    {
        $this->messages = [];
    }

    public function send(Message $message): void
    {
        $this->messages[] = $message;
    }

    public function getCountOfSentMessages(): int
    {
        return count($this->messages);
    }
}
```

#### 목-mock

mock은 공동 작업자의 호출를 확인하기 위해 설정하는 모조체입니다.

```php
$message = new Message('test@test.com', 'Test', 'Test test test');
$mailer = $this->createMock(MailerInterface::class);
$mailer
    ->expects($this->once())
    ->method('send')
    ->with($this->equalTo($message));
```

: exclamation : 들어오는 상호 작용을 확인하려면 스텁을 사용하고, 나가는 상호 작용을 확인하려면 mock을 사용하십시오. 더보기 : [Mock vs Stub](#mock-vs-stub)

<a name="naming"></a>
## 이름 짓기

:heavy_minus_sign: 좋지 않음 :

```php
public function test(): void
{
    $subscription = SubscriptionMother::new();

    $subscription->activate();

    self::assertSame(Status::activated(), $subscription->status());
}
```

:heavy_check_mark: **테스트 대상을 명시적으로 지정**

```php
public function sut(): void
{
    // sut = System under test
    $sut = SubscriptionMother::new();

    $sut->activate();

    self::assertSame(Status::activated(), $sut->status());
}
```

:heavy_minus_sign: 좋지 않음 :

```php
public function it_throws_invalid_credentials_exception_when_sign_in_with_invalid_credentials(): void
{

}

public function testCreatingWithATooShortPasswordIsNotPossible(): void
{

}

public function testDeactivateASubscription(): void
{

}
```

:heavy_check_mark: 더 좋음 :

- **밑줄을 사용하면 가독성이 향상됩니다.**
- **이름은 구현이 아닌 동작을 설명해야합니다.**
- **기술적인 키워드없이 이름을 사용하십시오. 프로그래머가 아닌 사람도 읽을 수 있어야합니다.**

```php
public function sign_in_with_invalid_credentials_is_not_possible(): void
{

}

public function creating_with_a_too_short_password_is_not_possible(): void
{

}

public function deactivating_an_activated_subscription_is_valid(): void
{

}

public function deactivating_an_inactive_subscription_is_invalid(): void
{

}
```

:information_source: 동작을 설명하는 것은 도메인 시나리오를 테스트하는 데 중요합니다. 코드가 유틸리티 코드라면 덜 중요합니다.

:question: 프로그래머가 아닌 사람이 단위 테스트를 읽는 것이 왜 유용할까요?

복잡한 도메인 로직이 있는 프로젝트의 경우 이 로직은 모든 사람에게 매우 명확해야 하므로 테스트는 기술적 키워드없이 도메인 세부 사항을 설명하면, 이러한 테스트와 같은 언어로 비즈니스적인 대화를 할 수 있습니다.

도메인과 관련된 모든 코드는 기술적인 세부 사항을 작성하지 않아야 합니다. 그렇지 않으면 프로그래머가 아닌 사람은 이 테스트를 읽을 수 없습니다. 그러면 도메인에 대해 이야기하고 싶을때, 테스트가  도메인이 하는 일이 무엇인지 아는 데 유용합니다. 기술적인 세부 사항을 설명하지마세요(예를 들어 null반환, 예외 발생 등). 이러한 종류의 정보는 도메인과 관련이 없으므로 이러한 키워드를 사용하면 안됩니다.

<a name="aaa-pattern"></a>
## AAA 패턴

일반적으로 Given, When, Then 이라고도 합니다.

:heavy_check_mark: 테스트의 세 섹션을 분리하십시오.

- **Arrange** : 테스트중인 시스템을 원하는 상태로 만듭니다. 의존성, 인수를 준비하고 마지막으로 SUT를 구성합니다.
- **Act** : 테스트 된 요소를 호출합니다.
- **Assert** : 결과, 최종 상태 또는 공동 작업자와의 커뮤니케이션을 확인합니다.

```php
public function aaa_pattern_example_test(): void
{
    //Arrange|Given
    $sut = SubscriptionMother::new();

    //Act|When
    $sut->activate();

    //Assert|Then
    self::assertSame(Status::activated(), $sut->status());
}
```

<a name="object-mother"></a>
## 부모 객체

이 패턴은 몇 가지 테스트에서 재사용 할 수있는 특정 개체를 만드는 데 도움이됩니다. 그 때문에 정렬 섹션이 간결하고 전체적으로 테스트가 더 읽기 쉬워집니다.

```php
final class SubscriptionMother
{
    public static function new(): Subscription
    {
        return new Subscription();
    }

    public static function activated(): Subscription
    {
        $subscription = new Subscription();
        $subscription->activate();
        return $subscription;
    }

    public static function deactivated(): Subscription
    {
        $subscription = self::activated();
        $subscription->deactivate();
        return $subscription;
    }
}
```

```php
final class ExampleTest
{
    public function example_test_with_activated_subscription(): void
    {
        $activatedSubscription = SubscriptionMother::activated();

        // do something

        // check something
    }

    public function example_test_with_deactivated_subscription(): void
    {
        $deactivatedSubscription = SubscriptionMother::deactivated();

        // do something

        // check something
    }
}
```

<a name="parameterized-test"></a>
## 매개 변수화 된 테스트

매개 변수화 된 테스트는 코드를 반복하지 않고 많은 매개 변수로 SUT를 테스트 할 수있는 좋은 옵션입니다.

:thumbsdown: 이런 종류의 테스트는 가독성이 떨어집니다. 가독성을 약간 높이려면 부정 및 긍정 예제를 서로 다른 테스트로 분리해야합니다.

```php
final class ExampleTest extends TestCase
{
    /**
     * @test
     * @dataProvider getInvalidEmails
     */
    public function detects_an_invalid_email_address(string $email): void
    {
        $sut = new EmailValidator();

        $result = $sut->isValid($email);

        self::assertFalse($result);
    }

    /**
     * @test
     * @dataProvider getValidEmails
     */
    public function detects_an_valid_email_address(string $email): void
    {
        $sut = new EmailValidator();

        $result = $sut->isValid($email);

        self::assertTrue($result);
    }

    public function getInvalidEmails(): array
    {
        return [
            ['test'],
            ['test@'],
            ['test@test'],
            //...
        ];
    }

    public function getValidEmails(): array
    {
        return [
            ['test@test.com'],
            ['test123@test.com'],
            ['Test123@test.com'],
            //...
        ];
    }
}
```

<a name="two-schools-of-unit-testing"></a>
## 두 학교의 단위 테스트

<a name="classical"></a>
### Classical (디트로이트 학교)

- 단위는 동작의 유닛 단위이며 몇 가지 관련 클래스가 될 수 있습니다.
- 모든 테스트는 다른 테스트와 격리되어야합니다. 따라서 병렬로 또는 임의의 순서로 호출 할 수 있어야합니다.

```php
final class TestExample extends TestCase
{
    /**
     * @test
     */
    public function suspending_an_subscription_with_can_always_suspend_policy_is_always_possible(): void
    {
        $canAlwaysSuspendPolicy = new CanAlwaysSuspendPolicy();
        $sut = new Subscription();

        $result = $sut->suspend($canAlwaysSuspendPolicy);

        self::assertTrue($result);
        self::assertSame(Status::suspend(), $sut->status());
    }
}
```

<a name="mockist"></a>
### Mockist (런던 학교)

- 단위는 단일 클래스입니다.
- 단위는 모든 공동 작업자로부터 격리되어야합니다.

```php
final class TestExample extends TestCase
{
    /**
     * @test
     */
    public function suspending_an_subscription_with_can_always_suspend_policy_is_always_possible(): void
    {
        $canAlwaysSuspendPolicy = $this->createStub(SuspendingPolicyInterface::class);
        $canAlwaysSuspendPolicy->method('suspend')->willReturn(true);
        $sut = new Subscription();

        $result = $sut->suspend($canAlwaysSuspendPolicy);

        self::assertTrue($result);
        self::assertSame(Status::suspend(), $sut->status());
    }
}
```

:information_source: **깨지기 쉬운 테스트를 피하려면 고전적인 접근 방식이 더 좋습니다.**

### 의존성

[TODO]

<a name="mock-vs-stub"></a>
## Mock vs Stub

예:

```php
final class NotificationService
{
    public function __construct(
        private MailerInterface $mailer,
        private MessageRepositoryInterface $messageRepository
    ) {}

    public function send(): void
    {
        $messages = $this->messageRepository->getAll();
        foreach ($messages as $message) {
            $this->mailer->send($message);
        }
    }
}
```

:x: 나쁨 :

- **스텁과의 상호 작용을 검증하면 깨지기 쉬운 테스트가 되어버립니다.**

```php
final class TestExample extends TestCase
{
    /**
     * @test
     */
    public function sends_all_notifications(): void
    {
        $message1 = new Message();
        $message2 = new Message();
        $messageRepository = $this->createMock(MessageRepositoryInterface::class);
        $messageRepository->method('getAll')->willReturn([$message1, $message2]);
        $mailer = $this->createMock(MailerInterface::class);
        $sut = new NotificationService($mailer, $messageRepository);

        $messageRepository->expects(self::once())->method('getAll');
        $mailer->expects(self::exactly(2))->method('send')
            ->withConsecutive([self::equalTo($message1)], [self::equalTo($message2)]);

        $sut->send();
    }
}
```

:heavy_check_mark: 좋음 :

```php
final class TestExample extends TestCase
{
    /**
     * @test
     */
    public function sends_all_notifications(): void
    {
        $message1 = new Message();
        $message2 = new Message();
        $messageRepository = $this->createStub(MessageRepositoryInterface::class);
        $messageRepository->method('getAll')->willReturn([$message1, $message2]);
        $mailer = $this->createMock(MailerInterface::class);
        $sut = new NotificationService($mailer, $messageRepository);

        // Removed asserting interactions with the stub
        $mailer->expects(self::exactly(2))->method('send')
            ->withConsecutive([self::equalTo($message1)], [self::equalTo($message2)]);

        $sut->send();
    }
}
```

<a name="three-styles-of-unit-testing"></a>
## 세 가지 스타일의 단위 테스트

<a name="output"></a>
### Output

: heavy_check_mark : 최상의 옵션 :

- **리팩토링에 대한 최고의 내구성**
- **최고의 정확성**
- **유지 보수 비용이 가장 낮음**
- **가능하다면 이런 종류의 테스트를 선호해야합니다**

```php
final class ExampleTest extends TestCase
{
    /**
     * @test
     * @dataProvider getInvalidEmails
     */
    public function detects_an_invalid_email_address(string $email): void
    {
        $sut = new EmailValidator();

        $result = $sut->isValid($email);

        self::assertFalse($result);
    }

    /**
     * @test
     * @dataProvider getValidEmails
     */
    public function detects_an_valid_email_address(string $email): void
    {
        $sut = new EmailValidator();

        $result = $sut->isValid($email);

        self::assertTrue($result);
    }

    public function getInvalidEmails(): array
    {
        return [
            ['test'],
            ['test@'],
            ['test@test'],
            //...
        ];
    }

    public function getValidEmails(): array
    {
        return [
            ['test@test.com'],
            ['test123@test.com'],
            ['Test123@test.com'],
            //...
        ];
    }
}
```

<a name="state"></a>
### State

:white_check_mark: 더 나쁜 옵션 :

- **리팩토링에 대한 내구성 저하**
- **더 나쁜 정확도**
- **높은 유지 보수 비용**

```php
final class ExampleTest extends TestCase
{
    /**
     * @test
     */
    public function adding_an_item_to_cart(): void
    {
        $item = new CartItem('Product');
        $sut = new Cart();

        $sut->addItem($item);

        self::assertSame(1, $sut->getCount());
        self::assertSame($item, $sut->getItems()[0]);
    }
}
```

<a name="communication"></a>
### Communication

:white_check_mark: 최악의 옵션 :

- **리팩토링에 대한 최악의 내구성**
- **최악의 정확도**
- **유지 관리 비용이 가장 높음**

```php
final class ExampleTest extends TestCase
{
    /**
     * @test
     */
    public function sends_all_notifications(): void
    {
        $message1 = new Message();
        $message2 = new Message();
        $messageRepository = $this->createStub(MessageRepositoryInterface::class);
        $messageRepository->method('getAll')->willReturn([$message1, $message2]);
        $mailer = $this->createMock(MailerInterface::class);
        $sut = new NotificationService($mailer, $messageRepository);

        $mailer->expects(self::exactly(2))->method('send')
            ->withConsecutive([self::equalTo($message1)], [self::equalTo($message2)]);

        $sut->send();
    }
}
```

<a name="functional-architecture-and-tests"></a>
## 기능적 아키텍처 및 테스트

:x: 나쁨 :

```php
final class NameService
{
    public function __construct(private CacheStorageInterface $cacheStorage) {}

    public function loadAll(): void
    {
        $namesCsv = array_map('str_getcsv', file(__DIR__.'/../names.csv'));
        $names = [];

        foreach ($namesCsv as $nameData) {
            if (!isset($nameData[0], $nameData[1])) {
                continue;
            }

            $names[] = new Name($nameData[0], new Gender($nameData[1]));
        }

        $this->cacheStorage->store('names', $names);
    }
}
```

**이와 같은 코드를 테스트하는 방법? 파일 시스템과 관련된 인프라 코드를 직접 사용하기 때문에 통합 테스트를 통해서만 가능합니다.**

: heavy_check_mark : 좋음 :

기능적 아키텍처와 마찬가지로 부작용이있는 코드와 논리 만 포함 된 코드를 분리해야합니다.

```php
final class NameParser
{
    /**
     * @param array $namesData
     * @return Name[]
     */
    public function parse(array $namesData): array
    {
        $names = [];

        foreach ($namesData as $nameData) {
            if (!isset($nameData[0], $nameData[1])) {
                continue;
            }

            $names[] = new Name($nameData[0], new Gender($nameData[1]));
        }

        return $names;
    }
}
```

```php
final class CsvNamesFileLoader
{
    public function load(): array
    {
        return array_map('str_getcsv', file(__DIR__.'/../names.csv'));
    }
}
```

```php
final class ApplicationService
{
    public function __construct(
        private CsvNamesFileLoader $fileLoader,
        private NameParser $parser,
        private CacheStorageInterface $cacheStorage
    ) {}

    public function loadNames(): void
    {
        $namesData = $this->fileLoader->load();
        $names = $this->parser->parse($namesData);
        $this->cacheStorage->store('names', $names);
    }
}
```

```php
final class ValidUnitExampleTest extends TestCase
{
    /**
     * @test
     */
    public function parse_all_names(): void
    {
        $namesData = [
            ['John', 'M'],
            ['Lennon', 'U'],
            ['Sarah', 'W']
        ];
        $sut = new NameParser();

        $result = $sut->parse($namesData);
        
        self::assertSame(
            [
                new Name('John', new Gender('M')),
                new Name('Lennon', new Gender('U')),
                new Name('Sarah', new Gender('W'))
            ],
            $result
        );
    }
}
```

<a name="observable-behavior-vs-implementation-details"></a>
## 관찰 가능한 동작과 구현 세부 정보

:x: 나쁨 :

```php
final class ApplicationService
{
    public function __construct(private SubscriptionRepositoryInterface $subscriptionRepository) {}

    public function renewSubscription(int $subscriptionId): bool
    {
        $subscription = $this->subscriptionRepository->findById($subscriptionId);

        if (!$subscription->getStatus()->isEqual(Status::expired())) {
            return false;
        }

        $subscription->setStatus(Status::active());
        $subscription->setModifiedAt(new \DateTimeImmutable());
        return true;
    }
}
```

```php
final class Subscription
{
    private Status $status;

    private \DateTimeImmutable $modifiedAt;

    public function __construct(Status $status, \DateTimeImmutable $modifiedAt)
    {
        $this->status = $status;
        $this->modifiedAt = $modifiedAt;
    }

    public function getStatus(): Status
    {
        return $this->status;
    }

    public function setStatus(Status $status): void
    {
        $this->status = $status;
    }

    public function getModifiedAt(): \DateTimeImmutable
    {
        return $this->modifiedAt;
    }

    public function setModifiedAt(\DateTimeImmutable $modifiedAt): void
    {
        $this->modifiedAt = $modifiedAt;
    }
}
```

```php
final class InvalidTestExample extends TestCase
{
    /**
     * @test
     */
    public function renew_an_expired_subscription_is_possible(): void
    {
        $modifiedAt = new \DateTimeImmutable();
        $expiredSubscription = new Subscription(Status::expired(), $modifiedAt);
        $repository = $this->createStub(SubscriptionRepositoryInterface::class);
        $repository->method('findById')->willReturn($expiredSubscription);
        $sut = new ApplicationService($repository);

        $result = $sut->renewSubscription(1);

        self::assertSame(Status::active(), $expiredSubscription->getStatus());
        self::assertGreaterThan($modifiedAt, $expiredSubscription->getModifiedAt());
        self::assertTrue($result);
    }

    /**
     * @test
     */
    public function renew_an_active_subscription_is_not_possible(): void
    {
        $modifiedAt = new \DateTimeImmutable();
        $activeSubscription = new Subscription(Status::active(), $modifiedAt);
        $repository = $this->createStub(SubscriptionRepositoryInterface::class);
        $repository->method('findById')->willReturn($activeSubscription);
        $sut = new ApplicationService($repository);

        $result = $sut->renewSubscription(1);

        self::assertSame($modifiedAt, $activeSubscription->getModifiedAt());
        self::assertFalse($result);
    }
}
```

:heavy_check_mark: 좋음 :

```php
final class ApplicationService
{
    public function __construct(private SubscriptionRepositoryInterface $subscriptionRepository) {}

    public function renewSubscription(int $subscriptionId): bool
    {
        $subscription = $this->subscriptionRepository->findById($subscriptionId);
        return $subscription->renew(new \DateTimeImmutable());
    }
}
```

```php
final class Subscription
{
    private Status $status;

    private \DateTimeImmutable $modifiedAt;

    public function __construct(\DateTimeImmutable $modifiedAt)
    {
        $this->status = Status::new();
        $this->modifiedAt = $modifiedAt;
    }

    public function renew(\DateTimeImmutable $modifiedAt): bool
    {
        if (!$this->status->isEqual(Status::expired())) {
            return false;
        }

        $this->status = Status::active();
        $this->modifiedAt = $modifiedAt;
        return true;
    }

    public function active(\DateTimeImmutable $modifiedAt): void
    {
        //simplified
        $this->status = Status::active();
        $this->modifiedAt = $modifiedAt;
    }

    public function expire(\DateTimeImmutable $modifiedAt): void
    {
        //simplified
        $this->status = Status::expired();
        $this->modifiedAt = $modifiedAt;
    }

    public function isActive(): bool
    {
        return $this->status->isEqual(Status::active());
    }
}
```

```php
final class ValidTestExample extends TestCase
{
    /**
     * @test
     */
    public function renew_an_expired_subscription_is_possible(): void
    {
        $expiredSubscription = SubscriptionMother::expired();
        $repository = $this->createStub(SubscriptionRepositoryInterface::class);
        $repository->method('findById')->willReturn($expiredSubscription);
        $sut = new ApplicationService($repository);

        $result = $sut->renewSubscription(1);

        // skip checking modifiedAt as it's not a part of observable behavior. To check this value we
        // would have to add a getter for modifiedAt, probably only for test purposes.
        self::assertTrue($expiredSubscription->isActive());
        self::assertTrue($result);
    }

    /**
     * @test
     */
    public function renew_an_active_subscription_is_not_possible(): void
    {
        $activeSubscription = SubscriptionMother::active();
        $repository = $this->createStub(SubscriptionRepositoryInterface::class);
        $repository->method('findById')->willReturn($activeSubscription);
        $sut = new ApplicationService($repository);

        $result = $sut->renewSubscription(1);

        self::assertTrue($activeSubscription->isActive());
        self::assertFalse($result);
    }
}
```

:information_source: 첫 번째 구독 모델의 디자인이 잘못되었습니다. 하나의 비즈니스 작업을 호출하려면 세 가지 메서드를 호출해야합니다. 또한 getter를 사용하여 작업을 확인하는 것은 좋은 방법이 아닙니다. `modifiedAt` 의 변경 확인을 건너 뛰고 `modifiedAt` 설정을 만료 비즈니스 작업으로 테스트 할 수 있습니다. `modifiedAt` 대한 getter는 필요하지 않습니다. 물론 테스트 용으로 만 제공되는 게터를 피할 수있는 가능성을 찾는 것이 매우 어려운 경우도 있지만 항상 도입하지 않도록 노력해야합니다.

<a name="unit-of-behavior"></a>
## 행동 단위

:x: 나쁨 :

```php
class CannotSuspendExpiredSubscriptionPolicy implements SuspendingPolicyInterface
{
    public function suspend(Subscription $subscription, \DateTimeImmutable $at): bool
    {
        if ($subscription->isExpired()) {
            return false;
        }

        return true;
    }
}
```

```php
class CannotSuspendExpiredSubscriptionPolicyTest extends TestCase
{
    /**
     * @test
     */
    public function it_returns_false_when_a_subscription_is_expired(): void
    {
        $policy = new CannotSuspendExpiredSubscriptionPolicy();
        $subscription = $this->createStub(Subscription::class);
        $subscription->method('isExpired')->willReturn(true);

        self::assertFalse($policy->suspend($subscription, new \DateTimeImmutable()));
    }

    /**
     * @test
     */
    public function it_returns_true_when_a_subscription_is_not_expired(): void
    {
        $policy = new CannotSuspendExpiredSubscriptionPolicy();
        $subscription = $this->createStub(Subscription::class);
        $subscription->method('isExpired')->willReturn(false);

        self::assertTrue($policy->suspend($subscription, new \DateTimeImmutable()));
    }
}
```

```php
class CannotSuspendNewSubscriptionPolicy implements SuspendingPolicyInterface
{
    public function suspend(Subscription $subscription, \DateTimeImmutable $at): bool
    {
        if ($subscription->isNew()) {
            return false;
        }

        return true;
    }
}
```

```php
class CannotSuspendNewSubscriptionPolicyTest extends TestCase
{
    /**
     * @test
     */
    public function it_returns_false_when_a_subscription_is_new(): void
    {
        $policy = new CannotSuspendNewSubscriptionPolicy();
        $subscription = $this->createStub(Subscription::class);
        $subscription->method('isNew')->willReturn(true);

        self::assertFalse($policy->suspend($subscription, new \DateTimeImmutable()));
    }

    /**
     * @test
     */
    public function it_returns_true_when_a_subscription_is_not_new(): void
    {
        $policy = new CannotSuspendNewSubscriptionPolicy();
        $subscription = $this->createStub(Subscription::class);
        $subscription->method('isNew')->willReturn(false);

        self::assertTrue($policy->suspend($subscription, new \DateTimeImmutable()));
    }
}
```

```php
class CanSuspendAfterOneMonthPolicy implements SuspendingPolicyInterface
{
    public function suspend(Subscription $subscription, \DateTimeImmutable $at): bool
    {
        $oneMonthEarlierDate = \DateTime::createFromImmutable($at)->sub(new \DateInterval('P1M'));

        return $subscription->isOlderThan(\DateTimeImmutable::createFromMutable($oneMonthEarlierDate));
    }
}
```

```php
class CanSuspendAfterOneMonthPolicyTest extends TestCase
{
    /**
     * @test
     */
    public function it_returns_true_when_a_subscription_is_older_than_one_month(): void
    {
        $date = new \DateTimeImmutable('2021-01-29');
        $policy = new CanSuspendAfterOneMonthPolicy();
        $subscription = new Subscription(new \DateTimeImmutable('2020-12-28'));

        self::assertTrue($policy->suspend($subscription, $date));
    }

    /**
     * @test
     */
    public function it_returns_false_when_a_subscription_is_not_older_than_one_month(): void
    {
        $date = new \DateTimeImmutable('2021-01-29');
        $policy = new CanSuspendAfterOneMonthPolicy();
        $subscription = new Subscription(new \DateTimeImmutable('2020-01-01'));

        self::assertTrue($policy->suspend($subscription, $date));
    }
}
```

```php
class Status
{
    private const EXPIRED = 'expired';
    private const ACTIVE = 'active';
    private const NEW = 'new';
    private const SUSPENDED = 'suspended';

    private string $status;

    private function __construct(string $status)
    {
        $this->status = $status;
    }

    public static function expired(): self
    {
        return new self(self::EXPIRED);
    }

    public static function active(): self
    {
        return new self(self::ACTIVE);
    }

    public static function new(): self
    {
        return new self(self::NEW);
    }

    public static function suspended(): self
    {
        return new self(self::SUSPENDED);
    }

    public function isEqual(self $status): bool
    {
        return $this->status === $status->status;
    }
}
```

```php
class StatusTest extends TestCase
{
    public function testEquals(): void
    {
        $status1 = Status::active();
        $status2 = Status::active();

        self::assertTrue($status1->isEqual($status2));
    }

    public function testNotEquals(): void
    {
        $status1 = Status::active();
        $status2 = Status::expired();

        self::assertFalse($status1->isEqual($status2));
    }
}
```

```php
class SubscriptionTest extends TestCase
{
    /**
     * @test
     */
    public function suspending_a_subscription_is_possible_when_a_policy_returns_true(): void
    {
        $policy = $this->createMock(SuspendingPolicyInterface::class);
        $policy->expects($this->once())->method('suspend')->willReturn(true);
        $sut = new Subscription(new \DateTimeImmutable());

        $result = $sut->suspend($policy, new \DateTimeImmutable());

        self::assertTrue($result);
        self::assertTrue($sut->isSuspended());
    }

    /**
     * @test
     */
    public function suspending_a_subscription_is_not_possible_when_a_policy_returns_false(): void
    {
        $policy = $this->createMock(SuspendingPolicyInterface::class);
        $policy->expects($this->once())->method('suspend')->willReturn(false);
        $sut = new Subscription(new \DateTimeImmutable());

        $result = $sut->suspend($policy, new \DateTimeImmutable());

        self::assertFalse($result);
        self::assertFalse($sut->isSuspended());
    }

    /**
     * @test
     */
    public function it_returns_true_when_a_subscription_is_older_than_one_month(): void
    {
        $date = new \DateTimeImmutable();
        $futureDate = $date->add(new \DateInterval('P1M'));
        $sut = new Subscription($date);

        self::assertTrue($sut->isOlderThan($futureDate));
    }

    /**
     * @test
     */
    public function it_returns_false_when_a_subscription_is_not_older_than_one_month(): void
    {
        $date = new \DateTimeImmutable();
        $futureDate = $date->add(new \DateInterval('P1D'));
        $sut = new Subscription($date);

        self::assertTrue($sut->isOlderThan($futureDate));
    }
}
```

:exclamation: **코드와 1:1, 클래스와 1:1 테스트를 작성하지 마십시오. 리팩토링을 어렵게 만드는 깨지기 쉬운 테스트로 이어집니다.**

:heavy_check_mark: 좋음 :

```php
final class CannotSuspendExpiredSubscriptionPolicy implements SuspendingPolicyInterface
{
    public function suspend(Subscription $subscription, \DateTimeImmutable $at): bool
    {
        if ($subscription->isExpired()) {
            return false;
        }

        return true;
    }
}
```

```php
final class CannotSuspendNewSubscriptionPolicy implements SuspendingPolicyInterface
{
    public function suspend(Subscription $subscription, \DateTimeImmutable $at): bool
    {
        if ($subscription->isNew()) {
            return false;
        }

        return true;
    }
}
```

```php
final class CanSuspendAfterOneMonthPolicy implements SuspendingPolicyInterface
{
    public function suspend(Subscription $subscription, \DateTimeImmutable $at): bool
    {
        $oneMonthEarlierDate = \DateTime::createFromImmutable($at)->sub(new \DateInterval('P1M'));

        return $subscription->isOlderThan(\DateTimeImmutable::createFromMutable($oneMonthEarlierDate));
    }
}
```

```php
final class Status
{
    private const EXPIRED = 'expired';
    private const ACTIVE = 'active';
    private const NEW = 'new';
    private const SUSPENDED = 'suspended';

    private string $status;

    private function __construct(string $status)
    {
        $this->status = $status;
    }

    public static function expired(): self
    {
        return new self(self::EXPIRED);
    }

    public static function active(): self
    {
        return new self(self::ACTIVE);
    }

    public static function new(): self
    {
        return new self(self::NEW);
    }

    public static function suspended(): self
    {
        return new self(self::SUSPENDED);
    }

    public function isEqual(self $status): bool
    {
        return $this->status === $status->status;
    }
}
```

```php
final class Subscription
{
    private Status $status;

    private \DateTimeImmutable $createdAt;

    public function __construct(\DateTimeImmutable $createdAt)
    {
        $this->status = Status::new();
        $this->createdAt = $createdAt;
    }

    public function suspend(SuspendingPolicyInterface $suspendingPolicy, \DateTimeImmutable $at): bool
    {
        $result = $suspendingPolicy->suspend($this, $at);
        if ($result) {
            $this->status = Status::suspended();
        }

        return $result;
    }

    public function isOlderThan(\DateTimeImmutable $date): bool
    {
        return $this->createdAt < $date;
    }

    public function activate(): void
    {
        $this->status = Status::active();
    }

    public function expire(): void
    {
        $this->status = Status::expired();
    }

    public function isExpired(): bool
    {
        return $this->status->isEqual(Status::expired());
    }

    public function isActive(): bool
    {
        return $this->status->isEqual(Status::active());
    }

    public function isNew(): bool
    {
        return $this->status->isEqual(Status::new());
    }

    public function isSuspended(): bool
    {
        return $this->status->isEqual(Status::suspended());
    }
}
```

```php
final class SubscriptionSuspendingTest extends TestCase
{
    /**
     * @test
     */
    public function suspending_an_expired_subscription_with_cannot_suspend_expired_policy_is_not_possible(): void
    {
        $sut = new Subscription(new \DateTimeImmutable());
        $sut->activate();
        $sut->expire();

        $result = $sut->suspend(new CannotSuspendExpiredSubscriptionPolicy(), new \DateTimeImmutable());

        self::assertFalse($result);
    }

    /**
     * @test
     */
    public function suspending_a_new_subscription_with_cannot_suspend_new_policy_is_not_possible(): void
    {
        $sut = new Subscription(new \DateTimeImmutable());

        $result = $sut->suspend(new CannotSuspendNewSubscriptionPolicy(), new \DateTimeImmutable());

        self::assertFalse($result);
    }

    /**
     * @test
     */
    public function suspending_an_active_subscription_with_cannot_suspend_new_policy_is_possible(): void
    {
        $sut = new Subscription(new \DateTimeImmutable());
        $sut->activate();

        $result = $sut->suspend(new CannotSuspendNewSubscriptionPolicy(), new \DateTimeImmutable());

        self::assertTrue($result);
    }

    /**
     * @test
     */
    public function suspending_an_active_subscription_with_cannot_suspend_expired_policy_is_possible(): void
    {
        $sut = new Subscription(new \DateTimeImmutable());
        $sut->activate();

        $result = $sut->suspend(new CannotSuspendExpiredSubscriptionPolicy(), new \DateTimeImmutable());

        self::assertTrue($result);
    }

    /**
     * @test
     */
    public function suspending_an_subscription_before_a_one_month_is_not_possible(): void
    {
        $sut = new Subscription(new \DateTimeImmutable('2020-01-01'));

        $result = $sut->suspend(new CanSuspendAfterOneMonthPolicy(), new \DateTimeImmutable('2020-01-10'));

        self::assertFalse($result);
    }

    /**
     * @test
     */
    public function suspending_an_subscription_after_a_one_month_is_possible(): void
    {
        $sut = new Subscription(new \DateTimeImmutable('2020-01-01'));

        $result = $sut->suspend(new CanSuspendAfterOneMonthPolicy(), new \DateTimeImmutable('2020-02-02'));

        self::assertTrue($result);
    }
}
```

<a name="humble-pattern"></a>
## 겸손한 패턴

이와 같은 클래스를 올바르게 단위 테스트하는 방법은 무엇입니까?

```php
class ApplicationService
{
    public function __construct(
        private OrderRepository $orderRepository,
        private FormRepository $formRepository
    ) {}

    public function changeFormStatus(int $orderId): void
    {
        $order = $this->orderRepository->getById($orderId);
        $soapResponse = $this->getSoapClient()->getStatusByOrderId($orderId);
        $form = $this->formRepository->getByOrderId($orderId);
        $form->setStatus($soapResponse['status']);
        $form->setModifiedAt(new \DateTimeImmutable());

        if ($soapResponse['status'] === 'accepted') {
            $order->setStatus('paid');
        }

        $this->formRepository->save($form);
        $this->orderRepository->save($order);
    }

    private function getSoapClient(): \SoapClient
    {
        return new \SoapClient('https://legacy_system.pl/Soap/WebService', []);
    }
}
```

:heavy_check_mark: 너무 복잡한 코드를 분리하여 클래스를 분리해야합니다.

```php
final class ApplicationService
{
    public function __construct(
        private OrderRepositoryInterface $orderRepository,
        private FormRepositoryInterface $formRepository,
        private FormApiInterface $formApi,
        private ChangeFormStatusService $changeFormStatusService
    ) {}

    public function changeFormStatus(int $orderId): void
    {
        $order = $this->orderRepository->getById($orderId);
        $form = $this->formRepository->getByOrderId($orderId);
        $status = $this->formApi->getStatusByOrderId($orderId);

        $this->changeFormStatusService->changeStatus($order, $form, $status);

        $this->formRepository->save($form);
        $this->orderRepository->save($order);
    }
}
```

```php
final class ChangeFormStatusService
{
    public function changeStatus(Order $order, Form $form, string $formStatus): void
    {
        $status = FormStatus::createFromString($formStatus);
        $form->changeStatus($status);

        if ($form->isAccepted()) {
            $order->changeStatus(OrderStatus::paid());
        }
    }
}
```

```php
final class ChangingFormStatusTest extends TestCase
{
    /**
     * @test
     */
    public function changing_a_form_status_to_accepted_changes_an_order_status_to_paid(): void
    {
        $order = new Order();
        $form = new Form();
        $status = 'accepted';
        $sut = new ChangeFormStatusService();

        $sut->changeStatus($order, $form, $status);

        self::assertTrue($form->isAccepted());
        self::assertTrue($order->isPaid());
    }

    /**
     * @test
     */
    public function changing_a_form_status_to_refused_not_changes_an_order_status(): void
    {
        $order = new Order();
        $form = new Form();
        $status = 'new';
        $sut = new ChangeFormStatusService();

        $sut->changeStatus($order, $form, $status);

        self::assertFalse($form->isAccepted());
        self::assertFalse($order->isPaid());
    }
}
```

그러나 ApplicationService는 모의 FormApiInterface 만 사용하는 통합 테스트로 테스트해야합니다.

<a name="trivial-test"></a>
## 사소한 테스트

:x: 나쁨 :

```php
final class Customer
{
    public function __construct(private string $name) {}

    public function getName(): string
    {
        return $this->name;
    }

    public function setName(string $name): void
    {
        $this->name = $name;
    }
}
```

```php
final class CustomerTest extends TestCase
{
    public function testSetName(): void
    {
        $customer = new Customer('Jack');

        $customer->setName('John');

        self::assertSame('John', $customer->getName());
    }
}
```

```php
final class EventSubscriber
{
    public static function getSubscribedEvents(): array
    {
        return ['event' => 'onEvent'];
    }

    public function onEvent(): void
    {

    }
}
```

```php
final class EventSubscriberTest extends TestCase
{
    public function testGetSubscribedEvents(): void
    {
        $result = EventSubscriber::getSubscribedEvents();

        self::assertSame(['event' => 'onEvent'], $result);
    }
}
```

:exclamation: 복잡한 로직없이 코드를 테스트하는 것은 무의미하지만 깨지기 쉬운 테스트로 이어집니다.

<a name="fragile-test"></a>
## 깨지기 쉬운 테스트

:x: 나쁨 :

```php
final class UserRepository
{
    public function __construct(
        private Connection $connection
    ) {}

    public function getUserNameByEmail(string $email): ?array
    {
        return $this
            ->connection
            ->createQueryBuilder()
            ->from('user', 'u')
            ->where('u.email = :email')
            ->setParameter('email', $email)
            ->execute()
            ->fetch();
    }
}
```

```php
final class TestUserRepository extends TestCase
{
    public function testGetUserNameByEmail(): void
    {
        $email = 'test@test.com';
        $connection = $this->createMock(Connection::class);
        $queryBuilder = $this->createMock(QueryBuilder::class);
        $result = $this->createMock(ResultStatement::class);
        $userRepository = new UserRepository($connection);
        $connection
            ->expects($this->once())
            ->method('createQueryBuilder')
            ->willReturn($queryBuilder);
        $queryBuilder
            ->expects($this->once())
            ->method('from')
            ->with('user', 'u')
            ->willReturn($queryBuilder);
        $queryBuilder
            ->expects($this->once())
            ->method('where')
            ->with('u.email = :email')
            ->willReturn($queryBuilder);
        $queryBuilder
            ->expects($this->once())
            ->method('setParameter')
            ->with('email', $email)
            ->willReturn($queryBuilder);
        $queryBuilder
            ->expects($this->once())
            ->method('execute')
            ->willReturn($result);
        $result
            ->expects($this->once())
            ->method('fetch')
            ->willReturn(['email' => $email]);

        $result = $userRepository->getUserNameByEmail($email);

        self::assertSame(['email' => $email], $result);
    }
}
```

:exclamation: 이런식으로 리포지토리를 테스트하면 깨지기 쉬운 테스트가 만들어지고 리팩토링이 어렵습니다. 리포지터리를 테스트하려면 통합 테스트를 작성하십시오.

<a name="test-fixtures"></a>
## 테스트 픽스처

:x: 나쁨 :

```php
final class InvalidTest extends TestCase
{
    private ?Subscription $subscription;

    public function setUp(): void
    {
        $this->subscription = new Subscription(new \DateTimeImmutable());
        $this->subscription->activate();
    }

    /**
     * @test
     */
    public function suspending_an_active_subscription_with_cannot_suspend_new_policy_is_possible(): void
    {
        $result = $this->subscription->suspend(new CannotSuspendNewSubscriptionPolicy(), new \DateTimeImmutable());

        self::assertTrue($result);
    }

    /**
     * @test
     */
    public function suspending_an_active_subscription_with_cannot_suspend_expired_policy_is_possible(): void
    {
        $result = $this->subscription->suspend(new CannotSuspendExpiredSubscriptionPolicy(), new \DateTimeImmutable());

        self::assertTrue($result);
    }

    /**
     * @test
     */
    public function suspending_a_new_subscription_with_cannot_suspend_new_policy_is_not_possible(): void
    {
        // Here we need to create a new subscription, it is not possible to change $this->subscription to a new subscription
    }
}
```

:heavy_check_mark: 좋음 :

```php
final class ValidTest extends TestCase
{
    /**
     * @test
     */
    public function suspending_an_active_subscription_with_cannot_suspend_new_policy_is_possible(): void
    {
        $sut = $this->createAnActiveSubscription();

        $result = $sut->suspend(new CannotSuspendNewSubscriptionPolicy(), new \DateTimeImmutable());

        self::assertTrue($result);
    }

    /**
     * @test
     */
    public function suspending_an_active_subscription_with_cannot_suspend_expired_policy_is_possible(): void
    {
        $sut = $this->createAnActiveSubscription();

        $result = $sut->suspend(new CannotSuspendExpiredSubscriptionPolicy(), new \DateTimeImmutable());

        self::assertTrue($result);
    }

    /**
     * @test
     */
    public function suspending_a_new_subscription_with_cannot_suspend_new_policy_is_not_possible(): void
    {
        $sut = $this->createANewSubscription();

        $result = $sut->suspend(new CannotSuspendNewSubscriptionPolicy(), new \DateTimeImmutable());

        self::assertFalse($result);
    }

    private function createANewSubscription(): Subscription
    {
        return new Subscription(new \DateTimeImmutable());
    }

    private function createAnActiveSubscription(): Subscription
    {
        $subscription = new Subscription(new \DateTimeImmutable());
        $subscription->activate();
        return $subscription;
    }
}
```

- 테스트간에 공유되는 상태를 피하는 것이 좋습니다.
- 몇 가지 테스트 사이에 요소를 재사용하려면 :
    - private 팩토리 메소드-하나의 클래스에서 재사용 (위와 같이)
    -  [부모 객체](https://gitlocalize.com/repo/6002/ko/(#object-mother)) -몇 가지 클래스에서 재사용

<a name="general-testing-anti-patterns"></a>
## 일반적인 테스트의 안티 패턴

<a name="exposing-private-state"></a>
### 비공개 상태 노출

:x: 나쁨 :

```php
final class Customer
{
    private CustomerType $type;

    private DiscountCalculationPolicyInterface $discountCalculationPolicy;

    public function __construct()
    {
        $this->type = CustomerType::NORMAL();
        $this->discountCalculationPolicy = new NormalDiscountPolicy();
    }

    public function makeVip(): void
    {
        $this->type = CustomerType::VIP();
        $this->discountCalculationPolicy = new VipDiscountPolicy();
    }

    public function getCustomerType(): CustomerType
    {
        return $this->type;
    }

    public function getPercentageDiscount(): int
    {
        return $this->discountCalculationPolicy->getPercentageDiscount();
    }
}
```

```php
final class InvalidTest extends TestCase
{
    public function testMakeVip(): void
    {
        $sut = new Customer();
        $sut->makeVip();

        self::assertSame(CustomerType::VIP(), $sut->getCustomerType());
    }
}
```

:heavy_check_mark: 좋음 :

```php
final class Customer
{
    private CustomerType $type;

    private DiscountCalculationPolicyInterface $discountCalculationPolicy;

    public function __construct()
    {
        $this->type = CustomerType::NORMAL();
        $this->discountCalculationPolicy = new NormalDiscountPolicy();
    }

    public function makeVip(): void
    {
        $this->type = CustomerType::VIP();
        $this->discountCalculationPolicy = new VipDiscountPolicy();
    }

    public function getPercentageDiscount(): int
    {
        return $this->discountCalculationPolicy->getPercentageDiscount();
    }
}
```

```php
final class ValidTest extends TestCase
{
    /**
     * @test
     */
    public function a_vip_customer_has_a_25_percentage_discount(): void
    {
        $sut = new Customer();
        $sut->makeVip();

        self::assertSame(25, $sut->getPercentageDiscount());
    }
}
```

:exclamation: 테스트에서 상태를 확인하기 위해서만 추가 프로덕션 코드 (예 : getter getCustomerType())를 추가하는 것은 나쁜 습관입니다. 다른 도메인 중요한 값 (이 경우 getPercentageDiscount ())으로 확인해야합니다. 물론 때로는 작업을 확인하는 다른 방법을 찾기가 어려울 수 있으며 테스트에서 정확성을 확인하기 위해 추가 프로덕션 코드를 추가해야 할 수 있지만 이를 피해야합니다.

<a name="leaking-domain-details"></a>
### 도메인 정보 유출

```php
final class DiscountCalculator
{
    public function calculate(int $isVipFromYears): int
    {
        Assert::greaterThanEq($isVipFromYears, 0);
        return min(($isVipFromYears * 10) + 3, 80);
    }
}
```

:x: 나쁨 :

```php
final class InvalidTest extends TestCase
{
    /**
     * @dataProvider discountDataProvider
     */
    public function testCalculate(int $vipDaysFrom, int $expected): void
    {
        $sut = new DiscountCalculator();

        self::assertSame($expected, $sut->calculate($vipDaysFrom));
    }

    public function discountDataProvider(): array
    {
        return [
            [0, 0 * 10 + 3], //leaking domain details
            [1, 1 * 10 + 3],
            [5, 5 * 10 + 3],
            [8, 80]
        ];
    }
}
```

:heavy_check_mark: 좋음 :

```php
final class ValidTest extends TestCase
{
    /**
     * @dataProvider discountDataProvider
     */
    public function testCalculate(int $vipDaysFrom, int $expected): void
    {
        $sut = new DiscountCalculator();

        self::assertSame($expected, $sut->calculate($vipDaysFrom));
    }

    public function discountDataProvider(): array
    {
        return [
            [0, 3],
            [1, 13],
            [5, 53],
            [8, 80]
        ];
    }
}
```

:information_source: 테스트에서 실제 서비스의 로직을 복제하지 마십시오. 하드 코딩 된 값으로 결과를 확인하십시오.

<a name="mocking-concrete-classes"></a>
### 구체적인 클래스 모킹-mocking

:x: 나쁨 :

```php
class DiscountCalculator
{
    public function calculateInternalDiscount(int $isVipFromYears): int
    {
        Assert::greaterThanEq($isVipFromYears, 0);
        return min(($isVipFromYears * 10) + 3, 80);
    }

    public function calculateAdditionalDiscountFromExternalSystem(): int
    {
        // get data from an external system to calculate a discount
        return 5;
    }
}
```

```php
class OrderService
{
    public function __construct(private DiscountCalculator $discountCalculator) {}

    public function getTotalPriceWithDiscount(int $totalPrice, int $vipFromDays): int
    {
        $internalDiscount = $this->discountCalculator->calculateInternalDiscount($vipFromDays);
        $externalDiscount = $this->discountCalculator->calculateAdditionalDiscountFromExternalSystem();
        $discountSum = $internalDiscount + $externalDiscount;
        return $totalPrice - (int) ceil(($totalPrice * $discountSum) / 100);
    }
}
```

```php
final class InvalidTest extends TestCase
{
    /**
     * @dataProvider orderDataProvider
     */
    public function testGetTotalPriceWithDiscount(int $totalPrice, int $vipDaysFrom, int $expected): void
    {
        $discountCalculator = $this->createPartialMock(DiscountCalculator::class, ['calculateAdditionalDiscountFromExternalSystem']);
        $discountCalculator->method('calculateAdditionalDiscountFromExternalSystem')->willReturn(5);
        $sut = new OrderService($discountCalculator);

        self::assertSame($expected, $sut->getTotalPriceWithDiscount($totalPrice, $vipDaysFrom));
    }

    public function orderDataProvider(): array
    {
        return [
            [1000, 0, 920],
            [500, 1, 410],
            [644, 5, 270],
        ];
    }
}
```

:heavy_check_mark: 좋음 :

```php
interface ExternalDiscountCalculatorInterface
{
    public function calculate(): int;
}
```

```php
final class InternalDiscountCalculator
{
    public function calculate(int $isVipFromYears): int
    {
        Assert::greaterThanEq($isVipFromYears, 0);
        return min(($isVipFromYears * 10) + 3, 80);
    }
}
```

```php
final class OrderService
{
    public function __construct(
        private InternalDiscountCalculator $discountCalculator,
        private ExternalDiscountCalculatorInterface $externalDiscountCalculator
    ) {}

    public function getTotalPriceWithDiscount(int $totalPrice, int $vipFromDays): int
    {
        $internalDiscount = $this->discountCalculator->calculate($vipFromDays);
        $externalDiscount = $this->externalDiscountCalculator->calculate();
        $discountSum = $internalDiscount + $externalDiscount;
        return $totalPrice - (int) ceil(($totalPrice * $discountSum) / 100);
    }
}
```

```php
final class ValidTest extends TestCase
{
    /**
     * @dataProvider orderDataProvider
     */
    public function testGetTotalPriceWithDiscount(int $totalPrice, int $vipDaysFrom, int $expected): void
    {
        $externalDiscountCalculator = $this->createStub(ExternalDiscountCalculatorInterface::class);
        $externalDiscountCalculator->method('calculate')->willReturn(5);
        $sut = new OrderService(new InternalDiscountCalculator(), $externalDiscountCalculator);

        self::assertSame($expected, $sut->getTotalPriceWithDiscount($totalPrice, $vipDaysFrom));
    }

    public function orderDataProvider(): array
    {
        return [
            [1000, 0, 920],
            [500, 1, 410],
            [644, 5, 270],
        ];
    }
}
```

:information_source: 행동의 일부를 대체하기 위해 구체적인 클래스를 모킹해야한다는 것은 이 클래스가 아마도 너무 복잡하고 단일 책임 원칙을 위반한다는 것을 의미합니다.

<a name="testing-private-methods"></a>
### 비공개 메서드 테스트

```php
final class OrderItem
{
    public function __construct(private int $total) {}

    public function getTotal(): int
    {
        return $this->total;
    }
}
```

```php
final class Order
{
    /**
     * @param OrderItem[] $items
     * @param int $transportCost
     */
    public function __construct(private array $items, private int $transportCost) {}

    public function getTotal(): int
    {
        return $this->getItemsTotal() + $this->transportCost;
    }

    private function getItemsTotal(): int
    {
        return array_reduce(
            array_map(fn (OrderItem $item) => $item->getTotal(), $this->items),
            fn (int $sum, int $total) => $sum += $total,
            0
        );
    }
}
```

:x: 나쁨 :

```php
final class InvalidTest extends TestCase
{
    /**
     * @test
     * @dataProvider ordersDataProvider
     */
    public function get_total_returns_a_total_cost_of_a_whole_order(Order $order, int $expectedTotal): void
    {
        self::assertSame($expectedTotal, $order->getTotal());
    }

    /**
     * @test
     * @dataProvider orderItemsDataProvider
     */
    public function get_items_total_returns_a_total_cost_of_all_items(Order $order, int $expectedTotal): void
    {
        self::assertSame($expectedTotal, $this->invokePrivateMethodGetItemsTotal($order));
    }

    public function ordersDataProvider(): array
    {
        return [
            [new Order([new OrderItem(20), new OrderItem(20), new OrderItem(20)], 15), 75],
            [new Order([new OrderItem(20), new OrderItem(30), new OrderItem(40)], 0), 90],
            [new Order([new OrderItem(99), new OrderItem(99), new OrderItem(99)], 9), 306]
        ];
    }

    public function orderItemsDataProvider(): array
    {
        return [
            [new Order([new OrderItem(20), new OrderItem(20), new OrderItem(20)], 15), 60],
            [new Order([new OrderItem(20), new OrderItem(30), new OrderItem(40)], 0), 90],
            [new Order([new OrderItem(99), new OrderItem(99), new OrderItem(99)], 9), 297]
        ];
    }

    private function invokePrivateMethodGetItemsTotal(Order &$order): int
    {
        $reflection = new \ReflectionClass(get_class($order));
        $method = $reflection->getMethod('getItemsTotal');
        $method->setAccessible(true);
        return $method->invokeArgs($order, []);
    }
}
```

:heavy_check_mark: 좋음 :

```php
final class ValidTest extends TestCase
{
    /**
     * @test
     * @dataProvider ordersDataProvider
     */
    public function get_total_returns_a_total_cost_of_a_whole_order(Order $order, int $expectedTotal): void
    {
        self::assertSame($expectedTotal, $order->getTotal());
    }

    public function ordersDataProvider(): array
    {
        return [
            [new Order([new OrderItem(20), new OrderItem(20), new OrderItem(20)], 15), 75],
            [new Order([new OrderItem(20), new OrderItem(30), new OrderItem(40)], 0), 90],
            [new Order([new OrderItem(99), new OrderItem(99), new OrderItem(99)], 9), 306]
        ];
    }
}
```

:exclamation: 테스트는 공개 API 만 확인해야합니다.

<a name="time-as-a-volatile-dependency"></a>
### 휘발성 의존성으로서의 시간

:information_source: 시간은 비결정적이므로 휘발성 의존성입니다. 각 호출은 다른 결과를 반환합니다.

:x: 나쁨 :

```php
final class Clock
{
    public static \DateTime|null $currentDateTime = null;

    public static function getCurrentDateTime(): \DateTime
    {
        if (null === self::$currentDateTime) {
            self::$currentDateTime = new \DateTime();
        }

        return self::$currentDateTime;
    }

    public static function set(\DateTime $dateTime): void
    {
        self::$currentDateTime = $dateTime;
    }

    public static function reset(): void
    {
        self::$currentDateTime = null;
    }
}
```

```php
final class Customer
{
    private \DateTime $createdAt;

    public function __construct()
    {
        $this->createdAt = Clock::getCurrentDateTime();
    }

    public function isVip(): bool
    {
        return $this->createdAt->diff(Clock::getCurrentDateTime())->y >= 1;
    }
}
```

```php
final class InvalidTest extends TestCase
{
    /**
     * @test
     */
    public function a_customer_registered_more_than_a_one_year_ago_is_a_vip(): void
    {
        Clock::set(new \DateTime('2019-01-01'));
        $sut = new Customer();
        Clock::reset(); // you have to remember about resetting the shared state

        self::assertTrue($sut->isVip());
    }

    /**
     * @test
     */
    public function a_customer_registered_less_than_a_one_year_ago_is_not_a_vip(): void
    {
        Clock::set((new \DateTime())->sub(new \DateInterval('P2M')));
        $sut = new Customer();
        Clock::reset(); // you have to remember about resetting the shared state

        self::assertFalse($sut->isVip());
    }
}
```

:heavy_check_mark: 좋음 :

```php
interface ClockInterface
{
    public function getCurrentTime(): \DateTimeImmutable;
}
```

```php
final class Clock implements ClockInterface
{
    private function __construct()
    {
    }

    public static function create(): self
    {
        return new self();
    }

    public function getCurrentTime(): \DateTimeImmutable
    {
        return new \DateTimeImmutable();
    }
}
```

```php
final class FixedClock implements ClockInterface
{
    private function __construct(private \DateTimeImmutable $fixedDate) {}

    public static function create(\DateTimeImmutable $fixedDate): self
    {
        return new self($fixedDate);
    }

    public function getCurrentTime(): \DateTimeImmutable
    {
        return $this->fixedDate;
    }
}
```

```php
final class Customer
{
    private \DateTimeImmutable $createdAt;

    public function __construct(\DateTimeImmutable $createdAt)
    {
        $this->createdAt = $createdAt;
    }

    public function isVip(\DateTimeImmutable $currentDate): bool
    {
        return $this->createdAt->diff($currentDate)->y >= 1;
    }
}
```

```php
final class ValidTest extends TestCase
{
    /**
     * @test
     */
    public function a_customer_registered_more_than_a_one_year_ago_is_a_vip(): void
    {
        $sut = new Customer(FixedClock::create(new \DateTimeImmutable('2019-01-01'))->getCurrentTime());

        self::assertTrue($sut->isVip(FixedClock::create(new \DateTimeImmutable('2020-01-02'))->getCurrentTime()));
    }

    /**
     * @test
     */
    public function a_customer_registered_less_than_a_one_year_ago_is_not_a_vip(): void
    {
        $sut = new Customer(FixedClock::create(new \DateTimeImmutable('2019-01-01'))->getCurrentTime());

        self::assertFalse($sut->isVip(FixedClock::create(new \DateTimeImmutable('2019-05-02'))->getCurrentTime()));
    }
}
```

:information_source: 시간과 난수는 도메인 코드에서 직접 생성하면 안됩니다. 동작을 테스트하려면 결정적인 결과가 있어야하므로 위의 예에서와 같이 이러한 값을 도메인 개체에 삽입해야합니다.

<a name="100-test-coverage-shouldnt-be-the-goal"></a>
## 100 % 테스트 커버리지가 목표가되어서는 안됩니다.

100 % 커버리지는 목표가 아니거나 심지어 바람직하지도 않습니다. 100 % 커버리지가 있다면 테스트가 매우 깨지기 쉬울 수 있으므로 리팩토링이 매우 어려울 것입니다. 돌연변이 테스트는 테스트의 품질에 대한 더 나은 피드백을 제공합니다. [더 읽어보기](https://sarvendev.com/en/2019/06/mutation-testing-we-are-testing-tests/)

<a name="recommended-books"></a>
## 추천 도서

- [테스트 주도 개발 : 사례 별 / Kent Beck-](https://www.amazon.com/gp/product/0321146530/) 클래식
- [단위 테스트 원리, 실습 및 패턴 / Vladimir Khorikov-](https://www.amazon.com/Unit-Testing-Principles-Practices-Patterns/dp/1617296279) 내가 읽은 테스트에 관한 최고의 책
