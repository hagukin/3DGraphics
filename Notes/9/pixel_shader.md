픽셀 셰이더란, 그래픽을 픽셀 단위로 색을 지정해주는 연산을 처리하는 프로그램이다.  
렌더링 파이프라인에서 가장 중요한 역할을 담당하는 부분 중 하나로 여겨진다.  

![image](https://user-images.githubusercontent.com/63915665/175776691-2dadf70c-82db-4150-9d94-07ed017a315b.png)  
지난 시간까지 작성한 우리의 파이프라인에서는 우리는 Vertex 클래스 안에 텍스쳐에 대한 정보를 넣어둔 채, 파이프라인의 Triangle Rasterizer에서 곧바로 텍스쳐를 읽어와 이를 PutPixel에 전해주는 방식을 사용했다.  
그러나 이 방식은 확장성에서의 문제가 있는데, 빛이라던가, 여러 수학적 공식들을 이용해 우리가 새로운 효과를 주고 싶어도 주지 못한다는 문제가 있다.  
또 애초에 모든 3D 모델들의 vertex들이 전부 텍스쳐 정보를 가지고 있을 필요도 없다. (메모리 낭비다)  

때문에 우리는 기존의 이 방식을 개선하기 위해 다음과 같은 접근법을 취했다.  (커밋 dba859d1d068e0a5353001efdbe83435ae4b4c23)
우선 Pipeline 객체 내부에 Vertex를 정의해두었는데 이를 삭제하고, Pipeline이 Effect라는 클래스를 템플릿으로 받게 변경한다.  
이제 Pipeline은 Effect에 따라 PutPixel()에서 다른 연산을 실행시켜준다. (텍스쳐를 입히거나, 빛을 입히거나, 색을 블렌딩하는 등)
Effect는 Vertex와 유사하다고 생각하면 편하다.

```c++
// Pipeline.h

// triangle drawing pipeline with programable
// pixel shading stage
template<class Effect>
class Pipeline
{
public:
	// vertex type used for geometry and throughout pipeline
	typedef typename Effect::Vertex Vertex;
public:
  // 모든 vertex 관련 연산 함수들도 삭제. 연산자들은 이제 각 Effects 내부에서 정의한다.
	Pipeline( Graphics& gfx )
		:
		gfx( gfx )
	{}
	void Draw( IndexedTriangleList<Vertex>& triList )
	{
		ProcessVertices( triList.vertices,triList.indices );
	}
	void BindRotation( const Mat3& rotation_in )
	{
		rotation = rotation_in;
	}
	void BindTranslation( const Vec3& translation_in )
	{
		translation = translation_in;
	}
  // BindTexture 삭제. (텍스쳐 바인딩은 이제 TextureEffect 클래스에서만 처리함. 나머지는 애초에 텍스쳐가 없을 수도 있으므로.)
  
  // ...
  // (생략)
  // ...
  
  			// prestep scanline interpolant
			iLine += diLine * (float( xStart ) + 0.5f - itEdge0.pos.x);

			for( int x = xStart; x < xEnd; x++,iLine += diLine )
			{
				// invoke pixel shader and write resulting color value
				gfx.PutPixel( x,y,effect.ps( iLine ) ); // effects 멤버의 pixel shader 함수를 호출한다. 
        // 즉 텍스쳐를 입히는 등의 모든 픽셀에 대한 내부적인 연산은 각 Effects들의 ps()함수에서 처리된다.
			}
		}
	}
public:
	Effect effect;
private:
	Graphics& gfx;
	PubeScreenTransformer pst;
	Mat3 rotation;
	Vec3 translation;
};
```
  
```c++
// TextureEffect.h
// 사실 기존의 Vertex 클래스의 기능들을 Pipeline에서 분리시켜 옮긴 것에 불과하다.

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
![image](https://user-images.githubusercontent.com/63915665/175777528-13c35a8b-9203-4e56-b687-ba9a09bf954d.png)  
Pipeline에서 v.t 대신 v 전체를 전달한다. 이제 Vertex(position, texture) 생성자 대신  
Vertex(position, const Vertex& src)를 제작해 사용하는 것으로 텍스쳐가 없는 vertex들도 모두 Pipeline에서 호환되도록 수정해준다.  

![image](https://user-images.githubusercontent.com/63915665/175777153-3289e707-b9fc-4c11-83b3-81279be4d17c.png)  
이러한 프레임워크의 수정으로 인한 몇몇 자잘한 수정들을 마치고 실행시켜보면 기존과 완전히 동일한 결과물이 나오는 걸 볼 수 있다.  
기능을 수정한 건 전혀 없고, 구조만 리팩토링 해준 것이기 때문이다.  
  
---  
  
![image](https://user-images.githubusercontent.com/63915665/175777184-8d3950d4-5fa8-420e-9b04-01c9ef228050.png)  
이번에는 이렇게 변경된 프레임워크에 맞게 새로운 효과들을 추가시켜보자.  
우선 위 사진에서 보이는 색을 블렌딩하는 효과를 추가해보자.  

![image](https://user-images.githubusercontent.com/63915665/175777344-83851667-9af7-4ca9-8983-584c440a40b1.png)  
방식 자체는 간단하다. 각 Vertex별로 색 하나를 지정해주고, 이 색들을 interpolate 시켜주는 방식이다.  
여기서 한가지 주의할 부분은, Color값을 interpolate해주기 위해서 integer 대신 float를 사용하는 게 더 계산에 용이하기 때문에 Vec3<float>로 색을 입력받고, 이를 실제로 렌더링할 때는 Color 객체로 Typecast해줘야 한다는 점이다.  
실제로 하드웨어에서도 일반적으로 색은 float로 처리되며, 마지막에 색을 그려야 할 때 int로 typecast되어 사용된다.  

```c++
// VertexColorEffect.h
#pragma once

#include "Pipeline.h"

// color gradient effect between vertices
class VertexColorEffect
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
			color( src.color ),
			pos( pos )
		{}
		Vertex( const Vec3& pos,const Vec3& color )
			:
			color( color ),
			pos( pos )
		{}
		Vertex& operator+=( const Vertex& rhs )
		{
			pos += rhs.pos;
			color += rhs.color;
			return *this;
		}
		Vertex operator+( const Vertex& rhs ) const
		{
			return Vertex( *this ) += rhs;
		}
		Vertex& operator-=( const Vertex& rhs )
		{
			pos -= rhs.pos;
			color -= rhs.color;
			return *this;
		}
		Vertex operator-( const Vertex& rhs ) const
		{
			return Vertex( *this ) -= rhs;
		}
		Vertex& operator*=( float rhs )
		{
			pos *= rhs;
			color *= rhs;
			return *this;
		}
		Vertex operator*( float rhs ) const
		{
			return Vertex( *this ) *= rhs;
		}
		Vertex& operator/=( float rhs )
		{
			pos /= rhs;
			color /= rhs;
			return *this;
		}
		Vertex operator/( float rhs ) const
		{
			return Vertex( *this ) /= rhs;
		}
	public:
		Vec3 pos;
		Vec3 color;
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
			return Color( in.color );
		}
	};
