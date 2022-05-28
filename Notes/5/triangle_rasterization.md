  
Occlusion은 앞에 있는 물체가 뒤에 있는 물체를 가리는 것을 말한다. 이 Occlusion이 있어야 우리는 어떤 공간을 쉽게 인식할 수 있다.  
![image](https://user-images.githubusercontent.com/63915665/170813920-12dee988-c4cc-408c-a34b-deb68c7df64e.png)  
Occlusion이 없이 지금처럼 wireframe rendering만 한다면 위 그림에서 빨간 큐브와 하얀 큐브 중 어떤게 더 앞에 있는 지를 알 수 없을 것이다.  
때문에 Occlusion이 포함된 Solid rendering은 굉장히 중요하다.  
  
이를 구현하기 위해 우리는 폴리곤을 사용한다.  
(모든 다각형은 여러 개의 작은 삼각형들로 분해할 수 있으므로, 컴퓨터그래픽스에서는 삼각형들로 모든 물체들을 구상해 그려낸다.)  
폴리곤의 각 verticies들로부터 선들을 그린 후 그 사이의 픽셀들의 색을 채우는 방식으로 어떤 면을 그릴 수 있다.  
![image](https://user-images.githubusercontent.com/63915665/170815553-f7915a02-47fd-4cd7-82ea-911bebf7a043.png)  
  
이렇게 폴리곤이 차지하는 픽셀에 색을 채워넣는 과정을 rasterization이라고 한다.  
그중 가장 널리 사용되는 lines rasterization을 살펴보자.  
  

  
<h2>Lines rasterization (=scanline rasterization)</h2>
  
어떤 폴리곤이 주어졌을 때 그 폴리곤이 차지하는 픽셀을 찾기 위해 line rasterization에서는 삼각형의 가장 작은 y좌표부터 가장 큰 y좌표까지 for loop을 돌면서 스캔한다.  
(주의: 2d 좌표계 좌상단이 0,0이라는 점 잊지 말 것)  
![image](https://user-images.githubusercontent.com/63915665/170815712-e8bd0740-4ce4-45a7-9427-033a71a548c5.png)  
이러한 방식 때문에 이 과정은 scanline conversion이라고도 불리운다.  
  
하지만 만약 삼각형의 한 변이 x축에 평행하지 않다면 어떨까?  
![image](https://user-images.githubusercontent.com/63915665/170815739-02590a90-2297-4776-84e3-3dca2d2fddc3.png)  
삼각형의 옆쪽 변(위 사진의 경우 좌측변)을 나타내는 직선 수식이 어느 지점 이후부터는 변해버리므로 어느 x점부터 스캔해야 하는지를 결정하는 게 난감해진다.  
쉽게 말해 좌우측에 변이 하나만 있을 때는 어떤 x범위를 확인해야 하는지가 명확했는데, 이렇게 한 쪽에 변이 두 개가 되어버리는 순간 x값을 하나의 식으로 나타낼 수 없게 되므로 난감해진다.  
  
이를 해결하기 위해 우리는 삼각형을 두 개로 분할한다. (flat-bottom, flat-top traingle로 분할한다)  
![image](https://user-images.githubusercontent.com/63915665/170815829-efa7b403-90cc-4324-9bc7-9421a19b41b4.png)  
이렇게 하는 것으로 이전에 발생했던 문제를 없앨 수 있다.  
구체적으로 어떻게 나누는 지는 뒤에서 코드와 함께 살펴보겠다.  
  
  
  
이번에는 lines rasterization 과정에서 발생할 수 있는 또 하나의 문제를 살펴보자.  
이는 바로 어떤 폴리곤이 어떤 픽셀을 차지하느냐? 의 문제이다.  
![image](https://user-images.githubusercontent.com/63915665/170816057-79393ebb-e99c-4d5e-9329-508481e08d79.png)  
이 예시를 살펴보면, 폴리곤의 변에 걸친 모든 픽셀을 해당 폴리곤의 소유로 했을 때의 문제를 보여준다.  1
서로 맞닿아 있는 폴리곤들의 지점에 있는 픽셀들이 양쪽 폴리곤 모두에게 소유당하는 현상이 발생한다.  
이는   
1. pixel flickering 등의 문제를 발생시킬 수 있다. (매 프레임마다 어떤 픽셀을 소유한 폴리곤이 변하면서 화면 일부가 깜빡거리는 현상)  
2. 같은 픽셀을 여러 폴리곤들이 저장하게 하는 것 자체가 엄청난 메모리의 낭비이다.  
와 같은 문제를 발생시킨다.  
  
그렇다고 이를 해결하겠답시고 폴리곤 내부에 있는 픽셀만 해당 폴리곤의 소유로 하게 되면 이번에는 접경 지점에 있는 픽셀들이 비는 현상이 발생한다.
![image](https://user-images.githubusercontent.com/63915665/170816131-f74d019f-00a5-4b7e-b706-b5ae5df7f6e0.png)  
이 역시 당연히 바람직하지 않다.  
  
그럼 어떻게 해야 할까?  
어떤 픽셀이 어떤 폴리곤의 소유인지를 명확히 하는 Rasterization rule을 만들어 이를 해결한다.  
우리는 DirectX11의 Rasterization rule을 사용할 것이다. ([참고](https://docs.microsoft.com/en-us/windows/win32/direct3d11/d3d10-graphics-programming-guide-rasterizer-stage-rules))  
    
dx11의 rasterization rule을 살펴보기 전에 dx11의 픽셀 좌표 시스템을 간략하게 살펴보자.  
![image](https://user-images.githubusercontent.com/63915665/170816286-b9107a04-dc46-4721-8237-a0cd69376588.png)  
([참고](https://docs.microsoft.com/en-us/windows/win32/direct3d10/d3d10-graphics-programming-guide-resources-coordinates))  
dx11에서 픽셀 한 칸은 x,y좌표 모두 \[0,1\) 범위를 차지한다. 즉 0은 포함하되 1은 포함하지 않는다. 이를 이해했다면 다시 rasterization rule도 돌아가자.  
  
Dx11의 rasterization rule은 ***top-left rule***을 사용한다.  
![image](https://user-images.githubusercontent.com/63915665/170816387-8f84dcee-deaa-4549-988d-8d036beaa66a.png)  
자세한 내용은 위 이미지를 살펴보거나 직접 도큐먼트를 읽어보는 걸 권장하지만, 간단히 요약하면 다음과 같다.  
어떤 폴리곤을 그릴 때,  
1) 픽셀의 중앙 (.5f, .5f)이 폴리곤 내에 있다면 그 픽셀을 그린다.  
2) 픽셀의 중앙이 폴리곤의 변 위에 있다면 다음 조건 하에서 그 픽셀을 그린다.  
    2-1) 삼각형의 윗쪽 변에 픽셀의 중앙이 걸쳐있고, 그 윗쪽 변이 horizontal(x축에 평행)하면 그린다
    2-2) 삼각형의 좌측 변에 픽셀의 중앙이 걸쳐있고, 그 좌측 변이 horizontal하지 않다면 그린다. (삼각형은 좌측 변을 하나 또는 두개 가질 수 있다.)  
  
---
  
이를 코드로 어떻게 구현하는지를 살펴보자.  
```c++
// Graphics.h
/* 
우리가 살펴봐야 할 함수는 DrawTriangle, DrawFlatTopTriangle, DrawFlatBottomTriangle이다.
DrawTriangle은 삼각형을 flat-top과 flat-bottom으로 쪼개 이를 색칠(rasterize)하고 draw하는 함수이다.
만약 쪼갤 필요가 없다면 둘 중 하나가 호출 될 것이다.
색칠할 때는 PutPixel 함수를 사용한다.
*/
#pragma once
#include <d3d11.h>
#include <wrl.h>
#include "GDIPlusManager.h"
#include "ChiliException.h"
#include "Surface.h"
#include "Colors.h"
#include "Vec2.h"

#define CHILI_GFX_EXCEPTION( hr,note ) Graphics::Exception( hr,note,_CRT_WIDE(__FILE__),__LINE__ )

class Graphics
{
... (생략)
public:
	Graphics( class HWNDKey& key );
	Graphics( const Graphics& ) = delete;
	Graphics& operator=( const Graphics& ) = delete;
	void EndFrame();
	void BeginFrame();
	void DrawTriangle( const Vec2& v0,const Vec2& v1,const Vec2& v2,Color c );
	void DrawLine( const Vec2& p1,const Vec2& p2,Color c )
	{
		DrawLine( p1.x,p1.y,p2.x,p2.y,c );
	}
	void DrawLine( float x1,float y1,float x2,float y2,Color c );
	void PutPixel( int x,int y,int r,int g,int b )
	{
		PutPixel( x,y,{ unsigned char( r ),unsigned char( g ),unsigned char( b ) } );
	}
	void PutPixel( int x,int y,Color c )
	{
		sysBuffer.PutPixel( x,y,c );
	}
	~Graphics();
private:
	void DrawFlatTopTriangle( const Vec2& v0,const Vec2& v1,const Vec2& v2,Color c );
	void DrawFlatBottomTriangle( const Vec2& v0,const Vec2& v1,const Vec2& v2,Color c );
... (생략)
};
```

```c++
// Graphics.cpp
void Graphics::DrawTriangle( const Vec2& v0,const Vec2& v1,const Vec2& v2,Color c )
{
	// 1. 인풋으로 주어지는 세 점 v0, v1, v2를 y축이 작은 순서대로(화면 윗쪽에 있는 순서대로) 정렬한다.
	// using pointers so we can swap (for sorting purposes)
	const Vec2* pv0 = &v0;
	const Vec2* pv1 = &v1;
	const Vec2* pv2 = &v2;

	// sorting vertices by y
	if( pv1->y < pv0->y ) std::swap( pv0,pv1 );
	if( pv2->y < pv1->y ) std::swap( pv1,pv2 );
	if( pv1->y < pv0->y ) std::swap( pv0,pv1 );

	// 2. 이렇게 정렬한 세 점을 통해 삼각형이 어떻게 생겼는지 파악한다.
	if( pv0->y == pv1->y ) // natural flat top (윗쪽 변이 x축과 평행)
	{
		// sorting top vertices by x
		if( pv1->x < pv0->x ) std::swap( pv0,pv1 ); // x축이 작은 게 먼저 오게 정렬한다.
		DrawFlatTopTriangle( *pv0,*pv1,*pv2,c );
	}
	else if( pv1->y == pv2->y ) // natural flat bottom (아랫쪽 변이 x축과 평행)
	{
		// sorting bottom vertices by x
		if( pv2->x < pv1->x ) std::swap( pv1,pv2 ); // x축이 작은 게 먼저 오게 정렬한다.
		DrawFlatBottomTriangle( *pv0,*pv1,*pv2,c );
	}
	else // general triangle (그 외의 거의 대부분의 경우)
	{
		// find splitting vertex
		const float alphaSplit =
			(pv1->y - pv0->y) /
			(pv2->y - pv0->y);
		const Vec2 vi = *pv0 + (*pv2 - *pv0) * alphaSplit;

		if( pv1->x < vi.x ) // major right
		{
			DrawFlatBottomTriangle( *pv0,*pv1,vi,c );
			DrawFlatTopTriangle( *pv1,vi,*pv2,c );
		}
		else // major left
		{
			DrawFlatBottomTriangle( *pv0,vi,*pv1,c );
			DrawFlatTopTriangle( vi,*pv1,*pv2,c );
		}
	}
}
```
보충설명 하자면,  
```c++
else // general triangle (그 외의 거의 대부분의 경우)
	{
		// find splitting vertex
		const float alphaSplit =
			(pv1->y - pv0->y) /
			(pv2->y - pv0->y);
		const Vec2 vi = *pv0 + (*pv2 - *pv0) * alphaSplit;

		if( pv1->x < vi.x ) // major right
		{
			DrawFlatBottomTriangle( *pv0,*pv1,vi,c );
			DrawFlatTopTriangle( *pv1,vi,*pv2,c );
		}
		else // major left
		{
			DrawFlatBottomTriangle( *pv0,vi,*pv1,c );
			DrawFlatTopTriangle( vi,*pv1,*pv2,c );
		}
	}
```  
이 부분은 삼각형을 앞서 언급한, 삼각형을 두 개로 나누는 과정이다.    
![image](https://user-images.githubusercontent.com/63915665/170815875-21067785-67af-4531-a4fc-d1d131c17662.png)  
더 자세히 말하면, 삼각형을 두개로 나눴을 때의 나눠지는 지점(코드에서는 vi)을 찾는 과정이기도 하다. (그래야 스캔할 x범위를 찾을 수 있음)  
  
삼각형이 major-left인지 major-right인지를 확인하는 걸 볼 수 있다.  
![image](https://user-images.githubusercontent.com/63915665/170815943-2bdd567f-3c74-4be9-b56d-ab41f0a8615d.png)  

이를 확인하기 위해 vi라는 값을 구하는데, vi를 구하기 위해서는 알파(코드에서는 alphaSplit)라는 값을 구해야 한다.  
![image](https://user-images.githubusercontent.com/63915665/170817102-f09793cf-6d83-4135-88ba-be630cacc059.png)  
이 알파는 v0, v1, v2의 y축 거리의 비를 이용해 구하는 값인데,  
알파 = 짧은 파랑색 길이/긴 파랑색 길이  
이다.  
  
삼각형의 닮음비는 x,y 모두 성립하므로 vi의 좌표는 다음과 같은 식으로 구할 수 있다.  
vi = (1-알파)v0 + 알파v2  
  
식을 전개해서 정리하면 다음과 같다.  
vi = v0 + (v2-v0)\*알파  
이 식을 사용중인 것을 코드에서도 볼 수 있다.  
  
이제 vi.x와 v0.x를 비교함으로써 major-right, major-left를 판별할 수 있다.  ]
![image](https://user-images.githubusercontent.com/63915665/170818432-4f654713-ad6a-4fd5-bd08-731d770fc47f.png)  
  
---
  
이번엔 DrawFlatTopTriangle, DrawFlatBottomTriangle 함수를 살펴보자.  
여기서 top-left rule이 구현된다.  
```c++
void Graphics::DrawFlatTopTriangle( const Vec2& v0,const Vec2& v1,const Vec2& v2,Color c )
{
	// calulcate slopes in screen space
	float m0 = (v2.x - v0.x) / (v2.y - v0.y);
	float m1 = (v2.x - v1.x) / (v2.y - v1.y);
	/* 여기서 중요한 포인트가 하나 있는데, 
	자세히 보면 우리는 일반적인 기울기 공식 dy/dx 대신 dx/dy를 사용하고 있음을 알 수 있다. 
	이는 m0, m1은 좌, 우변의 기울기를 나타내는 값인데 
	만약 dy/dx꼴로 사용해버리게 되면 만약 변이 x축에 수직일 경우 값이 무한대가 되어버린다는 문제가 발생한다. 
	문에 모든 가능한 삼각형 옆변의 기울기를 표현하기 위해 dx/dy를 대신 사용한다. 
	(어차피 옆의 변은 x축에 평행할 수가 없기 때문에 dx/dy꼴에서는 무한대 값이 나올 수가 없다.)
	*/

	// calculate start and end scanlines (어느 좌표부터 어느 좌표까지 스캔할 지 범위를 잡는다)
	const int yStart = (int)ceil( v0.y - 0.5f );
	const int yEnd = (int)ceil( v2.y - 0.5f ); // the scanline AFTER the last line drawn
	
	// 스캔
	for( int y = yStart; y < yEnd; y++ ) // 스캔 시에는 정수값만을 사용한다. (픽셀 좌표계가 정수 격자이므로)
	{
		// caluclate start and end points (x-coords)
		// add 0.5 to y value because we're calculating based on pixel CENTERS
		// 앞서 구한 기울기값을 이용해 x의 범위를 구한다.
		const float px0 = m0 * (float( y ) + 0.5f - v0.y) + v0.x;
		const float px1 = m1 * (float( y ) + 0.5f - v1.y) + v1.x;

		// calculate start and end pixels
		const int xStart = (int)ceil( px0 - 0.5f );
		const int xEnd = (int)ceil( px1 - 0.5f ); // the pixel AFTER the last pixel drawn

		for( int x = xStart; x < xEnd; x++ )
		{
			PutPixel( x,y,c );
		}
	}
}

void Graphics::DrawFlatBottomTriangle( const Vec2& v0,const Vec2& v1,const Vec2& v2,Color c )
{
	// calulcate slopes in screen space
	float m0 = (v1.x - v0.x) / (v1.y - v0.y);
	float m1 = (v2.x - v0.x) / (v2.y - v0.y);

	// calculate start and end scanlines
	const int yStart = (int)ceil( v0.y - 0.5f );
	const int yEnd = (int)ceil( v2.y - 0.5f ); // the scanline AFTER the last line drawn

	for( int y = yStart; y < yEnd; y++ )
	{
		// caluclate start and end points
		// add 0.5 to y value because we're calculating based on pixel CENTERS
		const float px0 = m0 * (float( y ) + 0.5f - v0.y) + v0.x;
		const float px1 = m1 * (float( y ) + 0.5f - v0.y) + v0.x;

		// calculate start and end pixels
		const int xStart = (int)ceil( px0 - 0.5f );
		const int xEnd = (int)ceil( px1 - 0.5f ); // the pixel AFTER the last pixel drawn

		for( int x = xStart; x < xEnd; x++ )
		{
			PutPixel( x,y,c );
		}
	}
}
```
  
---
  
lines rasterization을 구현했으니, 실제로 이를 프레임워크 단으로 가져와 적용시켜보자.  
일단 기존에 큐브를 선들의 집합으로 표현했지만 (IndexedLineList) 이제는 폴리곤을 사용할 수 있게 되었으니 폴리곤으로 표현해보자.  
```c++
//IndexedTraingleList.h
#pragma once

#include <vector>
#include "Vec3.h"

struct IndexedTriangleList
{
	std::vector<Vec3> vertices; //좌표점
	std::vector<size_t> indices; // 각 폴리곤(삼각형)들의 인덱스들.
};
```
  
![image](https://user-images.githubusercontent.com/63915665/170818824-3e7eddae-070f-46c0-b768-dd6163010ae8.png)  
```c++
//Cube.h
#pragma once

#include "Vec3.h"
#include <vector>
#include "IndexedLineList.h"
#include "IndexedTriangleList.h"

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
			0,1,  1,3,  3,2,  2,0,
			0,4,  1,5,	3,7,  2,6,
			4,5,  5,7,	7,6,  6,4 }
		};
	}
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
		}; // 사진참고
	}
private:
	std::vector<Vec3> vertices;
};
```

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

이제 코드를 실행하면 다음과 같은 큐브가 그려진다.  
![image](https://user-images.githubusercontent.com/63915665/170818979-8f8b6780-4ecd-401a-936c-fefdf4ed7ea1.png)  
폴리곤 처리 및 래스터라이제이션이 잘 되었지만 우리가 생각했던 것과는 약간 다른 결과물이 나온 것을 알 수 있는데, 왜 이런 못브이 나오는지는 다음 시간에 다뤄보겠다.  


