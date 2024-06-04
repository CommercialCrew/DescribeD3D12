# Direct3D 예제 해석하기

## HelloWindow

Direct3D 기술을 이용해 단색 화면을 표시하는 간단한 예제이다.
실제 사용자의 눈에 보이는 구현부는 OnRender()와 PopulateCommandList()인데,

OnRender()는 scene을 렌더링 하는 역할을, PopulateCommandList()는 렌더링할 scene의 커맨드들을 기록해두었다가 실제 렌더링시에 꺼내 쓰는 보관함 역할을 한다.

실제로 PopulateCommandList()에 화면을 어떻게 구성할 것인지에 대한 코드가 쓰여있다.

렌더링 구현부를 둘러싸고있는
```c++
m_commandList->ResourceBarrier(1, &CD3DX12_RESOURCE_BARRIER::Transition(m_renderTargets[m_frameIndex].Get(), D3D12_RESOURCE_STATE_PRESENT, D3D12_RESOURCE_STATE_RENDER_TARGET));

 CD3DX12_CPU_DESCRIPTOR_HANDLE rtvHandle(m_rtvHeap->GetCPUDescriptorHandleForHeapStart(), m_frameIndex, m_rtvDescriptorSize);
```
와

 ```c++ 
 m_commandList->ResourceBarrier(1, &CD3DX12_RESOURCE_BARRIER::Transition(m_renderTargets[m_frameIndex].Get(), D3D12_RESOURCE_STATE_RENDER_TARGET, D3D12_RESOURCE_STATE_PRESENT));
```

부분은 간단히 설명하면 다음과 같다.

우리 눈에 실제로 보이는 부분인 프론트 버퍼와 그렇지 않은 부분인 백 버퍼가 있는데, 백 버퍼에 보여줄 렌더링 사항들을 구현한 뒤에 프론트 버퍼로 보내서 실제로 보여준다. 그러는 사이에 백 버퍼에는 그 다음 프레임을 렌더링 하는 것이다.

즉, 그 사이에 실제로 사용자에게 보여줄 요소들을 적어넣는다.

```c++
const float clearColor[] = { 0.0f, 0.2f, 0.4f, 1.0f};
m_commandList->ClearRenderTargetView(rtvHandle, clearColor, 0, nullptr);
```
그 부분이 바로 이 부분인데, 부분이 화면을 단색으로 설정하는 부분이다.

 ## HelloTriangle
 
앞 프로젝트에서 화면을 단색으로 설정한 것에 이어서, 이번에는 무지개색의 2D 삼각형을 표시한다.

이전 프로그램에서 수정된 부분은 다음과 같다.

*1) (프로젝트명)::(프로젝트명)()*

이전 HelloWindow에서는 

DXSample(width, height, name),
m_frameIndex(0),
m_rtvDescriptorSize(0)

정도만 정의했던 것이 
```c++
m_viewport(0.0f, 0.0f, static_cast<float>(width), static_cast<float>(height)),
m_scissorRect(0, 0, static_cast<LONG>(width), static_cast<LONG>(height)),
```

라는 부분이 추가되었는데, 이 부분은 렌더링할 영역을 조정하는 부분이다.
viewpor는 짧게는 3D 영역이 표시되는 2D 사각형이라고 할 수 있다. 
조금 더 간단하게 말하면 '카메라의 방향을 조정하는 것'이라는 말이다.
화면을 뷰포트의 범위 내의 것만 표시시킨다.

scissorRect는 정한 범위 내의 부분만 렌더링 시키게 해준다.
굳이 렌더링 할 필요가 없는 부분을 잘라내는 것이다.



*2) LoadAssets()*

이 부분은 Direct3D 프로젝트에 필요한 텍스처, 사운드, 셰이더 등 필요한 리소스들을 로드하는데 사용된다.

이전 HelloWindow는 단색 화면만 표시하면 끝이었기 때문에 딱히 로드하는 것이 없어 메소드의 내용이 거의 비어있지만,

