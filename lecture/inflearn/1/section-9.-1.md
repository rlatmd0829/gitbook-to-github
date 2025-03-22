# Section 9. 생산자 소비자 문제1

## 생산자 소비자 문제1

생산자 소비자 문제는, 생산자 스레드와 소비자 스레드가 특정 자원을 함께 생산하고, 소비하면서 발생하는 문제이다.

![Image](https://github.com/user-attachments/assets/cf316f69-06bc-4b49-88ed-723be2b7eb95)

이 문제는 결국 중간에 있는 버퍼의 크기가 한정되어 있기 때문에 발생한다. 따라서 한정된 버퍼 문제라고도 한다.

### 생산자 소비자 문제 - 예제

생산자 소비자가 3개씩 있고 임계영역인 queue를 이용해 생성한 data를 넣고 가져가는 예제를 확인해보자.

#### 1. **생산자 소비자 바로 실행**

![Image](https://github.com/user-attachments/assets/e9b0f90e-5370-4f42-942c-735fcb64a274)

해당 예제에서 문제는 큐에 공간이 2개밖에 없어서 공간이 가득 차있을경우에는 데이터를 버리게 되는 문제가 있다.

해당 문제를 해결하기 위해 일정 시간동안 대기하는 로직을 넣으면 해결 할 수 있을 것 같다.

#### 2. 생산자 소비자 일정 시간 대기

![Image](https://github.com/user-attachments/assets/f234dd2e-4963-4a61-900a-8b0342aea846)

큐가 비었을때 소비하거나, 큐가 가득 찼을 때 생산해서 데이터를 버리는 문제를 해결하기 위해 스레드를 잠시 기다리게 변경하였다.

그러나 위의 그림을 보면 락을 가진 상태로 대기하기 때문에 다른 스레드가 임계영역에 진입하지 못해 계속 해서 대기하는 문제가 발생한다.

해당 문제를 해결하기 위해서 잠시 스레드가 대기하는 동안에는 다른 스레드에게 락을 양보하도록 해야한다.

#### 3. Object.wait(), Object.notify()

synchronized를 사용한 임계 영역 안에서 락을 가지고 무한 대기하는 문제는 Object 클래스에 wait(), notify() 메서드를 통해 해결할 수 있다.

![Image](https://github.com/user-attachments/assets/2032bfdb-7e2a-40bc-8b31-f7f1b4e856d4)

> 스레드 대기 집합(wait set)
>
> synchronized 임계 영역 안에서 Object.wait()를 호출하면 스레드는 대기(WAITING) 상태에 들어간다.
>
> 이렇게 대기 상태에 들어간 스레드를 관리하는 것을 대기집합 이라 한다. 모든 객체는 각자의 대기 집합을 가지고 있다.

**Object.wait()**

* 현재 스레드가 가진 락을 반납하고 대기(WAITING)한다.
* 이 메서드는 현재 스레드가 synchronized 블록이나 메서드에서 락을 소유하고 있을때만 호출 할 수있다.

**Object.notify()**

* 대기중인 스레드 중 하나를 깨운다.
* 이 메서드는 synchronized 블록이나 메서드에서 호출되어야 한다. 깨운 스레드는 락을 다시 획득할 기회를 얻게된다.

#### 4. **Object.wait(), Object.notify() 한계**

![Image](https://github.com/user-attachments/assets/9ba772ca-b42d-4e91-9269-49086c4784c9)

Object.wait(), Object.notify() 방식은 스레드 대기 집합 하나에 생산자, 소비자 스레드를 모두 관리한다.

그리고 notify()를 호출할 때 임의의 스레드가 선택되므로 큐에 데이터가 없는 상황에 소비자가 같은 소비자를 깨우는 비효율이 발생할 수 있다.
