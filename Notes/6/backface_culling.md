![image](https://user-images.githubusercontent.com/63915665/171601256-303a7def-4d61-4404-8ae1-72e7abbf0a12.png)
  
앞선 글에서 정육면체 폴리곤 색상들이 어딘가 이상한 형태로 렌더링되는 것을 살펴보았다.  
그 이유는 우리가 코드에서 폴리곤들에게 색상을 입힐 때 폴리곤의 위치와 관계없이 무조건 일정한 순서로 입혔기 때문인데,  
```c++
// Game.cpp
... (생략)
void Game::ComposeFrame()
{
	const Color colors[12] = {
		Colors::White,
		Colors::Blue,
		Colors::Cyan,
		Colors::Gray,
		Colors::Green,
		Colors::Magenta,
		Colors::LightGray,
		Colors::Red,
		Colors::Yellow,
		Colors::White,
		Colors::Blue,
		Colors::Cyan
	}; // 현재로썬 큐브의 변을 그릴 방법이 없으므로 일단 각 폴리곤을 색상으로 구분함. 
	auto triangles = cube.GetTriangles();
	const Mat3 rot =
		Mat3::RotationX( theta_x ) *
		Mat3::RotationY( theta_y ) *
		Mat3::RotationZ( theta_z );
	for( auto& v : triangles.vertices )
	{
		v *= rot;
		v += { 0.0f,0.0f,offset_z };
		pst.Transform( v );
	}
	for( auto i = triangles.indices.cbegin(),
		end = triangles.indices.cend();
		i != end; std::advance( i,3 ) )
	{
		gfx.DrawTriangle( triangles.vertices[*i],triangles.vertices[*std::next( i )],triangles.vertices[*std::next( i,2 )],
						  colors[std::distance( triangles.indices.cbegin(),i ) / 3] );
	}
}
```
때문에 가장 나중에 입혀지는 Blue와 Cyan 색은 정육면체를 어떻게 회전하던 항상 렌더링 되는 것을 볼 수 있다.  

이러한 문제를 해결하는 방법은 여러가지가 있는데,  
각 폴리곤의 위치에 따라 렌더링되는 순서를 조정해 렌더링하는 Painter's algorithm이 그중 하나이다.  
(가장 카메라에 가까운 폴리곤을 가장 나중에 렌더링함으로써 앞에 있는 폴리곤이 자연스럽게 뒤를 가리는 효과를 주는 것)  
  
![image](https://user-images.githubusercontent.com/63915665/171602331-aea17de9-6d28-4fe1-b607-eaac764bddcb.png)  
![image](https://user-images.githubusercontent.com/63915665/171602449-a60b9ae9-889b-40a2-93bc-443cdb8d7ee0.png)  
  
이번 글에서는 Painter's algorithm 대신 Backface culling을 다뤄볼 텐데, 원리는 굉장히 단순하다.  
우리가 폴리곤으로 구성된 오브젝트를 볼 때(렌더링할때)  
실제로 필요한 부분은 오브젝트 표면에 있는 "오브젝트 바깥쪽" 폴리곤들에 한정되고,
또 그 바깥쪽 폴리곤들 중에서도 카메라를 향하고 있는 폴리곤들로 더 좁혀진다는 성질을 이용한 방식이다.

위 사진에서 오브젝트 바깥쪽 폴리곤들은 총 5개가 있는데, 그 실제로 필요한 폴리곤은 세 개밖에 되지 않는다는 것을 알 수 있다.  
(편의상 2D로 나타낸 것이며, 실제로는 3D 오브젝트라고 생각하자)  
  
![image](https://user-images.githubusercontent.com/63915665/171603028-5721fdc9-fccd-45da-9873-b1137134bc53.png)  
Backface culling은 한 가지 문제가 있는데, concave shape(오목한 형태)의 경우 바깥쪽 면들끼리도 서로를 가릴 수 있기 때문에 이 방식이 완벽하다고 말할 수는 없다.  
이러한 예외 경우는 일단은 무시하고, Backface culling을 어떻게 구현할 지를 먼저 살펴보자.  
  
어떤 폴리곤이 렌더링되어야 하는지를 결정하기 위해서는 해당 폴리곤의 normal vector와 위치를 모두 고려해주어야 한다.  
![image](https://user-images.githubusercontent.com/63915665/171603704-eb94b3b3-43f7-4622-8953-5ba2bcf6f3d0.png)
  
위 사진의 경우, 두 폴리곤의 normal vector는 동일하지만 좌측 폴리곤만이 렌더링되어야 함을 알 수 있다.
이를 판정하는 방법은 다음과 같다.  
![image](https://user-images.githubusercontent.com/63915665/171604011-2d52b515-35df-4564-9d09-58d5b2e7fb81.png)  
좌측 폴리곤의 경우 viewing vector(빨간색)과 normal vector를 내적한 값이 음수이지만 (둔각)  
![image](https://user-images.githubusercontent.com/63915665/171604185-3976addb-ceaf-4a59-85d9-2cd07393f66b.png)  
우측 폴리곤의 경우 내적한 값이 양수임을 볼 수 있다. (예각)  
  
![image](https://user-images.githubusercontent.com/63915665/171604436-450dbf0d-78fa-434e-92c2-10f1ef4e5d7a.png)  
즉, 우리는 viewing vector와 normal vector를 내적해 봄으로써 어떤 폴리곤을 렌더링해야 하는지 판정할 수 있다.  
  
그렇다면 viewing vector와 normal vector(법선벡터)를 어떻게 구할까?  
viewing vector는 쉽게 구할 수 있는데, 카메라의 focal point(초점)에서 폴리곤 위의 아무 점까지 이은 벡터이다.  
폴리곤의 normal vector의 경우 외적(Cross product)을 사용해 구할 수 있다.  
폴리곤을 구성하는 세 점의 위치가 주어졌을 때, 세 점을 두 개씩 묶어 벡터 두개를 만들 수 있는데,  
이렇게 만든 벡터를 외적하면 폴리곤 면으로부터 수직인 normal vector를 구할 수 있다.  
(엄밀히 말하면 폴리곤의 normal vector는 이렇게 얻은 벡터를 본인 크기로 나눠 크기를 1로 만들어주어야 한다.  
자세한건 셰이딩을 다룰 때 다시 다루겠다.)  
  
![image](https://user-images.githubusercontent.com/63915665/171605716-bc570a92-7027-4dc0-a0fb-429591c0d4fb.png)  
이 때 주의해야 할 것은 법선벡터의 방향이 left-hand rule이냐 right-hand rule이냐에 따라 다르다는 것인데,  
수학에서는 보통 right-hand rule을 사용하지만 Dx3D에서는 left-hand rule을 사용한다는 것에 유의해야 한다.  
여기서는 위 사진과 같은 left-hand rule을 사용한다.  (참고: right-hand rule 시각화 [링크](https://mathinsight.org/cross_product))
  
![image](https://user-images.githubusercontent.com/63915665/171606499-df54277c-bfd0-40a1-8017-7d9395b81b1f.png)  
  
v X w  
= (vy * wz - vz * wy, vz * wx - vx * wz, vx * wy - vy * wx)  
  
|v X w|  
= |v| |w| sin 세타  
(주의: 이 식으로는 외적의 크기만을 알아낼 수 있다.)  
  
외적 관련 [블로그 글](https://gamesmith.tistory.com/127) 참고.  
  
  
코드와 함께 살펴보자.  
```c++
//Vec3.h
_Vec3	operator%( const _Vec3& rhs ) const //chili는 외적 연산자로 X랑 모양이 비슷한 %를 사용하긴 했으나 권장되는 방식은 아니다.
	{
		return _Vec3(
			y * rhs.z - z * rhs.y,
			z * rhs.x - x * rhs.z,
			x * rhs.y - y * rhs.x );
	}
```
  
```c++
//IndexedTriangleList.h
#pragma once

#include <vector>
#include "Vec3.h"

struct IndexedTriangleList
{
	IndexedTriangleList( std::vector<Vec3> verts_in,std::vector<size_t> indices_in )
		:
		vertices( std::move( verts_in ) ),
		indices( std::move( indices_in ) )
	{
		assert( vertices.size() > 2 );
		assert( indices.size() % 3 == 0 );
		cullFlags.resize( indices.size() / 3,false ); // 점의 갯수 / 3만큼의 크기(=삼각형 갯수)로 초기화
	}
	std::vector<Vec3> vertices;
	std::vector<size_t> indices;
	std::vector<bool> cullFlags; // 어떤 삼각형이 culling 되어야 하는지를 나타내는 값
};
```
  
```c++
// ... (생략)
void Game::ComposeFrame()
{
	const Color colors[12] = {
		Colors::White,
		Colors::Blue,
		Colors::Cyan,
		Colors::Gray,
		Colors::Green,
		Colors::Magenta,
		Colors::LightGray,
		Colors::Red,
		Colors::Yellow,
		Colors::White,
		Colors::Blue,
		Colors::Cyan
	};
  
	// generate indexed triangle list
	auto triangles = cube.GetTriangles();
  
	// generate rotation matrix from euler angles
	const Mat3 rot =
		Mat3::RotationX( theta_x ) *
		Mat3::RotationY( theta_y ) *
		Mat3::RotationZ( theta_z );
    
	// transform from model space -> world (/view) space
	for( auto& v : triangles.vertices )
	{
		v *= rot;
		v += { 0.0f,0.0f,offset_z };
	}
  
	// backface culling test (must be done in world (/view) space)
	for( size_t i = 0,
		 end = triangles.indices.size() / 3;
		 i < end; i++ )
	{
		const Vec3& v0 = triangles.vertices[triangles.indices[i * 3]];
		const Vec3& v1 = triangles.vertices[triangles.indices[i * 3 + 1]];
		const Vec3& v2 = triangles.vertices[triangles.indices[i * 3 + 2]];
		triangles.cullFlags[i] = (v1 - v0) % (v2 - v0) * v0 > 0.0f; // 외적해서 얻은 법선벡터와 v0(viewing vector)를 내적(*)해서 얻은 값을 0과 비교
    // 이때 삼각형 점의 위치를 나타내는 v0값을 바로 viewing vector로 쓸 수 있는 이유는 카메라의 focal point가 0,0,0이기 때문이다.
	}
  
	// transform to screen space (includes perspective transform)
	for( auto& v : triangles.vertices )
	{
		pst.Transform( v );
	}
  
	// draw the triangles
	for( size_t i = 0,
		 end = triangles.indices.size() / 3;
		 i < end; i++ )
	{
		// skip triangles previously determined to be back-facing
		if( !triangles.cullFlags[i] )
		{
			gfx.DrawTriangle( 
				triangles.vertices[triangles.indices[i * 3]],
				triangles.vertices[triangles.indices[i * 3 + 1]],
				triangles.vertices[triangles.indices[i * 3 + 2]],
				colors[i] );
		}
	}
}
```



