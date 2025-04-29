---
title:  "Implement a Compute Shader in Unreal5"
date:   2025-03-20 21:00
categories: [shaderposts]
tags: [material, Unreal, shader]
image: /post-img/shaderposts/compute-shader/cover-compute-shader.jpg
published: true
---

## Intro

<br />
A while back, I created a calibration tool via a plugin, which involved embedding a custom global shader in Unreal. Recently, while diving into Ubisoft's implementation of foliage animation, I found out that they use compute shaders to handle complex physics calculations. It’s a great example of how compute shaders can be applied effectively, and it inspired me to try something similar in UE.<br />

In Unreal, whether you're creating a new shading model or adding a custom compute or vertex shader, the whole implementation process can get pretty complicated and headache-inducing. Although I didn’t end up using this approach in the end, I still wanted to document the process here—might come in handy in the future.<br />

My initial idea was to export a JSON file from Houdini that contains all the necessary data for each type of tree—things like branch ID, pivot position, parent ID, etc. This JSON would then be imported into the project as a DataTable.<br />

From there, I planned to read the data in C++, pass it into a SceneViewExtension, and then forward it to the compute shader. After running the calculations in the compute shader, the resulting vertex positions would be passed into a Material Parameter Collection, which could then be read in the Material Editor.<br />

However, after testing it, I found that this kind of data transfer couldn't maintain a smooth per-frame update. So I had to drop the idea in the end.<br />

## Create a Plugin
Same as before, the shader, SceneViewExtension, and Subsystem code were all written inside a plugin. You can simply create a blank plugin through the engine and then add the necessary files. The folder structure is as follows:<br />
![structure](/post-img/shaderposts/ue5-compute-shader/file-structure.png){: width="80%"} <br />
In the `CTTree.cpp` file, I defined the new shader path by using `AddShaderSourceDirectoryMapping` in the `StartupModule`. 
![cttree](/post-img/shaderposts/ue5-compute-shader/cttree.png.png){: width="80%"} <br />
So that later I can use `"/Plugin/CTTree/CTTreeCS.usf"` to refer my compute shader or bind my shader. <br />
This is exactly where using a plugin becomes convenient: it allows you to register all necessary shader paths and initialize related logic right when the engine launches—avoiding the issues tied to game module execution timing. There are two main reasons for this:<br />
1. You can’t register shader paths inside the `Game Module`, because the `StartupModule()` of a game module is only executed before entering Play mode. So if your shader path isn't registered by then, it won’t be recognized or compiled in time for previewing or editor-time usage.<br />

2. then you'll need to set up a dedicated shader module, which involves adding dependencies like "ShaderCore", "RHI", etc., in your .Build.cs file, and something like this in `.uproject` file: <br />

```c++
//.uproject
"Modules": [
    {
        "Name": "ComputeTree", // Game Module
    "Type": "Runtime",
    "LoadingPhase": "Default"
  },
  {
      "Name": "ComputeTreeShaders", // Additional Shader Module
    "Type": "Developer",
    "LoadingPhase": "PostEngineInit"
  }
],

```

But at that point, it’s basically equivalent—if not more complicated—than setting things up in a plugin. So it makes more sense to just do it all inside a plugin module. It’s much cleaner and easier to manage. <br />



### Module Type and LoadingPhase

![uplugin](/post-img/shaderposts/ue5-compute-shader/uplugin.png){: width="80%"} <br />
After creating a plugin, you'll have a `.uplugin` file that includes a key called `Modules`, which contains three main configurations: `Name`, `Type`, and `LoadingPhase`. <br />

`Name` is the name of the plugin module. This name will also appear in the `.Build.cs` file and within the plugin's `.cpp` file accordingly. <br />

`Type` has several options: `Runtime`, `Editor`, `Developer`, `CookedOnly`, `Program`. <br />
For example, I’m currently using `Runtime`, which allows the module to be used during gameplay and also ensures it's included in the build. <br />

`LoadingPhase` options include: `Default`, `PostEngineInit`, `PostConfigInit`, etc. <br />
I’m using `PostConfigInit`, which means the module will load right after reading the ini configuration files. <br />

If the `LoadingPhase` is set incorrectly, it will cause errors, for example: <br />
![loadingphase](/post-img/shaderposts/ue5-compute-shader/loadingphase.png){: width="100%"} <br />




## Data Table File Setup