public:
	PixelShader ps;
}; 
```
![image](https://user-images.githubusercontent.com/63915665/175777733-4b87d17c-641a-4200-8565-ff312dd08708.png)  
그 밖에 Cube.h에서 몇가지 구조적인 수정만 해주면 우리가 원하는 블렌딩된 큐브를 얻을 수 있다.  
(씬 정보는 Engine/CubeVertexColorScene.h을 참고하면 된다.)  

---  
  
![image](https://user-images.githubusercontent.com/63915665/175777766-223fd6ea-dc65-43ed-be3b-4d7859a3449d.png)  
이번에는 각 면이 단색으로 이루어진 Effect를 만들어보자.  

![image](https://user-images.githubusercontent.com/63915665/175777843-819073cd-8255-457f-af01-ab19b558cc5f.png)  
단순하게 생각하면 그냥 vertex에 같은 색들을 주고 interpolate하면 된다고 착각할 수 있지만, 폴리곤의 겹치는 부분들이 있기 떄문에 그렇게 해버리면 원하는 결과를 얻을 수 없다.  
  
![image](https://user-images.githubusercontent.com/63915665/175777857-e4a5ee88-4956-4dd7-840f-b6df68799a89.png)  
때문에 이런 상황에서는 어쩔 수 없이 각 면들의 vertex들을 다 분리시켜 사진처럼 별도의 vertex들로 만들어야 한다.  

```c++
// 
#pragma once