HelloTriangle은 도형을 표시해야 하기에 그에 따른 리소스를 로드한다. 

hlsl 셰이더를 로드하기도 하고, 삼각형의 꼭짓점의 좌표를 정의하는 부분도 있는 등 다양한 부분이 정의되어 있는 모습을 볼 수 있다.



*3)PopulateCommandList()*

렌더링을 위한 명령들을 모아두는 곳인 만큼 차이가 날 수 밖에 없다.

HelloWindow는 clearColor[]를 이용해 배경의 색만 지정했다면, 여기서는 

m_commandList->IASetPrimitiveTopology

m_commandList->IASetVertexBuffers

m_commandList->DrawInstanced

라는 부분이 추가되었다. 

 'm_commandList->IASetPrimitiveTopology' 라는 부분은 점, 선, 도형 등 앞으로 그릴 프리미티브의 유형을 결정하는 부분으로, 'D3D_PRIMITIVE_TOPOLOGY_TRIANGLELIST' 를 넣어 '삼각형을 그릴 것이다' 라고 입력 어셈블러에게 알려준 것이다.



'm_commandList->IASetVertexBuffers' 는 그래픽 데이터를 저장할 버퍼를 설정하는 부분이고,

'm_commandList->DrawInstanced' 는 그릴 정점의 개수 등을 정하는 등 그래픽 명령을 내려 렌더링하는 부분이다.


나머지 이하의 부분들은 기존과 동일하다.


## HelloTexture

기본 바탕이 되는 것은 단색 배경에 2D 삼각형을 표시하는 것이므로 앞부분은 수정사항이 없고, LoadAssets()와 PopulateCommandList() 같은 본격적으로 렌더링의 상세를 정의하는 부분이 수정된다.



*1) LoadAssets()*

먼저, 텍스처를 적용하는 부분이 추가되었다.

```c++
ComPtr<ID3D12Resource> textureUploadHeap;

// Create the texture.
{
    // Describe and create a Texture2D.
    D3D12_RESOURCE_DESC textureDesc = {};
    textureDesc.MipLevels = 1;
    textureDesc.Format = DXGI_FORMAT_R8G8B8A8_UNORM;
    textureDesc.Width = TextureWidth;
    textureDesc.Height = TextureHeight;
    textureDesc.Flags = D3D12_RESOURCE_FLAG_NONE;
    textureDesc.DepthOrArraySize = 1;
    textureDesc.SampleDesc.Count = 1;
    textureDesc.SampleDesc.Quality = 0;
    textureDesc.Dimension = D3D12_RESOURCE_DIMENSION_TEXTURE2D;

    ThrowIfFailed(m_device->CreateCommittedResource(
        &CD3DX12_HEAP_PROPERTIES(D3D12_HEAP_TYPE_DEFAULT),
        D3D12_HEAP_FLAG_NONE,
        &textureDesc,
        D3D12_RESOURCE_STATE_COPY_DEST,
        nullptr,
        IID_PPV_ARGS(&m_texture)));

    const UINT64 uploadBufferSize = GetRequiredIntermediateSize(m_texture.Get(), 0, 1);

    // Create the GPU upload buffer.
    ThrowIfFailed(m_device->CreateCommittedResource(
        &CD3DX12_HEAP_PROPERTIES(D3D12_HEAP_TYPE_UPLOAD),
        D3D12_HEAP_FLAG_NONE,
        &CD3DX12_RESOURCE_DESC::Buffer(uploadBufferSize),
        D3D12_RESOURCE_STATE_GENERIC_READ,
        nullptr,
        IID_PPV_ARGS(&textureUploadHeap)));

    // Copy data to the intermediate upload heap and then schedule a copy 
    // from the upload heap to the Texture2D.
    std::vector<UINT8> texture = GenerateTextureData();

    D3D12_SUBRESOURCE_DATA textureData = {};
    textureData.pData = &texture[0];
    textureData.RowPitch = TextureWidth * TexturePixelSize;
    textureData.SlicePitch = textureData.RowPitch * TextureHeight;

    UpdateSubresources(m_commandList.Get(), m_texture.Get(), textureUploadHeap.Get(), 0, 0, 1, &textureData);
    m_commandList->ResourceBarrier(1, &CD3DX12_RESOURCE_BARRIER::Transition(m_texture.Get(), D3D12_RESOURCE_STATE_COPY_DEST, D3D12_RESOURCE_STATE_PIXEL_SHADER_RESOURCE));

    // Describe and create a SRV for the texture.
    D3D12_SHADER_RESOURCE_VIEW_DESC srvDesc = {};
    srvDesc.Shader4ComponentMapping = D3D12_DEFAULT_SHADER_4_COMPONENT_MAPPING;
    srvDesc.Format = textureDesc.Format;
    srvDesc.ViewDimension = D3D12_SRV_DIMENSION_TEXTURE2D;
    srvDesc.Texture2D.MipLevels = 1;
    m_device->CreateShaderResourceView(m_texture.Get(), &srvDesc, m_srvHeap->GetCPUDescriptorHandleForHeapStart());
}
```
가 해당 부분이다. 