My plan is to import the JSON file exported from Houdini into Unreal and convert it into a `Data Table` asset. <br />
To create a Data Table that matches the structure of the JSON file, you first need to define a corresponding struct. <br />
This can be done either through Blueprints or directly using C++ code, below is mine: <br />
![tree-data-struct](/post-img/shaderposts/ue5-compute-shader/tree-data-struct.png){: width="80%"} <br />

The first key `Name` is necessary in the json data even if I don't need it, because the `TreeDataRow` struct is inherited from `FTableRowBase`, and there is a `FName RowName` already there that will be inherited, without `Name` key, the import of the json data will fail. <br />
![tree-json](/post-img/shaderposts/ue5-compute-shader/json.png){: width="80%"} <br />

Then when import the json file, I can select the row struct I just defined: <br />
![row-struct](/post-img/shaderposts/ue5-compute-shader/row-struct.png){: width="80%"} <br />

Then this is the Data Table looks like: <br />
![tree-data-table](/post-img/shaderposts/ue5-compute-shader/tree-data-table.png){: width="80%"} <br />


## MyGameInstance

So how and when should I read this DataTable?
My idea is to create a `UGameInstance` method that reads the `TreeDataTable` at game startup, then passes the processed data to the `SceneViewExtension` code, and finally sends it to the shader.

Here’s my GameInstance code:

```c++
// MyGameInstance.h
#pragma once
#include "TreeDataTableStruct.h"
#include "CoreMinimal.h"
#include "Engine/GameInstance.h"
#include "RHIResources.h"   
#include "RenderGraphResources.h"   
#include "RenderGraphDefinitions.h"
#include "MyGameInstance.generated.h"

UCLASS()
class COMPUTETREE_API UMyGameInstance : public UGameInstance
{
    GENERATED_BODY()

public:
    virtual void Init() override;

    TArray<FTreeDataRow> TreeDataArray;
    TArray<FTreeDataRowGPU> TreeDataArrayGPU;

    FBufferRHIRef TreeDataBufferRHI;

    UPROPERTY()
    UMaterialParameterCollection* MyMPC;

    UMyGameInstance();

    // Function to load tree data from DataTable
    void LoadTreeDataFromDataTable();

    void GetTreeData(TArray<FTreeDataRowGPU>& OutTreeData) const;
    void LoadMPC();

    // Debugging function to print tree data
    UFUNCTION(BlueprintCallable, Category = "Debug")
    void PrintTreeData();
};
```

```c++
// MyGameInstance.cpp
#include "MyGameInstance.h"

#include "Engine/DataTable.h"
#include "ShaderParameterUtils.h"

#include "Misc/Paths.h"
#include "Misc/FileHelper.h"
#include "Serialization/JsonSerializer.h"
#include "ShaderParameterUtils.h"
#include "RenderGraphBuilder.h"
#include "Materials/MaterialParameterCollectionInstance.h"
#include "Materials/MaterialParameterCollection.h"
//#include "CTTree/CTTreeCS.h"


UMyGameInstance::UMyGameInstance()
{
    // Constructor logic

}
void UMyGameInstance::Init()
{
    Super::Init();
    UE_LOG(LogTemp, Warning, TEXT("CT UMyGameInstance Initialized!"));
    LoadTreeDataFromDataTable();
    UE_LOG(LogTemp, Warning, TEXT("CT UMyGameInstance Loaded Data!"));
    LoadMPC();

}


void UMyGameInstance::LoadTreeDataFromDataTable()
{
	if (UDataTable* DataTable = LoadObject<UDataTable>(nullptr, TEXT("DataTable'/Game/CTTree/CTTreeData.CTTreeData'")))
	{
		const TArray<FName> RowNames = DataTable->GetRowNames();

        TreeDataArrayGPU.Empty();

		for (const FName& RowName : RowNames)
		{
			if (const FTreeDataRow* Row = DataTable->FindRow<FTreeDataRow>(RowName, TEXT("UMyGameInstance LoadTreeDataFromDataTable")))
			{
                TreeDataArrayGPU.Add(FTreeDataRowGPU(*Row));
			}
		}
	}
}

void UMyGameInstance::GetTreeData(TArray<FTreeDataRowGPU>& OutTreeData) const
{
    OutTreeData = TreeDataArrayGPU;
}

void UMyGameInstance::LoadMPC()
{
	if (IsInGameThread())
	{
        MyMPC = LoadObject<UMaterialParameterCollection>(nullptr, TEXT("/Game/CTTree/MPC_CTTree.MPC_CTTree"));
        UE_LOG(LogTemp, Warning, TEXT("CT UMyGameInstance Loaded MPC!"));
    }
	else
	{
		AsyncTask(ENamedThreads::GameThread, [this]()
			{
                MyMPC = LoadObject<UMaterialParameterCollection>(nullptr, TEXT("/Game/CTTree/MPC_CTTree.MPC_CTTree"));
			});
        UE_LOG(LogTemp, Warning, TEXT("CT UMyGameInstance Loaded MPC!"));
	}
    if (!MyMPC)
    {
        UE_LOG(LogTemp, Error, TEXT("Failed to load Material Parameter Collection!"));
    }

}

```

