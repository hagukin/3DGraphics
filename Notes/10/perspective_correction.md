![image](https://user-images.githubusercontent.com/63915665/178100329-90532864-9e4e-4aa2-9a77-1542274ba5b3.png)  
현재 우리의 파이프라인에는 한 가지 큰 문제가 있는데, 바로 텍스쳐가 뒤틀려 보이는 현상이다.  
왜 이런 현상이 발생하는 지를 이해하려면 우리가 텍스쳐를 처리할 때 어떤 식으로 접근하는 지를 다시 떠올려보면 된다. (Notes 참고) 

![image](https://user-images.githubusercontent.com/63915665/178100692-56b32caa-95da-403d-9f61-89e5102f70d3.png)  
![image](https://user-images.githubusercontent.com/63915665/178101303-b176381f-7eab-4bc4-80b4-ebbca16db83d.png)  
우리는 텍스쳐를 입힐 때 다음과 같은 과정들을 거친다. (파이프라인의 PST단계부터 살펴보자)   

1. 물체의 정점들의 3D 좌표를 원근을 고려해 변형시킨다. 
편의상 이 변형된 좌표를 원근좌표라고 하자. 원근좌표는 3D좌표에 z를 나누는 것으로 구해준다.
이러한 연산은 PubeScreenTransformer의 Transform()에서 처리되며, 
Pipeline의 PostProcessTriangleVertices에서 각 정점들에 대해 .Transform()을 해주는 것으로 처리된다.
```c++
// PubeScreenTransformer.h
#pragma once
#include "Vec3.h"
#include "Graphics.h"

class PubeScreenTransformer
{
public:
	PubeScreenTransformer()
		:
		xFactor( float( Graphics::ScreenWidth ) / 2.0f ),
		yFactor( float( Graphics::ScreenHeight ) / 2.0f )
	{}
	Vec3& Transform( Vec3& v ) const
	{
		const float zInv = 1.0f / v.z;
		v.x = (v.x * zInv + 1.0f) * xFactor;
		v.y = (-v.y * zInv + 1.0f) * yFactor;
		return v;
	}
	Vec3 GetTransformed( const Vec3& v ) const
	{
		return Transform( Vec3( v ) );
	}
private:
	float xFactor;
	float yFactor;
};
```
```c++
// Pipeline.h
// ...
// vertex post-processing function
// perform perspective and viewport transformations
void PostProcessTriangleVertices( Triangle<Vertex>& triangle )
{
  // perspective divide and screen transform for all 3 vertices
  pst.Transform( triangle.v0.pos );
  pst.Transform( triangle.v1.pos );
  pst.Transform( triangle.v2.pos );

  // draw the triangle
  DrawTriangle( triangle );
}
// ...
```
.PostProcessTriangleVertices() -> .Transform() 이후 어떤 폴리곤(삼각형)의 pos값에는 3D좌표가 아닌 이 원근 좌표가 저장됨에 주의하자.  
코드를 살펴보면 실질적으로 변하는 값은 x, y 뿐이라는 것을 알 수 있다. z값에 따라 x, y값이 변하는 것인데, 이는 예전 Note에서도 다뤘듯이 Perspective projection의 과정이다.

2. 파이프라인에서 어떤 픽셀을 그리기 직전에 pixelShader(=ps)를 통해 셰이더 연산을 한다. 텍스쳐를 입히는 경우에도 이곳에서 연산이 처리된다.  
```c++
// Pipeline.h
// ...
// prestep scanline interpolant
iLine += diLine * (float( xStart ) + 0.5f - itEdge0.pos.x);

for( int x = xStart; x < xEnd; x++,iLine += diLine )
{
  // invoke pixel shader and write resulting color value
  gfx.PutPixel( x,y,effect.ps( iLine ) ); //.ps()는 셰이더 연산을 호출하는 함수이다.
}
// ...
```
```c++
// TextureEffect.h
#pragma once

#include "Pipeline.h"

// basic texture effect
class TextureEffect
{
public:
	// the vertex type that will be input into the pipeline
	class Vertex
	{
	public:
		Vertex() = default;
		Vertex( const Vec3& pos )
			:
			pos( pos )
		{}
		Vertex( const Vec3& pos,const Vertex& src )
			:
			t( src.t ),
			pos( pos )
		{}
		Vertex( const Vec3& pos,const Vec2& t )
			:
			t( t ),
			pos( pos )
		{}
		Vertex& operator+=( const Vertex& rhs )
		{
			pos += rhs.pos;
			t += rhs.t;
			return *this;
		}
		Vertex operator+( const Vertex& rhs ) const
		{
			return Vertex( *this ) += rhs;
		}
		Vertex& operator-=( const Vertex& rhs )
		{
			pos -= rhs.pos;
			t -= rhs.t;
			return *this;
		}
		Vertex operator-( const Vertex& rhs ) const
		{
			return Vertex( *this ) -= rhs;
		}
		Vertex& operator*=( float rhs )
		{
			pos *= rhs;
			t *= rhs;
			return *this;
		}
		Vertex operator*( float rhs ) const
		{
			return Vertex( *this ) *= rhs;
		}
		Vertex& operator/=( float rhs )
		{
			pos /= rhs;
			t /= rhs;
			return *this;
		}
		Vertex operator/( float rhs ) const
		{
			return Vertex( *this ) /= rhs;
		}
	public:
		Vec3 pos;
		Vec2 t;
	};
	// invoked for each pixel of a triangle
	// takes an input of attributes that are the
	// result of interpolating vertex attributes
	// and outputs a color
	class PixelShader
	{
	public:
		template<class Input>
		Color operator()( const Input& in ) const
		{
			return pTex->GetPixel(
				(unsigned int)std::min( in.t.x * tex_width + 0.5f,tex_xclamp ),
				(unsigned int)std::min( in.t.y * tex_height + 0.5f,tex_yclamp )
			);
		}
		void BindTexture( const std::wstring& filename )
		{
			pTex = std::make_unique<Surface>( Surface::FromFile( filename ) );
			tex_width = float( pTex->GetWidth() );
			tex_height = float( pTex->GetHeight() );
			tex_xclamp = tex_width - 1.0f;
			tex_yclamp = tex_height - 1.0f;
		}
	private:
		std::unique_ptr<Surface> pTex;
		float tex_width;
		float tex_height;
		float tex_xclamp;
		float tex_yclamp;
	};
public:
	PixelShader ps;
}; 
```
---  
  
![image](https://user-images.githubusercontent.com/63915665/178101408-2f002a5f-05b5-44c5-a212-ccc43af9851f.png)  
그런데 이러한 과정을 거치게 되면 멀리 있는 물체의 경우 Draw하는 정점들이 뺵빽히 들어차있는 반면,  
가까히 있는 물체는 그 밀도가 더 낮은 것을 알 수 있다.  
더 쉽게 말하면, 위 사진에서 빨간 부분과 파란 부분의 실제 면적의 크기는 같지만(=정점의 갯수는 같지만), 화면상에 그려지는 면적의 크기는 다르다.(=pixel의 수가 다르다)  
![image](https://user-images.githubusercontent.com/63915665/178101582-3ff27f6d-cdcb-41e0-aad2-01c7fb566d15.png)  
이유는 텍스쳐 좌표와 물체의 정점들과 맵핑하기 위해 iterate할 때 z축 거리와 상관없이 루프를 돌며 interpolate하기 떄문이다. (tc1 ~ tc2의 좌표값을 linear interpolate)  
텍스쳐가 멀리 있던 가까히 있던 관계없이 똑같은 밀도로 분포시키기 때문에 좁은 면적에 픽셀이 전부 커버하지 못할 정도로 많은 텍스쳐점을 그리려고 해서 텍스쳐가 뒤틀려보이는 이상한 현상이 관찰되는 것이다.  

![image](https://user-images.githubusercontent.com/63915665/178101810-1b8e4e69-4db3-48ee-a266-6f929fc83e5c.png)  
이를 해결하기 위해서는  
폴리곤을 interpolate하면서 각 점들을 Draw()를 해주는 밀도와,  
텍스쳐에 대해 interpolate를 해주는 밀도를 일치시켜주면 된다. (만들어낸 표현이라 정확하지 않을 수 있는데, 직관적으로 이해하고 넘어가자.)  
일치시켜주기 위해서는 텍스쳐 좌표들을 interpolate 할때도 텍스쳐 좌표를 그 텍스쳐와 맵핑된 정점의 z값으로 나눈 후 interpolate 해주면 된다.  
물론 1/z를 해준 텍스쳐 좌표값을 그대로 사용할 수 없으므로,이렇게 interpolate해서 구한 텍스쳐 좌표에 다시 z를 곱해주어 제대로 된 텍스쳐 좌표값 (=tc값)을 구해야 한다.  
  
그러기 위해서는 우리는 텍스쳐의 모든 점들에 대해 그 점과 맵핑되어있는 물체의 정점들의 z값들을 알아야 한다.  
단순히 생각하면 z1과 z2 사이를 linear interpolate하는 걸로 그 사이의 zk 값들을 전부 구할 수 있어보이지만, 이렇게 하면 또 한 번 밀도가 달라지기 때문에 우리는 대신 1/z1, 1/z2를 구한 후 그 사이를 interpolate해주어 1/zk를 구하고, zk가 필요할 때 이때 구한 값을 다시 한번 역수를 취해줌으로써 zk를 구하면 된다.  
(말로 설명하기 복잡하고, 또 수학적 증명 과정 또한 생략되어 있지만, 직관적으로 이해하고 넘어가자. 수학적 증명과정에 대해서는 이 자료를 참고하자.  
[자료1](https://www.scratchapixel.com/lessons/3d-basic-rendering/rasterization-practical-implementation/visibility-problem-depth-buffer-depth-interpolation)  
[자료2](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.3.211&rep=rep1&type=pdf))  

---  

정리하면, 이제부터 텍스쳐를 입힐 때는 다음과 같은 과정을 거친다. 코드와 함께 살펴보자. (커밋 ba9a0d48fc402a7eacc13f518f9a9422c25e5b19)  
기존과 달라졌거나 추가된 부분들은 볼드체로 강조해놓았다.  

1. 물체의 정점들의 3D 좌표를 원근을 고려해 변형시킨다.  
![image](https://user-images.githubusercontent.com/63915665/178102086-19dbfb05-2840-48e4-9307-4ee9d446e84a.png)  
**이때 기존에는 3D 좌표(Vec3)만을 받아 이 좌표값에 1/z를 곱해 원근좌표로 변환해주는 것만을 했는데, 이제부터는 정점 오브젝트(Vertex) 자체를 받아와 오브젝트 자체에 1/z를 계산해준다. 코드에서 보이는 v \*= zInv; 부분이다.  
TextureEffects의 경우 이 과정에서 텍스쳐 좌표들에 1/z를 곱하는 과정이 포함된다. 또 좌표값의 z(=Vertex.pos.z)값에 1/z를 곱하면 1이 되버리므로 쓸모가 없기 때문에 pos.z에는 1/z를 저장해둔다. 물론 vertex 내부에 zInv를 따로 저장해도 되겠지만 메모리를 아끼기 위해 이와 같은 방법을 취한다. (이는 구조적으로 좋은 방ㅂ버이 아니다. 하나의 변수가 여러 역할을 할 뿐더러 변수명과 다른 역할을 하기 때문이다. 이 부분은 차후 수정될 것이다.) **  

2. 파이프라인에서 어떤 픽셀을 그리기 직전에 셰이더 연산을 한다. 텍스쳐를 입히는 경우에도 이곳에서 연산이 처리된다.  
![image](https://user-images.githubusercontent.com/63915665/178102329-415ef1e1-4f08-4c9b-81cf-0fc26bf7f463.png)  
**이때 아까 1/z를 해주었던 텍스쳐 좌표값을 그대로 사용할 수 없으므로, 여기에 다시 z를 곱해주어 제대로 된 텍스쳐 좌표값을 구한다.**  


![image](https://user-images.githubusercontent.com/63915665/178102460-20301c3f-5ca0-4a3a-8677-dfe80fe593dd.png)  
실행해보면, 더이상 텍스쳐가 뒤틀리지 않는 것을 확인할 수 있다.

---  
![image](https://user-images.githubusercontent.com/63915665/178100358-4040254f-d22c-4bf5-b8f3-f6db84369db5.png)  
