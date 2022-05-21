2차원 회전 행렬을 먼저 살펴보자.  
![image](https://user-images.githubusercontent.com/63915665/169639936-0d14b945-2d2a-48b2-a114-42fcae213ba6.png)

여기서 주의해야 할 점은, 위 사진은 xA 꼴로 곱하고 있기 때문에 Ax꼴일때의 회전행렬을 transpose 해주어야 한다는 것이다.
(블로그 [글](https://gamesmith.tistory.com/139)을 살펴보면 행렬이 다르다는 것을 알 수 있다.   
이는 블로그 글은 Ax꼴이라 행렬이 transpose 되어서 그렇다. 보통 수학에서는 행렬곱을 Ax꼴로 표현하므로 주의할 것.)

또 한가지 주의할 점은 그래픽스에서는 x, y 좌표계의 양의 방향을 x 오른쪽, __y 왼쪽__
으로 나타내기 때문에 크기가 양인 세타가 주어졌을 때회전의 방향이 반시계 방향이 아닌 시계 방향이다. [(참고)](https://youtu.be/cN97hkDrzcc?list=PLqCJpWy5Fohe8ucwhksiv9hTF5sfid8lA)  

2차원 회전행렬은 3차원에서 z축을 회전축으로 잡고 회전시키는 것과 동일하다.
고로 이를 3차원 행렬로 나타내면 다음과 같다.
![image](https://user-images.githubusercontent.com/63915665/169640419-9aabff1c-e9d8-4dd9-91cf-c034818efb56.png)  
  
x, y축을 축으로 잡고 회전시키는 것 또한 동일하다.
때문에 간단한 논리적인 사고를 통해 식을 도출해 낼 수 있다. [(참고)](https://youtu.be/cN97hkDrzcc?list=PLqCJpWy5Fohe8ucwhksiv9hTF5sfid8lA&t=629)  

![image](https://user-images.githubusercontent.com/63915665/169640703-c497f4d1-aac4-41a9-9e8d-ce451c0a7f5c.png)  
z축을 축으로 잡고 양의 세타만큼 하는 회전은 x축을 y축 쪽으로 회전시키는 것으로 이해할 수 있다.
마찬가지로,  
y축을 축으로 잡고 양의 세타만큼 하는 회전은 z축을 x축 쪽으로 회전시키는 것으로 이해할 수 있다.
이 둘은 축의 이름만 다를 뿐 본질적으로 동일한 행위이므로, 우리는 회전행렬의 행과 열에 x, y, z라고 이름을 붙이고 이들의 위치를 변환해주는 것으로  
y축 회전의 회전행렬 또한 쉽게 구할 수 있다.

```c++
static _Mat3 RotationZ( T theta )
	{
		const T sinTheta = sin( theta );
		const T cosTheta = cos( theta );
		return{
			 cosTheta, sinTheta, (T)0.0,
			-sinTheta, cosTheta, (T)0.0,
			(T)0.0,    (T)0.0,   (T)1.0
		};
	}
	static _Mat3 RotationY( T theta )
	{
		const T sinTheta = sin( theta );
		const T cosTheta = cos( theta );
		return{
			 cosTheta, (T)0.0,-sinTheta,
			 (T)0.0,   (T)1.0, (T)0.0,
			 sinTheta, (T)0.0, cosTheta
		};
	}
	static _Mat3 RotationX( T theta )
	{
		const T sinTheta = sin( theta );
		const T cosTheta = cos( theta );
		return{
			(T)1.0, (T)0.0,   (T)0.0,
			(T)0.0, cosTheta, sinTheta,
			(T)0.0,-sinTheta, cosTheta,
		};
	}
```