Eventually, I call `UMyGameInstance::GetTreeData()` inside the `SceneViewExtension` to retrieve the output array, then store it in a buffer and pass it to the Compute Shader. <br />

I also wrote a `UMyGameInstance::LoadMPC()` function to load the `MaterialParameterCollection` object I had created beforehand, so that the `SceneViewExtension` can pass the output processed by the compute shader to the MPC and I get the value in the material editor by using this MPC. <br /> 

This approach clearly separates the CPU-side logic from the GPU-side processing, helping to avoid conflicts between the GameThread and RenderThread. <br />
Otherwise, if you operate directly inside the SceneViewExtension, you might run into errors like: <br />
`Assertion failed: IsInGameThread() [File:D:\build\++UE5\Sync\Engine\Source\Runtime\CoreUObject\Private\UObject\UObjectGlobals.cpp] [Line: 1702] Unable to load /Game/CTTree/CTTreeData. Objects and Packages can only be loaded from the game thread with the currently active loader 'LegacyLoader'` <br />

## SceneViewExtension
After configuring the required data, I move on to the `SceneViewExtension` file. Inside the `PrePostProcessPass_RenderThread()` function, the first step is to call the function from `MyGameInstance` to get the data.
To do that, first need to get the current `World`, and then retrieve the GameInstance from it:

```c++
//CTTreeSceneViewExtension.cpp
void FCTTreeSceneViewExtension::PrePostProcessPass_RenderThread(
    FRDGBuilder& GraphBuilder,
    const FSceneView& View,
    const FPostProcessingInputs& Inputs)
{
    //...
    UWorld* World = View.Family->Scene->GetWorld();
    
    UMyGameInstance* GameInstance = Cast<UMyGameInstance>(World->GetGameInstance());
    //...
}
```
Then I can easily get my tree data:
```c++
// get Data
TArray<FTreeDataRowGPU> TreeDataArray;
GameInstance->GetTreeData(TreeDataArray);
```

### Pass Array Data to the Compute Shader

At this point, I ran into a problem. I know how to pass a single `integer`, `vector`, or `float` to a shader. For example, if I want to pass a `time` variable to the shader, I just need to:
1. Declare a `time` variable in the target shader:
```hlsl
//CTTreeCS.h
...
float TimeCS;
...
```
2. Declare a shader parameter for it in the shader’s header file:
```c++
    BEGIN_SHADER_PARAMETER_STRUCT(FParameters, )
        ...
        SHADER_PARAMETER(float, TimeCS)
        ...
    END_SHADER_PARAMETER_STRUCT()
```
3. Retrieve the time variable in the `SceneViewExtension` and bind it to the shader parameter:
```c++
    FCTTreeCS::FParameters* Params = GraphBuilder.AllocParameters<FCTTreeCS::FParameters>();
    Params->TimeCS = World->GetRealTimeSeconds();
```

### Create the SRV (Shader Resource View)
But now I need to pass an array of data. On the C++ side, the type is `TArray<FTreeDataRowGPU>`, but on the shader side, there is no `TArray`. How should I handle this?
If you want to pass a batch of `structured data` to the shader, you first need to convert the data `eg. TArray<FTreeDataRowGPU>` into a buffer `FRDGBufferRef`, and then create an `SRV (Shader Resource View)` from that buffer. An SRV is a `read-only view` of GPU resources used in DirectX / Unreal, which allows the shader to **read** the contents of the buffer. There's also a similar `writable view` called `FRDGBufferUAVRef`, which I’ll be using later as well.

Below is the part of code that I convert my tree data array to an SRV:

