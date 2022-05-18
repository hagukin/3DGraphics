* 참고: 벡터 * 행렬 순으로 곱셈을 하는 방법  
(x, y) * ((a, b), (c, d)) = ((x, y)*(a, c), (x, y)*(b, d))  
![image](https://user-images.githubusercontent.com/63915665/169009928-14d3e4f9-a8c8-43dd-8bf9-e843a2161f80.png)
  
  
3차원 공간을 나타내보자.  
일단 벡터3 클래스가 필요하다.  

Vec3 클래스를 정의할 때 Vec2를 상속받아 정의하자. 단 모든 연산 관련 함수는 오버라이딩 해야한다.
```c++
#pragma once

#include <algorithm>
#include "ChiliMath.h"
#include "Vec2.h"

template <typename T>
class _Vec3 : public _Vec2<T>
{
public:
	_Vec3() = default;
	_Vec3( T x,T y,T z )
		:
		_Vec2( x,y ),
		z( z )
	{}
	template <typename T2>
	explicit operator _Vec3<T2>() const
	{
		return{ (T2)x,(T2)y,(T2)z };
	}
	T		LenSq() const
	{
		return sq( *this );
	}
	T		Len() const
	{
		return sqrt( LenSq() );
	}
	_Vec3&	Normalize()
	{
		const T length = Len();
		x /= length;
		y /= length;
		z /= length;
		return *this;
	}
  // ... (생략)
  ```
이렇게 하면 Vec2를 인자로 받는 함수들에 Vec3를 넘겨줄 수 있고,  
넘겨주더라도 내부 연산에서 Vec3의 멤버함수만을 사용한다면 호환이 가능하다는 장점을 가진다.

벡터 말고도 3차원 공간을 표현하려면 3x3 행렬이 필요하다.  
3x3 행렬은 벡터와는 다르게 그냥 간단하게 상속해버릴 수가 없다.  
고로 새로운 클래스 Mat3를 정의해 각 연산멤버함수들을 정의해주면 된다.  


이렇게 만든 벡터와 행렬을 실제로 사용하기 전에 우리가 어떤 3차원 좌표계를 사용할 것인지를 정해야 한다.  
![image](https://user-images.githubusercontent.com/63915665/169012829-51d087ce-1227-4752-9280-5bd3aff1ed43.png)

여기서는 Direct3D를 사용하므로 left-handed 좌표계를 사용할 것이다.  
이때 같은 왼손좌표계이더라도 다르게 표현가능하다는 것을 볼 수 있는데(유니티와 언리얼 비교),  
여기서는 유니티의 방식, 즉 z가 화면 안쪽으로 들어가는 방향의 좌표계를 사용한다.  

좌표계를 정했다면, 원점이 어디에 놓여야 하는지를 정해야 한다.  
원점은 화면의 정중앙에 놓이는 게 좋은데, scaling시 물체가 화면 중앙에서 멀어지는 현상을 막기 위해서이다.  
(원점이 어느 한 구석에 있을 경우 화면 정 중앙에서 멀어짐)  

그 다음은 화면의 음수와 양수 끝지점을 어떤 값으로 할 지를 정해야 하는데,  
-1과 +1을 사용하는 게 권장된다. 물론 화면의 픽셀사이즈/2 로 할 수도 있겠지만 (-240~+240 = 480px 형태로)  
-1과 +1을 사용하면 화면의 픽셀 갯수와 무관하게 음수 끝과 양수 끝을 파악할 수 있다.  
이러한 좌표 체계를 Normalized Device Coordinates (NDC)라고 하며, opengl과 dx3d에서 사용한다.  
  
화면 좌표를 NDC에 맞게 -1, 1로 interpolate하는 과정은 어렵지 않다.
다음과 같다.
```c++
// branch: Tut2
// 3차원 좌표를 화면상의 x, y로 변환해주는 과정 (일단은 z좌표 무시)
Vec3& Transform( Vec3& v ) const
	{
		v.x = (v.x + 1.0f) * xFactor;
		v.y = (-v.y + 1.0f) * yFactor;
		return v;
	}
```
여기서 2차원에서는 좌상단이 0,0이고 우측이 x+, 하단이 y+인 좌표계를 사용한다는 점에 주의해야한다. (그래서 y값에 -1을 먼저 곱해준다)  
  
  
  
3차원 좌표계를 만들 준비가 되었으니, 실제로 무언가를 그려보자.  
일단 큐브 하나를 그려보자.  
Cube라는 클래스를 정의하자. 총 12개의 모서리가 있으니 점 24개를 저장해도 되지만, 그래픽스 라이브러리와 유사하게 IndexArray를 사용해  
겹치는 점을 없애줌으로써 최적화가 가능하다.

```c++
#pragma once

#include "Vec3.h"
#include <vector>
#include "IndexedLineList.h"

class Cube
{
public:
	Cube( float size )
	{
		const float side = size / 2.0f;
		vertices.emplace_back( -side,-side,-side );
		vertices.emplace_back( side,-side,-side );
		vertices.emplace_back( -side,side,-side );
		vertices.emplace_back( side,side,-side );
		vertices.emplace_back( -side,-side,side );
		vertices.emplace_back( side,-side,side );
		vertices.emplace_back( -side,side,side );
		vertices.emplace_back( side,side,side );
	}
	IndexedLineList GetLines() const
	{
		return{ 
			vertices,{
			0,1,  1,3,  3,2,  2,0, // 첫번째 모서리가 0, 1번 정점을 이었다는 뜻
			0,4,  1,5,  3,7,  2,6,
			4,5,  5,7,  7,6,  6,4 }
		};
	}
private:
	std::vector<Vec3> vertices;
};
```

```c++
#pragma once

#include <vector>
#include "Vec3.h"

struct IndexedLineList
{
	std::vector<Vec3> vertices;
	std::vector<size_t> indices;
};
```
이제 이걸 실제로 프레임워크를 사용해 화면 상에 그려보자.

```c++
void Game::ComposeFrame()
{
	auto lines = cube.GetLines();
	for( auto& v : lines.vertices ) //IndexedLineList.vertices
	{
		v += { 0.0f,0.0f,1.0f }; 
		// 현재로썬 z축을 무시중이므로 현재로썬 시각적으로는 차이가 없지만, 나중을 위해 일단 카메라를 정육면체 밖에서 뺀다.
		
		pst.Transform( v );
	}
	for( auto i = lines.indices.cbegin(),
		end = lines.indices.cend();
		i != end; std::advance( i,2 ) ) // std::advance를 사용하면 i가 1 늘어날 때 iterator를 2씩 이동한다.
		// 각 선에 대한 정보는 2개의 정점으로 이루어져 있으므로 2개씩 뛰어넘어간다.
	{
		gfx.DrawLine( lines.vertices[*i],lines.vertices[*std::next( i )],Colors::White );
	}
}
```
* 참고: std::advance 자료: https://ence2.github.io/2020/11/stdadvance-%EC%98%88%EC%A0%9C/

![image](https://user-images.githubusercontent.com/63915665/169018463-b513cf52-b8a3-4b43-abaf-94e7908fb337.png)

정사각형도 아니고 직사각형 하나가 그려지는 걸 볼 수 있는데, 
일단 3차원 도형이 아닌 이유는 우리가 아직 z축에 의한 원근을 고려하지 않고 있기 때문이고(orthographic projection),  
정사각형이 아닌 직사각형이 그려진 이유는 우리는 현재 가로세로 모두 -1에서 1까지로 나누고 있지만  
실제 화면 픽셀은 비율이 다르기 때문이다. (만약 창 크기를 1:1 비율로 설정하면 정사각형이 그려진다)  

이 문제는 Transformation Matrix를 사용해서 해결이 가능하며, 이는 차후 다루겠다.
