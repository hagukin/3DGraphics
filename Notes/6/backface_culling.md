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
  
![image](https://user-images.githubusercontent.com/63915665/171610350-8763df04-2006-4d34-ba2e-b2e930d0578d.png)  
실행해보면 정상 동작하는 모습을 볼 수 있다.  
  
![image](https://user-images.githubusercontent.com/63915665/171611056-30634a9a-b383-40e1-950c-b65d8aa3b20a.png)  
외적을 코드로 구현할 때 한 가지 중요한 내용이 있는데,  
바로 삼각형의 정점의 index 순서이다.  
  
![image](https://user-images.githubusercontent.com/63915665/171611703-8b017083-39cd-4d0e-854f-28592fc0aba3.png)  
우리는 left-hand rule을 사용중이므로 마찬가지로 여기서는 left-hand-winding rule을 사용하는데, 간단하게 설명하면  
삼각형의 "앞면"이 엄지 방향일 때 삼각형 정점들은 나머지 손가락들이 꺾이는 방향 순서대로 나타낸다는 규칙이다.  
  
위 사진에서 우리가 법선벡터를 구하기 위해 (v1-v0) X (v2-v0) 이라는 공식을 쓴다고 해 보자.  
이때 삼각형의 어떤 정점에 어떤 번호를 붙여주느냐에 따라 좌측 삼각형과 우측 삼각형의 외적 벡터의 방향이 반대가 되는 것을 볼 수 있는데,  
때문에 우리는 항상 left-hand-winding rule을 사용한다고 통일함으로써 폴리곤이 투명하게 렌더링되는 불상사를 방지한다.  
  
우리 코드에서도 이미 이러한 규칙을 적용해 사용하고 있는데,  
```c++
//Cube.h
IndexedTriangleList GetTriangles() const
{
	return{
		vertices,{
		0,2,1, 2,3,1,
		1,3,5, 3,7,5,
		2,6,3, 3,6,7,
		4,5,7, 4,7,6,
		0,4,2, 2,4,6,
		0,1,4, 1,5,4 }
	};
}
```
![image](https://user-images.githubusercontent.com/63915665/171612005-26f25f1b-9a48-4051-84ba-47807c10979e.png)  
이 코드에서 정점 세 개로 폴리곤을 표현할 때   
각 정점의 순서는 위 사진의 정점들의 번호를 left-hand-winding rule을 이용해 나타낸 것이라는 것을 알 수 있다.  
(큐브 바깥쪽으로 나가는 폴리곤을 렌더링하는 것이므로 엄지를 큐브 바깥쪽으로 둔 채 0,1,2점을 정렬하면 0,2,1 순인 것을 알 수 있다.)  
  
  
마무리짓기 전에, 현재 구현한 backface culling이 어떤 문제점들을 가지고 있는지 알아보자.  
(편의를 위해 프레임워크 단에서 Scene이라는 객체를 추가해 사용한다. 게임씬과 유사한 개념으로, 코드를 보면 크게 어려운 내용은 없다.)  
```c++
// Scene.h
#pragma once
#include "Keyboard.h"
#include "Mouse.h"
#include "Graphics.h"

class Scene
{
public:
	virtual void Update( Keyboard& kbd,Mouse& mouse,float dt ) = 0;
	virtual void Draw( Graphics& gfx ) const = 0;
	virtual ~Scene() = default;
};
```
  
씬 예제:  
```c++
// SolidCubeScene.h
// 기존의 큐브 하나만 있는 환경을 씬 형태로 묶은 것으로, 앞에서 다룬 코드들과 완전히 동일하다.
#pragma once

#include "Scene.h"
#include "Cube.h"
#include "PubeScreenTransformer.h"
#include "Mat3.h"

class SolidCubeScene : public Scene
{
public:
	SolidCubeScene() = default;
	virtual void Update( Keyboard& kbd,Mouse& mouse,float dt ) override
	{
		if( kbd.KeyIsPressed( 'Q' ) )
		{
			theta_x = wrap_angle( theta_x + dTheta * dt );
		}
		if( kbd.KeyIsPressed( 'W' ) )
		{
			theta_y = wrap_angle( theta_y + dTheta * dt );
		}
		if( kbd.KeyIsPressed( 'E' ) )
		{
			theta_z = wrap_angle( theta_z + dTheta * dt );
		}
		if( kbd.KeyIsPressed( 'A' ) )
		{
			theta_x = wrap_angle( theta_x - dTheta * dt );
		}
		if( kbd.KeyIsPressed( 'S' ) )
		{
			theta_y = wrap_angle( theta_y - dTheta * dt );
		}
		if( kbd.KeyIsPressed( 'D' ) )
		{
			theta_z = wrap_angle( theta_z - dTheta * dt );
		}
		if( kbd.KeyIsPressed( 'R' ) )
		{
			offset_z += 2.0f * dt;
		}
		if( kbd.KeyIsPressed( 'F' ) )
		{
			offset_z -= 2.0f * dt;
		}
	}
	virtual void Draw( Graphics& gfx ) const override
	{
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
			triangles.cullFlags[i] = (v1 - v0) % (v2 - v0) * v0 > 0.0f;
		}
		// transform to screen space (includes perspective transform)
		for( auto& v : triangles.vertices )
		{
			pst.Transform( v );
		}
		// draw the mf triangles!
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
private:
	PubeScreenTransformer pst;
	Cube cube = Cube( 1.0f );
	static constexpr Color colors[12] = {
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
	static constexpr float dTheta = PI;
	float offset_z = 2.0f;
	float theta_x = 0.0f;
	float theta_y = 0.0f;
	float theta_z = 0.0f;
};
```
코드가 너무 길어지니까 씬 코드를 다 기입하지는 않겠지만, 우리의 backface culling의 문제점을 아래 씬들을 통해 확인할 수 있다.  
  
SolidCubeScene.h  
![image](https://user-images.githubusercontent.com/63915665/171613578-1b62b1a9-8d8a-4f07-a6f3-95771c417e5d.png)  
큐브 하나짜리 씬으로, 아무 문제 없이 동작한다.  
  
CubeOrderScene.h  
![image](https://user-images.githubusercontent.com/63915665/171613658-7ce4ab94-6351-4278-b852-d9d5f89a4668.png)  
큐브 두개짜리 씬으로, 큐브 하나를 멀리 이동시켰는데도 가려지지 않고 계속 렌더링 되는 것을 볼 수 있다.  
  
ConcaveHexahedron.h (ConHexWireScene.h로 색상을 입히지 않은 wireframe 형태로 볼 수 있다)  
![image](https://user-images.githubusercontent.com/63915665/171613767-8cffadbb-b74b-4c0d-bcef-fc5c0a55c0e0.png)  
![image](https://user-images.githubusercontent.com/63915665/171614033-db9b9953-2d3a-4417-9138-503ccc651f30.png)  
글 초반에 언급한 오목한 형태의 오브젝트로, 폴리곤이 잘못 렌더링되는 문제를 볼 수 있다.  

XMutualScene.h  
![image](https://user-images.githubusercontent.com/63915665/171614161-afa19c09-6652-471c-b3fa-c7873ad7191c.png)  
서로 겹쳐있는 사각형으로, 폴리곤이 서로 겹쳐있을 경우 잘못 렌더링되는 문제를 볼 수 있다.  
이 문제는 backface culling 및 painter's algorithm으로도 해결할 수 없다! (삼각형을 거리순으로 정렬하더라도 서로 겹쳐있으면 한쪽이 한쪽을 가리게 되어 있으므로)  
이런일이 일어나는 걸 막겠답시고 폴리곤(삼각형)이 서로 아예 intersect하지 못하게 만들어버리더라도,  
![image](https://user-images.githubusercontent.com/63915665/171614461-7194842e-654c-40bd-b22d-a3998fca418c.png)  
다음과 같은 형태처럼 경우 서로 intersect하지는 않지만 서로가 서로를 일부분 가리는 도형의 경우 여전히 painter's algorithm으로 제대로 렌더링을 할 수 없다.  
즉, backface culling이나 painter's algorithm, 혹은 폴리곤을 sorting하는 방법으로는 모든 렌더링 문제(특히 overlap problem)를 해결할 수 없고, 결국 이것만 가지고는 제대로 된 occlusion을 구현할 수 없음을 보여준다.  
  
그렇다면 제대로 occlusion을 구현하는 방법은 무엇일까?  
이는 나중에 다시 다뤄보도록 하겠다.  
