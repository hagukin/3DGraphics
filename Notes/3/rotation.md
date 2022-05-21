2차원 회전 행렬을 먼저 살펴보자.  
![image](https://user-images.githubusercontent.com/63915665/169639936-0d14b945-2d2a-48b2-a114-42fcae213ba6.png)

여기서 주의해야 할 점은, 위 사진은 xA 꼴로 곱하고 있기 때문에 Ax꼴일때의 회전행렬을 transpose 해주어야 한다는 것이다.
(블로그 [글](https://gamesmith.tistory.com/139)을 살펴보면 행렬이 다르다는 것을 알 수 있다.   
이는 블로그 글은 Ax꼴이라 행렬이 transpose 되어서 그렇다. 보통 수학에서는 행렬곱을 Ax꼴로 표현하므로 주의할 것.)

또 한가지 주의할 점은 2D그래픽스에서는 x, y 좌표계의 양의 방향을 x 오른쪽, __y 아랫쪽__
으로 나타내기 때문에 크기가 양인 세타가 주어졌을 때회전의 방향이 반시계 방향이 아닌 시계 방향이다. 
[(참고)](https://youtu.be/cN97hkDrzcc?list=PLqCJpWy5Fohe8ucwhksiv9hTF5sfid8lA)  
단 3D 회전에서 z축 기준 회전일 경우에는 x양이 오른쪽, y양이 위쪽 동일하므로 반시계 방향 그대로이다.  

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

코드로 나타내면 다음과 같다. (Mat3.h)
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
  
Game.h와 Game.cpp에 약간의 코드를 추가해 회전하는 큐브를 그려보자.  
```c++
void Game::Go()
{
	gfx.BeginFrame();
	UpdateModel();
	ComposeFrame();
	gfx.EndFrame();
}

void Game::UpdateModel()
{
	const float dt = 1.0f / 60.0f;
	if( wnd.kbd.KeyIsPressed( 'Q' ) )
	{
		theta_x = wrap_angle( theta_x + dTheta * dt );
	}
	if( wnd.kbd.KeyIsPressed( 'W' ) )
	{
		theta_y = wrap_angle( theta_y + dTheta * dt );
	}
	if( wnd.kbd.KeyIsPressed( 'E' ) )
	{
		theta_z = wrap_angle( theta_z + dTheta * dt );
	}
	if( wnd.kbd.KeyIsPressed( 'A' ) )
	{
		theta_x = wrap_angle( theta_x - dTheta * dt );
	}
	if( wnd.kbd.KeyIsPressed( 'S' ) )
	{
		theta_y = wrap_angle( theta_y - dTheta * dt );
	}
	if( wnd.kbd.KeyIsPressed( 'D' ) )
	{
		theta_z = wrap_angle( theta_z - dTheta * dt );
	}
}

void Game::ComposeFrame()
{
	auto lines = cube.GetLines();
	const Mat3 rot =
		Mat3::RotationX( theta_x ) *
		Mat3::RotationY( theta_y ) *
		Mat3::RotationZ( theta_z );
	for( auto& v : lines.vertices )
	{
		v *= rot;
		v += { 0.0f,0.0f,1.0f };
		pst.Transform( v );
	}
	for( auto i = lines.indices.cbegin(),
		end = lines.indices.cend();
		i != end; std::advance( i,2 ) )
	{
		gfx.DrawLine( lines.vertices[*i],lines.vertices[*std::next( i )],Colors::White );
	}
}
```
![image](https://user-images.githubusercontent.com/63915665/169640881-70cde479-57b7-4459-a602-394a9f366974.png)

큐브를 회전시키면 큐브의 회전이 좌우 모두로 인식될 수 있음을 알 수 있는데,  
이는 큐브의 어느 면이 앞인지를 시각적으로 구분해주지 않았을 뿐더러, 원근에 따른 크기변화도 없기 때문이다.  
이 문제는 나중 포스트에서 해결해보자.  
  
회전행렬에 대해 주의해줘야 할 점이 있는데, 이는 바로 회전행렬을 곱하는 순서에 따라 결과가 변한다는 것이다.    
DirectXMath 라이브러리에서도 마찬가지이다. XMMATRIx XMMatrixRoatationRollPitchYaw()라는 함수의 레퍼런스를 읽어보면  
Roll, Pitch, Yaw 순으로 회전시킨다는 사실을 알 수 있다.  

3차원 회전 방향을 편하게 알 수 있는 방법은, 엄지가 축의 양의 방향으로 가게 왼손으로 축을 감싸쥔 상태에서  
손목을 몸 안쪽으로 돌려주면 그게 양의 세타에 대해 회전하는 방향이다. (이 원리로 z축을 잡고 돌려보면 반시계임을 알 수 있다)   
![image](https://user-images.githubusercontent.com/63915665/169641126-9efa9de1-9755-46ff-89cb-c7d22d063bec.png)  
  
회전은 periodic 하다.  
즉 360도 회전은 0도와 똑같고, 이를 라디안으로 나타내면 2pi마다 원래 위치로 돌아온다고 이해할 수 있다.  
그러나 컴퓨터 연산에서는 2pi, 4pi ... 이렇게 회전을 반복하다 2^N pi까지 가게 되면 이는 2 * (3.141592...^N)인데,  
컴퓨터의 floating point는 오차가 있으므로 오차가 누적되어 초기 상태와 미묘하게 위치가 달라질 수 있다.   
때문에 회전각의 범위를 0pi ~ 2pi 또는 -pi~pi 같은 꼴로 제한하는 게 좋다.  
이를 코드로 구현하면 아래와 같다.  
```c++
#pragma once

#include <math.h>

constexpr float PI = 3.14159265f;
constexpr double PI_D = 3.1415926535897932;

template <typename T>
inline auto sq( const T& x )
{
	return x * x;
}

template<typename T>
inline T wrap_angle( T theta )
{
	const T modded = fmod( theta,(T)2.0 * (T)PI_D ); // float의 모듈로 연산 (std::fmod)
	return (modded > (T)PI_D) ?
		(modded - (T)2.0 * (T)PI_D) :
		modded; // -pi~pi로 제한. (2pi로 mod한 후 나머지가 pi보다 크면 2pi를 빼고, 작으면 그대로 놔둔다)
}
```