#include "Pipeline.h"

// solid color attribute not interpolated
class SolidEffect
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
			color( src.color ),
			pos( pos )
		{}
		Vertex( const Vec3& pos,const Color& color )
			:
			color( color ),
			pos( pos )
		{}
		Vertex& operator+=( const Vertex& rhs )
		{
			pos += rhs.pos;
        
      // VertexColorEffect와는 다르게 pos만 더해주고 color는 건드리지 않는 것을 볼 수 있다.
  
			return *this;
		}
		Vertex operator+( const Vertex& rhs ) const
		{
			return Vertex( *this ) += rhs;
		}
		Vertex& operator-=( const Vertex& rhs )
		{
			pos -= rhs.pos;
			return *this;
		}
		Vertex operator-( const Vertex& rhs ) const
		{
			return Vertex( *this ) -= rhs;
		}
		Vertex& operator*=( float rhs )
		{
			pos *= rhs;
			return *this;
		}
		Vertex operator*( float rhs ) const
		{
			return Vertex( *this ) *= rhs;
		}
		Vertex& operator/=( float rhs )
		{
			pos /= rhs;
			return *this;
		}
		Vertex operator/( float rhs ) const
		{
			return Vertex( *this ) /= rhs;
		}
	public:
		Vec3 pos;
		Color color;
	};
	// invoked for each pixel of a triangle
	// takes an input of attributes that are the
	// result of interpolating vertex attributes
	// and outputs a color
	class PixelShader
	{
	public:
		template<class I>
		Color operator()( const I& in ) const
		{
			return in.color;
		}
	};
public:
	PixelShader ps;
}; 
```
  
```
// Cube.h
// TextureEffect, VertexColorEffect, SolidEffect에 해당하는 static한 함수들이 나뉘어있음을 볼 수 있다.
#pragma once

#include "Vec3.h"
#include "IndexedTriangleList.h"

