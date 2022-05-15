본 학습의 목표는 PlanetChili의 프레임워크나 WinAPI, Dx를 이해하는 것보다는 그래픽스의 기반이 되는 지식들을 이해하는 데 있다.
때문에 코드를 세세히 살펴보기보다는 핵심이 되는 이론적 개념들을 이해하고 정리하는 것에 집중할 것이다.

프레임워크에 대한 대략적인 설명은 [여기](https://youtu.be/O7dKCaWzvzA?list=PLqCJpWy5Fohe8ucwhksiv9hTF5sfid8lA)에서 확인 가능하다.  

또한 본 프레임워크에서는 행렬을 곱할 때 통상적인 Ax 순서 (행렬 * 벡터)가 아닌 xA(벡터 * 행렬)을 사용한다.  
이는 Dx11이 해당 방식을 사용하기 때문이며, 때문에 v * A * B * C의 경우 (((v*A)*B)*C) 순서로 처리된다.

프레임워크의 실제 사용 모습을 아래 별 그리기 예제와 함께 살펴보자.
(세부적인 내용은 굳이 이해할 필요X)
```c++:Game.cpp
/******************************************************************************************
*	Chili DirectX Framework Version 16.10.01											  *
*	Game.cpp																			  *
*	Copyright 2016 PlanetChili.net <http://www.planetchili.net>							  *
*																						  *
*	This file is part of The Chili DirectX Framework.									  *
*																						  *
*	The Chili DirectX Framework is free software: you can redistribute it and/or modify	  *
*	it under the terms of the GNU General Public License as published by				  *
*	the Free Software Foundation, either version 3 of the License, or					  *
*	(at your option) any later version.													  *
*																						  *
*	The Chili DirectX Framework is distributed in the hope that it will be useful,		  *
*	but WITHOUT ANY WARRANTY; without even the implied warranty of						  *
*	MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the						  *
*	GNU General Public License for more details.										  *
*																						  *
*	You should have received a copy of the GNU General Public License					  *
*	along with The Chili DirectX Framework.  If not, see <http://www.gnu.org/licenses/>.  *
******************************************************************************************/
#include "MainWindow.h"
#include "Game.h"
#include "Mat2.h"

Game::Game( MainWindow& wnd )
	:
	wnd( wnd ),
	gfx( wnd )
{
	const float dTheta = 2.0f * PI / float( nflares * 2 );
	for( int i = 0; i < nflares * 2; i++ )
	{
		const float rad = (i % 2 == 0) ? radOuter : radInner;
		star.emplace_back(
			rad * cos( float( i ) * dTheta ),
			rad * sin( float( i ) * dTheta )
		);
	}
}

void Game::Go()
{
	gfx.BeginFrame();
	UpdateModel();
	ComposeFrame();
	gfx.EndFrame();
}

void Game::UpdateModel()
{
	if( !wnd.kbd.KeyIsPressed( VK_SPACE ) )
	{
		theta += vRot;
	}
}

void Game::ComposeFrame()
{
	const Vec2 trl = { float( gfx.ScreenWidth ) / 2.0f,float( gfx.ScreenHeight ) / 2.0f };
	const Mat2 trf = Mat2::Rotation( theta ) * Mat2::Scaling( size );
	auto vtx( star );
	for( auto& v : vtx )
	{
		v *= trf; // transformation
		v += trl; // translation
	}
	for( auto i = vtx.cbegin(),end = std::prev( vtx.cend() ); i != end; i++ )
	{
		gfx.DrawLine( *i,*std::next( i ),Colors::White );
	}
	gfx.DrawLine( vtx.front(),vtx.back(),Colors::White );
}
```