```c++
//CTTreeSceneViewExtension.cpp
FRDGBufferRef TreeDataBufferRDG = GraphBuilder.CreateBuffer(
    FRDGBufferDesc::CreateStructuredDesc(sizeof(FTreeDataRowGPU), TreeDataArray.Num()),
    TEXT("SceneViewExtension CT_TreeDataBuffer")
);
void* CPUData = GraphBuilder.Alloc(TreeDataArray.Num() * sizeof(FTreeDataRowGPU), 16);
FMemory::Memcpy(CPUData, TreeDataArray.GetData(), TreeDataArray.Num() * sizeof(FTreeDataRowGPU));
GraphBuilder.QueueBufferUpload(TreeDataBufferRDG, CPUData, TreeDataArray.Num() * sizeof(FTreeDataRowGPU), ERDGInitialDataFlags::None);
FRDGBufferSRVRef TreeDataSRV = GraphBuilder.CreateSRV(TreeDataBufferRDG);
...
Params->TreeData = TreeDataSRV; // Pass to the shader parameter TreeData
```

```c++
//CTTreeCS.h
BEGIN_SHADER_PARAMETER_STRUCT(FParameters, )
    SHADER_PARAMETER_RDG_BUFFER_SRV(StructuredBuffer<FTreeDataRowGPU>, TreeData) // Declare TreeData
END_SHADER_PARAMETER_STRUCT()
```

```hlsl
//CTTreeCS.usf
...
StructuredBuffer<FTreeDataRowGPU> TreeData; // Read registered TreeData
```

But the process wasn’t over yet—I ran into an issue where the data I was reading appeared corrupted. At first, my shader looked like this: 

```hlsl
struct FTreeDataRowGPU
{
    int BranchID; 
    int ParentID; 
    float3 PivotPos; 
};

// Read register
StructuredBuffer<FTreeDataRowGPU> TreeData; 
int TotalBranches;
float TimeCS;
RWStructuredBuffer<float> DebugOutput; 

[numthreads(64,1,1)]
void CTTreeCS(
    uint3 DispatchThreadID : SV_DispatchThreadID,
    uint3 GroupId : SV_GroupID,
    uint3 GroupThreadId : SV_GroupThreadID)
{   
    const uint ThreadId = DispatchThreadID.x;
    
    DebugOutput[ThreadId] = TreeData[0].BranchID;
}
```
When I output the 0th TreeData `TreeData[0].BranchID`, I get the correct BranchID. But when I output the 1st entry `TreeData[1].BranchID`, it returns a negative value, which indicates that the data is being read incorrectly. <br />

Then I found that the size of `FTreeDataRow` was *40 bytes*, which didn’t match the data I had input. I had provided two int32 and one vector, which should be `2 * 4 + 24 = 32 bytes`. So where did the extra 8 bytes come from? <br />

Going back to the `TreeDataTableStruct.h` file I mentioned above, you would see that there are two structs defined. Initially, I only had the first `struct FTreeDataRow`, which inherits from `FTableRowBase`. This struct was used specifically for the DataTable file, to import the JSON file. So, the extra 8 bytes likely came from `FTableRowBase`. <br />

To keep the data clean and minimal, I created a second struct `FTreeDataRowGPU`, ensuring it only includes the two int32 and one float3, with size 20 bytes. For the two int32, in shader it's still gonna be int32, however the `FVector PivotPos` has to be converted to an array with three floats so that in shader I can use `float3 PivotPos`: <br />

```c++
//TreeDataTableStruct.h
struct FTreeDataRowGPU
{
public:
    FTreeDataRowGPU(const FTreeDataRow& rawData)
        : BranchID(rawData.BranchID)
        , ParentID(rawData.ParentID)
    {
        PivotPos[0] = rawData.PivotPos.X;
        PivotPos[1] = rawData.PivotPos.Y;
        PivotPos[2] = rawData.PivotPos.Z;
    }

    const int32 BranchID{ 0 };
    int32 ParentID{ 0 };
    float PivotPos[3]{ 0.0f, 0.0f, 0.0f };
};

```

By converting the FVector to float3, the struct in shader can natually matching the size and alignment, without adding any paddings. <br />
The struct in the shader still like this: <br />
```hlsl
struct FTreeDataRowGPU
{
    int BranchID; // 4 bytes
    int ParentID; // 4 b
    float3 PivotPos; // 12 b
};
```
This aligns perfectly with the 20 bytes on the C++ side. With the alignment correct, reading the second, third, and all subsequent arrays will no longer result in corrupted data due to misalignment. <br />

