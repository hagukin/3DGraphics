우리가 개발한 기능들을 일련의 과정으로써 통합해 하나의 파이프라인을 구축해보자.  

우리가 구축하려는 파이프라인은 사실 현대의 그래픽스 파이프라인과 비교하면 큰 차이가 있다.  
![image](https://user-images.githubusercontent.com/63915665/174307999-1f5bd89f-cb97-4bb9-ad55-461b411071e4.png)
우리는 90년대 그래픽스에서 사용하던 Fixed Function Pipeline을 만들어 나갈 것인데, (이미 어느 정도 만들어 둔 상태이긴 하다)
이 방식은 개발 편의성의 문제를 가진다. 현대에는 온갖 시각 효과들이 존재하는 데 이러한 방식으로는 이런 효과들 하나하나마다 개별적인 함수를 일일히 다 작성해주어야 하고, 이는 효율적이지 못한 방식이다. 
flameEffect1(), flameEffect2(), ... 이런 식으로 모든 효과들마다 하나하나 함수를 작성해야 한다면 일의 능률이 떨어질 수 밖에 없고, 그 함수들을 일일히 디버깅하는 데도 많은 자원이 소모될 것이다.  

때문에 현대의 그래픽스 파이프라인은 프로그래머에게 보다 더 많은 자유를 주기 위해 GPU에서 돌아가는 커스텀 프로그램의 기능을 제공고 있다. (=Programmable Shader)  
내부적인 연산은 미리 정의해두고, 프로그래머는 값들을 조정하기만 하는 것으로 다양한 시각효과들을 표현할 수 있게 된다.  
  
![image](https://user-images.githubusercontent.com/63915665/174309463-f78b33eb-e87b-4312-8383-a0c0907022d1.png)
현대의 파이프라인은 위 사진과 조금 더 유사하며, 앞으로 이러한 파이프라인의 구축 또한 해볼 것이지만, 지금은 일단 넘어가도록 하자.  
  
---

다시 돌아와서, 우리가 만들어나갈 파이프라인의 과정을 살펴보자.  
![image](https://user-images.githubusercontent.com/63915665/174307999-1f5bd89f-cb97-4bb9-ad55-461b411071e4.png)

Indexed Triangle list는 Vertices(정점들/Vertex들)와 Indices(인덱스들/Index들)의 정보를 담고있는 배열이다.  
우리는 이 배열에서 Vertices와 Indices의 정보를 분리해 추출해내어 각각 stream으로 전달할 것인데,  
우선 정점들의 경우 Vertex Transformer로 보내 정점들의 물리적 위치에 대한 연산을 처리해준다. 즉 월드 좌표계 등에 맞게 회전이나 이동 등을 여기서 처리해준다.  
그 후 삼각형 점들의 인덱스들을 담고 있는 stream과 이렇게 변환된 정점들의 stream을 토대로 Triangle Assembler에서 삼각형(폴리곤)들을 만들어낼 것이다. 이 과정에서 Backface culling 또한 처리된다.   
  
이 과정을 Index stream이 텅 빌 때까지(=모든 인덱스들에 대해서) 반복해주고, 이렇게 생성된 삼각형들은 Perspective/Screen Transformer로 보내 각 삼각형 및 그 정점마다 Perspective division(원근처리. 우리는 이를 z축 거리만큼 나눠주는 방식으로 구현한 적 있다.)
및 Screen transform을 해 준다.  
  
이 과정을 거친 삼각형들은 Triangle Rasterizer에서 Rasterize해서 렌더링 할 폴리곤들을 결정하고, 이렇게 결정된 폴리곤들은 그 색상과 함께 Rasterization space에 저장된다.  
이때 텍스쳐 픽셀과 폴리곤 위치를 mapping 해 텍스쳐를 입히는 것도 처리한다. (이전 글에서 다뤘던 과정을 거친다)  
  
최종적으로 Rasterization space의 값들을 PutPixel을 통해 화면 상에 렌더링한다.  
  
---

코드와 함께 살펴보자.  
사실 기능적인 부분들은 이미 대부분 구현이 완료된 상태이므로, 이를 파이프라인화 하는 과정을 살펴보자. (커밋 399244fbf1ea160ff7625ccce6f3db0fde9e8bd9)  
  
```c++
//Pipeline.h
// ... (생략)

class Pipeline
{
public:
  /*
  Vertex 클래스 정의
  */
	// vertex type used for geometry and throughout pipeline
  class Vertex
	{
	public:
		// Vertex 관련 연산자들 정의 
    // ... (코드 생략)
    
	public:
    // 실제 값 저장되는 부분
		Vec3 pos;
		Vec2 t;
	};
  
public:
	Pipeline( Graphics& gfx )
		:
		gfx( gfx )
	{}
  
  
	void Draw( IndexedTriangleList<Vertex>& triList )
	{
    // 가장 핵심적인 함수.
    // 앞서 파이프라인에서 살펴보았던 Split 과정을 거쳐 Vertex Transformer로 전달하는 과정이 여기서 이뤄진다.
		ProcessVertices( triList.vertices,triList.indices );
	}
  
  
  // 멤버변수 초기화
	void BindRotation( const Mat3& rotation_in )
	{
		rotation = rotation_in;
	}
  
  
  // 멤버변수 초기화
	void BindTranslation( const Vec3& translation_in )
	{
		translation = translation_in;
	}
  
  
  // 멤버변수 초기화
	void BindTexture( const std::wstring& filename )
	{
		pTex = std::make_unique<Surface>( Surface::FromFile( filename ) );
	}
  
  
private:
	// vertex processing function 
	// transforms vertices and then passes vtx & idx lists to triangle assembler
  // Vertex Transformer의 역할이다.
	void ProcessVertices( const std::vector<Vertex>& vertices,const std::vector<size_t>& indices )
	{
		// create vertex vector for vs output
		std::vector<Vertex> verticesOut; // 반환값

		// transform vertices using matrix + vector
    // 입력받은 정점들에 대해 위치 관련 연산을 해주어 반환값에 emplace_back 한다. (회전 및 이동 처리)
		for( const auto& v : vertices )
		{
			verticesOut.emplace_back( v.pos * rotation + translation,v.t ); 
      // 회전행렬 적용 및 위치 이동 처리. 이때 texture coordinate(v.t)는 당연히 변하지 않는다. 
      // 월드에서의 물리적 이동이 텍스쳐 맵핑에 영향을 주지 않기 때문이다. 
		}

		// assemble triangles from stream of indices and vertices
    // 파이프라인의 다음 단계로 전달한다.
		AssembleTriangles( verticesOut,indices );
	}
  
  
	// triangle assembler
	// assembles indexed vertex stream into triangles and passes them to post process
	// culls (does not send) back facing triangles
	void AssembleTriangles( const std::vector<Vertex>& vertices,const std::vector<size_t>& indices )
	{
		// assemble triangles in the stream and process
		for( size_t i = 0,end = indices.size() / 3;
			 i < end; i++ )
		{
			// determine triangle vertices via indexing
      // 인덱싱을 통해 삼각형을 하나씩 만든다.
			const auto& v0 = vertices[indices[i * 3]];
			const auto& v1 = vertices[indices[i * 3 + 1]];
			const auto& v2 = vertices[indices[i * 3 + 2]];
      
			// cull backfacing triangles with cross product (%) shenanigans
      // backface culling 처리
			if( (v1.pos - v0.pos) % (v2.pos - v0.pos) * v0.pos <= 0.0f )
			{
				// process 3 vertices into a triangle
        // 세 정점에 대해 파이프라인의 다음 단계로 넘겨주기 전 삼각형으로 만드는 과정을 거쳐준다.
				ProcessTriangle( v0,v1,v2 );
			}
		}
	}
  
  
  // 실제로 합쳐주는 과정. 나중에 이 과정을 거치면서 Geometry shader를 통해 Geometry shading을 거친다.
	// triangle processing function
	// takes 3 vertices to generate triangle
	// sends generated triangle to post-processing
	void ProcessTriangle( const Vertex& v0,const Vertex& v1,const Vertex& v2 )
	{
    // 여기서 Geometry shading
		// generate triangle from 3 vertices using gs
		// and send to post-processing
    
    // PostProcessTriangleVertices은 Perspective/Screen Transformer의 역할이다
		PostProcessTriangleVertices( Triangle<Vertex>{ v0,v1,v2 } ); // Triangle.h를 보면 Triangle은 그냥 세 Vertex로 이루어진 간단한 Struct임을 알 수 있다.
	}
  
  
	// vertex post-processing function
	// perform perspective and viewport transformations
	void PostProcessTriangleVertices( Triangle<Vertex>& triangle )
	{
		// perspective divide and screen transform for all 3 vertices
    // pst는 코드 맨 아랫쪽에 정의되어 있으며, Pipeline 오브젝트의 private한 멤버변수이다.
    // 여기서 원근처리 등을 해준다.
		pst.Transform( triangle.v0.pos );
		pst.Transform( triangle.v1.pos );
		pst.Transform( triangle.v2.pos );

		// draw the triangle
		DrawTriangle( triangle );
	}
  
  
  // 예전에 다뤘던 삼각형 그리는 과정이다. 자세한 설명은 해당 내용 참고
  // 한가지 변한 점은, 이전에는 우리는 xy좌표만 interpolate 했지만, 지금은 xyz 다 interpolate한다. (정확한 의미는 아래 코드 및 나중에 제작할 셰이더 내용을 참고)
	// === triangle rasterization functions ===
	//   it0, it1, etc. stand for interpolants
	//   (values which are interpolated across a triangle in screen space)
	//
	// entry point for tri rasterization
	// sorts vertices, determines case, splits to flat tris, dispatches to flat tri funcs
	void DrawTriangle( const Triangle<Vertex>& triangle )
	{
		// using pointers so we can swap (for sorting purposes)
		const Vertex* pv0 = &triangle.v0;
		const Vertex* pv1 = &triangle.v1;
		const Vertex* pv2 = &triangle.v2;

		// sorting vertices by y
		if( pv1->pos.y < pv0->pos.y ) std::swap( pv0,pv1 );
		if( pv2->pos.y < pv1->pos.y ) std::swap( pv1,pv2 );
		if( pv1->pos.y < pv0->pos.y ) std::swap( pv0,pv1 );

		if( pv0->pos.y == pv1->pos.y ) // natural flat top
		{
			// sorting top vertices by x
			if( pv1->pos.x < pv0->pos.x ) std::swap( pv0,pv1 );

			DrawFlatTopTriangle( *pv0,*pv1,*pv2 );
		}
		else if( pv1->pos.y == pv2->pos.y ) // natural flat bottom
		{
			// sorting bottom vertices by x
			if( pv2->pos.x < pv1->pos.x ) std::swap( pv1,pv2 );

			DrawFlatBottomTriangle( *pv0,*pv1,*pv2 );
		}
		else // general triangle
		{
			// find splitting vertex interpolant
			const float alphaSplit =
				(pv1->pos.y - pv0->pos.y) /
				(pv2->pos.y - pv0->pos.y);
			const auto vi = interpolate( *pv0,*pv2,alphaSplit );

			if( pv1->pos.x < vi.pos.x ) // major right
			{
				DrawFlatBottomTriangle( *pv0,*pv1,vi );
				DrawFlatTopTriangle( *pv1,vi,*pv2 );
			}
			else // major left
			{
				DrawFlatBottomTriangle( *pv0,vi,*pv1 );
				DrawFlatTopTriangle( vi,*pv1,*pv2 );
			}
		}
	}
  
  
	// does flat *TOP* tri-specific calculations and calls DrawFlatTriangle
	void DrawFlatTopTriangle( const Vertex& it0,
							  const Vertex& it1,
							  const Vertex& it2 )
	{
		// calulcate dVertex / dy
		// change in interpolant for every 1 change in y
		const float delta_y = it2.pos.y - it0.pos.y;
		const auto dit0 = (it2 - it0) / delta_y;
		const auto dit1 = (it2 - it1) / delta_y;

		// create right edge interpolant
		auto itEdge1 = it1;

		// call the flat triangle render routine
		DrawFlatTriangle( it0,it1,it2,dit0,dit1,itEdge1 );
	}
  
  
	// does flat *BOTTOM* tri-specific calculations and calls DrawFlatTriangle
	void DrawFlatBottomTriangle( const Vertex& it0,
								 const Vertex& it1,
								 const Vertex& it2 )
	{
		// calulcate dVertex / dy
		// change in interpolant for every 1 change in y
		const float delta_y = it2.pos.y - it0.pos.y;
		const auto dit0 = (it1 - it0) / delta_y;
		const auto dit1 = (it2 - it0) / delta_y;

		// create right edge interpolant
		auto itEdge1 = it0;

		// call the flat triangle render routine
		DrawFlatTriangle( it0,it1,it2,dit0,dit1,itEdge1 );
	}
  
  
	// does processing common to both flat top and flat bottom tris
	// texture lookup and pixel written here
	void DrawFlatTriangle( const Vertex& it0,
						   const Vertex& it1,
						   const Vertex& it2,
						   const Vertex& dv0,
						   const Vertex& dv1,
						   Vertex itEdge1 )
	{
		// create edge interpolant for left edge (always v0)
		auto itEdge0 = it0;

		// calculate start and end scanlines
		const int yStart = (int)ceil( it0.pos.y - 0.5f );
		const int yEnd = (int)ceil( it2.pos.y - 0.5f ); // the scanline AFTER the last line drawn

		// do interpolant prestep
		itEdge0 += dv0 * (float( yStart ) + 0.5f - it0.pos.y);
		itEdge1 += dv1 * (float( yStart ) + 0.5f - it0.pos.y);

		// prepare clamping constants
		const float tex_width = float( pTex->GetWidth() );
		const float tex_height = float( pTex->GetHeight() );
		const float tex_xclamp = tex_width - 1.0f;
		const float tex_yclamp = tex_height - 1.0f;
    
    /*
    변경된 부분들. xy만이 아니라 xyz를 모두 interpolate하는 이유는 앞으로 추가될 셰이더 관련 기능들 때문이다.
    */
		for( int y = yStart; y < yEnd; y++,itEdge0 += dv0,itEdge1 += dv1 )
		{
			// calculate start and end pixels
			const int xStart = (int)ceil( itEdge0.pos.x - 0.5f );
			const int xEnd = (int)ceil( itEdge1.pos.x - 0.5f ); // the pixel AFTER the last pixel drawn

			// create scanline interpolant startpoint
			// (some waste for interpolating x,y,z, but makes life easier not having
			//  to split them off, and z will be needed in the future anyways...)
			auto iLine = itEdge0;

			// calculate delta scanline interpolant / dx
			const float dx = itEdge1.pos.x - itEdge0.pos.x;
			const auto diLine = (itEdge1 - iLine) / dx;

			// prestep scanline interpolant
			iLine += diLine * (float( xStart ) + 0.5f - itEdge0.pos.x);

			for( int x = xStart; x < xEnd; x++,iLine += diLine )
			{
				// perform texture lookup, clamp, and write pixel
				gfx.PutPixel( x,y,pTex->GetPixel(
					(unsigned int)std::min( iLine.t.x * tex_width + 0.5f,tex_xclamp ),
					(unsigned int)std::min( iLine.t.y * tex_height + 0.5f,tex_yclamp )
				) );
			}
		}
	}
private:
  // 멤버들에 대한 처리(초기화)는 BindRotation, BindTranslation, BindTexture에서 처리
	Graphics& gfx; // Graphics 오브젝트에 대한 레퍼런스 저장
	PubeScreenTransformer pst; // PST
	Mat3 rotation; // 회전행렬
	Vec3 translation; // 위치
	std::unique_ptr<Surface> pTex; // 텍스쳐
};
```
  
그 외 변경된 파이프라인에 맞게 기존 Scene 헤더파일들은 삭제되고 새로 제작해 추가되었다.  
또한 TexVertex 역시 새로운 Vertex 클래스로 대체되었으므로 삭제되었다.  
Vec3, Vec2의 .interpolateTo() 대신 ChiliMath.h에 general한 interpolate()함수가 추가되었다.  
Cube 오브젝트는 기존의 하드코딩된 값들이 지워졌고 GetSkinned()라는 정육면체 텍스쳐와 맵핑할 때 쓸 수 있는 함수가 추가되었다.  
  
![image](https://user-images.githubusercontent.com/63915665/174319519-13370360-2d9d-490b-a9d5-a427887ac4c2.png)  
사진으로 보듯이 파이프라인화를 함으로써 씬을 만들기가 훨씬 쉬워졌다.  
  