앞에서부터 textureDesc.~ 부분들은 텍스처를 3D로 할건지 2D로 할건지, 크기는 얼만큼으로 설정할지 등의 요소들을 정의하고, 



CreateCommittedResource 부분은 텍스처 데이터를 GPU로 업로드하기 위해 데이터를 담을 임시 버퍼를 만드는 부분이다.




'std::vector<UINT8> texture = GenerateTextureData();' 이하의 부분은 텍스처 데이터를 생성하고 GPU로 업로드 한 뒤, 텍스처를 픽셀 셰이더 리소스로 전환하는 부분이다.


GenerateTextureData()가 실제로 흰색과 검은색의 체크무늬를 그려내는 부분인데, 이를 texture 변수에 담아 진행한다.
UpdateSubresources()를 통해 텍스처 데이터를 텍스처 리소스로 복사하고,
ResoureBarrier()에서 텍스처의 상태를 'D3D12_RESOURCE_STATE_PIXEL_SHADER_RESOURCE'로 전환시킨다.
이렇게 하면 픽셀 셰이더에서 텍스처 데이터를 읽을 수 있게된다.


D3D12_SHADER_RESOURCE_VIEW_DESC srvDesc = {}; 이하의 부분은 텍스처를 위한 SRV(셰이더 자원 뷰)를 생성하는 부분이다.


이후 m_CommandList->Close 이하의 부분들로 CommandList와 관련된 부분이 끝난다.



*2) GenerateTextureData()*

텍스처 자체를 구현하는 부분이다. 각 칸의 크기와 패턴의 RGBA 값 등을 정의하여 흰색과 검은색의 체크무늬 패턴을 만들어낸다.



*3) PopluateCommandList()*
```c++
ID3D12DescriptorHeap* ppHeaps[] = { m_srvHeap.Get() };
m_commandList->SetDescriptorHeaps(_countof(ppHeaps), ppHeaps);

m_commandList->SetGraphicsRootDescriptorTable(0, m_srvHeap->GetGPUDescriptorHandleForHeapStart());
```
부분이 추가되었다.

디스트립터 힙을 설정해 GPU가 텍스처가 적용된 리소스(SRV)를 참조하도록 지시하는 부분이다.

SetDescriptorHeaps()를 통해 SRV를 포인터 배열에 추가하고 디스크립터 힙을 설정한다.

SetGraphicsRootDescriptorTable()을 통해 그래픽 루트 서명이라는 것을 설정해 GPU에게 특정 리소스를 참조하게 지시한다. 여기서는 SRV의 시작주소를 0번째 루트 파라미터에 설정해 SRV의 리소스를 참조하도록 만들었다.

이렇게 하면 GPU가 텍스처를 참조해 삼각형에 텍스처가 적용된다.