### Add a Compute Shader Pass
Now that the issue with reading the data array is resolved, let’s return to the `SceneViewExtension`. After retrieving the data and binding the shader parameters, the next crucial step is to `dispatch the compute shader`: <br />
```c++
// Use CS
FComputeShaderUtils::AddPass(
    GraphBuilder,
    RDG_EVENT_NAME("CTTreeAnimationCS"),
    ERDGPassFlags::Compute,
    ComputeShader,
    Params,
    GroupCount
);
```

### Create the UAV (Unordered Access View)

As I mentioned earlier, the writable view is `FRDGBufferUAVRef`, and it's created in a similar way to `FRDGBufferSRVRef`: <br />

1. Create the buffer and bind it to a shader parameter:
```c++
//CTTreeSceneViewExtension.cpp
FRDGBufferRef DebugOutputBuffer = GraphBuilder.CreateBuffer(
    FRDGBufferDesc::CreateStructuredDesc(sizeof(float), TotalBranches),
    TEXT("CT_DebugOutputBuffer") 
); // create a gpu buffer to store DebugOutput
// create DebugOutput's UAV(Unordered Access View) to store the data
FRDGBufferUAVRef DebugOutputUAV = GraphBuilder.CreateUAV(DebugOutputBuffer);
...
Params->DebugOutput = DebugOutputUAV;
```
2. Declare the shader parameter in the compute shader’s header file:
```c++
SHADER_PARAMETER_RDG_BUFFER_UAV(RWStructuredBuffer<float>, DebugOutput)
```
3. In the shader, read from the registered parameter and write to it：
```hlsl
RWStructuredBuffer<float> DebugOutput; 
...
DebugOutput[ThreadId] = TreeData[ThreadId].BranchID;
```


### Get the Readback

After creating the UAV and the compute shader is dispatched, I can add code to retrieve the output data written from the shader. <br />
```c++

    const uint32 NumOfBytes = sizeof(float) * TotalBranches;

    AddEnqueueCopyPass(GraphBuilder, DebugOutputReadback, DebugOutputBuffer, NumOfBytes);

    if (DebugOutputReadback->IsReady())
    {
        DebugOutputArray.Empty();
        float* Buffer = (float*)DebugOutputReadback->Lock(sizeof(float));

        for (int i = 0; i < TotalBranches; i++)
        {
            DebugOutputArray.Add(Buffer[i]);
        }
        DebugOutputReadback->Unlock();
    }
    else
    {
        UE_LOG(LogTemp, Warning, TEXT("Readback DebugOutput Failed!"));
        return;
    }
```

The key step here is `AddEnqueueCopyPass`, which is a tool provided by RDG to queue the data copy to CPU memory after the pass is executed. <br />
I use `if (DebugOutputReadback->IsReady())` to check if the copy is completed, because the copy is performed *asynchronously*, and you have to wait until it’s finished before calling `Lock()`. <br />
Once the readback buffer is locked, you can get a pointer to access the data, iterate over all the returned data, and add it to `DebugOutputArray`. <br />
After obtaining the `DebugOutputArray`, it’s simply a matter of passing it to the `MaterialParameterCollection` so that you can retrieve the parameters in the Material Editor.

Here is the code for passing parameters to the MPC: <br />

```c++

UMaterialParameterCollectionInstance* MPCInstance = World->GetParameterCollectionInstance(GameInstance->MyMPC);
if (MPCInstance)
{
    if (DebugOutputArray.Num() > 0)
    {
        for (int i = 0; i < TotalBranches; i++)
        {
            MPCInstance->SetScalarParameterValue("Test", DebugOutputArray[i]);
            UE_LOG(LogTemp, Error, TEXT("Set Material Parameter Collection : %f"), DebugOutputArray[i]);
        }
        
    }
    else
    {
        UE_LOG(LogTemp, Error, TEXT("DebugOutputArray is empty!"));
    }

}
```


## Some Thoughts about *Thread*
Here I want to record some notes about the concept of "threads". <br />

For vertex shaders and pixel shaders, the shader operation is executed per vertex and per pixel, respectively — this is easy to understand: each vertex and pixel runs the shader code once individually. <br />
However, you cannot group some vertices or pixels together for execution. <br />

