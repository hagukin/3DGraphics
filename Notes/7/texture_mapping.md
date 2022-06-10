텍스쳐를 사용하는 이유는 우리가 현실적으로 질감을 이루는 사물의 표면들을 폴리곤만으로 다 모델링해 나타내기가 어렵기 때문이다.  
때문에 평면에 이미지를 입히는 것으로 질감 및 표면의 외형을 나타낸다.  

텍스쳐를 폴리곤들의 표면에 입히기 위해 텍스쳐 맵핑을 사용하는데, 텍스쳐 맵핑의 원리를 알아보기 전에 어떻게 하면 안되는지를 먼저 알아보자.  
2D 게임을 만든다고 가정해보자. 어떤 스프라이트 이미지 파일을 가지고 있는데, 이걸 화면 상에서 확대시키려면 어떻게 해야 할까?  
![image](https://user-images.githubusercontent.com/63915665/172857061-b6c7d7f3-544c-4733-bb33-b14a2b17a260.png)
  
![image](https://user-images.githubusercontent.com/63915665/172857107-5191fbcb-921a-4c7b-9cb7-27dfae865c66.png)
  
스프라이트(이미지) 자체를 선형사상을 사용해 확대시키면 될 것 같지만, 실제로 이런 방식으로 구현하게 되면 위 사진처럼 이미지 사이사이 빈 공간이 생기게 된다.  
이는 스프라이트(이미지)의 픽셀의 수는 한정되어 있음에도 이를 억지로 더 넓은 범위의 픽셀로 확대시키려고 했기 때문이다. 즉 이는 잘못된 방법이다.  
  
---
  
그렇다면 올바른 방법은 무엇일까?  
어떤 폴리곤이 주어졌다고 하자. 이 폴리곤 위에 입히고 싶은 이미지(텍스쳐) 또한 주어졌다고 하자.  
![image](https://user-images.githubusercontent.com/63915665/172858964-3ff84253-6139-4b67-9642-0ce436350559.png)  
이 때 이 텍스쳐 상의 점들(uv coordinates)을 폴리곤 표면 상의 점들과 맵핑해주는 과정이 필요하다.  
이를 구현하기 위해 우선 텍스쳐 상에 필요한 수만큼의 기준점들을 찍고, 그 기준점들을 폴리곤의 꼭짓점들과 대응시켜준다. (즉 텍스쳐 상의 파란 기준점은 폴리곤의 파란 꼭짓점과 대응된다)  
이렇게 기준점을 찍었다면, 이제 텍스쳐 위의 점(픽셀)들을 각 기준점들에 대한 상대적인 위치관계로 나타내어 그 위치관계를 그대로 폴리곤에 가져와 폴리곤의 위에 그려내면 된다.  
  
우선 기준점을 찍어보자.  
위 사진 예시에서는 점이 4개가 찍혀있는 것을 볼 수 있는데, 이는 우리가 예전에 rasterization을 다뤘을 때처럼 삼각형을 flat top, flat bottom으로 나눠야 하기 때문이다. 
그래야 x,y축 좌표계에서 축을 따라 값을 증가시키며 iterate가 가능하다.  
    
기준점을 찍었다면, 우리는 이제 폴리곤 표면의 픽셀 하나하나를 iterate 할 것이다.  
iterate하기 위해서 우리는 세 점의 interpolation을 사용할 것이다.  
(정확히 말하면 삼각형을 flat top과 flat bottom으로 나눴으므로 세 점의 interpolation을 두 번 진행할 것이다)  
  
![image](https://user-images.githubusercontent.com/63915665/172861555-90bce1a6-016c-48a0-ab14-b83297228f55.png)  
우선 flat top 삼각형에 속하는 픽셀들부터 iterate한다고 가정하면, 위 사진처럼 빨강,하늘,연두색 점들 사이를 iterate한다고도 해석할 수 있다.  
점 ABC가 있을 때 이 세 점 사이를 iterate하는 방법은 다음과 같다. 우선 점 A와 B를 iterate하는 점 M, 그리고 점 A와 C를 iterate하는 점 N을 잡는다. 그 후 점 M과 점 N 사이를 iterate하면 된다.  
  
![image](https://user-images.githubusercontent.com/63915665/172863532-3878432f-63c3-4cf6-ab0c-ccd196f1eda4.png)
이걸 우리가 가진 flat top 삼각형에 적용해보면, 위 사진처럼 하얀 선(scanline)을 그려나가면서 그 scanline 위의 점들을 iterate하는 꼴이 되는 것을 알 수 있다.  
이 과정을 그대로 텍스쳐의 uv 좌표들에도 적용하면서 텍스쳐 이미지 픽셀들도 스캔해나가면서(흰색 화살표 방향), 같은 상대적 위치의 점으로 픽셀을 칠하면 텍스쳐를 폴리곤 위에 그려낼 수 있다.  
    
  
텍스쳐 맵핑 과정을 실제로 코드로 나타내기 전에, pre-stepping이라는 개념을 먼저 짚고 넘어가자.  
![image](https://user-images.githubusercontent.com/63915665/172863843-336f6324-05c1-404a-8599-43b32e315739.png)
  
![image](https://user-images.githubusercontent.com/63915665/172864693-500da02e-c861-4e60-98f8-96248139fa91.png)
  
우리가 픽셀을 그릴 때 우리는 항상 픽셀의 중심을 기준으로 판단한다. (rasterization에서도 그러했듯이)  
때문에 텍스쳐를 폴리곤 상에 그릴 때도, 우리는 폴리곤의 꼭짓점 위치에서부터 interpolation을 시작하며 그리는 게 아니라, 폴리곤으로부터 약간 떨어진 픽셀의 중심지점에서부터 시작해야 한다. (pre-stepping)  
텍스쳐도 마찬가지이다. (사진의 X지점이 아니라 O지점에서부터 스캔을 시작해야 한다)  
    
---
  
코드를 살펴보자. (커밋 17d036beacacf96f1e48fc146ac29bd8aa64059d)  
살펴보기 전에 rasterization의 내용을 다시 살펴보는 것이 권장된다.  
우선 텍스쳐와 폴리곤 좌표를 맵핑하기 위해 TexVertex라는 새 오브젝트 타입을 정의해준다.  
```c++
// TexVertex.h
#pragma once

#include "Vec2.h"
#include "Vec3.h"

class TexVertex
{
public:
	TexVertex( const Vec3& pos,const Vec2& tc )
		:
		pos( pos ),
		tc( tc )
	{}
	TexVertex InterpolateTo( const TexVertex& dest,float alpha ) const
	{
		return{
			pos.InterpolateTo( dest.pos,alpha ), // Vec3.InterpolateTo()
			tc.InterpolateTo( dest.tc,alpha ) // Vec2.InterpolateTo()
		};
	}
	TexVertex& operator+=( const TexVertex& rhs )
	{
		pos += rhs.pos;
		tc += rhs.tc;
		return *this;
	}
	TexVertex operator+( const TexVertex& rhs ) const
	{
		return TexVertex( *this ) += rhs;
	}
	TexVertex& operator-=( const TexVertex& rhs )
	{
		pos -= rhs.pos;
		tc -= rhs.tc;
		return *this;
	}
	TexVertex operator-( const TexVertex& rhs ) const
	{
		return TexVertex( *this ) -= rhs;
	}
	TexVertex& operator*=( float rhs )
	{
		pos *= rhs;
		tc *= rhs;
		return *this;
	}
	TexVertex operator*( float rhs ) const
	{
		return TexVertex( *this ) *= rhs;
	}
	TexVertex& operator/=( float rhs )
	{
		pos /= rhs;
		tc /= rhs;
		return *this;
	}
	TexVertex operator/( float rhs ) const
	{
		return TexVertex( *this ) /= rhs;
	}
public:
	Vec3 pos;
	Vec2 tc;
};
```
  
이렇게 정의한 TexVertex를 프레임워크 상에서 기존 Vertex(Vec3) 대신 사용할 수 있게 프레임워크를 수정해준다.  
  
![image](https://user-images.githubusercontent.com/63915665/172866291-6c476218-8a88-48ae-b82f-493475a3bba1.png)  
IndexedTriangleList에서 Vec3 대신 템플릿을 사용해 TexVertex를 사용할 수 있게 한다.  
![image](https://user-images.githubusercontent.com/63915665/172866489-520b7023-4e9f-45f0-8842-17260124f03b.png)  
이에 맞게 Cube도 수정해준다. 이때 tc.emplace_back()이 된 것을 볼 수 있는데, 이는 uv 좌표와 관련되있다. (이후 후술)  
![image](https://user-images.githubusercontent.com/63915665/172867020-f57d46c5-a5dd-46ec-b1d1-56461b9628eb.png)  
Cube에 GetTriangle()과 별도로 GetTriangleTex()라는 함수도 추가해준다.  
  
  
Graphics.h/cpp 또한 수정해준다.  
![image](https://user-images.githubusercontent.com/63915665/172867549-e629fb78-ffdc-4678-86f9-fe8abf0f71b4.png)  
텍스쳐를 실제로 렌더링하는 부분은 Graphics::DrawTriangleTex() 함수에서 이뤄지는데, 코드와 함께 살펴보자.  
(기존의 DrawTriangle과 거의 완전히 동일하다, 다른 부분에만 별도의 주석을 적어두었다)
``` c++
void Graphics::DrawTriangleTex( const TexVertex& v0,const TexVertex& v1,const TexVertex& v2,const Surface& tex )
{
	// using pointers so we can swap (for sorting purposes)
	const TexVertex* pv0 = &v0;
	const TexVertex* pv1 = &v1;
	const TexVertex* pv2 = &v2;

	// sorting vertices by y
  // DrawTriangle과 사실상 동일. 오브젝트 형태가 변했으므로 y대신 pos.y를 사용한다.
	if( pv1->pos.y < pv0->pos.y ) std::swap( pv0,pv1 );
	if( pv2->pos.y < pv1->pos.y ) std::swap( pv1,pv2 );
	if( pv1->pos.y < pv0->pos.y ) std::swap( pv0,pv1 );

	if( pv0->pos.y == pv1->pos.y ) // natural flat top
	{
		// sorting top vertices by x
		if( pv1->pos.x < pv0->pos.x ) std::swap( pv0,pv1 );
		DrawFlatTopTriangleTex( *pv0,*pv1,*pv2,tex );
	}
	else if( pv1->pos.y == pv2->pos.y ) // natural flat bottom
	{
		// sorting bottom vertices by x
		if( pv2->pos.x < pv1->pos.x ) std::swap( pv1,pv2 );
		DrawFlatBottomTriangleTex( *pv0,*pv1,*pv2,tex );
	}
	else // general triangle
	{
		// find splitting vertex
		const float alphaSplit =
			(pv1->pos.y - pv0->pos.y)
			(pv2->pos.y - pv0->pos.y);
		const TexVertex vi = pv0->InterpolateTo( *pv2,alphaSplit ); // 기존에는 vi = *pv0 + (*pv2 - *pv0) * alphaSplit; 을 사용해 직접 벡터 계산식을 적었다.
    // 여기서는 TexVertex를 사용중이므로 미리 만들어둔 함수를 사용한다.

		if( pv1->pos.x < vi.pos.x ) // major right
		{
			DrawFlatBottomTriangleTex( *pv0,*pv1,vi,tex );
			DrawFlatTopTriangleTex( *pv1,vi,*pv2,tex );
		}
		else // major left
		{
			DrawFlatBottomTriangleTex( *pv0,vi,*pv1,tex );
			DrawFlatTopTriangleTex( vi,*pv1,*pv2,tex );
		}
	}
}
```

하지만 DrawFlatTopTriangleTex()의 경우 DrawFlatTopTriangle()과 다소 차이가 있는데, 코드로 살펴보자.  
![image](https://user-images.githubusercontent.com/63915665/172870757-9d98e5b0-a5ad-49ca-ae9a-e13a83f7e165.png)  
<사진 1>  
  
![image](https://user-images.githubusercontent.com/63915665/172873501-41e7c211-5cd3-4eed-ac0e-10dac0a5cf4e.png)  
![image](https://user-images.githubusercontent.com/63915665/172873576-6f978cb4-5ebd-4a15-a773-b1cfbf9f6e6e.png)  
<사진 2, 3>  
  
![image](https://user-images.githubusercontent.com/63915665/172874848-b98cafd2-7e20-4ba7-a954-806f50f8965e.png)  
<사진 4>  
  
``` c++
void Graphics::DrawFlatTopTriangleTex( const TexVertex& v0,const TexVertex& v1,const TexVertex& v2,const Surface& tex )
{
/*
A. 좌변과 우변을 각각 iterate하면서 scanline의 좌측점과 우측점을 구해준다.
*/
	// calulcate slopes in screen space
	const float m0 = (v2.pos.x - v0.pos.x) / (v2.pos.y - v0.pos.y);
	const float m1 = (v2.pos.x - v1.pos.x) / (v2.pos.y - v1.pos.y);

	// calculate start and end scanlines
	const int yStart = (int)ceil( v0.pos.y - 0.5f );
	const int yEnd = (int)ceil( v2.pos.y - 0.5f ); // the scanline AFTER the last line drawn

	// init tex coord edges
	Vec2 tcEdgeL = v0.tc; // FlatTop이므로 텍스쳐 사진 상의 하늘색 점에 해당한다.
	Vec2 tcEdgeR = v1.tc; // 텍스쳐의 연두색 점에 해당한다.
	const Vec2 tcBottom = v2.tc; // 텍스쳐의 파란색 점에 해당한다.

	// calculate tex coord edge unit steps
	const Vec2 tcEdgeStepL = (tcBottom - tcEdgeL) / (v2.pos.y - v0.pos.y); 
  // 하늘색에서 파란색까지의 텍스쳐 상에서의 거리 변화를 폴리곤 상에서의 거리 변화로 나눈 값으로,
  // 사진에서 폴리곤쪽에 있는 빨간 화살표 한 칸(거리1) 당 텍스쳐에서는 점과 점 사이를 따라 얼마만큼의 거리를 이동하는지를 나타낸다.
  
	const Vec2 tcEdgeStepR = (tcBottom - tcEdgeR) / (v2.pos.y - v1.pos.y);
  // 마찬가지로 연두~파랑도 해준다.

	// do tex coord edge prestep
  // 이렇게 구한 값들로 prestep을 처리해준다. yStart는 폴리곤 픽셀의 첫 위치, 즉 픽셀 의 y값이다.
	tcEdgeL += tcEdgeStepL * (float( yStart ) + 0.5f - v1.pos.y); 
  // (float( yStart ) + 0.5f - v1.pos.y) 는 꼭짓점과 픽셀 중앙까지의 거리 차이를 반환한다. (사진2 하얀 화살표)
  // 왜 0.5를 더하는가에 대해서는 rasterization에서 우리가 어떻게 yStart를 정의했는가를 살펴보면 알 수 있다.
  // 여기에 tcEdgeStepL을 곱해주어 폴리곤 단위에서 텍스쳐 단위로 크기를 변경해준 뒤,
  // tcEdgeL에 이 값을 더해주면 텍스쳐의 시작 좌표를 prestep된 곳(폴리곤에서 픽셀의 센터에 해당하는 곳)으로 지정할 수 있다. (사진3)
  
	tcEdgeR += tcEdgeStepR * (float( yStart ) + 0.5f - v1.pos.y);

	// init tex width/height and clamp values
	const float tex_width = float( tex.GetWidth() );
	const float tex_height = float( tex.GetHeight() );
	const float tex_clamp_x = tex_width - 1.0f;
	const float tex_clamp_y = tex_height - 1.0f;

	for( int y = yStart; y < yEnd; y++,
		 tcEdgeL += tcEdgeStepL,tcEdgeR += tcEdgeStepR ) 
     // 매 루프마다 세 값을 increment한다. 
     // 폴리곤을 iterate하면서 동시에 텍스쳐도 iterate해야 하기 때문이다.
     // 이는 텍스쳐의 시작지점인 tcEdgeL/R에 tcEdgeStepL/R을 더해주는 것으로 이뤄진다. (사진 4)
	{
  /*
  B. 매 루프마다 이번엔 이렇게 잡은 좌측점과 우측점 사이를 iterate해준다. (scanline 위의 점들을 iterate)
     전체적으로 A 과정과 유사한 것을 볼 수 있는데, 이는 실제로 원리가 같기 때문이다.
     (폴리곤과 텍스쳐에서 동시에 어떤 두 점 사이를 interpolate하기 위해 폴리곤에서의 거리를 텍스쳐 상에서의 거리로 변환시켜주고 픽셀들을 iterate)
  */
		// caluclate start and end points (x-coords)
		// add 0.5 to y value because we're calculating based on pixel CENTERS
		const float px0 = m0 * (float( y ) + 0.5f - v0.pos.y) + v0.pos.x;
		const float px1 = m1 * (float( y ) + 0.5f - v1.pos.y) + v1.pos.x;

		// calculate start and end pixels
		const int xStart = (int)ceil( px0 - 0.5f );
		const int xEnd = (int)ceil( px1 - 0.5f ); // the pixel AFTER the last pixel drawn

		// calculate tex coord scanline unit step
		const Vec2 tcScanStep = (tcEdgeR - tcEdgeL) / (px1 - px0);

		// do tex coord scanline prestep
		Vec2 tc = tcEdgeL + tcScanStep * (float( xStart ) + 0.5f - px0);

		for( int x = xStart; x < xEnd; x++,tc += tcScanStep )
		{
			PutPixel( x,y,tex.GetPixel( 
				int( std::min( tc.x * tex_width,tex_clamp_x ) ),
				int( std::min( tc.y * tex_height,tex_clamp_y ) ) ) );
			// need std::min b/c tc.x/y == 1.0, we'll read off edge of tex
			// and with fp err, tc.x/y can be > 1.0 (by a tiny amount)
		}
	}
}
```

TODO 20:11