class Cube
{
public:
	template<class V>
	static IndexedTriangleList<V> GetPlain( float size = 1.0f )
	{
		const float side = size / 2.0f;

		std::vector<Vec3> vertices;

		vertices.emplace_back( -side,-side,-side ); // 0
		vertices.emplace_back( side,-side,-side ); // 1
		vertices.emplace_back( -side,side,-side ); // 2
		vertices.emplace_back( side,side,-side ); // 3
		vertices.emplace_back( -side,-side,side ); // 4
		vertices.emplace_back( side,-side,side ); // 5
		vertices.emplace_back( -side,side,side ); // 6
		vertices.emplace_back( side,side,side ); // 7

		std::vector<V> verts( vertices.size() );
		for( size_t i = 0; i < vertices.size(); i++ )
		{
			verts[i].pos = vertices[i];
		}
		return{
			std::move( verts ),{
				0,2,1, 2,3,1,
				1,3,5, 3,7,5,
				2,6,3, 3,6,7,
				4,5,7, 4,7,6,
				0,4,2, 2,4,6,
				0,1,4, 1,5,4
			}
		};
	}
	template<class V>
	static IndexedTriangleList<V> GetPlainIndependentFaces( float size = 1.0f )
	{
		const float side = size / 2.0f;

		std::vector<Vec3> vertices;

		vertices.emplace_back( -side,-side,-side ); // 0 near side
		vertices.emplace_back( side,-side,-side ); // 1
		vertices.emplace_back( -side,side,-side ); // 2
		vertices.emplace_back( side,side,-side ); // 3
		vertices.emplace_back( -side,-side,side ); // 4 far side
		vertices.emplace_back( side,-side,side ); // 5
		vertices.emplace_back( -side,side,side ); // 6
		vertices.emplace_back( side,side,side ); // 7
		vertices.emplace_back( -side,-side,-side ); // 8 left side
		vertices.emplace_back( -side,side,-side ); // 9
		vertices.emplace_back( -side,-side,side ); // 10
		vertices.emplace_back( -side,side,side ); // 11
		vertices.emplace_back( side,-side,-side ); // 12 right side
		vertices.emplace_back( side,side,-side ); // 13
		vertices.emplace_back( side,-side,side ); // 14
		vertices.emplace_back( side,side,side ); // 15
		vertices.emplace_back( -side,-side,-side ); // 16 bottom side
		vertices.emplace_back( side,-side,-side ); // 17
		vertices.emplace_back( -side,-side,side ); // 18
		vertices.emplace_back( side,-side,side ); // 19
		vertices.emplace_back( -side,side,-side ); // 20 top side
		vertices.emplace_back( side,side,-side ); // 21
		vertices.emplace_back( -side,side,side ); // 22
		vertices.emplace_back( side,side,side ); // 23

		std::vector<V> verts( vertices.size() );
		for( size_t i = 0; i < vertices.size(); i++ )
		{
			verts[i].pos = vertices[i];
		}
		return{
			std::move( verts ),{
				0,2, 1,    2,3,1,
				4,5, 7,    4,7,6,
				8,10, 9,  10,11,9,
				12,13,15, 12,15,14,
				16,17,18, 18,17,19,
				20,23,21, 20,22,23
			}
		};
	}
	template<class V>
	static IndexedTriangleList<V> GetSkinned( float size = 1.0f )
	{
		const float side = size / 2.0f;
		const auto ConvertTexCoord = []( float u,float v )
		{
			return Vec2{ (u + 1.0f) / 3.0f,v / 4.0f };
		};

		std::vector<Vec3> vertices;
		std::vector<Vec2> tc;

		vertices.emplace_back( -side,-side,-side ); // 0
		tc.emplace_back( ConvertTexCoord( 1.0f,0.0f ) );
		vertices.emplace_back( side,-side,-side ); // 1
		tc.emplace_back( ConvertTexCoord( 0.0f,0.0f ) );
		vertices.emplace_back( -side,side,-side ); // 2
		tc.emplace_back( ConvertTexCoord( 1.0f,1.0f ) );
		vertices.emplace_back( side,side,-side ); // 3
		tc.emplace_back( ConvertTexCoord( 0.0f,1.0f ) );
		vertices.emplace_back( -side,-side,side ); // 4
		tc.emplace_back( ConvertTexCoord( 1.0f,3.0f ) );
		vertices.emplace_back( side,-side,side ); // 5
		tc.emplace_back( ConvertTexCoord( 0.0f,3.0f ) );
		vertices.emplace_back( -side,side,side ); // 6
		tc.emplace_back( ConvertTexCoord( 1.0f,2.0f ) );
		vertices.emplace_back( side,side,side ); // 7
		tc.emplace_back( ConvertTexCoord( 0.0f,2.0f ) );
		vertices.emplace_back( -side,-side,-side ); // 8
		tc.emplace_back( ConvertTexCoord( 1.0f,4.0f ) );
		vertices.emplace_back( side,-side,-side ); // 9
		tc.emplace_back( ConvertTexCoord( 0.0f,4.0f ) );
		vertices.emplace_back( -side,-side,-side ); // 10
		tc.emplace_back( ConvertTexCoord( 2.0f,1.0f ) );
		vertices.emplace_back( -side,-side,side ); // 11
		tc.emplace_back( ConvertTexCoord( 2.0f,2.0f ) );
		vertices.emplace_back( side,-side,-side ); // 12
		tc.emplace_back( ConvertTexCoord( -1.0f,1.0f ) );
		vertices.emplace_back( side,-side,side ); // 13
		tc.emplace_back( ConvertTexCoord( -1.0f,2.0f ) );

		std::vector<V> verts( vertices.size() );
		for( size_t i = 0; i < vertices.size(); i++ )
		{
			verts[i].pos = vertices[i];
			verts[i].t = tc[i];
		}

		return{
			std::move( verts ),{
				0,2,1,   2,3,1,
				4,8,5,   5,8,9,
				2,6,3,   3,6,7,
				4,5,7,   4,7,6,
				2,10,11, 2,11,6,
				12,3,7,  12,7,13
			}
		};
	}
};
```

![image](https://user-images.githubusercontent.com/63915665/175778048-7b37c135-cabe-488c-8dbd-aa53f2263e07.png)  
실행해보면 원하는 결과가 나오는 걸 확인할 수 있다.  
(Engine/CubeSolidScene.h 참고)  
  
---