For compute shaders, the execution order and count are determined by how many thread groups you dispatch and the thread index within each group. <br />

On the C++ side, you decide how many groups to dispatch. In UE syntax, this is done via the `FIntVector GroupCount(XGroupCount, YGroupCount, ZGroupCount)` parameter inside `FComputeShaderUtils::AddPass()`, which corresponds to `Dispatch(x, y, z)`. <br />

On the HLSL side, before the shader’s void function, you declare how many small threads per group with `[numthreads(x,y,z)]`. <br />

Thus: <br />
`Total Threads = numthreads(x,y,z) × GroupCount(x,y,z)` <br />

For single-dimensional threads — like in my code where I wrote: `FIntVector GroupCount(FMath::DivideAndRoundUp(TotalBranches, 64), 1, 1);` paired with `[numthreads(64,1,1)]` — then in the shader, `SV_DispatchThreadID.x` will be arranged from
`0 ~ ceil(TotalBranches/64) × 64`. for example: `0, 1, 2, 3, 4... 63`. <br />
The maximum value `TotalBranches` depends on how many tree branch data entries I have. <br />
Thus inside the shader, `SV_DispatchThreadID.x` can be used as the branch index — letting each thread operate per branch. <br />
Every time the compute shader is called, it runs 64 threads per group. <br />

For example, if I output my debug like this: <br />
`DebugOutput[ThreadId] = TreeData[ThreadId].BranchID;` <br />

then when printing the result on the CPU side: <br />
![set-mat.png](/post-img/shaderposts/ue5-compute-shader/set-mat.png){: width="80%"} <br />
→ I will see 5 outputs each call because there is only 1 thread group and 5 threads. <br />

This is something vertex shaders or pixel shaders cannot achieve! <br />



## Compute Shader .cpp

To bind the compute shader, just use the `IMPLEMENT_SHADER_TYPE` macro in its .cpp file. <br />
![implement-shader](/post-img/shaderposts/ue5-compute-shader/implement-shader.png){: width="90%"} <br />


## Subsystem 

`UEngineSubsystem` is a subsystem class in Unreal Engine used for managing engine lifecycle events (such as initialization, shutdown, etc.). <br />
By registering the `SceneViewExtension` inside a subsystem, you can ensure that it is properly managed throughout the engine's lifecycle and activated when needed. <br />

I added my SceneViewExtension in `FWorldDelegates::OnPostWorldInitialization`instead of directly during subsystem initialization. <br />
The reason is that I need to access the current World inside the SceneViewExtension, and the World isn't ready yet during early initialization — it must wait until World initialization is complete before registering the SceneViewExtension, otherwise the World would be null. <br />

```c++
void UCTTreeEngineSubsystem::OnWorldInitialized(UWorld* NewWorld, const UWorld::InitializationValues IVS)
{
    if (NewWorld && NewWorld->IsGameWorld())
    {
        UE_LOG(LogTemp, Warning, TEXT("CT ComputeShader Subsystem: World Initialized! Registering SceneViewExtension."));

        // makesure SceneViewExtension only register once
        if (!CTTreeSceneViewExtension.IsValid())
        {
            CTTreeSceneViewExtension = FSceneViewExtensions::NewExtension<FCTTreeSceneViewExtension>();
        }
    }
}

```


## Unreal Debug from game package

First, you need to package the project inside the engine by going to **Platform → Windows → DebugGame → Package Project**. <br />
After packaging, the built game will be generated under your project directory. <br />
![package-game.png](/post-img/shaderposts/ue5-compute-shader/package-game.png){: width="80%"} <br />
Then, switch back to Visual Studio and change the configuration to *DebugGame*. This way, you can set breakpoints and debug the packaged game. <br />
![debug-game](/post-img/shaderposts/ue5-compute-shader/debug-game.png){: width="80%"} <br />




## Use PIX to Debug

To debug in PIX, you would need the built game package I mentioned above. Then within PIX, set two line arguments: <br />
![pix](/post-img/shaderposts/ue5-compute-shader/pix.png){: width="90%"} <br />
Also I used `RDG_EVENT_SCOPE()` whintin the SceneViewExtension and `RDG_EVENT_NAME()` in `FComputeShaderUtils::AddPass()`, and `TEXT()` in `GraphicBuilder.CreateBuffer()` to show the name label: <br />
![pix-scope](/post-img/shaderposts/ue5-compute-shader/pix-scope.png){: width="100%"} <br />
